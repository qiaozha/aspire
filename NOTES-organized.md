# Aspire Codebase — Learning Notes

A reference document for getting up to speed with the `dotnet/aspire` repository.

---

## Table of Contents

1. [What is Aspire?](#1-what-is-aspire)
2. [Repository Layout](#2-repository-layout)
3. [Core Concepts: Application Model](#3-core-concepts-application-model)
4. [Architecture Overview](#4-architecture-overview)
5. [DCP Executor — Local Orchestration](#5-dcp-executor--local-orchestration)
6. [Dashboard](#6-dashboard)
7. [Publishing Pipeline](#7-publishing-pipeline)
8. [Azure Integration & Bicep](#8-azure-integration--bicep)
9. [Client Integration Components](#9-client-integration-components)
10. [Aspire CLI](#10-aspire-cli)
11. [Polyglot AppHost (ATS System)](#11-polyglot-apphost-ats-system)
12. [VS Code Extension](#12-vs-code-extension)
13. [Key Architectural Principles](#13-key-architectural-principles)
14. [Local Development Setup](#14-local-development-setup)

---

## 1. What is Aspire?

`dotnet/aspire` is a distributed application orchestration and deployment platform for .NET. It provides:

- **App Host orchestration** — A code-first model where you declare services, containers, databases, and cloud resources in C#. During local dev, Aspire starts everything (Docker containers, processes) and wires them together automatically.
- **A monitoring Dashboard** — Blazor-based UI for traces, logs, and metrics via OpenTelemetry.
- **40+ platform integrations** — Redis, PostgreSQL, SQL Server, MongoDB, RabbitMQ, Kafka, all Azure services, etc.
- **A polyglot CLI** — Supports non-.NET languages (TypeScript, Python, Go, Rust, Java) as app host authors.
- **Deployment publishing** — Generates infrastructure-as-code artifacts (Bicep, Kubernetes YAML) ready for production.

```csharp
var builder = DistributedApplication.CreateBuilder(args);

var db = builder.AddPostgres("pg");
var api = builder.AddProject("api").WithReference(db);
var web = builder.AddNpmApp("web").WithReference(api);

builder.Build().Run();
```

---

## 2. Repository Layout

```
aspire/
├── src/            # All production source code
│   ├── Aspire.Hosting/                  # Core orchestration framework
│   ├── Aspire.Dashboard/                # Blazor monitoring dashboard
│   ├── Aspire.Cli/                      # CLI tool (native AOT)
│   ├── Aspire.Hosting.Azure.*/          # 30+ Azure integration packages
│   ├── Aspire.Hosting.CodeGeneration.*/ # Polyglot SDK code generators
│   └── Components/Aspire.*/            # 38+ client-side integration packages
├── tests/          # Unit & integration tests (mirrors src/ structure)
├── playground/     # Sample apps for manual verification
├── extension/      # VS Code extension (TypeScript)
├── docs/specs/     # Architecture specs & design documents
├── eng/            # Build infrastructure (Arcade SDK, CI pipelines)
└── tools/          # Developer tooling (QuarantineTools, etc.)
```

---

## 3. Core Concepts: Application Model

The **application model** is the central data structure everything else reads and transforms.

### `DistributedApplicationModel` — The Resource Graph

```csharp
[DebuggerDisplay("Resources = {Resources.Count}")]
public class DistributedApplicationModel(IResourceCollection resources)
{
    public IResourceCollection Resources { get; } = resources;
}
```

`IResourceCollection` is just `IList<IResource>` — a **flat list**, not a tree. The graph structure (the dependency DAG) is **implicit**: it is encoded in annotations and `ReferenceExpression` objects attached to resources. Aspire never stores an explicit edge list; edges are discovered at runtime by traversing annotations.

It is registered as a **singleton in DI**. Everything — `DcpExecutor`, `AzureResourcePreparer`, `BicepProvisioner`, manifest writers, the Dashboard — gets it from DI and queries `model.Resources`.

### `IResource` — The Fundamental Unit

```csharp
public interface IResource
{
    string Name { get; }
    ResourceAnnotationCollection Annotations { get; }
}
```

That's the entire interface. Two properties. Resources are **pure inert data objects** — they don't start, stop, or manage themselves. All runtime behavior is encoded in `Annotations`.

**Resource type hierarchy:**

```
IResource
  └── Resource (abstract base, validates name)
        ├── ContainerResource        — runs a Docker image
        ├── ProjectResource          — runs a .NET project
        ├── ExecutableResource       — runs an arbitrary binary
        ├── ParameterResource        — external input / secret
        ├── ConnectionStringResource — a literal connection string
        └── [integration-specific]
              ├── RedisResource            : ContainerResource
              ├── PostgresServerResource   : ContainerResource
              ├── AzureBicepResource       : Resource
              └── AzureProvisioningResource: AzureBicepResource
```

The three **primitive compute types** (`ContainerResource`, `ProjectResource`, `ExecutableResource`) are what DCP knows about natively. Everything else is modeled on top of them.

### Annotations — The Extensibility Mechanism

`IResourceAnnotation` is an empty marker interface. Any POCO implementing it can be attached to any resource:

```csharp
// Examples of built-in annotations:
ContainerImageAnnotation             // → image name, tag, registry
EndpointAnnotation                   // → port, scheme, name
EnvironmentCallbackAnnotation        // → lambda to inject env vars at startup
ContainerMountAnnotation             // → volume or bind mount
HealthCheckAnnotation                // → health probe configuration
PipelineStepAnnotation               // → deploy pipeline step (Bicep, push, etc.)
ManifestPublishingCallbackAnnotation // → how to write resource to azd manifest
WaitAnnotation                       // → WaitFor dependency ordering
```

Extension methods like `AddRedis("cache")` are syntactic sugar over:
1. `new RedisResource("cache")` (a `ContainerResource` subclass)
2. `resource.Annotations.Add(new ContainerImageAnnotation { Image = "redis", Tag = "7.4" })`
3. `resource.Annotations.Add(new EndpointAnnotation(ProtocolType.Tcp, targetPort: 6379))`
4. Wrap in `IResourceBuilder<RedisResource>` and return

### `IResourceBuilder<T>` — Fluent Configuration

`IResourceBuilder<T>` is a thin fluent wrapper around a resource. All `WithXyz()` methods just add more annotations:

```
IDistributedApplicationBuilder
  └── .AddRedis("cache")
        └── IResourceBuilder<RedisResource>
              ├── .Resource            → the RedisResource instance
              ├── .WithEnvironment()   → adds EnvironmentCallbackAnnotation
              ├── .WithDataVolume()    → adds ContainerMountAnnotation
              └── .WithHealthCheck()   → adds HealthCheckAnnotation
```

### Standard Capability Interfaces

Resources opt into behaviors by implementing interfaces (polymorphism over type-checking):

| Interface | What it enables |
|---|---|
| `IResourceWithEnvironment` | `.WithEnvironment()`, env var injection |
| `IResourceWithEndpoints` | `.WithEndpoint()`, `GetEndpoint()`, URL generation |
| `IResourceWithConnectionString` | `.WithReference(db)` wires connection string into consumers |
| `IResourceWithServiceDiscovery` | Registers in DNS-style service discovery |
| `IResourceWithArgs` | CLI argument injection |
| `IResourceWithWaitSupport` | `WaitFor()` — blocks startup until dependency is ready |
| `IResourceWithParent` | Lifecycle containment (child stops when parent stops) |
| `IComputeResource` | Marks as a compute unit (project / container / exec) |

`ContainerResource` implements nearly all of them. `ParameterResource` implements almost none — it's a value, not a runnable thing.

### Values & References — How the DAG Forms

The graph is a **heterogeneous DAG** encoding both dependency order and value flows. Edges are structured value objects, not strings:

```
web ──► EndpointReference ──► api
                               │
                               └──► ConnectionStringReference ──► postgres
```

| Type | Run mode | Publish mode |
|---|---|---|
| `EndpointReference` | `http://localhost:5000` | `{api.bindings.http.url}` |
| `ConnectionStringReference` | `Host=localhost;Port=5432;...` | `{postgres.connectionString}` |
| `ParameterResource` | value from user secrets / env | `${PARAM}` placeholder |
| `ReferenceExpression` | interpolated concrete string | interpolated manifest expression |

**`ReferenceExpression`** is the key glue type. It captures structured value objects inside a C# interpolated string handler, deferring resolution:

```csharp
// This does NOT resolve "localhost:5432" at build time.
// It stores the reference to pg.PrimaryEndpoint for later evaluation.
public ReferenceExpression ConnectionStringExpression =>
    ReferenceExpression.Create(
        $"Host={pg.PrimaryEndpoint.Property(EndpointProperty.Host)};" +
        $"Port={pg.PrimaryEndpoint.Property(EndpointProperty.Port)}");
```

- **Run time**: `IValueProvider.GetValueAsync()` resolves all references to concrete strings.
- **Publish time**: `IManifestExpressionProvider.ValueExpression` emits `{pg.bindings.tcp.host}` placeholders into the azd manifest.

---

## 4. Architecture Overview

```
┌──────────────────────────────────────────────────────────────────────┐
│                      User-Facing Entry Points                        │
│   aspire CLI (Aspire.Cli)  │  VS Code Extension  │  App Host C#     │
└────────────┬───────────────┴────────────┬─────────┴──────┬──────────┘
             │                            │                │
             ▼                            ▼                ▼
  ┌──────────────────┐       ┌──────────────────┐  ┌───────────────────────┐
  │  CLI Orchestrator│       │  VS Code Ext.    │  │  App Host SDK         │
  │  (run/publish)   │       │  (debug, attach) │  │  (Aspire.Hosting)     │
  └────────┬─────────┘       └──────────────────┘  └──────────┬────────────┘
           │                                                  │
           │                                   ┌──────────────▼────────────┐
           │                                   │     Application Model     │
           │                                   │  IResource DAG + Annotations│
           │                                   └──────────────┬────────────┘
           │                                                  │
           │              ┌───────────────────────────────────┤
           │              │               │                   │
           ▼              ▼               ▼                   ▼
    ┌──────────┐  ┌─────────────┐  ┌──────────┐  ┌─────────────────────┐
    │  AzDev   │  │DCP Executor │  │Dashboard │  │  Publishing Pipeline│
    │  CLI (azd│  │(local run)  │  │(Blazor)  │  │  (Bicep / manifest) │
    └──────────┘  └─────────────┘  └──────────┘  └─────────────────────┘
```

**Data flow summary:**

```
Developer writes C# App Host (or TypeScript/Python via JSON-RPC)
         │  AddRedis / AddProject / AddAzureKeyVault / ...
         ▼
  Application Model (IResource flat list with Annotation-encoded edges)
         │
    ─────┼──────────────────────────────────
    │                   │                  │
    ▼                   ▼                  ▼
DCP Executor      Dashboard (Blazor)  Publishing Pipeline
(local run)       (monitoring)        (aspire publish)
    │                   │                  │
    ▼             OTLP telemetry     ┌─────┴─────┐
Containers         from services     ▼           ▼
Processes ──────► Resource State  Bicep files manifest.json
Docker/Podman      Events (gRPC)   (for Azure)  (for azd/K8s)
```

---

## 5. DCP Executor — Local Orchestration

### What is DCP?

**DCP (Developer Control Plane)** is a separate native binary shipped with the Aspire workload. It runs on the same machine as the App Host and exposes a **Kubernetes-style CRD API** over a local Unix socket (or named pipe on Windows). DCP abstracts the actual container runtime (Docker, Podman) and process launcher behind that K8s-like API surface.

```
App Host Process
  └─ DcpHost (starts DCP subprocess)
      └─ DcpExecutor
          ├─ Walks IResource graph → creates DCP CRD objects
          ├─ KubernetesService → HTTP calls to DCP's local K8s API
          ├─ Watch loops → ResourceNotificationService → Dashboard
          └─ Log streams → ResourceLoggerService → Dashboard
```

### What Annotations `DcpExecutor` Reads

`DcpExecutor` walks `DistributedApplicationModel.Resources` and maps annotations to DCP CRD objects:

| Annotation / Resource Type | DCP CRD produced | Key fields mapped |
|---|---|---|
| `ContainerImageAnnotation` | `Container` (CRD) | image, ports, env, volumeMounts, command, args |
| `DockerfileBuildAnnotation` | `Container.Spec.Build` | Dockerfile path, context, build args |
| `EndpointAnnotation` | `Service` (CRD) | address, port, protocol, addressAllocationMode |
| `ProjectResource` / `ExecutableResource` | `Executable` (CRD) | executablePath, workingDirectory, args, env |
| `ContainerMountAnnotation` | `Container.Spec.VolumeMounts` | source, target, type (bind/volume) |
| `EnvironmentCallbackAnnotation` | env on Container or Executable | resolved key/value pairs |
| `HealthCheckAnnotation` | `Executable.Spec.HealthProbes` | HTTP/TCP probe config |
| `WaitAnnotation` | Ordering between `Service` objects | which services must be ready first |

### The `AppResource` Bridge Type

`AppResource` is the in-memory coupling between the Aspire model and the DCP model. It lives only inside `DcpExecutor`'s internal list — never sent over the wire.

```
IResource (Aspire app model)
    │
    └─ RenderedModelResource : AppResource
            ├─ ModelResource       → the original IResource
            ├─ DcpResource         → the CRD object (Container or Executable)
            ├─ ServicesProduced    → List<ServiceWithModelResource>  (endpoints exposed)
            └─ ServicesConsumed    → List<ServiceWithModelResource>  (endpoints depended on)

ServiceWithModelResource : RenderedModelResource
    ├─ Service             → the DCP Service CRD
    └─ EndpointAnnotation  → the original EndpointAnnotation from the model
```

One `RenderedModelResource` per runnable Aspire resource. One `ServiceWithModelResource` per `EndpointAnnotation`. `ServicesConsumed` drives startup ordering.

### `KubernetesService` — The Only DCP Channel

`IKubernetesService` (implemented by `KubernetesService`) is the **only communication channel** between `DcpExecutor` and the DCP binary. It wraps the official Kubernetes .NET client library pointed at DCP's local endpoint.

```
DcpExecutor                                 DCP Binary (native process)
    │                                              │
    │  ── HTTP over Unix socket ──────────────►   │
    │                                              │
    │  CreateAsync<Container>(spec)            K8s-style CRD API
    │  CreateAsync<Executable>(spec)          (list, get, create, patch, delete, watch)
    │  CreateAsync<Service>(spec)                  │
    │  WatchAsync<Container>()    ◄─────────────── │ (long-poll watch stream)
    │  GetLogStreamAsync(...)     ◄─────────────── │ (stdout/stderr as HTTP stream)
```

### State Change Propagation

After creating CRDs, `DcpExecutor` runs concurrent watch loops:

```
WatchKubernetesResourceAsync<Executable>()
WatchKubernetesResourceAsync<Container>()   →  ProcessResourceChange()
WatchKubernetesResourceAsync<Service>()              │
WatchKubernetesResourceAsync<Endpoint>()             ▼
                                            ResourceSnapshotBuilder.ToSnapshot()
                                                     │
                                                     ▼
                                            ResourceNotificationService.PublishUpdateAsync()
                                                     │
                                                     ▼
                                            Dashboard (via gRPC stream) + CLI backchannel
```

`ResourceNotificationService` is an in-process pub/sub bus. The Dashboard subscribes and re-renders resource state in real time.

---

## 6. Dashboard

A **Blazor Server** web app for monitoring running resources. Runs in-process with the App Host (or as a standalone container).

```
Aspire.Dashboard/
  ├─ Otlp/          ← OTLP gRPC receiver (traces, metrics, logs)
  ├─ Model/         ← In-memory state of resources, spans, logs
  ├─ Components/    ← Blazor UI components & pages
  │    └─ Pages/    ← ResourcesPage, TraceDetail, Logs, Metrics
  ├─ ServiceClient/ ← gRPC client for App Host resource API
  ├─ Api/           ← REST API consumed by the VS Code extension
  └─ Mcp/           ← MCP (Model Context Protocol) server endpoint
```

The Dashboard subscribes to `ResourceNotificationService` via gRPC streaming from the App Host and renders live resource state. It also receives OTLP telemetry directly from instrumented services.

---

## 7. Publishing Pipeline

Publishing translates the app model into deployment artifacts. Runs when `aspire publish` or `azd` invokes the App Host in publish mode.

```
DistributedApplicationPipeline
  ├─ PipelineStep (topologically ordered, dependency-aware)
  │   ├─ "provision-redis"     ← AzureBicepResource pipeline step
  │   ├─ "provision-keyvault"  ← AzureBicepResource pipeline step
  │   └─ "write-manifest"      ← ManifestPublisher step
  └─ PipelineExecutor (runs steps in dependency order)

ManifestPublisher
  └─ Calls ManifestPublishingCallbackAnnotation per resource
      └─ Each resource writes its JSON block to aspire-manifest.json
```

`PipelineStepAnnotation` on a resource registers it into the pipeline. Steps declare `DependsOn` relationships, so provisioning steps wait for the Azure credentials step to complete first. `aspire-manifest.json` is the intermediate format consumed by `azd`, Kubernetes publishers, or custom tooling.

---

## 8. Azure Integration & Bicep

### C# Integration API — Context-Aware Pattern

Every Azure resource type has a dedicated project (`src/Aspire.Hosting.Azure.*/`) with a context-aware API:

```csharp
// Declare the Azure resource
var cache = builder.AddAzureManagedRedis("cache");

// Optionally swap in a local Docker container for dev
var cache = builder.AddAzureManagedRedis("cache").RunAsContainer();

// Wire to services exactly like any other resource
builder.AddProject<Projects.Api>().WithReference(cache);
```

The same resource behaves differently depending on execution mode:

| Context | Behavior |
|---|---|
| `aspire run` with `RunAsContainer()` | Local Docker container, no Azure required |
| `aspire publish` | Generates Bicep files + manifest entries, no containers started |
| `aspire run` with Azure provisioner | Deploys to Azure via ARM API inside the App Host process |

### What is the Azure.Provisioning SDK?

`Azure.Provisioning.*` NuGet packages are a **C#-first infrastructure-as-code SDK**. They let you declare Azure infrastructure as typed C# objects that compile to Bicep text — similar in concept to AWS CDK or Pulumi.

| SDK Type | Role |
|---|---|
| `ProvisionableResource` | Base for all Azure resources (e.g., `RedisResource`, `StorageAccount`) |
| `ProvisioningParameter` | A Bicep `param` declaration |
| `ProvisioningOutput` | A Bicep `output` declaration |
| `BicepValue<T>` | A typed Bicep expression (literal, reference, or function call) |
| `BicepFunction.Interpolate(...)` | Produces Bicep string interpolation |
| `Infrastructure.Build().Compile()` | Runs the full graph and emits Bicep text |

### How Bicep Gets Generated

`AzureProvisioningResource` stores a `ConfigureInfrastructure` callback at construction time. When `GetBicepTemplateString()` is called:

```csharp
// Example: ConfigureRedisInfrastructure (from AzureRedisExtensions.cs)
private static void ConfigureRedisInfrastructure(AzureResourceInfrastructure infrastructure)
{
    var redis = AzureProvisioningResource.Create<CdkRedisResource>(infrastructure, ...);
    infrastructure.Add(redis);

    redis.IsAccessKeyAuthenticationDisabled = true;

    // Bicep outputs become connection string references
    infrastructure.Add(new ProvisioningOutput("connectionString", typeof(string))
    {
        Value = BicepFunction.Interpolate($"{redis.HostName},ssl=true")
    });
}

// At publish/provision time:
var infrastructure = new AzureResourceInfrastructure(this, Name);
ConfigureInfrastructure(infrastructure);   // runs the callback
var compilation = infrastructure.Build(options).Compile(); // → Bicep text
```

No Bicep files are pre-authored in the repo — they are all generated at runtime from C# objects.

### The Three Deployment Layers

```
Layer 1: C# Azure.Provisioning SDK → Bicep text
  ConfigureInfrastructure(infra) → infra.Build().Compile() → "cache.module.bicep"

Layer 2: Bicep text → ARM JSON
  BicepCliCompiler: `az bicep build --file cache.module.bicep --stdout` → ARM JSON

Layer 3: ARM JSON → Azure deployment
  BicepProvisioner → Azure.ResourceManager SDK → ARM REST API → Azure resource created
  → Reads back outputs (connectionString, hostName) → stored in resource.Outputs
```

### Two Deployment Paths

**Path A — `aspire run` (live provisioning):**
- The pipeline step runs inside the App Host process.
- `BicepProvisioner.GetOrCreateResourceAsync()` → generates Bicep → compiles → deploys via ARM API.
- Deployment is checksum-cached — re-runs skip provisioning when nothing changed.

**Path B — `azd up` (Azure Developer CLI):**
- `AzureBicepResource.WriteToManifest()` writes to `aspire-manifest.json`:
  ```json
  { "type": "azure.bicep.v0", "path": "cache.module.bicep", "params": { "location": "{azure.location}" } }
  ```
- `azd` reads the manifest, takes the `.bicep` files, and runs its own ARM deployment pipeline.
  Aspire only produces artifacts; `azd` executes the deployment.

### Summary: Who Does What

| Who | What |
|---|---|
| `Azure.Provisioning SDK` | Generates Bicep text from C# DSL |
| `BicepCliCompiler` | Shells out to `az bicep build` to compile Bicep → ARM JSON |
| `BicepProvisioner` | Submits ARM JSON to Azure via `Azure.ResourceManager` SDK, reads back outputs |
| `azd` | Alternative deployer — reads Aspire manifest + Bicep files, runs its own deployment |

---

## 9. Client Integration Components

These are the **application-side packages** consumed by individual microservices, not the App Host:

```
src/Components/Aspire.*/
  ├─ Aspire.StackExchange.Redis
  ├─ Aspire.Npgsql
  ├─ Aspire.Azure.Storage.Blobs
  └─ ... (38 packages total)
```

Each integration package handles:
- **DI registration** — e.g., `builder.AddRedisClient()` adds `IConnectionMultiplexer` to the service container
- **Health checks** — standard .NET health checks wired to the infrastructure
- **Resilience** — Polly pipelines for retry, circuit breaker
- **Telemetry** — OpenTelemetry tracing automatically configured
- **Configuration binding** — reads from `appsettings.json` / environment variables

> **Key distinction:** `src/Aspire.Hosting.*` is the **App Host** (orchestration, declares the resource graph). `src/Components/Aspire.*` is the **client library** (consumed by individual microservices to connect to those resources). They are independent NuGet packages with different dependency graphs.

---

## 10. Aspire CLI

A **native AOT** .NET CLI. Being native AOT means it cannot dynamically load arbitrary .NET assemblies at runtime — which is why the CLI communicates with the App Host server process via JSON-RPC (the server can load any assembly).

### Commands

| Command | Purpose |
|---|---|
| `aspire run` | Launch App Host + DCP + Dashboard |
| `aspire publish` | Generate deployment artifacts (Bicep, K8s YAML, azd manifest) |
| `aspire add` | Add NuGet integration packages to a project |
| `aspire new` | Scaffold from templates |
| `aspire deploy` | Deploy to Azure (via `azd`) |
| `aspire exec` | Run a command inside the Aspire environment |
| `aspire ps / logs / restart` | Resource lifecycle management |
| `aspire telemetry` | View OTLP traces/spans/logs |
| `aspire mcp` | Start MCP server for AI tool integration |
| `aspire sdk generate` | Generate typed guest-language SDK for an integration library |
| `aspire sdk dump` | Dump capability list from the running App Host server |

### Why the CLI Talks to the App Host Server via RPC

The CLI is a thin native AOT orchestrator. Every language/integration-specific capability lives inside NuGet packages only loaded inside the App Host Server process:

| RPC method | Why the CLI can't do it locally |
|---|---|
| `getRuntimeSpec` | `ILanguageSupport` implementations (how to run `npx tsx`) come from NuGet packages, not the CLI |
| `scaffoldAppHost` | Scaffold templates are embedded resources in NuGet packages |
| `generateCode` | Requires reflecting over loaded integration assemblies (e.g., `Aspire.Hosting.Redis.dll`) |
| `getCapabilities` | Needs reflection over NuGet assemblies to enumerate `[AspireExport]` attributes |

---

## 11. Polyglot AppHost (ATS System)

Allows TypeScript, Python, Go, Rust, and Java to write Aspire App Hosts. All 40+ integrations stay in .NET — guest code calls into the .NET server via JSON-RPC and gets back opaque handles to .NET objects.

### Process Architecture

```
┌──────────────────────┐
│     Aspire CLI       │  ← orchestrator; starts both processes; provides socket path
└───────┬──────────────┘
        │ spawns (via env REMOTE_APP_HOST_SOCKET_PATH)
        ▼                         ▼
┌─────────────────────┐   ┌───────────────────────────────┐
│  AppHost Server     │   │  Guest Process                │
│  (.NET)             │◄─►│  (e.g., `npx tsx apphost.ts`) │
│                     │   │                               │
│  CapabilityDispatch │   │  Generated SDK (.modules/)    │
│  App Model & DCP    │   │  User's apphost.ts            │
│  ILanguageSupport   │   │  ATS transport (vscode-rpc)   │
└─────────────────────┘   └───────────────────────────────┘
   JSON-RPC over Unix socket
```

### Startup Sequence

```
aspire run (in TypeScript apphost directory)
  │
  ▼ [1] CLI detects language
  │   → calls getRuntimeSpec RPC → TypeScriptLanguageSupport checks for apphost.ts + package.json
  │
  ▼ [2] CLI installs guest dependencies
  │   → runs: npm install
  │
  ▼ [3] CLI builds AppHost Server .NET project
  │   → Scaffolds temp project in $TMPDIR/.aspire/hosts/<hash>/
  │   → Program.cs: just `await RemoteHostServer.RunAsync(args)`
  │   → .csproj references all Aspire packages from aspire.json
  │   → dotnet build
  │
  ▼ [4] CLI runs code generation
  │   → starts AppHost Server, calls generateCode RPC
  │   → AtsCapabilityScanner reflects all loaded assemblies for [AspireExport]
  │   → AtsTypeScriptCodeGenerator emits .modules/aspire.ts, base.ts, transport.ts
  │   → CLI writes files, saves .modules/.codegen-hash (SHA-256 of package versions)
  │
  ▼ [5] CLI starts the guest process
      → runs: npx tsx apphost.ts
      → injects REMOTE_APP_HOST_SOCKET_PATH into guest environment
```

### JSON-RPC Wire Format

Uses LSP-style framing (same as VS Code language servers) over a Unix socket:

```
Content-Length: 127\r\n
\r\n
{"jsonrpc":"2.0","id":1,"method":"invokeCapability","params":["Aspire.Hosting.Redis/addRedis",{"builder":{"$handle":"1"},"name":"cache"}]}
```

The TypeScript SDK uses the `vscode-jsonrpc` npm package (the same package VS Code language servers use).

### Handle System — How .NET Objects Cross the Boundary

Every .NET object referenced by guest code is an **opaque handle** — a string ID + type name tuple. The guest never deserializes the object's internals; it just passes the handle back in future calls.

```
Guest                                          .NET AppHost Server
  │── invokeCapability("createBuilder") ───►│
  │◄── { "$handle": "1",                   │  Creates IDistributedApplicationBuilder
  │      "$type": "Aspire.Hosting/..." }   │  Stores in HandleRegistry
  │
  │── invokeCapability("Aspire.Hosting.Redis/addRedis",
  │      { "builder": {"$handle":"1"}, "name": "cache" }) ──►│
  │                                                           │  HandleRegistry.TryGet("1") → builder
  │                                                           │  method.Invoke(null, [builder, "cache"])
  │◄── { "$handle": "2", "$type": "...RedisResource" }       │  Stores result, returns handle
```

The `HandleRegistry` is a per-client `ConcurrentDictionary<string, object>`. `CapabilityDispatcher` resolves handles, unmarshals JSON args, calls the real C# method via `MethodInfo.Invoke()`, and marshals the result.

**ATS type ID format:** `{AssemblyName}/{FullTypeName}`
Example: `Aspire.Hosting.Redis/Aspire.Hosting.ApplicationModel.RedisResource`

### How Code Generation Works

`AtsCapabilityScanner.ScanAssemblies()` performs 5 passes over all loaded assemblies:

1. **Collect** — finds all `[AspireExport]` methods/types, `[AspireDto]` DTOs, enum types
2. **Resolve Unknown** — flags capability parameters whose types weren't found in the exported set
3. **Filter Invalid** — drops capabilities with unresolved types
4. **Expand Interface Targets** — `withEnvironment<T>` where `T : IResourceWithEnvironment` becomes one entry per concrete implementing type
5. **Filter Overloads** — ATS capability IDs are unique; C# overloads that collide are removed with diagnostics

The resulting `AtsContext` (capabilities, handle types, DTOs, enums) is passed to `ICodeGenerator.GenerateDistributedApplication()`, which templates out the guest-language SDK files.

**Hash-based cache:** `.modules/.codegen-hash` is a SHA-256 hash of all loaded package `id:version` pairs. If the hash changes (you ran `aspire add redis`), the next `aspire run` regenerates the SDK so `addRedis()` appears in TypeScript. This means the SDK cannot be committed to the repo.

### Generated TypeScript SDK (Simplified)

```typescript
// .modules/aspire.ts  (generated — do not edit)
export class DistributedApplicationBuilder {
    async addRedis(name: string, port?: number): Promise<RedisResourceBuilder> {
        const result = await this._client.invokeCapability(
            'Aspire.Hosting.Redis/addRedis',
            { builder: this._handle, name, port }
        );
        return new RedisResourceBuilder(result, this._client);
    }
}

// User's apphost.ts:
import { createBuilder } from './.modules/aspire.js';

const builder = await createBuilder();
const cache = await builder.addRedis("cache");
const api   = await builder.addProject("api", "../Api/Api.csproj");
await api.withReference(cache);
await builder.build().run();
```

### Callbacks — Host Calls Back into Guest

When the .NET runtime needs to evaluate a guest-registered callback (e.g., `withEnvironment` closure):

```
Guest                                          .NET AppHost Server
  │── invokeCapability("withEnvironmentCallback",
  │       { resource: {...}, callback: "cb_1" }) ──►│  stores "cb_1" → C# delegate
  │
  │◄── invokeCallback("cb_1", { p0: {"$handle":"5"} }) ───── at startup, evaluates env
  │
  │  runs user's async closure
  │  ctx.environmentVariables.set("KEY", "v")
  │    ── invokeCapability("set", ...) ────────────►│
```

### How to Add a New Guest Language (e.g., Python)

Two server-side implementations, both auto-discovered from a NuGet package:

**Step 1: Implement `ILanguageSupport`**
```csharp
public sealed class PythonLanguageSupport : ILanguageSupport
{
    public string Language => "python/cpython";

    public DetectionResult Detect(string directoryPath) =>
        File.Exists(Path.Combine(directoryPath, "apphost.py"))
            ? DetectionResult.Found("python/cpython", "apphost.py")
            : DetectionResult.NotFound;

    public RuntimeSpec GetRuntimeSpec() => new()
    {
        Execute = new CommandSpec { Command = "python", Args = ["{appHostFile}"] },
        InstallDependencies = new CommandSpec { Command = "pip", Args = ["install", "-r", "requirements.txt"] },
    };
}
```

**Step 2: Implement `ICodeGenerator`**
```csharp
public sealed class PythonCodeGenerator : ICodeGenerator
{
    public string Language => "Python";

    // Receives AtsContext (all capabilities, handles, DTOs, enums)
    // Returns filename → content dictionary
    public Dictionary<string, string> GenerateDistributedApplication(AtsContext context)
    {
        // Generate typed Python SDK from context.Capabilities ...
        return new Dictionary<string, string> { [".modules/aspire.py"] = generatedCode };
    }
}
```

**Step 3: Register via DI**
```csharp
services.AddSingleton<ILanguageSupport, PythonLanguageSupport>();
services.AddSingleton<ICodeGenerator, PythonCodeGenerator>();
```

No CLI changes needed. Adding the NuGet package is the only integration step.

---

## 12. VS Code Extension

TypeScript extension providing developer experience inside VS Code:

```
extension/
  src/
    extension.ts      ← entry point, registers all providers
    capabilities.ts   ← feature flags
    server/           ← connects to Dashboard REST API
    dcp/              ← DCP resource state watcher
    views/            ← Tree views (Resources, Logs, etc.)
    commands/         ← VS Code command palette entries
    debugger/         ← Debug session integration
    editor/           ← Code lens, completion providers
  loc/
    strings.ts        ← All localized UI strings (must match package.nls.json)
  schemas/
    aspire-settings.schema.json
```

The extension communicates with the Dashboard's REST API (`Aspire.Dashboard/Api/`) to display resource state, logs, and traces directly in the editor.

**Localization rule:** All user-visible strings must appear in both `extension/loc/strings.ts` **and** `extension/package.nls.json`.

---

## 13. Key Architectural Principles

1. **Annotation-driven extensibility** — All resource metadata is injected as annotations. Third-party integrations extend the model without forking core types. New behaviors = new annotation type + new service that scans for it.

2. **Passive data model** — `DistributedApplicationModel` is a passive data store. Resources are pure data objects. All behavior (starting, deploying, logging) lives in external services that scan the model and react to annotations.

3. **Context-aware evaluation** — `DistributedApplicationExecutionContext` tells every service whether it is in Run, Publish, or Test mode. The same app model code (`AddAzureRedis`) behaves differently in each context.

4. **Deferred value resolution** — `IValueProvider`, `ReferenceExpression`, and `EndpointReference` are lazy; they are evaluated asynchronously at startup, not at model construction time. This allows deferred config and near-circular references between services.

5. **Pipeline as first-class concept** — Provisioning, deployment, and publishing are steps in a dependency-ordered `DistributedApplicationPipeline`, not ad-hoc lifecycle hooks.

6. **No IDL for polyglot** — ATS uses C# reflection as the schema source. `[AspireExport]` attributes are the contract, making polyglot SDKs a pure output of the existing C# type system. No separate `.proto` or OpenAPI schema is needed.

7. **Hosting vs. client separation** — `Aspire.Hosting.*` is the App Host (orchestration). `Components/Aspire.*` is the client library (consumed by individual microservices). They are independent NuGet packages with different dependency graphs.

---

## 14. Local Development Setup

### Prerequisites

The repo pins a specific local .NET SDK (in `global.json`). Always use the repo's scripts or the `.dotnet/dotnet` binary, not the system `dotnet`.

```bash
# First time: installs the local .NET SDK (~30 seconds)
./restore.sh
```

### Daily Workflow

**Launch VS Code** (sets up correct PATH for the local SDK — do not use `code .` directly):
```bash
./start-code.sh
```

**Build after code changes:**
```bash
# Full build: restore + build (~3-5 minutes)
./build.sh

# Build only, skipping restore (faster once restore has been run)
./build.sh --build

# Skip native AOT compilation (~1-2 minutes faster)
./build.sh --build /p:SkipNativeBuild=true

# Full clean build
./build.sh --rebuild
```

**Run the sample app to verify setup:**
```bash
./.dotnet/dotnet run --project playground/TestShop/TestShop.AppHost/TestShop.AppHost.csproj
```

**Run tests for a specific project:**
```bash
dotnet test tests/Aspire.Hosting.Tests/Aspire.Hosting.Tests.csproj \
  -- --filter-not-trait "quarantined=true" --filter-not-trait "outerloop=true"
```

**Run a specific test method:**
```bash
dotnet test tests/Aspire.Hosting.Tests/Aspire.Hosting.Tests.csproj \
  -- --filter-method "*.TestMethodName" \
     --filter-not-trait "quarantined=true" --filter-not-trait "outerloop=true"
```

> **Critical:** This repo uses **Microsoft.Testing.Platform (MTP)**, not VSTest. Always put filters **after `--`**. The classic `--filter` argument (before `--`) **will hang**.

### VS Code Extension Development

```bash
cd extension
npm install
npm run build
# Then press F5 in VS Code to launch the Extension Development Host
```

### Recovering a Corrupted SDK

```bash
./restore.sh       # re-installs the local .NET SDK from scratch
./build.sh --rebuild
```
