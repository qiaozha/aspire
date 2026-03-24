# dotnet/aspire — Architecture Notes

A structured study guide built from Q&A sessions when getting familiar with the Aspire codebase.
Organized for progressive understanding: start with the big picture, then dive into each subsystem.

---

## Table of Contents

1. [What Is Aspire?](#1-what-is-aspire)
2. [High-Level Architecture](#2-high-level-architecture)
   - [Layered Diagram](#layered-diagram)
   - [Repository Layout](#repository-layout)
   - [Key Architectural Principles](#key-architectural-principles)
3. [The Application Model (`DistributedApplicationModel`)](#3-the-application-model)
   - [IResource — The Fundamental Unit](#iresource--the-fundamental-unit)
   - [Annotations — The Extensibility Mechanism](#annotations--the-extensibility-mechanism)
   - [Standard Capability Interfaces](#standard-capability-interfaces)
   - [The Value/Reference System — How the DAG Forms](#the-valuereference-system--how-the-dag-forms)
   - [Build Phase vs. Deployment Phase: Deferred Evaluation](#build-phase-vs-deployment-phase-deferred-evaluation)
   - [Design Philosophy: Inert Data + External Orchestration](#design-philosophy-inert-data--external-orchestration)
4. [Local Orchestration: DCP Executor](#4-local-orchestration-dcp-executor)
   - [What Is DCP?](#what-is-dcp)
   - [What DcpExecutor Reads From the Resource Graph](#what-dcpexecutor-reads-from-the-resource-graph)
   - [The AppResource Bridge Type](#the-appresource-bridge-type)
   - [How KubernetesService Fits In](#how-kubernetesservice-fits-in)
   - [State Change Propagation](#state-change-propagation)
5. [Monitoring Dashboard](#5-monitoring-dashboard)
6. [Publishing Pipeline](#6-publishing-pipeline)
7. [Azure Integration & Bicep Generation](#7-azure-integration--bicep-generation)
   - [The C# Integration API](#the-c-integration-api)
   - [What Is the Azure.Provisioning SDK?](#what-is-the-azureprovisioning-sdk)
   - [How the Provisioning CDK Connects to the AppHost API](#how-the-provisioning-cdk-connects-to-the-apphost-api)
   - [How Bicep Is Generated: The Three-Layer Stack](#how-bicep-is-generated-the-three-layer-stack)
   - [Post-CDK Modification: Extending Bicep Generation](#post-cdk-modification-extending-bicep-generation)
   - [How the Provisioning Library Communicates (In-Process, Not RPC)](#how-the-provisioning-library-communicates-in-process-not-rpc)
   - [Two Deployment Paths](#two-deployment-paths)
   - [Full Azure Resource Lifecycle Mapping](#full-azure-resource-lifecycle-mapping)
   - [Why the AppHost Must Run for Bicep Generation](#why-the-apphost-must-run-for-bicep-generation)
   - [The Two-Description Problem: CDK vs. App Model](#the-two-description-problem-cdk-vs-app-model)
   - [Why Not Unify Them as a Single Source of Truth?](#why-not-unify-them-as-a-single-source-of-truth)
   - [What Can the Synthesis World Infer from the Orchestration World?](#what-can-the-synthesis-world-infer-from-the-orchestration-world)
   - [Can the Orchestration World Be Inferred from the Synthesis World?](#can-the-orchestration-world-be-inferred-from-the-synthesis-world)
8. [Polyglot AppHost: Guest Runtime Interaction (ATS)](#8-polyglot-apphost-guest-runtime-interaction)
   - [The Key Design Principle](#the-key-design-principle)
   - [The 3-Process Architecture](#the-3-process-architecture)
   - [Startup Sequence Step by Step](#startup-sequence-step-by-step)
   - [JSON-RPC Protocol: Wire Format](#json-rpc-protocol-wire-format)
   - [Handle System: How Objects Cross the Process Boundary](#handle-system-how-objects-cross-the-process-boundary)
   - [Inside the AppHost Server: Capability Dispatch](#inside-the-apphost-server-capability-dispatch)
   - [What the Generated TypeScript SDK Looks Like](#what-the-generated-typescript-sdk-looks-like)
   - [Callbacks: Guest Code Invoked by .NET](#callbacks-guest-code-invoked-by-net)
   - [Adding a New Guest Language (e.g., Python)](#adding-a-new-guest-language-eg-python)
   - [Guest Runtime Interaction Summary Table](#guest-runtime-interaction-summary-table)
9. [SDK Code Generator](#9-sdk-code-generator)
   - [The Core Problem: No Runtime Discovery](#the-core-problem-no-runtime-discovery)
   - [Why It Must Run on First Use](#why-it-must-run-on-first-use)
   - [The Two-Phase Split: Where Code Lives](#the-two-phase-split-where-code-lives)
   - [The `aspire sdk generate` Command](#the-aspire-sdk-generate-command)
   - [Hash-Based Cache: When Regeneration Is Skipped](#hash-based-cache-when-regeneration-is-skipped)
   - [The 5-Pass Scanner](#the-5-pass-scanner)
   - [Assembly Loading in the AppHost Server](#assembly-loading-in-the-apphost-server)
10. [Aspire CLI Architecture](#10-aspire-cli-architecture)
    - [Why RPC Instead of In-Process](#why-rpc-instead-of-in-process)
    - [CLI Commands](#cli-commands)
11. [VS Code Extension](#11-vs-code-extension)
12. [Local Development Setup](#12-local-development-setup)
    - [Prerequisites](#prerequisites)
    - [Daily Workflow](#daily-workflow)
    - [VS Code Extension Development](#vs-code-extension-development)
    - [If You Rebuild from Scratch](#if-you-rebuild-from-scratch)
13. [Debugging Guide](#13-debugging-guide)
    - [Understanding the 3-Process Architecture for Debugging](#understanding-the-3-process-architecture-for-debugging)
    - [Debugging the AppHost Server (.NET Process)](#debugging-the-apphost-server-net-process)
    - [Debugging the Guest Process (TypeScript/Python)](#debugging-the-guest-process-typescriptpython)
    - [Debugging Bicep Generation](#debugging-bicep-generation)
    - [Summary: What to Debug Where](#summary-what-to-debug-where)

---

## 1. What Is Aspire?

At its core, `dotnet/aspire` is a **distributed application orchestration and deployment platform** for .NET. It provides:

- **App Host orchestration** — A code-first model where you declare services, containers, databases, and cloud resources in C#. During local dev, Aspire starts everything (Docker containers, processes) and wires them together with service discovery and environment variables.
- **A monitoring Dashboard** — Blazor-based UI for traces, logs, and metrics via OpenTelemetry.
- **40+ platform integrations** — Redis, PostgreSQL, SQL Server, MongoDB, RabbitMQ, Kafka, all Azure services, Kubernetes, etc.
- **A polyglot CLI** — Evolving from a `dotnet run` wrapper to a full orchestration CLI supporting non-.NET languages (TypeScript, Python, Go, Rust, Java).
- **Deployment publishing** — Generating infrastructure-as-code artifacts (Bicep, Kubernetes YAML) ready for production.

---

## 2. High-Level Architecture

### Layered Diagram

```
┌────────────────────────────────────────────────────────────────────────┐
│                        User-Facing Entry Points                        │
│                                                                        │
│   aspire CLI (Aspire.Cli)   VS Code Extension   App Host (Program.cs)  │
└──────────────┬─────────────────────┬─────────────────────┬─────────────┘
               │                     │                     │
               ▼                     ▼                     ▼
  ┌────────────────────┐  ┌──────────────────┐  ┌────────────────────────┐
  │  CLI Orchestration │  │  VS Code Ext.    │  │   App Host SDK         │
  │  (run/publish/add) │  │  (debug, attach) │  │   (Aspire.Hosting)     │
  └─────────┬──────────┘  └──────────────────┘  └──────────┬─────────────┘
            │                                              │
            │                               ┌──────────────▼──────────────┐
            │                               │       Application Model     │
            │                               │  IResource, Annotations DAG │
            │                               └──────────────┬──────────────┘
            │                                              │
            │             ┌────────────────────────────────┼─────────────┐
            │             │              │                 │             │
            ▼             ▼              ▼                 ▼             ▼
  ┌──────────┐  ┌──────────────┐  ┌──────────┐  ┌──────────────┐  ┌──────────┐
  │  AzDev   │  │DCP Executor  │  │Dashboard │  │  Publishing  │  │  Azure   │
  │  CLI     │  │(local run)   │  │(Blazor)  │  │  Pipeline    │  │Provision │
  └──────────┘  └──────────────┘  └──────────┘  └──────────────┘  └──────────┘
```

### Repository Layout

```
aspire/
├── src/                    # All production source code
├── tests/                  # Unit & integration tests (mirrors src/)
├── playground/             # Sample apps for manual verification
├── extension/              # VS Code extension (TypeScript)
├── docs/specs/             # Architecture specs & design documents
├── eng/                    # Build infrastructure (Arcade SDK, pipelines)
└── tools/                  # Developer tooling (QuarantineTools, etc.)
```

### Key Architectural Principles

| Principle | Description |
|-----------|-------------|
| **Annotation-driven extensibility** | All resource metadata is injected as annotations. Third-party integrations extend the model without forking core types. |
| **Context-aware behavior** | `DistributedApplicationExecutionContext` tells every resource whether it is in Run, Publish, or Test mode — the same app model behaves differently in each mode. |
| **Lazy value resolution** | `IValueProvider`, `ReferenceExpression`, and `EndpointReference` are evaluated asynchronously at startup, allowing deferred config and circular-ish references. |
| **Pipeline as first-class concept** | Provisioning, deployment, and publishing are steps in a dependency-ordered `DistributedApplicationPipeline`, not ad-hoc lifecycle hooks. |
| **No IDL for polyglot** | ATS uses C# reflection as the schema source; `[AspireExport]` attributes are the contract, making polyglot SDKs a pure output of the existing C# type system. |
| **Separation of hosting vs. client** | `src/Aspire.Hosting.*` is the app host (orchestration); `src/Components/Aspire.*` is the client library (consumed by individual microservices). They are independent NuGet packages. |

---

## 3. The Application Model

> **Q: What exactly is `DistributedApplicationModel` and how does everything connect?**

`DistributedApplicationModel` is the central data structure of an Aspire AppHost. It is intentionally simple:

```csharp
[DebuggerDisplay("Resources = {Resources.Count}")]
public class DistributedApplicationModel(IResourceCollection resources)
{
    public IResourceCollection Resources { get; } = resources;
}
```

`IResourceCollection` is just `IList<IResource>` — **a flat list, not a tree**. The graph structure (the DAG of dependencies) is implicit, encoded in annotations and `ReferenceExpression` objects attached to resources. Aspire never stores an explicit edge list; edges are discovered at runtime by traversing annotations.

It is registered as a singleton in DI. Every subsystem — `DcpExecutor`, `AzureResourcePreparer`, `BicepProvisioner`, manifest writers, the Dashboard — gets it from DI and queries `model.Resources`.

### IResource — The Fundamental Unit

```csharp
public interface IResource
{
    string Name { get; }
    ResourceAnnotationCollection Annotations { get; }
}
```

Resources are **pure inert data objects** — they don't start, stop, or manage themselves. Everything about their runtime behavior is carried in `Annotations`.

#### Resource Hierarchy

```
IResource
  └── Resource (abstract)
        ├── ContainerResource        — runs a Docker image
        ├── ProjectResource          — runs a .NET project
        ├── ExecutableResource       — runs an arbitrary binary
        ├── ParameterResource        — an external input / secret
        ├── ConnectionStringResource — a literal connection string
        └── [integration-specific]
              ├── RedisResource            : ContainerResource
              ├── PostgresServerResource   : ContainerResource
              ├── AzureProvisioningResource: AzureBicepResource
              └── ...
```

The three primitive compute types (`ContainerResource`, `ProjectResource`, `ExecutableResource`) are what DCP (DcpExecutor) knows about natively. Everything else is modeled on top of them.

### Annotations — The Extensibility Mechanism

`IResourceAnnotation` is an empty marker interface. Any POCO implementing it can be attached to any resource:

```csharp
ContainerImageAnnotation        // → image name, tag, registry
EndpointAnnotation              // → port, scheme, name
EnvironmentCallbackAnnotation   // → lambda to inject env vars
ContainerMountAnnotation        // → volume or bind mount
HealthCheckAnnotation           // → health probe config
PipelineStepAnnotation          // → deploy pipeline step
ManifestPublishingCallbackAnnotation // → how to write to azd manifest
WaitAnnotation                  // → WaitFor dependency
```

Extension methods like `AddRedis("cache")` are really just:

1. `new RedisResource("cache")` (a `ContainerResource` subclass)
2. `resource.Annotations.Add(new ContainerImageAnnotation { Image = "redis", Tag = "7.4" })`
3. `resource.Annotations.Add(new EndpointAnnotation(ProtocolType.Tcp, ..., targetPort: 6379))`
4. Wrap in `IResourceBuilder<RedisResource>` and return

`IResourceBuilder<T>` is a fluent wrapper:

```
IDistributedApplicationBuilder     (top-level, owns model + DI)
  └── .AddRedis("cache")
        └── IResourceBuilder<RedisResource>
              ├── .Resource          → the RedisResource instance
              ├── .WithEnvironment() → adds EnvironmentCallbackAnnotation
              ├── .WithDataVolume()  → adds ContainerMountAnnotation
              └── .WithHealthCheck() → adds HealthCheckAnnotation
```

### Standard Capability Interfaces

Resources opt into behaviors by implementing interfaces (polymorphism over type checks):

| Interface | What it enables |
|-----------|----------------|
| `IResourceWithEnvironment` | `.WithEnvironment()`, env var injection |
| `IResourceWithEndpoints` | `.WithEndpoint()`, `GetEndpoint()`, URL generation |
| `IResourceWithConnectionString` | `.WithReference(db)` wires connection string into consumers |
| `IResourceWithServiceDiscovery` | Registers service in DNS-style discovery |
| `IResourceWithArgs` | CLI argument injection |
| `IResourceWithWaitSupport` | `WaitFor()` — blocks startup until dependency is ready |
| `IResourceWithParent` | Lifecycle containment (child stops when parent stops) |
| `IComputeResource` | Marks as a compute unit (project/container/exec) |

### The Value/Reference System — How the DAG Forms

The graph is a heterogeneous DAG encoding both dependency order and value flows. Edges are encoded as structured value objects:

```
web ──► EndpointReference ──► api
                               │
                               └──► ConnectionStringReference ──► postgres
```

| Type | Run mode | Publish mode |
|------|----------|--------------|
| `EndpointReference` | `http://localhost:5000` | `{api.bindings.http.url}` |
| `ConnectionStringReference` | `Host=localhost;Port=5432;...` | `{postgres.connectionString}` |
| `ParameterResource` | value from user secrets / env | `${PARAM}` placeholder |
| `ReferenceExpression` | interpolated concrete string | interpolated manifest expression |

`ReferenceExpression` is the key glue type — it wraps an interpolated string handler that captures structured value objects, not their resolved values. At run time, `IValueProvider.GetValueAsync()` resolves all references to concrete strings. At publish time, `IManifestExpressionProvider.ValueExpression` emits `{pg.bindings.tcp.host}` placeholders into the azd manifest.

### Build Phase vs. Deployment Phase: Deferred Evaluation

> **Q: Is `DistributedApplicationModel` built before resource deployment happens? If so, how can it know resource identifiers and link resources without them being created yet?**

**Yes — the model is fully built before any deployment occurs.** The key design that makes this work is *deferred evaluation*: the model is a graph of intentions, not resolved values.

**Build phase (`Program.cs` startup):**

Every call to `AddAzurePostgresFlexibleServer()`, `AddProject()`, `WithReference()` etc. constructs an in-memory DAG of `IResource` objects with annotations attached. No containers are started, no ARM deployments are triggered. `DistributedApplicationModel` is a snapshot of this graph — pure data, no side effects.

**The two-interface pattern — how references cross the deployment boundary:**

Every structured value in the DAG implements two interfaces:

| Interface | Mode | Purpose |
|-----------|------|---------|
| `IManifestExpressionProvider` | Publish | Emits a structured placeholder like `{pg.outputs.connectionString}` into Bicep/manifest — no real value needed |
| `IValueProvider` | Run | Asynchronously resolves to a real value **after** the resource is provisioned |

`BicepOutputReference` is a concrete example — it wraps the name of an ARM deployment output:

```csharp
// In run mode: blocks until ARM deployment finishes, then returns the actual ARM output value
public async ValueTask<string?> GetValueAsync(CancellationToken cancellationToken = default)
{
    var provisioning = Resource.ProvisioningTaskCompletionSource;
    if (provisioning is not null)
    {
        await provisioning.Task.WaitAsync(cancellationToken);  // ← waits for ARM deployment
    }
    return Value;  // ← reads actual output after provisioning completes
}
```

`ProvisioningTaskCompletionSource` is a `TaskCompletionSource` that `BicepProvisioner` completes once the ARM deployment finishes and outputs are read back. Any consumer process that has `WithReference(db)` blocks on this task before its environment variables are resolved and injected.

**The full flow:**

```
Program.cs build phase
  └── DAG constructed (IResource objects + annotations, BicepOutputReferences as placeholders)
      └── DistributedApplicationModel frozen (no resources created yet)

Run/Deploy phase (after build)
  ├── Publish mode: IManifestExpressionProvider.ValueExpression
  │     → emits "{pg.outputs.connectionString}" into Bicep params as a symbolic expression
  │     (no actual values needed — module-to-module wiring is purely symbolic)
  └── Run mode: IValueProvider.GetValueAsync()
        → BicepProvisioner deploys the ARM template
        → sets resource.Outputs["connectionString"] from ARM deployment outputs
        → completes ProvisioningTaskCompletionSource
        → unblocks all consumers waiting for that reference
        → environment variables injected into child processes
```

**Why resource identifiers work without resources existing:**

In publish mode, identifiers are never needed. A `BicepOutputReference` for `pg.outputs.connectionString` becomes the literal string `{pg.outputs.connectionString}` — a Bicep parameter that wires modules together symbolically. In run mode, resources are provisioned in dependency order derived from the DAG, and consumer processes never start until their dependencies' `TaskCompletionSource` fires. The model itself never holds actual IDs — it holds handles to future values that resolve at the right time.

### Design Philosophy: Inert Data + External Orchestration

The design principle is:

- `DistributedApplicationModel` is a **passive data store** — just the resource list
- **Resources are pure data objects** — annotation bags with a name
- **All behavior** (starting, stopping, deploying, manifest-writing, logging) lives in external services that scan the model and react to annotations

New capabilities (like a new deployment target or dashboard widget) are added by creating a new annotation type + a new service that scans for it — zero changes to the core model.

---

## 4. Local Orchestration: DCP Executor

> **Q: What is DCP and what resource graph information does DcpExecutor consume? How is KubernetesService related?**

### What Is DCP?

DCP (Developer Control Plane) is a **separate native binary** shipped as part of the Aspire workload. It runs on the same machine as the App Host and exposes a **Kubernetes-style CRD API** over a local Unix socket / named pipe.

From DCP's perspective, resources are Kubernetes custom objects (CRDs), not containers or processes directly — DCP abstracts the actual container runtime (Docker, Podman) and process launcher behind that API surface.

```
App Host Process
  └─ DcpHost (starts DCP subprocess)
      └─ DcpExecutor
          ├─ Translates IResource graph → DCP Kubernetes objects
          │   (Container, Executable, Service CRDs)
          ├─ DcpKubernetesClient → HTTP to DCP's local K8s API
          ├─ Watches resource state changes → ResourceNotificationService
          └─ Streams logs → ResourceLoggerService → Dashboard
```

### What DcpExecutor Reads From the Resource Graph

`DcpExecutor` walks `DistributedApplicationModel` and reads these annotations from each `IResource`:

| Annotation / Resource Type | DCP CRD Produced | Key Fields Mapped |
|---------------------------|------------------|-------------------|
| `ContainerImageAnnotation` | `Container` (CRD) | image, ports, env, volumeMounts, command, args, restartPolicy, networks |
| `DockerfileBuildAnnotation` | `Container.Spec.Build` | Dockerfile path, context, build args |
| `EndpointAnnotation` | `Service` (CRD) | address, port, protocol, addressAllocationMode |
| `ProjectResource` / `ExecutableResource` | `Executable` (CRD) | executablePath, workingDirectory, args, env, healthProbes |
| `ContainerMountAnnotation` | `Container.Spec.VolumeMounts` | Source, target, type (bind/volume) |
| `EnvironmentAnnotation` / `EnvironmentCallbackAnnotation` | `Container.Spec.Env` / `Executable.Spec.Env` | Resolved key/value EnvVar list |
| `HealthCheckAnnotation` | `Executable.Spec.HealthProbes` | HTTP/TCP probe config |
| `WaitAnnotation` | Ordering between Service objects | Which services must be ready before this resource starts |
| `ProbeAnnotation` | `Container.Spec.HealthProbes` | Liveness/readiness probes |

### The AppResource Bridge Type

`AppResource` (in `AppResource.cs`) is the coupling object between the Aspire app model and the DCP model. It lives only in the executor's `_appResources` list — never sent over the wire:

```
IResource (Aspire app model)
    │
    └─ RenderedModelResource : AppResource
            ├─ ModelResource         → the original IResource
            ├─ DcpResource           → the CustomResource CRD object (Container/Executable)
            ├─ ServicesProduced      → List<ServiceWithModelResource>  (endpoints this resource exposes)
            └─ ServicesConsumed      → List<ServiceWithModelResource>  (endpoints this resource depends on)

ServiceWithModelResource : RenderedModelResource
    ├─ Service                → the DCP Service CRD
    └─ EndpointAnnotation     → the original EndpointAnnotation from the model
```

One `RenderedModelResource` is created per runnable Aspire resource. One `ServiceWithModelResource` is created per `EndpointAnnotation`. The `ServicesConsumed` list drives startup ordering (wait conditions).

### How KubernetesService Fits In

`IKubernetesService` (implemented by `KubernetesService`) is the **only channel** between the executor and DCP:

```
DcpExecutor                               DCP Binary (local process)
    │                                           │
    │  ─── HTTP over unix socket ──────────►    │
    │                                           │
    │  CreateAsync<Container>(spec)         K8s-style CRD API
    │  CreateAsync<Executable>(spec)        (list, get, create, patch, delete, watch)
    │  CreateAsync<Service>(spec)               │
    │  WatchAsync<Container>()    ◄──────────── │ (long-poll watch stream)
    │  WatchAsync<Executable>()   ◄──────────── │
    │  GetLogStreamAsync(...)     ◄──────────── │ (stdout/stderr as HTTP stream)
```

It uses the official k8s .NET client library (the same used for real Kubernetes), pointed at DCP's local endpoint. Every DCP CRD type (`Container`, `Executable`, `Service`, `Endpoint`, `ContainerExec`, `ContainerNetwork`) implements `IKubernetesStaticMetadata` so the generic `CreateAsync<T>` / `WatchAsync<T>` API works uniformly.

### State Change Propagation

After resource creation, `DcpExecutor` runs concurrent watch loops:

```
WatchKubernetesResourceAsync<Executable>(...)
WatchKubernetesResourceAsync<Container>(...)   →  ProcessResourceChange()
WatchKubernetesResourceAsync<Service>(...)              │
WatchKubernetesResourceAsync<Endpoint>(...)             ▼
                                              ResourceSnapshotBuilder.ToSnapshot()
                                                        │
                                                        ▼
                                              ResourceNotificationService.PublishUpdateAsync()
                                                        │
                                                        ▼
                                              Dashboard (gRPC stream) + CLI backchannel
```

`ResourceNotificationService` is an in-process pub/sub bus. The Dashboard subscribes to it and re-renders resource state in real time.

---

## 5. Monitoring Dashboard

A Blazor Server web app for monitoring running resources:

```
Dashboard Web App (runs in-process with App Host or standalone)
  ├─ Otlp/          ← OTLP gRPC receiver (traces, metrics, logs)
  │    └─ OtlpGrpcService.cs
  ├─ Model/         ← In-memory state of resources, spans, logs
  ├─ Components/    ← Blazor UI components & pages
  │    ├─ Pages/ResourcesPage, TraceDetail, Logs, Metrics
  │    └─ Controls/
  ├─ ServiceClient/ ← gRPC client for talking to App Host resource API
  ├─ Api/           ← REST API for the VS Code extension to consume
  └─ Mcp/           ← MCP (Model Context Protocol) server endpoint
```

The Dashboard subscribes to `ResourceNotificationService` via gRPC streaming from the App Host and renders live resource state. It also receives OTLP telemetry directly from instrumented services.

---

## 6. Publishing Pipeline

Publishing translates the app model into deployment artifacts. It runs when `aspire publish` or `azd` invokes the App Host in **publish mode**.

```
DistributedApplicationPipeline
  ├─ PipelineStep (ordered, dependency-aware)
  │   ├─ "provision-redis"        (AzureBicepResource step)
  │   ├─ "provision-keyvault"     (AzureBicepResource step)
  │   └─ "write-manifest"         (ManifestPublisher step)
  └─ PipelineExecutor (runs steps in topological order)

ManifestPublisher
  └─ Calls ManifestPublishingCallbackAnnotation per resource
      └─ Each resource writes its JSON block to aspire-manifest.json
```

The manifest (`aspire-manifest.json`) is the intermediate format consumed by `azd`, Kubernetes publishers, or custom tooling.

---

## 7. Azure Integration & Bicep Generation

> **Q: How is the resource graph used in generating Bicep files? How does provisioning communicate with the AppHost?**

### Short Answer

The resource graph (from `DistributedApplicationModel.Resources`) is the source of truth for what Azure resources exist. **Bicep content is generated per-resource** using the Azure.Provisioning SDK — not from a graph traversal. Cross-resource wiring is handled through a flat `moduleMap` dictionary.

The provisioning library does **not** communicate with the AppHost server via RPC — they run **in the same process**.

---

### The C# Integration API

Every Azure resource type has a dedicated project (`src/Aspire.Hosting.Azure.*/`) with a consistent fluent API:

```csharp
// 1. Declare the Azure resource
var cache = builder.AddAzureManagedRedis("cache");

// 2. Optionally swap in a local Docker container for dev mode
var cache = builder.AddAzureManagedRedis("cache")
                   .RunAsContainer();

// 3. Configure auth, role assignments, networking
var cache = builder.AddAzureManagedRedis("cache")
                   .WithAccessKeyAuthentication();

// 4. Wire it to services like any other resource
builder.AddProject<Projects.Api>()
       .WithReference(cache);
```

**Context-awareness pattern** — the same declaration behaves differently based on mode:

| Context | Behavior |
|---------|----------|
| `aspire run` (local) | `RunAsContainer()` — removes Azure resource, adds local Docker container instead |
| `aspire publish` | Emits Bicep + manifest, no containers started |
| Azure provisioning (`aspire run` with Azure provisioner) | Actually deploys to Azure via ARM |

---

### What Is the Azure.Provisioning SDK?

`Azure.Provisioning.*` NuGet packages (by the Azure SDK team — e.g., `Azure.Provisioning.Redis`, `Azure.Provisioning.KeyVault`) are a **C#-first infrastructure-as-code SDK**. They let you declare Azure infrastructure as typed C# objects that compile to Bicep text. Think of it as the Azure equivalent of CDK or Pulumi's native SDK.

| SDK Type | Role |
|----------|------|
| `ProvisionableResource` | Base class for all Azure resources (e.g. `RedisResource`, `StorageAccount`) |
| `ProvisioningParameter` | A Bicep `param` declaration |
| `ProvisioningOutput` | A Bicep `output` declaration |
| `BicepValue<T>` | A typed Bicep expression (literal, reference, or function call) |
| `BicepFunction.Interpolate(...)` | Produces Bicep string interpolation (`'${...}'`) |
| `Infrastructure.Build().Compile()` | Runs the entire graph and emits Bicep file content |
| `ResourceX.FromExisting(...)` | References an already-deployed Azure resource by name |

Example — what `ConfigureRedisInfrastructure` does internally:

```csharp
private static void ConfigureRedisInfrastructure(AzureResourceInfrastructure infrastructure)
{
    // 1. Declare an Azure Redis (Azure.Provisioning.Redis.RedisResource)
    var redis = AzureProvisioningResource.Create<CdkRedisResource>(infrastructure, ...);
    infrastructure.Add(redis);

    // 2. Add auth configuration
    redis.IsAccessKeyAuthenticationDisabled = true;
    redis.RedisConfiguration = new RedisCommonConfiguration { IsAadEnabled = "true" };

    // 3. Declare Bicep outputs — these become connection string references
    infrastructure.Add(new ProvisioningOutput("connectionString", typeof(string))
    {
        Value = BicepFunction.Interpolate($"{redis.HostName},ssl=true")
    });
    infrastructure.Add(new ProvisioningOutput("name", typeof(string))
    {
        Value = redis.Name.ToBicepExpression()
    });

    // 4. Optionally store secret in Key Vault
    var secret = new KeyVaultSecret("connectionString")
    {
        Parent = keyVault,
        Properties = new SecretProperties
        {
            Value = BicepFunction.Interpolate(
                $"{redis.HostName},ssl=true,password={redis.GetKeys().PrimaryKey}")
        }
    };
    infrastructure.Add(secret);
}
```

---

### How Bicep Is Generated: The Three-Layer Stack

```
Layer 1: C# code (Azure.Provisioning SDK) → Bicep text
Layer 2: Bicep text → ARM JSON (bicep CLI binary)
Layer 3: ARM JSON → Azure deployment (Azure Resource Manager REST API)
```

**Layer 1 — `AzureProvisioningResource.GetBicepTemplateFile()`:**

```csharp
var infrastructure = new AzureResourceInfrastructure(this, Name);
ConfigureInfrastructure(infrastructure);      // user's callback runs here
var plan = infrastructure.Build(options);
var compilation = plan.Compile();             // Azure.Provisioning SDK → Bicep text
File.WriteAllText(moduleSourcePath, compiledBicep.Value);
```

No Bicep files are pre-authored — they are all generated at runtime from C#.

**Layer 2 — `BicepCliCompiler.CompileBicepToArmAsync()`:**

Shells out to `bicep build --stdout` (or `az bicep build` as fallback). The `bicep` binary must be on PATH.

**Layer 3 — `BicepProvisioner.GetOrCreateResourceAsync()`:**

```csharp
var deployments = resourceGroup.GetArmDeployments();
var operation = await deployments.CreateOrUpdateAsync(
    WaitUntil.Started, deploymentName,
    new ArmDeploymentContent(new(ArmDeploymentMode.Incremental)
    {
        Template = BinaryData.FromString(armTemplateContents),
        Parameters = BinaryData.FromObjectAsJson(parameters),
    }), cancellationToken);
await operation.WaitForCompletionAsync(cancellationToken);
```

---

### Post-CDK Modification: Extending Bicep Generation

> **Q: Can Aspire modify the Bicep content after the CDK libraries have defined it?**

Yes — but not by text manipulation. Aspire modifies the **CDK object graph** before synthesis. The synthesis window is strictly between the CDK callbacks running and `Build()` being called. After `File.WriteAllText` the generated Bicep string is cached in `_generatedBicep` and never touched again.

**The exact sequence inside `GetBicepTemplateFile()`** ([AzureProvisioningResource.cs](src/Aspire.Hosting.Azure/AzureProvisioningResource.cs)):

```
1. new AzureResourceInfrastructure(this, Name)   ← empty CDK graph, "location" param pre-added
2. ConfigureInfrastructure(infrastructure)        ← all registered CDK callbacks run (multicast)
3. EnsureParametersAlign(infrastructure)          ← Aspire post-processing (see below)
4. infrastructure.Build(ProvisioningBuildOptions) ← CDK synthesizes to Bicep text
5. File.WriteAllText(...)                         ← text written once, never modified again
```

**`ConfigureInfrastructure` is a composable multicast delegate:**

`AzureProvisioningResource.ConfigureInfrastructure` is a plain `Action<AzureResourceInfrastructure>` property. The `ConfigureInfrastructure<T>()` extension method (in `AzureProvisioningResourceExtensions.cs`) appends additional callbacks using `+=`:

```csharp
// AzureProvisioningResourceExtensions.cs
public static IResourceBuilder<T> ConfigureInfrastructure<T>(
    this IResourceBuilder<T> builder,
    Action<AzureResourceInfrastructure> configure)
    where T : AzureProvisioningResource
{
    builder.Resource.ConfigureInfrastructure += configure;   // ← appended, not replaced
    return builder;
}
```

This means every `.ConfigureInfrastructure(infra => { ... })` call on a resource builder adds another pass over the CDK object graph that runs at synthesis time. The integration's own initial callback runs first; user-supplied customizations run after. Example:

```csharp
var redis = builder.AddAzureManagedRedis("cache")
    .ConfigureInfrastructure(infra =>
    {
        // This runs after the integration's built-in ConfigureInfrastructure callback
        var redisCache = infra.GetProvisionableResources()
                              .OfType<RedisCache>()
                              .Single();
        redisCache.Sku = new RedisSkuInfo { Name = RedisSkuName.Premium, Capacity = 1 };
    });
```

**`EnsureParametersAlign` — post-CDK parameter reconciliation (step 3):**

After all CDK callbacks run, Aspire reconciles the CDK's synthesized `ProvisioningParameter` list with the Aspire-level `Parameters` dictionary:

- CDK parameter exists but Aspire `Parameters` doesn't know about it → added to `Parameters` so it appears in the manifest
- Aspire `Parameters` has an entry but the CDK is missing the corresponding parameter → a CDK `ProvisioningParameter` is injected (backwards compatibility for old-style `WithParameter` usage)

**What does NOT happen:**

- No regex or string rewriting of the `.bicep` text after synthesis
- No post-`Build()` Bicep modification
- The generated Bicep string is cached (`_generatedBicep`) after the first call to `GetBicepTemplateString()` — it is frozen from that point on

---

### How the Provisioning Library Communicates (In-Process, Not RPC)

The provisioning library is **not** a separate server — it is loaded as a regular .NET assembly inside the AppHost process:

```
AzureBicepResource constructor
  ├── ManifestPublishingCallbackAnnotation → WriteToManifest() (for azd manifest)
  ├── PipelineStepAnnotation → creates "provision-{name}" step
  └── PipelineConfigurationAnnotation → wires up dependency ordering

DistributedApplicationPipeline
  └── walks all resources → collects PipelineStepAnnotations → sorts topologically → executes

"provision-redis" step:
  bicepProvisioner = context.Services.GetRequiredService<IBicepProvisioner>()  // DI, in-process
  provisioningContext = await azureEnvironment.ProvisioningContextTask.Task    // TaskCompletionSource
  await bicepProvisioner.GetOrCreateResourceAsync(resource, provisioningContext, ...)
```

| From → To | Mechanism |
|-----------|-----------|
| Pipeline executor → BicepProvisioner | `IServiceProvider.GetRequiredService<IBicepProvisioner>()` |
| BicepProvisioner → Dashboard/UI | `ResourceNotificationService.PublishUpdateAsync()` (in-memory) |
| BicepProvisioner → bicep CLI | `Process.Start("bicep build ...")` (subprocess) |
| BicepProvisioner → Azure ARM API | `Azure.ResourceManager` SDK over HTTPS |
| One Bicep resource waits on another | `TaskCompletionSource` on `AzureEnvironmentResource.ProvisioningContextTask` |

---

### Two Deployment Paths

**Path A: `aspire run` (local dev provisioning)**

The provisioning pipeline runs inside the AppHost process:

```
AzureBicepResource.constructor
  → adds PipelineStepAnnotation
  → pipeline step calls ProvisionAzureBicepResourceAsync
  → BicepProvisioner.GetOrCreateResourceAsync()
  → ARM deployment
```

Deployment state is cached in `~/.aspire/deployments/<hash>.json` so re-runs skip provisioning unless the resource configuration has changed.

**Path B: `azd up` (Azure Developer CLI deployment)**

`AzureBicepResource.WriteToManifest()` writes to the Aspire manifest that `azd` reads:

```json
{
  "type": "azure.bicep.v0",
  "path": "redis.module.bicep",
  "params": { "location": "{azure.location}" }
}
```

`azd` reads this manifest and the `.bicep` files alongside it, then runs its own ARM deployment pipeline. Aspire produces the artifacts; `azd` handles the actual deployment.

---

### Full Azure Resource Lifecycle Mapping

```
C# API call: builder.AddAzureManagedRedis("cache")
    │
    ├─ Creates AzureManagedRedisResource (: AzureProvisioningResource)
    │   └─ Stores ConfigureInfrastructure callback (a C# lambda/closure)
    │
    ├─ Attaches ManifestPublishingCallbackAnnotation
    │   └─ WriteToManifest() → writes "type: azure.bicep.v0" to manifest.json
    │
    ├─ Attaches PipelineStepAnnotation
    │   └─ "provision-cache" step → calls BicepProvisioner.GetOrCreateResourceAsync()
    │
    └─ Attaches PipelineConfigurationAnnotation
        └─ Evaluates Bicep template to discover cross-resource dependencies

At run/publish time:
    GetBicepTemplateFile()
        → new AzureResourceInfrastructure(this, Name)
        → ConfigureInfrastructure(infra)          ← Azure.Provisioning objects added here
        → infra.Build(options).Compile()           ← Azure.Provisioning emits Bicep text
        → writes .bicep file to temp dir

BicepProvisioner (run mode):
    → BicepCliCompiler: `az bicep build --file x.bicep --stdout` → ARM JSON
    → ArmDeploymentContent → Azure ARM REST API (incremental deployment)
    → Reads ARM deployment outputs back (hostName, connectionString, name)
    → Injects into resource SecretOutputs / connection string

AzurePublishingContext (publish mode):
    → Assembles all .bicep files → output directory
    → Writes main.bicep orchestrating all modules
    → Writes azd-compatible parameters + manifest
```

---

### Why the AppHost Must Run for Bicep Generation

This is not a quirk — it is the standard **synthesis model** used by all code-as-infrastructure tools. In AWS CDK, you run `cdk synth` to execute your CDK app and produce CloudFormation. In Pulumi, `pulumi preview` runs your Pulumi program. In Aspire, `aspire publish` runs the AppHost in publish mode to produce Bicep.

**The AppHost is the synthesis runtime. Running it is the synthesis step.**

```
AWS CDK:    cdk synth        → executes your CDK app code    → emits CloudFormation YAML
Pulumi:     pulumi preview   → executes your Pulumi program  → emits resource plan
Aspire:     aspire publish   → executes the AppHost          → emits Bicep modules
```

The `ConfigureInfrastructure` lambda is the infrastructure definition — the equivalent of a CDK construct body. It is a closure that reads configuration state accumulated by the full infrastructure definition code before it runs. This state (e.g., which blob containers were declared, whether HNS is enabled, whether private endpoints are attached) is the output of the synthesis-time execution, not something that needs to pre-exist in Azure.

This is also why the AppHost starts in a special `PublishMode` — DCP is skipped (no local containers, no dashboard), because synthesis does not require the local runtime. Only the infrastructure definition code needs to execute.

**The publish flow end-to-end:**

```
aspire publish (CLI)
    │
    ├─► launches: dotnet run AppHost.csproj --operation publish --publisher azure
    │       ↓
    │   AppHost starts in PublishMode (DcpHost skipped — no containers, no dashboard)
    │   ↓
    │   Pipeline assembled from PipelineStepAnnotations
    │   ↓
    │   AzureEnvironmentResource.PublishAsync() fires
    │   ↓
    │   AzurePublishingContext.WriteModelAsync(model, environment)
    │       ├─ iterates model.Resources (fully materialized in-memory list)
    │       ├─ calls resource.GetBicepTemplateFile() → runs ConfigureInfrastructure()
    │       └─ writes .bicep files to disk
    │
    └─► backchannel (Unix socket, JSON-RPC)
            ← App Host streams progress updates back to CLI for display
```

The backchannel is for **progress reporting only** — all Bicep generation happens inside the App Host process.

---

### How the Provisioning CDK Connects to the AppHost API

The bridge is `AzureResourceInfrastructure`, which **inherits directly from `Azure.Provisioning.Infrastructure`**:

```
Azure.Provisioning.Infrastructure          ← from azure-sdk-for-net/sdk/provisioning/
        ▲
        │ inherits
AzureResourceInfrastructure               ← src/Aspire.Hosting.Azure/AzureResourceInfrastructure.cs
        │
        └── AspireResource property        ← links back to AzureProvisioningResource (the Aspire node)
```

There are two key handoff points:

**Handoff 1: `ConfigureInfrastructure` stores CDK objects (at registration time)**

```csharp
// src/Aspire.Hosting.Azure.Storage/AzureStorageExtensions.cs
var configureInfrastructure = (AzureResourceInfrastructure infrastructure) =>
{
    // Azure.Provisioning.Storage.StorageAccount ← CDK type from azure-sdk-for-net
    var storageAccount = new StorageAccount(...)
    {
        Kind = StorageKind.StorageV2,
        Sku = new StorageSku() { Name = StorageSkuName.StandardGrs },
        ...
    };
    infrastructure.Add(storageAccount);   // ← Infrastructure.Add() — pure Azure.Provisioning call

    // CDK outputs declared here — names must match BicepOutputReference properties on the resource class
    infrastructure.Add(new ProvisioningOutput("blobEndpoint", typeof(string))
        { Value = storageAccount.PrimaryEndpoints.BlobUri.ToBicepExpression() });
};

// The lambda is stored; the CDK objects inside are NOT created yet
var resource = new AzureStorageResource(name, configureInfrastructure);
return builder.AddResource(resource);
```

The lambda is only executed when `GetBicepTemplateFile()` is called at deploy/publish time (see [How Bicep Is Generated](#how-bicep-is-generated-the-three-layer-stack) and [Full Azure Resource Lifecycle Mapping](#full-azure-resource-lifecycle-mapping) for the full execution flow).

**Handoff 2: `BicepOutputReference` reads ARM deployment outputs back into the Aspire model**

After ARM deployment completes, the provisioner reads the output values by name into `resource.Outputs[key]`. `BicepOutputReference("blobEndpoint", resource)` resolves to that value when any consumer calls `GetValueAsync()`. This is the return path: CDK defines what outputs exist; the resource class declares typed properties that consume them by name.

---

### The Two-Description Problem: CDK vs. App Model

An Aspire integration author must maintain **two parallel descriptions** of the same Azure resource:

| | CDK `ConfigureInfrastructure` callback | Aspire resource class |
|---|---|---|
| **Purpose** | Emit Bicep text (what Azure infrastructure exists) | App model node (DAG, references, DI wiring) |
| **Executes when** | Publish/deploy time only | Registration time (always) |
| **Types used** | `Azure.Provisioning.*` CDK types | `IResource`, `IResourceWithConnectionString`, etc. |
| **What it produces** | `.bicep` file on disk | Entry in `DistributedApplicationModel.Resources` |
| **Coupling point** | emits `ProvisioningOutput("blobEndpoint")` | reads `new BicepOutputReference("blobEndpoint", this)` |

The coupling between the two is a **plain string** — the output name. There is no compiler enforcement. If an author adds `ProvisioningOutput("connectionString")` in the CDK callback but forgets the matching `BicepOutputReference("connectionString", this)` property on the resource class, it compiles fine and fails only at runtime.

**Storage as concrete example:**

```
ConfigureInfrastructure (CDK side)               AzureStorageResource class (Aspire side)
─────────────────────────────────────            ─────────────────────────────────────────
ProvisioningOutput("blobEndpoint", ...)   ←→     BicepOutputReference("blobEndpoint", this)
ProvisioningOutput("queueEndpoint", ...)  ←→     BicepOutputReference("queueEndpoint", this)
ProvisioningOutput("tableEndpoint", ...)  ←→     BicepOutputReference("tableEndpoint", this)
ProvisioningOutput("dataLakeEndpoint", .) ←→     BicepOutputReference("dataLakeEndpoint", this)
ProvisioningOutput("name", ...)           ←→     BicepOutputReference("name", this)
ProvisioningOutput("id", ...)             ←→     BicepOutputReference("id", this)
```

The two layers are independent — CDK types are never directly converted into Aspire `IResource` objects. They serve different roles:

```
CDK types used inside ConfigureInfrastructure    Role
──────────────────────────────────────────────   ───────────────────────────────────────────────
StorageAccount                                   ARM resource; emits blobEndpoint/queueEndpoint outputs
BlobService                                      ARM hierarchy node (parent of BlobContainers in Bicep)
QueueService                                     ARM hierarchy node (parent of Queues in Bicep)
TableService                                     ARM hierarchy node (parent of Tables in Bicep)
FileShareService                                 Not used in Aspire's storage integration

Aspire IResource registrations (via builder.AddResource())
──────────────────────────────────────────────────────────
AzureStorageResource                             Top-level storage node; consumes blobEndpoint/queueEndpoint outputs
AzureBlobStorageResource                         Exposes blob endpoint to app consumers  (via AddBlobs())
AzureQueueStorageResource                        Exposes queue endpoint                  (via AddQueues())
AzureTableStorageResource                        Exposes table endpoint                  (via AddTables())
AzureBlobStorageContainerResource                Individual container access              (via AddBlobContainer())
```

Notice that no CDK type maps directly to an Aspire `IResource` type — they are parallel designs. The only link between the two columns is the `ProvisioningOutput`/`BicepOutputReference` string coupling shown in the table above.

---

### Why Not Unify Them as a Single Source of Truth?

**The root cause: Aspire serves two masters, CDK serves one.**

The `cdk synth` analogy explains the publish path, but CDK only has one output target — CloudFormation. A CDK `Bucket` construct doesn't need to do anything at development time: it doesn't appear in a dashboard, it doesn't resolve connection strings for local dev, it doesn't participate in `WithReference()` wiring.

Aspire has two completely separate output targets:

```
aspire publish  →  ConfigureInfrastructure lambda  →  Bicep file on disk (for Azure deployment)
aspire run      →  AzureStorageResource class       →  live app graph node (for local dev, dashboard,
                                                        service discovery, WithReference())
```

`aspire run` never executes the CDK lambda at all. The `AzureStorageResource` resolves its blob endpoint from a local Azurite container. `aspire publish` runs the CDK lambda and ignores local containers entirely. These are two fundamentally different modes of the same resource type, with no overlap at runtime.

A `StorageAccount` CDK construct cannot express "I am an Azurite container in local mode" — it is a Bicep emission tool, not a runtime-aware app node. This is the irreducible reason two descriptions must exist: the orchestration world (`aspire run`) and the synthesis world (`aspire publish`) need different types.

The three tensions below are consequences of this root split:

**Tension 1: Different lifetimes**

The Aspire resource class must exist from the moment `AddAzureStorage()` is called — it enters `DistributedApplicationModel` immediately and is used by DCP, the dashboard, `WithReference()`, and health checks. The CDK callback only runs at deploy/publish time. A single unified type would need to be simultaneously a live in-memory app node and a deferred Bicep description.

**Tension 2: `BicepOutputReference` values exist only after ARM deployment**

`BicepOutputReference("blobEndpoint")` resolves to a concrete URL only after ARM deployment completes. During `aspire run` with a local Azurite emulator, there is no deployment — the same `AzureStorageResource` resolves connection strings from the container's ports. A CDK type has no way to express "I'm a `StorageAccount` under Bicep AND I'm an Azurite endpoint in local mode."

**Tension 3: The CDK is a Bicep emission tool, not an app model**

`Azure.Provisioning.Storage.StorageAccount` exposes ~200 ARM properties. The Aspire resource class consciously surfaces only 4 (`BlobEndpoint`, `QueueEndpoint`, `TableEndpoint`, `DataLakeEndpoint`). That reduction is editorial judgment — it cannot be derived from the ARM schema.

**What partial unification already exists:**

The `AzurePublishingContext` does traverse `IValueWithReferences` on the Aspire model to discover which `BicepOutputReference` values are actually consumed, so the main Bicep module's outputs are demand-driven rather than exhaustive. But the reverse (CDK → auto-generating typed resource properties) has no equivalent mechanism, because `ConfigureInfrastructure` is a runtime lambda that cannot be statically analyzed.

A source generator that runs the AppHost in "schema mode" to introspect the CDK output graph and auto-generate `BicepOutputReference` properties is theoretically possible but introduces the same complexity as `aspire sdk generate` — at that point you'd also need the server to be running to reflect on the lambda, defeating the purpose.

**The current design is a deliberate tradeoff**: accept string-coupling between `ProvisioningOutput` names and `BicepOutputReference` names in exchange for keeping two fundamentally different concerns (Bicep emission vs. app model) in clearly separate layers.

---

### What Can the Synthesis World Infer from the Orchestration World?

**User intent (topology) is already inferred — the irreducible gap is ARM schema knowledge.**

The `ConfigureInfrastructure` lambda already reads heavily from the orchestration-world resource:

```csharp
// All of these come from the AzureStorageResource populated at registration time:
azureResource.IsHnsEnabled           → StorageAccount.IsHierarchicalNamespaceEnabled
azureResource.BlobContainers         → loop: blobContainer.ToProvisioningEntity() per container
azureResource.DataLakeFileSystems    → loop: dataLakeFileSystem.ToProvisioningEntity() per FS
azureResource.Queues                 → loop: queue.ToProvisioningEntity() per queue
azureResource.TableStorageBuilder    → creates TableService if not null
azureResource.HasAnnotation<PrivateEndpointTargetAnnotation>() → NetworkRuleSet.DefaultAction
```

`ToProvisioningEntity()` on leaf resource types (`AzureBlobStorageContainerResource`, `AzureQueueStorageQueueResource`) is already a mechanical bridge — it converts the Aspire representation directly into a CDK object. This works because those leaf types have a 1:1 obvious correspondence with their ARM equivalents.

**What the lambda must supply that cannot be inferred:**

```csharp
// ARM schema choices — not expressible in the app model:
Kind = StorageKind.StorageV2         // what "storage" means in ARM taxonomy
Sku = StorageSkuName.StandardGrs    // default SKU — editorial judgment
MinimumTlsVersion = Tls1_2         // security compliance default
AllowSharedKeyAccess = false        // security default

// ARM hierarchy knowledge:
new BlobService("blobs") { Parent = storageAccount }
    // BlobService is a required ARM parent node for BlobContainer;
    // the app model has no concept of this intermediate node

// ARM property path knowledge:
storageAccount.PrimaryEndpoints.BlobUri.ToBicepExpression()
    // which of ~200 StorageAccount properties maps to "blobEndpoint"
    // — requires reading ARM API documentation, not derivable from app model types
```

The orchestration world says: *"there is an Azure Storage resource with 3 blob containers and HNS off."* It says nothing about ARM resource `kind`, SKU, TLS policy, the `BlobService` hierarchy node that ARM requires as a parent for containers, or which `PrimaryEndpoints.*` property path emits the blob URL.

**The structural reason the gap is irreducible:**

The Aspire app model intentionally stays abstract — it expresses developer intent (*"I want blob storage"*), not Azure ARM vocabulary. `StorageV2`, `StandardGrs`, `Tls1_2` are ARM taxonomy terms with no Aspire equivalents. The CDK lambda is precisely where that ARM vocabulary gets introduced. The work of encoding ARM schema knowledge doesn't disappear if you change the representation — it just moves somewhere else (a declarative config file, a code generator, a convention table). The C# lambda is the current encoding of that knowledge.

This is also why `ToProvisioningEntity()` only exists on simple leaf resources where the mapping is 1:1 and mechanical, while the top-level `AzureStorageResource` has no `ToProvisioningEntity()` — its CDK creation requires authorial judgment about ARM defaults that cannot be mechanically derived.

---

### Can the Orchestration World Be Inferred from the Synthesis World?

**No — the synthesis world is a lossy, one-way projection of the orchestration world.**

**Reason 1: The CDK lambda is a field on the orchestration-world object and takes it as input**

The `configureInfrastructure` lambda captures `azureResource` by closure and is stored as a field on `AzureStorageResource`. You cannot call the lambda without the orchestration-world object already existing. There is no synthesis world without the orchestration world — the dependency is structurally one-directional.

**Reason 2: The sub-resource consumer hierarchy is completely absent from CDK**

`AzureBlobStorageResource` is an Aspire `IResource` with `IResourceWithConnectionString` and `IResourceWithParent<AzureStorageResource>` — it exists so consuming services can call `WithReference(blobStorage)` and receive the correct endpoint injected as an environment variable.

In the CDK world, `BlobService` is an ARM hierarchy node required as a parent for `BlobContainer` entries in Bicep. It has no consumer-endpoint semantics. Nothing in the emitted Bicep says "there is a blob consumer endpoint that other services reference". You cannot derive the existence of `AzureBlobStorageResource` (or `AzureQueueStorageResource`, `AzureTableStorageResource`) from the CDK output:

```
Orchestration world (Aspire IResource hierarchy)    Synthesis world (ARM hierarchy in Bicep)
────────────────────────────────────────────────    ─────────────────────────────────────────
AzureStorageResource                                StorageAccount
  AzureBlobStorageResource          ✗ no equivalent → (BlobService is an ARM structural node,
  AzureQueueStorageResource         ✗ no equivalent      not a consumer endpoint resource)
  AzureTableStorageResource         ✗ no equivalent
  AzureBlobStorageContainerResource ←→ BlobContainer  (1:1 leaf — ToProvisioningEntity() exists)
  AzureQueueStorageQueueResource    ←→ StorageQueue    (1:1 leaf — ToProvisioningEntity() exists)
```

The three sub-resources for blob, queue, and table access simply do not exist in the CDK world.

**Reason 3: Dual-mode endpoint resolution lives only in the orchestration world**

`AzureStorageResource.BlobUriExpression` switches at runtime:

```csharp
public ReferenceExpression BlobUriExpression => IsEmulator
    ? ReferenceExpression.Create($"{EmulatorBlobEndpoint.Property(EndpointProperty.Url)}")
    : ReferenceExpression.Create($"{BlobEndpoint}");
```

`EmulatorBlobEndpoint` is an `EndpointReference(this, "blob")` that resolves to the Azurite container's port 10000. The synthesis world only knows about the Azure path (`BlobEndpoint` → `ProvisioningOutput("blobEndpoint")`). There is nothing in any CDK type or emitted Bicep that encodes "port 10000 is the local blob endpoint". `aspire run` uses this path for every service that has `WithReference(blobStorage)` — it is entirely invisible to synthesis.

**Reason 4: Annotations are consumed and discarded by the lambda**

`PrivateEndpointTargetAnnotation` is read by the lambda and changes `NetworkRuleSet.DefaultAction = Deny` in the Bicep output. After that read, the annotation is gone — it never appears in any CDK type or in the emitted Bicep text. You cannot reconstruct the presence or absence of this annotation (or any other annotation: health check config, resource relationship annotations, emulator annotations) from the synthesis output.

**Reason 5: Connection string format knowledge is orchestration-world-only**

The `AzureBlobStorageResource` implements `IResourceWithConnectionString` with connection string formats that differ across three modes:
- Emulator: full connection string with blob/queue/table ports
- Azure standard client: `Endpoint={blobEndpoint}`
- Azure Functions: separate `blobServiceUri` + `queueServiceUri` environment variable keys

None of this formatting logic appears in the CDK or the emitted Bicep. The synthesis world outputs a raw URL string — the orchestration world knows where and how different SDK clients expect to consume it.

**Summary: what the synthesis world permanently loses**

| Orchestration-world concept | Synthesis-world equivalent |
|---|---|
| `AzureBlobStorageResource` (consumer endpoint IResource) | Nothing — `BlobService` is an unrelated ARM node |
| `EndpointReference` to Azurite port 10000 | Not representable in CDK/Bicep |
| `PrivateEndpointTargetAnnotation` annotation | Consumed, not emitted |
| Connection string format per SDK / per mode | Not representable in CDK/Bicep |
| `WithReference()` injection wiring | Not representable in CDK/Bicep |
| Azure Functions env var naming (`__blobServiceUri`) | Not representable in CDK/Bicep |

---

## 8. Polyglot AppHost: Guest Runtime Interaction

> **Q: How does the AppHost interact with TypeScript/Python/etc. guest runtimes? How does the guest build the resource graph?**

### The Key Design Principle

All 40+ integrations stay in .NET. Guest language code **never reimplements them**. Instead:

- Guest code calls into the .NET server via JSON-RPC
- Gets back opaque handles to live .NET objects
- Passes those handles forward in future calls

The guest language's JSON-RPC calls are the **equivalent of `Program.cs`** in a .NET AppHost. They populate `DistributedApplicationModel` with resources. The provisioning library then consumes that model — these are sequential phases, not competing approaches.

---

### The 3-Process Architecture

At runtime there are always three separate processes:

```
┌──────────────────────┐
│     Aspire CLI       │  ← orchestrator: starts both, passes socket path
└───────┬──────────────┘
        │ spawns                               spawns │
        ▼                                             ▼
┌──────────────────────┐          ┌─────────────────────────────┐
│  AppHost Server      │          │  Guest Process              │
│  (.NET RemoteHost)   │◄────────►│  (e.g., `npx tsx apphost.ts`)│
│                      │ JSON-RPC │                             │
│  - ILanguageSupport  │          │  - Generated SDK (.modules/)│
│  - ICodeGenerator    │          │  - User's apphost.ts        │
│  - CapabilityDispatch│          │  - ATS Client (transport.ts)│
│  - App Model & DCP   │          └─────────────────────────────┘
│  - Provisioning SDK  │
└──────────────────────┘
        ▲
        JSON-RPC over Unix socket (REMOTE_APP_HOST_SOCKET_PATH env var)
```

**Critical point for debugging**: For a Guest AppHost (TypeScript/Python), the **Provisioning SDK (`Azure.Provisioning`) runs in the AppHost Server (.NET RemoteHost)**, NOT in the guest process. The guest calls `builder.addAzureKeyVault("kv")` which becomes a JSON-RPC call to the server. The server creates the real `AzureKeyVaultResource` in its in-memory `DistributedApplicationModel`, and `GetBicepTemplateFile()` / `ConfigureInfrastructure()` all run in the AppHost Server's .NET process.

---

### Startup Sequence Step by Step

```
aspire run (in TypeScript apphost directory)
      │
[1]   CLI detects language
      │  Calls server RPC: detect(directoryPath)
      │  TypeScriptLanguageSupport.Detect() checks: apphost.ts? package.json?
      │  Returns: language = "typescript/nodejs"
      │
[2]   CLI requests RuntimeSpec from server
      │  TypeScriptLanguageSupport.GetRuntimeSpec() returns:
      │    Execute:      { command: "npx", args: ["tsx", "{appHostFile}"] }
      │    WatchExecute: { command: "npx", args: ["nodemon", "--exec", "npx tsx {appHostFile}"] }
      │    InstallDeps:  { command: "npm", args: ["install"] }
      │
[3]   CLI installs guest dependencies
      │  Runs: npm install (in the apphost directory)
      │
[4]   CLI scaffolds & builds the AppHost Server .NET project
      │  Creates hidden project in $TMPDIR/.aspire/hosts/<hash>/
      │  Program.cs: just `await RemoteHostServer.RunAsync(args)`
      │  .csproj references every Aspire NuGet package from aspire.json
      │  dotnet build (PrepareAsync in DotNetBasedAppHostServerProject.cs)
      │
[5]   Code generation
      │  AtsContextFactory scans all loaded assemblies for [AspireExport]
      │  AtsTypeScriptCodeGenerator generates .modules/aspire.ts + aspire.d.ts
      │  Typed builder classes with invokeCapability() under the hood
      │
[6]   CLI starts AppHost Server (.NET)
      │  Sets REMOTE_APP_HOST_SOCKET_PATH=/tmp/aspire/<session>.sock
      │  .NET server begins listening on Unix socket
      │
[7]   CLI starts Guest Process
      │  Runs: npx tsx apphost.ts
      │  Injects REMOTE_APP_HOST_SOCKET_PATH into guest's environment
      │
[8]   Guest connects and calls capabilities
```

---

### JSON-RPC Protocol: Wire Format

Uses LSP-style framing (same as VS Code extensions) over a Unix socket:

```
Content-Length: 127\r\n
\r\n
{"jsonrpc":"2.0","id":1,"method":"invokeCapability","params":["Aspire.Hosting/createBuilder",{}]}
```

The TypeScript SDK's `transport.ts` uses the `vscode-jsonrpc` npm package — the same package VS Code itself uses.

---

### Handle System: How Objects Cross the Process Boundary

Every .NET object that guest code holds is an **opaque handle** — a string ID + type ID tuple. The guest never deserializes the object's internals; it only passes the handle back for future calls.

```
Guest                                            .NET AppHost Server
  │                                                     │
  │── invokeCapability("Aspire.Hosting/createBuilder", {}) ──►│
  │                                                     │  Creates IDistributedApplicationBuilder
  │◄── { "$handle": "1",                                │  Stores in handle registry [1 → builder]
  │      "$type": "Aspire.Hosting/.../IDistributedApplicationBuilder" }
  │
  │── invokeCapability("Aspire.Hosting.Redis/addRedis", {
  │       "builder": { "$handle": "1", "$type": "..." },
  │       "name": "cache"
  │   }) ──►│
  │         │  Looks up handle 1 → actual IDistributedApplicationBuilder object
  │         │  Calls RedisExtensions.AddRedis(builder, "cache")  ← real C# extension method
  │         │  Stores result in HandleRegistry [2 → IResourceBuilder<RedisResource>]
  │◄── { "$handle": "2", "$type": "Aspire.Hosting.Redis/.../RedisResource" }
```

The `HandleRegistry` is a per-client `ConcurrentDictionary<string, object>` that stores live .NET objects by ID.

**ATS type ID format:** `{AssemblyName}/{FullTypeName}`

```
Aspire.Hosting/Aspire.Hosting.IDistributedApplicationBuilder
Aspire.Hosting.Redis/Aspire.Hosting.ApplicationModel.RedisResource
```

---

### Inside the AppHost Server: Capability Dispatch

When the guest sends `invokeCapability("Aspire.Hosting.Redis/addRedis", args)`:

```
RemoteAppHostService.InvokeCapabilityAsync()
    └── CapabilityDispatcher.InvokeAsync("Aspire.Hosting.Redis/addRedis", args)
            │
            │  At startup, RegisterFromCapability() bound this capability ID to
            │  the real MethodInfo of AddRedis() (via AtsCapabilityScanner reflection scan)
            │
            ├─ Unmarshal args:
            │     "builder" → HandleRegistry.TryGet("1") → returns actual IDistributedApplicationBuilder
            │     "name"    → JsonNode → string "cache"
            │
            ├─ Invoke via reflection:
            │     method.Invoke(null, [builder, "cache"])
            │     == RedisExtensions.AddRedis(builder, "cache")  ← real C# extension method
            │
            └─ Marshal result:
                  IResourceBuilder<RedisResource> → HandleRegistry.Register(obj) → handle "2"
                  Return: {"$handle": "2", "$type": "..."}
```

The `RemoteHostServer` (in `src/Aspire.Hosting.RemoteHost/`) hosts the `JsonRpcServer` as a background service, which listens on a Unix socket (or named pipe on Windows) and scopes a `RemoteAppHostService` + `CapabilityDispatcher` per client connection.

The `CapabilityDispatcher` scans assemblies listed in `AtsAssemblies` (from `appsettings.json`) for `[AspireExport]` attributes and registers typed `CapabilityHandler` delegates that unmarshal JSON args, resolve handles, and invoke the real .NET methods via reflection.

---

### What the Generated TypeScript SDK Looks Like

The code generator produces typed TypeScript classes that call `invokeCapability()` under the hood:

```typescript
// .modules/aspire.ts  (generated — do not edit)
export class DistributedApplicationBuilder {
    constructor(
        private readonly _handle: IDistributedApplicationBuilderHandle,
        private readonly _client: AspireClient) {}

    async addRedis(name: string, port?: number): Promise<RedisResourceBuilder> {
        const result = await this._client.invokeCapability(
            'Aspire.Hosting.Redis/addRedis',
            { builder: this._handle, name, port }
        );
        return new RedisResourceBuilder(result as RedisResourceBuilderHandle, this._client);
    }
}

export async function createBuilder(): Promise<DistributedApplicationBuilder> {
    const socketPath = process.env['REMOTE_APP_HOST_SOCKET_PATH']!;
    const client = new AspireClient(socketPath);
    await client.connect();
    const handle = await client.invokeCapability('Aspire.Hosting/createBuilder', {});
    return new DistributedApplicationBuilder(handle, client);
}
```

The user's `apphost.ts`:

```typescript
import { createBuilder } from './.modules/aspire.js';

const builder = await createBuilder();
const cache = await builder.addRedis("cache");
const api = await builder.addProject("api", "../Api/Api.csproj");
await api.withReference(cache);
await builder.build().run();
```

---

### Callbacks: Guest Code Invoked by .NET

When .NET needs to call back into the guest (e.g., `withEnvironmentCallback`):

```
Guest registers a closure, sending a callback ID string to the server.
Server stores: callbackId → C# delegate.
When the environment is evaluated, the server calls back:

Server → invokeCallback("callback_1_1234567890", { "p0": { "$handle": "5", ... } })
Guest → wraps handle in typed class → runs user's async closure
Guest → ctx.environmentVariables.set("MY_KEY", "hello")
      → invokeCapability("set", { dict: handle, key, value }) to the server
```

---

### Adding a New Guest Language (e.g., Python)

Two interfaces to implement, both discovered automatically from the NuGet package:

**Step 1: `ILanguageSupport`** — tells the CLI how to detect, scaffold, and run the guest:

```csharp
public sealed class PythonLanguageSupport : ILanguageSupport
{
    public string Language => "python/cpython";

    public DetectionResult Detect(string directoryPath)
    {
        var hasAppHost = File.Exists(Path.Combine(directoryPath, "apphost.py"));
        var hasRequirements = File.Exists(Path.Combine(directoryPath, "requirements.txt"));
        return hasAppHost && hasRequirements
            ? DetectionResult.Found("python/cpython", "apphost.py")
            : DetectionResult.NotFound;
    }

    public RuntimeSpec GetRuntimeSpec() => new()
    {
        Execute = new CommandSpec { Command = "python", Args = ["{appHostFile}"] },
        WatchExecute = new CommandSpec { Command = "watchdog", Args = ["watchmedo", "auto-restart", "--", "python", "{appHostFile}"] },
        InstallDependencies = new CommandSpec { Command = "pip", Args = ["install", "-r", "requirements.txt"] }
    };

    public Dictionary<string, string> Scaffold(ScaffoldRequest request) =>
        new() { ["apphost.py"] = "from aspire import create_builder\n..." };
}
```

**Step 2: `ICodeGenerator`** — produces the guest-language SDK from `AtsContext`:

```csharp
public sealed class PythonCodeGenerator : ICodeGenerator
{
    public string Language => "Python";

    public Dictionary<string, string> Generate(AtsContext context)
    {
        // Template each capability in context.Capabilities into Python classes
        return new() { [".modules/aspire.py"] = generatedCode };
    }
}
```

**Step 3: Register via DI** (one line, in the NuGet package's hosting extension):

```csharp
services.AddSingleton<ILanguageSupport, PythonLanguageSupport>();
services.AddSingleton<ICodeGenerator, PythonCodeGenerator>();
```

No CLI changes needed — adding the NuGet package to `aspire.json` is the only integration step.

---

### Guest Runtime Interaction Summary Table

| Concern | Owned By | Mechanism |
|---------|----------|-----------|
| Socket path distribution | CLI | `REMOTE_APP_HOST_SOCKET_PATH` env var |
| Language detection | `ILanguageSupport.Detect()` on server | CLI calls `detect` RPC |
| Run command | `ILanguageSupport.GetRuntimeSpec()` on server | CLI calls `getRuntimeSpec` RPC |
| Scaffolding new project | `ILanguageSupport.Scaffold()` on server | CLI calls `scaffoldAppHost` RPC |
| SDK code generation | `ICodeGenerator` on server | CLI calls `generateCode` RPC |
| Guest → Host calls | ATS JSON-RPC `invokeCapability` | Guest connects to Unix socket |
| Host → Guest callbacks | ATS JSON-RPC `invokeCallback` | Reverse direction on same connection |
| Handle lifecycle | AppHost server handle registry | Valid until server exits |
| All Aspire integrations | .NET server process | Guest sees them as typed SDK methods |

---

### How TypeScript Provisioning Libraries Interact with AppHost Server APIs

> **Q: If I define infrastructure with TypeScript provisioning libraries, how do TypeScript CDK models get recognized by server APIs like `addStorageAccount` and actually add a storage account?**

TypeScript provisioning libraries are **not standalone CDKs that define resources directly**. They are thin proxy wrappers — every API call crosses the JSON-RPC boundary to the .NET AppHost Server, which holds the real resource model and the only copy of the `Azure.Provisioning` CDK.

**What makes a method available in TypeScript:**

The binding mechanism is `[AspireExport]` on the C# side:

```csharp
// src/Aspire.Hosting.Azure.Storage/AzureStorageExtensions.cs
[AspireExport("addAzureStorage", Description = "Adds an Azure Storage resource")]
public static IResourceBuilder<AzureStorageResource> AddAzureStorage(
    this IDistributedApplicationBuilder builder, [ResourceName] string name)
{ ... }
```

At startup, `AtsCapabilityScanner` reflects over all loaded assemblies, finds every `[AspireExport]`-annotated method, and registers it in `CapabilityDispatcher` as `"Aspire.Hosting.Azure.Storage/addAzureStorage"`. The TypeScript SDK is generated from this same reflection pass — the TypeScript method name is always in sync with the .NET capability ID.

**Call trace for `builder.addAzureStorage("my-storage")`:**

```
TypeScript apphost.ts                      .NET AppHost Server (RemoteHost)
─────────────────────                      ────────────────────────────────
const storage = await
  builder.addAzureStorage("my-storage")
        │
        │  invokeCapability(
        │    "Aspire.Hosting.Azure.Storage/addAzureStorage",
        │    { builder: { $handle: "1" }, name: "my-storage" }
        │  )  ──── JSON-RPC over Unix socket ────────────────►
        │                                                     │
        │                               CapabilityDispatcher.InvokeAsync()
        │                                                     │
        │                               1. Looks up handle "1"
        │                                  → actual IDistributedApplicationBuilder
        │                               2. Invokes via reflection:
        │                                  AzureStorageExtensions.AddAzureStorage(
        │                                    builder, "my-storage")
        │                               3. Creates AzureStorageResource in the
        │                                  in-memory DistributedApplicationModel
        │                               4. Attaches ConfigureInfrastructure callback
        │                                  (a C# lambda → generates Bicep later)
        │                               5. Registers result handle "2"
        │
        │◄─── { "$handle": "2", "$type": "...AzureStorageResource" }
        │
    TypeScript holds opaque handle "2".
    Azure.Provisioning CDK, Bicep generation, and
    ARM deployment run 100% in .NET. TypeScript never
    sees or instantiates any CDK objects.
```

**TypeScript is always a proxy — it never holds deserialized resource objects:**

- Every `await builder.addSomethingAzure(...)` → JSON-RPC → real C# extension method runs in .NET server → `AzureXResource` enters .NET `DistributedApplicationModel`
- TypeScript holds only an opaque `$handle` string
- `ConfigureInfrastructure` callbacks, Bicep generation, and ARM deployment all execute exclusively in the .NET process
- The generated `.modules/aspire.ts` is the complete TypeScript surface; every method in it corresponds 1:1 to a `[AspireExport]`-annotated C# method

**To add a new Azure resource type accessible from TypeScript**, you must ship both sides:

| Layer | What to write |
|-------|--------------|
| **C# (.NET)** | `MyResource : AzureProvisioningResource`, extension method with `[AspireExport("addMyResource")]`, `ConfigureInfrastructure` callback using `Azure.Provisioning.*` CDK types |
| **TypeScript (generated)** | Run `aspire sdk generate` — reflects the C# assembly and generates the TypeScript wrapper that calls `invokeCapability("YourAssembly/addMyResource", ...)` |

The TypeScript library is then purely a proxy — it holds handles and routes calls. All provisioning logic stays in .NET.

---

## 9. SDK Code Generator

> **Q: How does the Aspire code generator work and why does it trigger when first running with the CLI?**

### The Core Problem: No Runtime Discovery

Guest languages (TypeScript, Python, etc.) can't use .NET reflection at runtime. Every capability (`addRedis`, `withEnvironment`, etc.) that comes from a NuGet package must be represented as **statically-typed code in the guest language** before user code can call it. That's what SDK generation produces: the `.modules/aspire.ts` (and `base.ts`, `transport.ts`) files.

---

### Why It Must Run on First Use

The SDK cannot be committed to the repo because it is derived from whatever NuGet packages are installed. If you run `aspire add redis`, the Redis package is added, and on the next `aspire run` the hash will change and regeneration produces `addRedis()` in the TypeScript SDK.

**The generation pipeline:**

```
[1] CLI scaffolds hidden .NET project in $TMPDIR/.aspire/hosts/<hash>/
    Program.cs: await RemoteHostServer.RunAsync(args)
    .csproj references every Aspire NuGet package from aspire.json

[2] CLI builds it (PrepareAsync in DotNetBasedAppHostServerProject.cs)
    → returns NeedsCodeGeneration: true

[3] CLI starts the server
    → server binary loads all Aspire assemblies into the process

[4] CLI calls generateCode RPC
    → AtsContextFactory lazily scans all loaded assemblies for [AspireExport]
    → AtsTypeScriptCodeGenerator produces aspire.ts, base.ts, transport.ts
    → CLI writes files to .modules/ in the apphost directory
    → CLI saves .codegen-hash (SHA-256 of sorted packageId:version pairs)

[5] CLI starts guest process (npx tsx apphost.ts)
    → TypeScript code: import { createBuilder } from './.modules/aspire.js'
```

---

### The Two-Phase Split: Where Code Lives

**Server side (AppHost Server process):**

```
AssemblyLoader → AtsContextFactory → AtsCapabilityScanner → AtsContext
                                                              ↓
                                               CodeGenerationService.generateCode()
                                                              ↓
                                                    ICodeGenerator.GenerateDistributedApplication()
                                                              ↓
                                               Dictionary<string, string>  (filename → content)
```

**Client side (CLI):**

- `GuestAppHostProject.cs` orchestrates: build server → start it → call `GenerateCodeAsync` → write files → save `.codegen-hash`
- `AppHostRpcClient.cs` sends `{"method": "generateCode", "params": ["TypeScript"]}` over the Unix socket

---

### The `aspire sdk generate` Command

A standalone variant for integration library authors:

```bash
aspire sdk generate ./MyIntegration.csproj -l typescript -o ./output
```

Same pipeline in isolation: temp AppHost server project → add `.csproj` as `additionalProjectReference` → build → connect → call `GenerateCodeAsync` → write files. Lets integration authors generate typed SDKs for publishing to npm/PyPI.

---

### Hash-Based Cache: When Regeneration Is Skipped

The `.modules/.codegen-hash` file stores the SHA-256 of `packageId:version;packageId:version;...` (sorted). When the hash matches, code generation is skipped. The hash is recomputed every run from the packages in `aspire.json`.

---

### The 5-Pass Scanner

`AtsCapabilityScanner.ScanAssemblies()` runs these passes across all assemblies:

1. **Collect** — per-assembly scan for `[AspireExport]` on methods and types, `[AspireDto]` on DTOs, gathers enum types from signatures
2. **Resolve Unknown types** — any capability parameter that references a type not in the exported set is marked `Unknown`
3. **Filter invalid** — capabilities with unresolved `Unknown` types are dropped (with diagnostics)
4. **Expand interface targets** — `withEnvironment<T> where T : IResourceWithEnvironment` becomes a capability entry for every concrete type that implements `IResourceWithEnvironment`; `ExpandedTargetTypes` is pre-computed here
5. **Filter overloads** — C# allows overloaded methods; ATS capability IDs must be unique, so duplicates emit errors and are removed

---

### Assembly Loading in the AppHost Server

The assembly list is driven by two things:

**1. `AtsAssemblies` in `appsettings.json`** (generated by the CLI):

```json
{
  "AtsAssemblies": [
    "Aspire.Hosting",
    "Aspire.Hosting.Redis",
    "Aspire.Hosting.CodeGeneration.TypeScript"
  ]
}
```

`AssemblyLoader` reads this array at startup and calls `Assembly.Load(name)` for each.

**2. `ASPIRE_INTEGRATION_LIBS_PATH` env var** (prebuilt / bundled mode):

For the prebuilt CLI bundle, `PrebuiltAppHostServer` restores NuGet packages to a temp folder and passes that path via `ASPIRE_INTEGRATION_LIBS_PATH`. `AssemblyLoader` registers a custom `AssemblyLoadContext.Default.Resolving` handler that probes that directory for DLLs.

**How to inspect what's loaded:**

- Check `$TMPDIR/.aspire/hosts/<hash>/appsettings.json` — that's the exact list
- Use `--debug` to enable `Debug` log level and see "Loaded assembly: {AssemblyName}" log lines
- Use `aspire sdk dump` (calls `getCapabilities` RPC) — capability IDs like `Aspire.Hosting.Redis/addRedis` confirm which assemblies were scanned

---

## 10. Aspire CLI Architecture

> **Q: Why does the CLI need to communicate with the AppHost server via RPC?**

### Why RPC Instead of In-Process

The CLI is a **native AOT-compiled binary** that cannot dynamically load arbitrary .NET assemblies. ALL language-specific knowledge lives inside NuGet packages that are only present in the server process:

| RPC Method | Why the CLI Can't Do It Locally |
|------------|--------------------------------|
| `getRuntimeSpec` | Returned by `ILanguageSupport` implementations from NuGet packages (`Aspire.Hosting.CodeGeneration.TypeScript`) — the CLI doesn't reference those packages |
| `scaffoldAppHost` | Scaffold templates are embedded resources inside those same NuGet packages, accessed via `ILanguageSupport.Scaffold()` |
| `generateCode` | Code generation requires `AtsCapabilityScanner` to reflect over loaded integration assemblies — only the server can do this |
| `getCapabilities` | Same — needs reflection over NuGet package assemblies to enumerate `[AspireExport]` attributes |

There is also a **process separation reason**: the AppHost Server manages DCP, containers, and the Dashboard. The CLI is the user-facing shell. Separate processes mean the CLI can monitor the server, restart it, and display its exit code independently.

```
CLI (native AOT, thin orchestrator)
  │── JSON-RPC over Unix socket ──→ AppHost Server (.NET, loads all NuGet packages)
                                         │── JSON-RPC over Unix socket ──→ Guest (TypeScript/Python)
```

### CLI Commands

| Command | Purpose |
|---------|---------|
| `aspire run` | Launch App Host + DCP + Dashboard |
| `aspire publish` | Generate deployment artifacts (Bicep, K8s YAML) |
| `aspire add` | Add NuGet integration packages to a project |
| `aspire new` | Scaffold from templates |
| `aspire deploy` | Deploy to Azure (via `azd`) |
| `aspire exec` | Run a command inside the Aspire environment |
| `aspire ps / logs / restart` | Resource lifecycle management |
| `aspire telemetry` | View OTLP traces/spans/logs from CLI |
| `aspire mcp` | Start MCP server for AI tool integration |
| `aspire sdk generate` | Generate guest-language SDK from an integration library |
| `aspire sdk dump` | Dump all registered capabilities |
| `aspire docs` | Search documentation |

---

## 11. VS Code Extension

TypeScript extension providing developer experience inside VS Code:

```
extension/
  src/
    extension.ts          ← entry point, registers providers
    capabilities.ts       ← feature flags
    server/               ← connects to Dashboard HTTP API
    dcp/                  ← DCP resource state watcher
    views/                ← Tree views (Resources, Logs, etc.)
    commands/             ← VS Code command palette entries
    debugger/             ← Debug session integration
    editor/               ← Code lens, completion providers
  loc/
    strings.ts            ← All localized UI strings (paired with package.nls.json)
  schemas/
    aspire-settings.schema.json ← Settings schema
```

The extension communicates with the Dashboard's REST API (`Api/`) to show resource state, logs, and traces directly in the editor.

---

## 12. Local Development Setup

### Prerequisites

The local `.NET SDK` is managed by the repo — do not use your system `dotnet` for builds.

```bash
# Install local SDK and set up the repo (run first, ~30 seconds)
./restore.sh
```

### Daily Workflow

**Launch VS Code correctly** (sets correct PATH so terminals use the local SDK):

```bash
./start-code.sh
```

Never open VS Code via `code .` — that won't have `.dotnet` on PATH.

**Build after code changes:**

```bash
# Full build (restore + build, ~3-5 minutes)
./build.sh

# Build only (skip restore, faster after first setup)
./build.sh --build

# Skip native AOT compilation (~1-2 minutes saved)
./build.sh --build /p:SkipNativeBuild=true

# Clean rebuild
./build.sh --rebuild
```

**Run the sample app to verify setup:**

```bash
./dotnet.sh run --project playground/TestShop/TestShop.AppHost/TestShop.AppHost.csproj
```

**Run tests for a specific project:**

```bash
dotnet test tests/Aspire.Hosting.Tests/Aspire.Hosting.Tests.csproj -- \
  --filter-not-trait "quarantined=true" --filter-not-trait "outerloop=true"
```

**Run a specific test by name:**

```bash
dotnet test tests/Aspire.Hosting.Tests/Aspire.Hosting.Tests.csproj -- \
  --filter-method "*.MyTestMethodName" \
  --filter-not-trait "quarantined=true" --filter-not-trait "outerloop=true"
```

> **Important:** Tests use Microsoft.Testing.Platform (MTP), not VSTest. All filters must go **after** `--`. Never use `--filter` before `--`.

### VS Code Extension Development

```bash
cd extension
npm install
npm run build
```

Then use `start-code.sh` from the repo root and press `F5` to launch the Extension Development Host.

### If You Rebuild from Scratch

```bash
# Re-install local SDK (if .dotnet/ gets corrupted)
./restore.sh

# Full clean build
./build.sh --rebuild
```

---

## 13. Debugging Guide

> **Q: How to debug the Guest AppHost server and Bicep generation?**

### Understanding the 3-Process Architecture for Debugging

For a Guest AppHost (TypeScript/Python), there are **three distinct debuggable processes**:

1. **CLI** (`aspire` binary) — thin orchestrator
2. **AppHost Server** (`.NET RemoteHost` process) — Provisioning SDK and Bicep generation run here
3. **Guest process** (`node apphost.ts`, `python apphost.py`, etc.) — user's configuration code runs here

These are **two separate debuggable processes** on the .NET side (CLI and AppHost Server). The guest is a third.

---

### Debugging the AppHost Server (.NET Process)

**Enable verbose output:**

```bash
aspire run --debug --project apphost.ts
```

`--debug` sets `Logging__LogLevel__Default=Debug` on the AppHost Server process, forwarding all stdout/stderr from both processes to the CLI log.

**Attach a .NET debugger:**

```bash
aspire run --wait-for-debugger --project apphost.ts
```

This sets `ASPIRE_WAIT_FOR_DEBUGGER=true`, which causes `DistributedApplication.WaitForDebugger()` to spin in a loop printing the PID. Attach to that PID in Visual Studio or VS Code with "Attach to Process". There is a 30-second timeout (configurable via `ASPIRE_WAIT_FOR_DEBUGGER_TIMEOUT`).

This is the process to attach to when debugging **Provisioning SDK execution** and **Bicep generation** — since those run inside the AppHost Server, not the guest.

**Log files:**

All AppHost Server stdout/stderr is written to `<log-dir>/apphost.log` (the log dir is printed on startup with `--debug`).

---

### Debugging the Guest Process (TypeScript/Python)

The guest process is started by `GuestRuntime.ExecuteCommandAsync()` as a plain child process with no special wait loop. To debug it:

1. Add a language-level debugger break at the start of your AppHost code (e.g., `debugger;` in TypeScript)
2. Use `--debug` to see the PID in the CLI output
3. Attach the language-specific debugger to that PID

---

### Debugging Bicep Generation

Bicep generation happens inside the **AppHost Server** process during `PublishAsync → WriteModelAsync`. Call stack:

```
GetBicepTemplateFile()
  → ConfigureInfrastructure(infrastructure)   ← your lambda closure runs here
  → infrastructure.Build(ProvisioningBuildOptions)  ← Azure.Provisioning SDK
  → plan.Compile()                             ← emits Bicep text
```

**Option 1: Inspect generated files**

```bash
aspire publish --debug --output-path ./infra
```

The generated `.bicep` files are written to `--output-path` — inspect the final result.

**Option 2: Attach .NET debugger to the publish run**

```bash
aspire publish --wait-for-debugger --project AppHost.csproj
```

Set a breakpoint in `AzureProvisioningResource.cs` → `GetBicepTemplateFile()`. The `plan` variable (the `ProvisioningPlan` from `Azure.Provisioning`) is inspectable under the debugger — it exposes all `ProvisionableResource` objects, their properties, and the module structure before compilation.

**Option 3: Unit test Bicep generation directly (fastest)**

You don't need to start any server:

```csharp
var builder = DistributedApplication.CreateBuilder(
    new DistributedApplicationOptions { Args = ["--operation", "publish"] });

var myResource = builder.AddAzureKeyVault("kv");
var app = builder.Build();

var resource = (AzureProvisioningResource)myResource.Resource;
var bicep = resource.GetBicepTemplateString();
Console.WriteLine(bicep);
```

This is exactly what the test projects in `tests/Aspire.Hosting.Azure.*` do — they call `GetBicepTemplateString()` directly and compare output with snapshots via Verify.

---

### Summary: What to Debug Where

| What You're Investigating | Which Process to Attach To |
|---------------------------|---------------------------|
| CLI logic (command parsing, orchestration) | CLI process |
| Language detection, RuntimeSpec, scaffolding | AppHost Server (.NET) |
| Capability dispatch, handle registry | AppHost Server (.NET) |
| Provisioning SDK, Bicep text generation | AppHost Server (.NET) |
| ARM deployment, Azure SDK calls | AppHost Server (.NET) |
| User's TypeScript/Python config code | Guest process (node/python) |
| Generated SDK behavior (.modules/) | Guest process, inspect source |
