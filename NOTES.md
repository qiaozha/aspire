
What Aspire Is Working On
Architecture
Bicep Relationship
Path 1: Hand-authored Bicep
Path 2: Programmatic generation (C# → Bicep)
Code Generation
1. ATS (Aspire Type System) – Polyglot AppHost SDK generation
2. Manifest code generation (for deployment tooling)
Summary Table
dotnet/aspire — Code Architecture Overview
1. Repository Layout
2. Major Subsystems
3. Core Layer: Application Model (ApplicationModel)
4. Orchestration Layer: DCP Executor (Dcp)
5. Dashboard (Aspire.Dashboard)
6. Publishing Pipeline (Pipelines + Publishing/)
7. Azure Integration & Bicep (Aspire.Hosting.Azure)
8. Client Integration Components (Components)
9. Aspire CLI (Aspire.Cli)
10. Polyglot AppHost — ATS System (Ats)
11. VS Code Extension (extension)
12. Full Dependency / Data Flow Diagram
13. Key Architectural Principles
4. Orchestration Layer: DCP Executor (Dcp) — Expanded
What is DCP?
What resource graph information does DcpExecutor consume?
The AppResource bridge type
How KubernetesService fits in
State change propagation after creation
7. Azure Integration & Bicep — Expanded
The C# Integration API
What is the Azure.Provisioning SDK?
Full Azure resource lifecycle mapping
AppHost ↔ Guest Runtime Interaction
The Big Picture
1. Process Architecture
2. Startup Sequence Step by Step
3. JSON-RPC Protocol: Wire Format
4. Handle System: How Objects Cross the Boundary
5. What the Generated TypeScript SDK Looks Like
6. Callbacks: Guest Code Invoked by .NET
7. How to Add a New Guest Language (e.g., Python)
Step 1: Implement ILanguageSupport
Step 2: Implement ICodeGenerator
Step 3: Register via DI (one line in the NuGet package's hosting extension)
8. Summary Table
Local Development Setup Status
Daily Workflow
Launch VS Code correctly
Build after code changes
Run the sample app to verify setup
Run tests for a specific project
If You Rebuild from Scratch
Extension Development (VS Code Extension)
How the Aspire Code Generator Works
The Core Problem: No Runtime Discovery
The Two-Phase Split: Where Code Lives
Why It Must Run on First Use
The aspire sdk generate Command
Hash-Based Cache: When Regeneration Is Skipped
The 5-Pass Scanner
The Three-Layer Stack
Layer 1: Azure.Provisioning SDK → Bicep text (C# code in your AppHost)
Layer 2: Bicep text → ARM JSON (the bicep CLI)
Layer 3: ARM template → Azure deployment (Azure Resource Manager)
Two Different Deployment Paths
Path A: aspire run (local dev provisioning)
Path B: azd up (Azure Developer CLI deployment)
Summary
The Provisioning Library Is Part of the AppHost Process
How It Flows: The Pipeline
Communication Channels (all in-process):
The Sequence
The Analogy
Why Not Generate Bicep From the Guest Directly?
Phase 1: The Guest Builds the Resource Graph via JSON-RPC
The invocation chain
Phase 2: What the Provisioning Library Reads From the Model
The Complete Picture
DistributedApplicationModel — The Application Graph
IResource — The Fundamental Unit
Annotations — The Extensibility Mechanism
Standard Capability Interfaces
The Value/Reference System — How the DAG Forms
The Full Architecture Overview
Key Insight: Inert Data + External Orchestration
Why the AppHost must run
The publish flow end-to-end
What the CLI does
Why not just have the SDK generate Bicep directly?
Debugging the Guest AppHost Server
1. Enable verbose CLI output
2. Attach a debugger to the AppHost Server (.NET process)
3. Attach a debugger to the Guest process
4. Log file
Debugging Bicep Generation
1. Run publish with --debug and inspect the temp files
2. Set a breakpoint directly in GetBicepTemplateFile
3. Unit test Bicep generation directly
4. Inspect intermediate ProvisioningPlan

What Aspire Is Working On
At its core, dotnet/aspire is a distributed application orchestration and deployment platform for .NET. It provides:

App Host orchestration – A code-first model where you declare services, containers, databases, and cloud resources in C#. During local dev, Aspire starts everything (Docker containers, processes) and wires them together with service discovery and environment variables.
A monitoring Dashboard – Blazor-based UI for traces, logs, and metrics via OpenTelemetry.
40+ platform integrations – Redis, PostgreSQL, SQL Server, MongoDB, RabbitMQ, Kafka, all Azure services, Kubernetes, etc.
A polyglot CLI – Evolving from a dotnet run wrapper to a full orchestration CLI supporting non-.NET languages.
Deployment publishing – Generating infrastructure-as-code artifacts (Bicep, Kubernetes YAML) ready for production.


Architecture
The architecture has several clear layers:
┌────────────────────────────────────────────────────────┐
│            App Host (C# Program.cs)                    │
│   builder.AddRedis("cache")                            │
│   builder.AddProject("api").WithReference(cache)       │
└────────────────────────────┬───────────────────────────┘
                             │ produces
                    ┌────────▼─────────┐
                    │  App Model (DAG) │  ← IResource graph with Annotations
                    └────────┬─────────┘
           ┌─────────────────┼──────────────────────┐
           ▼                 ▼                      ▼
   ┌──────────────┐  ┌──────────────┐   ┌──────────────────────┐
   │ DCP Executor │  │  Dashboard   │   │ Publishing Pipeline  │
   │ (local run)  │  │ (monitoring) │   │ (Bicep / K8s YAML)   │
   └──────────────┘  └──────────────┘   └──────────────────────┘


Key concepts:

IResource – The base unit. Everything (Redis, Postgres, a .NET project, an Azure Key Vault) is a resource.
Annotations – Metadata attached to resources (endpoints, environment vars, health checks, publish callbacks). The system is annotation-driven and extensible.
IResourceBuilder<T> – A fluent builder for configuring resources. Extension methods like AddRedis(), WithReference(), WithEnvironment() are the public API surface.
Pipeline steps – A dependency-ordered pipeline (provisioning, publishing) that runs at different lifecycle stages.


Bicep Relationship
Bicep is the primary way Aspire models Azure cloud resources. The relationship has two paths:
Path 1: Hand-authored Bicep
builder.AddBicepTemplate("myStorage", "storage.bicep")
       .WithParameter("location", "eastus");



AzureBicepResource wraps a .bicep file and its parameters.
At publish time, the file is copied to the output with references resolved.
At run time, BicepProvisioner compiles Bicep → ARM via the az bicep build CLI, then deploys via Azure ARM SDK.
Path 2: Programmatic generation (C# → Bicep)
builder.AddAzureRedis("cache");      // No bicep file needed
builder.AddAzureKeyVault("kv");      // Bicep is generated at runtime



Each Azure integration (e.g., Aspire.Hosting.Azure.Redis) uses the Azure.Provisioning SDK to declare infrastructure in C#.
AzureProvisioningResource calls GetBicepTemplateFile() → calls infrastructure.Build().Compile() → emits .bicep text.
During aspire publish, AzurePublishingContext collects all AzureBicepResource instances and writes .bicep files + a main main.bicep orchestrator to disk.
The full flow:
C# resource definition
    → Azure.Provisioning SDK (C# infra model)
        → AzureProvisioningResource.GetBicepTemplateString()
            → .bicep file (written to temp dir or output dir)
                → BicepCliCompiler → ARM JSON
                    → Azure ARM deployment




Code Generation
This is the most novel active work in the repo. There are two distinct code-gen systems:
1. ATS (Aspire Type System) – Polyglot AppHost SDK generation
The goal: let developers write Aspire app hosts in TypeScript, Python, Go, Rust, Java. Since all 40+ integrations are C#, they communicate via JSON-RPC with a .NET server instead of re-implementing everything.
How it works:
C# extension methods + [AspireExport] attribute
    → AtsCapabilityScanner (reflection-based, scans assemblies)
        → AtsContext (flat capability model: handles, DTOs, enums)
            → ICodeGenerator implementations
                → TypeScript SDK / Python SDK / Go SDK / etc.



[AspireExport("addRedis")] on a C# method tells the scanner: expose this as a capability Aspire.Hosting.Redis/addRedis.
[AspireDto] marks types to be serialized as JSON.
[AspireExport] on a type creates an opaque "handle" type — guest code holds a reference, all method calls go back to .NET via RPC.
The AtsTypeScriptCodeGenerator produces typed TypeScript classes with fluent methods that call invokeCapability() under the hood.
Similar generators exist in Aspire.Hosting.CodeGeneration.Go/, .Java/, .Python/, .Rust/, .TypeScript/.
Why no IDL? Unlike gRPC/protobuf, ATS has no separate schema file — the C# code is the schema. The scanner infers everything from reflected assemblies.
2. Manifest code generation (for deployment tooling)
When you run aspire publish, each resource writes itself to a aspire-manifest.json via ManifestPublishingCallbackAnnotation. Deployment tools (like the Azure Developer CLI azd) read this manifest and drive deployment. The Azure resources emit Bicep files as the deployment artifact.


Summary Table

Area	Technology	Purpose
Local orchestration	DCP (internal), Docker	Starts containers/processes during dev
App model	C# resource graph, annotations	Declares distributed app topology
Azure cloud resources	Bicep + Azure.Provisioning SDK	Infrastructure-as-code for Azure
Bicep provisioning	Azure ARM API	Deploys Azure resources during aspire run
Publish artifacts	Bicep files + manifest JSON	Output for azd, K8s, etc.
Polyglot app hosts	ATS + JSON-RPC	Lets TypeScript/Python/etc. write app hosts
Language SDKs	Code generation (ATS → TS/Py/Go/Rust/Java)	Typed wrappers for each guest language

User: Can you generate a comprehensive overview of the code architect of this repo
GitHub Copilot: Read
Read
Read
Read
Read
Read
Read
Read
Here is a comprehensive architectural overview of the Aspire repository:


dotnet/aspire — Code Architecture Overview


1. Repository Layout
aspire/
├── src/                    # All production source code
├── tests/                  # Unit & integration tests (mirrors src/)
├── playground/             # Sample apps for manual verification
├── extension/              # VS Code extension (TypeScript)
├── docs/specs/             # Architecture specs & design documents
├── eng/                    # Build infrastructure (Arcade SDK, pipelines)
└── tools/                  # Developer tooling (QuarantineTools, etc.)




2. Major Subsystems
┌──────────────────────────────────────────────────────────────────────────┐
│                          User-Facing Entry Points                        │
│                                                                          │
│  aspire CLI (Aspire.Cli)   VS Code Extension  App Host (C# Program.cs)  │
└─────────────┬──────────────────────┬─────────────────────┬──────────────┘
              │                      │                      │
              ▼                      ▼                      ▼
┌─────────────────────┐  ┌──────────────────┐  ┌──────────────────────────┐
│  CLI Orchestration  │  │  VS Code Ext.    │  │   App Host SDK           │
│  (run/publish/add)  │  │  (debug, attach) │  │   (Aspire.Hosting)       │
└──────────┬──────────┘  └──────────────────┘  └──────────┬───────────────┘
           │                                               │
           │                               ┌──────────────▼───────────────┐
           │                               │       Application Model      │
           │                               │  IResource, Annotations, DAG │
           │                               └──────────────┬───────────────┘
           │                                              │
           │              ┌───────────────────────────────┼────────────────┐
           │              │               │               │                │
           ▼              ▼               ▼               ▼                ▼
    ┌──────────┐  ┌──────────────┐ ┌──────────┐ ┌──────────────┐ ┌──────────────┐
    │  AzDev   │  │DCP Executor  │ │Dashboard │ │  Publishing  │ │   Azure      │
    │  CLI     │  │(local run)   │ │(Blazor)  │ │  Pipeline    │ │ Provisioning │
    └──────────┘  └──────────────┘ └──────────┘ └──────────────┘ └──────────────┘




3. Core Layer: Application Model (ApplicationModel)
The app model is the central graph that everything else reads and transforms.
Key types:

Type	Role
IResource / Resource	Base of all resources (projects, containers, databases, cloud services)
IResourceAnnotation	Metadata attached to resources. Over 70 annotation types exist.
ResourceAnnotationCollection	Per-resource bag of annotations
DistributedApplicationModel	The complete resource graph (IEnumerable<IResource>)
IResourceBuilder<T>	Fluent builder wrapping a resource; extension methods extend this
EndpointAnnotation / EndpointReference	Model network endpoints and references between them
ReferenceExpression	Lazy string expression resolving at runtime (e.g., connection strings)
IValueProvider	Async-evaluated string values used in env vars, params

Key interfaces used on resources:

Interface	Meaning
IResourceWithConnectionString	Has a connection string expression
IResourceWithEndpoints	Exposes network endpoints
IResourceWithEnvironment	Accepts environment variable injection
IResourceWithParameters	Takes typed parameters (e.g., Bicep params)
IResourceWithServiceDiscovery	Participates in service discovery
IAzureResource	Is an Azure cloud resource



4. Orchestration Layer: DCP Executor (Dcp)
DCP (Developer Control Plane) is Aspire's local execution engine. It is a separate binary (shipped with the Aspire workload) that presents a Kubernetes-style API.
App Host Process
  └─ DcpHost (starts DCP subprocess)
      └─ DcpExecutor
          ├─ Translates IResource graph → DCP Kubernetes objects
          │   (Container, Executable, Service CRDs)
          ├─ DcpKubernetesClient → HTTP to DCP's local K8s API
          ├─ Watches resource state changes → ResourceNotificationService
          └─ Streams logs → ResourceLoggerService → Dashboard



DcpExecutor is the main orchestrator: it walks the resource graph and creates DCP custom resources via KubernetesService.
Port allocation, container pulls, environment variable injection, endpoint allocation all happen here.
Resource state changes flow back via watch loops to ResourceNotificationService, which distributes events to the Dashboard and other subscribers.


5. Dashboard (Aspire.Dashboard)
A Blazor Server web app for monitoring running resources.
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


The Dashboard subscribes to ResourceNotificationService via gRPC streaming from the App Host and renders live resource state. It also receives OTLP telemetry directly from instrumented services.


6. Publishing Pipeline (Pipelines + Publishing/)
Publishing translates the app model into deployment artifacts. It runs when aspire publish or azd invokes the app host in publish mode.
DistributedApplicationPipeline
  ├─ PipelineStep (ordered, dependency-aware)
  │   ├─ "provision-redis"        (AzureBicepResource step)
  │   ├─ "provision-keyvault"     (AzureBicepResource step)
  │   └─ "write-manifest"         (ManifestPublisher step)
  └─ PipelineExecutor (runs steps in topological order)

ManifestPublisher
  └─ Calls ManifestPublishingCallbackAnnotation per resource
      └─ Each resource writes its JSON block to aspire-manifest.json


The manifest (aspire-manifest.json) is the intermediate format consumed by azd, Kubernetes publishers, or custom tooling.


7. Azure Integration & Bicep (Aspire.Hosting.Azure)
Azure resources follow a two-layer design:
C# Integration API (e.g., AddAzureRedis, AddAzureKeyVault)
   │
   ▼
AzureProvisioningResource  ─────────────────────────────────┐
   │ (uses Azure.Provisioning SDK to declare infra in C#)   │
   │                                                        │
   ▼                                                        │
GetBicepTemplateString()                                    │
   │ → infrastructure.Build().Compile()                     │
   │ → Emits .bicep text                                    │
   │                                                        │
   ▼                ─────────────────── Publish mode ───────┘
AzurePublishingContext
   │→ Writes per-resource .bicep files to output dir
   │→ Writes main.bicep orchestrator
   │→ Writes azd-compatible manifest
   │
   ▼
BicepProvisioner (run mode)
   │→ BicepCliCompiler: calls `az bicep build` → ARM JSON
   │→ Azure ARM SDK: ArmDeploymentContent → deploys to Azure
   └→ Reads outputs back → injects into resource connection strings


Two Bicep paths:

AzureProvisioningResource – C# code (Azure.Provisioning SDK) generates Bicep at runtime. Used by all built-in Azure integrations.
AzureBicepResource – Points to a hand-authored .bicep file. For custom or advanced scenarios (builder.AddBicepTemplate(...)).
30+ Azure integrations each in their own project (src/Aspire.Hosting.Azure.*/), all building on AzureProvisioningResource.


8. Client Integration Components (Components)
These are the application-side packages — consumed by the microservices themselves (not the App Host). They handle:

Dependency injection registration (e.g., builder.AddRedisClient())
Health checks (standard .NET health checks wired to infrastructure)
Resilience (Polly pipelines for retry, circuit breaker)
Telemetry (OpenTelemetry tracing automatically configured)
Configuration binding from appsettings.json / environment
Examples: Aspire.StackExchange.Redis, Aspire.Npgsql, Aspire.Azure.Storage.Blobs, etc. (38 packages total).


9. Aspire CLI (Aspire.Cli)
A native AOT .NET CLI with these top-level commands:

Command	Purpose
aspire run	Launch app host + DCP + Dashboard
aspire publish	Generate deployment artifacts (Bicep, K8s YAML)
aspire add	Add NuGet integration packages to a project
aspire new	Scaffold from templates
aspire deploy	Deploy to Azure (via azd)
aspire exec	Run a command inside the Aspire environment
aspire ps / logs / restart	Resource lifecycle management
aspire telemetry	View OTLP traces/spans/logs from CLI
aspire mcp	Start MCP server for AI tool integration
aspire docs	Search documentation

The CLI communicates with the running App Host via a backchannel (gRPC/JSON-RPC) defined in Backchannel and Backchannel.


10. Polyglot AppHost — ATS System (Ats)
The newest major architectural addition: allows TypeScript, Python, Go, Rust, Java to write app hosts.
Guest Language (e.g., TypeScript)
                     app-host.ts
              builder.addRedis("cache")          ← from generated SDK
                     │
                     │ JSON-RPC over stdio/socket
                     ▼
              .NET AppHost Server (Aspire.Hosting)
                     │
                     │ Dispatches via CapabilityDispatcher
                     ▼
              Actual C# extension method (AddRedis)
                     │
                     ▼
              Shared Application Model (same DCP, Dashboard, Bicep)


ATS (Aspire Type System):
[AspireExport("addRedis")]          ← attribute on C# method
    │
    ▼
AtsCapabilityScanner                ← reflects assemblies at build time
    │  → discovers capabilities, handle types, DTOs, enums
    ▼
AtsContext                          ← flat, portable type model
    │
    ▼
ICodeGenerator implementations:
  ├─ AtsTypeScriptCodeGenerator     → typed TS SDK classes
  ├─ Go / Python / Rust / Java generators
  └─ (one per src/Aspire.Hosting.CodeGeneration.*/ project)


Type categories in ATS:

Category	Attribute	Behavior
Handle	[AspireExport] on type	Opaque reference; method calls RPC back to .NET
DTO	[AspireDto]	Serialized as JSON, passed by value
Enum	(auto-discovered)	Serialized as string
Primitive	(auto-discovered)	string, number, boolean, null



11. VS Code Extension (extension)
TypeScript extension providing developer experience inside VS Code:
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
    aspire-settings.schema.json ← Settings schema (currently open file)


The extension communicates with the Dashboard's REST API (Api) to show resource state, logs, and traces directly in the editor.


12. Full Dependency / Data Flow Diagram
Developer writes C# App Host
         │
         ▼
  DistributedApplicationBuilder
         │ AddRedis / AddProject / AddAzureKeyVault / ...
         ▼
  Application Model (IResource DAG with Annotations)
         │
    ─────┼─────────────────────────────────────────
    │                    │                    │
    ▼                    ▼                    ▼
DCP Executor       Dashboard (Blazor)   Publishing Pipeline
(local run)        (monitoring)         (aspire publish)
    │                    │                    │
    │              OTLP telemetry       ┌─────┴──────┐
    │              from services        │            │
    ▼                                   ▼            ▼
Containers            ┌──────────────┐  Bicep     manifest.json
Processes     ──────► │ Resource     │  files     (for azd/K8s)
Docker/Podman         │ State Events │
    │                 └──────────────┘
    ▼
Service Discovery
(OTEL, env vars, connection strings)




13. Key Architectural Principles

Annotation-driven extensibility — All resource metadata is injected as annotations. Third-party integrations extend the model without forking core types.
Context-aware behavior — DistributedApplicationExecutionContext tells every resource whether it is in Run, Publish, or Test mode, allowing the same app model to behave differently per context.
Lazy value resolution — IValueProvider, ReferenceExpression, and EndpointReference are evaluated asynchronously at startup, allowing circular-ish references and deferred config.
Pipeline as first-class concept — Provisioning, deployment, and publishing are steps in a dependency-ordered DistributedApplicationPipeline, not ad-hoc lifecycle hooks.
No IDL for polyglot — ATS uses C# reflection as the schema source; [AspireExport] attributes are the contract, making polyglot SDKs a pure output of the existing C# type system.
Separation of hosting vs. client — src/Aspire.Hosting.* is the app host (orchestration); src/Components/Aspire.* is the client library (consumed by individual microservices). They are independent NuGet packages.
User: 1. add more details about DCP executor and what resource graph information it needed? And how KubernetesService is related? 2. Add more details about C# integration API in Azure Integration & Bicep section and what is Azure.Provisioning SDK?
GitHub Copilot: Read , lines 1 to 80
Read
Read , lines 1 to 120
Read , lines 1 to 80
Read , lines 1 to 80
Read , lines 1 to 80
Searched for regex CreateContainerAsync\|CreateExecutableAsync\|CreateServiceAsync\|CreateAsync.*Container\|CreateAsync.*Executable (**/src/Aspire.Hosting/Dcp/DcpExecutor.cs), no results
Read
Searched for regex _kubernetesService\.CreateAsync|kubernetesService\.CreateAsync (**/src/Aspire.Hosting/Dcp/DcpExecutor.cs), 4 results
Read , lines 1 to 80
Read , lines 2210 to 2300
Read , lines 80 to 200
Searched for text ConfigureRedisInfrastructure (**/src/Aspire.Hosting.Azure.Redis/AzureRedisExtensions.cs), 2 results
Read , lines 1 to 120
Read , lines 270 to 333
Read , lines 300 to 450
Read , lines 1 to 80
Searched for regex ContainerImageAnnotation|EndpointAnnotation|WaitAnnotation|ProjectResource|ExecutableResource (**/src/Aspire.Hosting/Dcp/DcpExecutor.cs), 20 results
Now I have all the details needed. Here are the two expanded sections:


4. Orchestration Layer: DCP Executor (Dcp) — Expanded
What is DCP?
DCP (Developer Control Plane) is a separate native binary shipped as part of the Aspire workload. It runs on the same machine as the app host and exposes a Kubernetes-style CRD API over a local Unix socket / named pipe. From DCP's perspective, resources are Kubernetes custom objects (CRDs), not containers or processes directly — DCP abstracts the actual container runtime (Docker, Podman) and process launcher behind that API surface.
What resource graph information does DcpExecutor consume?
DcpExecutor walks DistributedApplicationModel and reads the following annotations from each IResource:

Annotation / Resource Type	DCP CRD produced	Key fields mapped
ContainerImageAnnotation	Container (CRD)	image, ports, env, volumeMounts, command, args, restartPolicy, networks, persistent, start/stop
DockerfileBuildAnnotation	Container.Spec.Build	Dockerfile path, context, build args
EndpointAnnotation	Service (CRD)	address, port, protocol, addressAllocationMode
ProjectResource / ExecutableResource	Executable (CRD)	executablePath, workingDirectory, args, env, healthProbes, ambientEnvironment, pemCertificates
ContainerMountAnnotation	Container.Spec.VolumeMounts	Source, target, type (bind/volume)
EnvironmentAnnotation / EnvironmentCallbackAnnotation	Container.Spec.Env / Executable.Spec.Env	Resolved key/value EnvVar list
HealthCheckAnnotation	Executable.Spec.HealthProbes	HTTP/TCP probe config
WaitAnnotation	Ordering between Service objects	Which services must be ready before this resource starts
ProbeAnnotation	Container.Spec.HealthProbes	Liveness/readiness probes

The AppResource bridge type
AppResource (in AppResource.cs) is the coupling object between the Aspire app model and the DCP model. It is never sent over the wire — it lives only in the executor's _appResources list.
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


One RenderedModelResource is created per runnable Aspire resource. One ServiceWithModelResource is created per EndpointAnnotation. The ServicesConsumed list drives startup ordering (wait conditions).
How KubernetesService fits in
DcpExecutor                               DCP Binary (local process)
    │                                           │
    │  ─── HTTP over unix socket ──────────►   │
    │                                           │
    │  CreateAsync<Container>(spec)         K8s-style CRD API
    │  CreateAsync<Executable>(spec)        (list, get, create, patch, delete, watch)
    │  CreateAsync<Service>(spec)               │
    │  WatchAsync<Container>()    ◄──────────── │ (long-poll watch stream)
    │  WatchAsync<Executable>()   ◄──────────── │
    │  GetLogStreamAsync(...)     ◄──────────── │ (stdout/stderr as HTTP stream)
    │
    ├─ kubernetesService.CreateAsync(dcpContainerResource)   // line 2267
    ├─ kubernetesService.CreateAsync(exe)                   // line 1841
    └─ kubernetesService.PatchAsync(...)                    // resource lifecycle updates


IKubernetesService (implemented by KubernetesService) is the only channel between the executor and DCP. It uses the official k8s .NET client library (the same used for real Kubernetes), pointed at DCP's local endpoint. Every DCP CRD type (Container, Executable, Service, Endpoint, ContainerExec, ContainerNetwork) implements IKubernetesStaticMetadata so the generic CreateAsync<T> / WatchAsync<T> API works uniformly.
State change propagation after creation
After creation, DcpExecutor runs three concurrent watch loops:
WatchKubernetesResourceAsync<Executable>(...)
WatchKubernetesResourceAsync<Container>(...)     →  ProcessResourceChange()
WatchKubernetesResourceAsync<Service>(...)               │
WatchKubernetesResourceAsync<Endpoint>(...)              ▼
                                               ResourceSnapshotBuilder.ToSnapshot()
                                                         │
                                                         ▼
                                               ResourceNotificationService.PublishUpdateAsync()
                                                         │
                                                         ▼
                                               Dashboard (via gRPC stream) + CLI backchannel


ResourceNotificationService is an in-process pub/sub bus. The Dashboard subscribes to it and re-renders resource state in real time.


7. Azure Integration & Bicep — Expanded
The C# Integration API
Every Azure resource type has a dedicated project (src/Aspire.Hosting.Azure.*/) that exposes a fluent C# API for declaring that Azure resource. The pattern is consistent across all 30+ integrations:
// 1. Declare the Azure resource (always returns a typed IResourceBuilder)
var cache = builder.AddAzureManagedRedis("cache");

// 2. Optionally swap in a local Docker container for dev mode
//    (context-aware: does nothing in publish mode)
var cache = builder.AddAzureManagedRedis("cache")
                   .RunAsContainer();

// 3. Configure auth, role assignments, networking
var cache = builder.AddAzureManagedRedis("cache")
                   .WithAccessKeyAuthentication();

// 4. Wire it to services like any other resource
builder.AddProject<Projects.Api>()
       .WithReference(cache);


The triple context-awareness pattern is central to all Azure integrations:

Context	Behavior
Run mode (local aspire run)	RunAsContainer() — removes Azure resource, adds local Docker container instead
Publish mode (aspire publish)	Emits Bicep + manifest, no containers started
Azure provisioning (run with azure provisioner)	Actually deploys to Azure via ARM

What is the Azure.Provisioning SDK?
Azure.Provisioning.* (NuGet packages by the Azure SDK team, e.g. Azure.Provisioning.Redis, Azure.Provisioning.KeyVault) is a C#-first infrastructure-as-code SDK. It lets you declare Azure infrastructure as typed C# objects that compile to Bicep text. Think of it as the Azure equivalent of CDK or Pulumi's native SDK.
Key types in the Azure.Provisioning SDK:

SDK Type	Role
ProvisionableResource	Base class for all Azure resources (e.g. RedisResource, StorageAccount)
ProvisioningParameter	A Bicep param declaration
ProvisioningOutput	A Bicep output declaration
BicepValue<T>	A typed Bicep expression (can be a literal, a reference, or a function call)
BicepFunction.Interpolate(...)	Produces Bicep string interpolation ('${...}')
Infrastructure.Build().Compile()	Runs the entire graph and emits Bicep file content
ResourceX.FromExisting(...)	References an already-deployed Azure resource by name

Example — what ConfigureRedisInfrastructure does:
// From AzureRedisExtensions.cs
private static void ConfigureRedisInfrastructure(AzureResourceInfrastructure infrastructure)
{
    // 1. Declare an Azure Redis resource (Azure.Provisioning.Redis.RedisResource)
    var redis = AzureProvisioningResource.Create<CdkRedisResource>(infrastructure, ...);
    infrastructure.Add(redis);

    // 2. Add auth configuration
    redis.IsAccessKeyAuthenticationDisabled = true;
    redis.RedisConfiguration = new RedisCommonConfiguration { IsAadEnabled = "true" };

    // 3. Declare Bicep outputs — these become connection string references
    infrastructure.Add(new ProvisioningOutput("connectionString", typeof(string))
    {
        Value = BicepFunction.Interpolate(
AreaTechnologyPurpose
Local orchestrationDCP (internal), DockerStarts containers/processes during devApp modelC# resource graph, annotationsDeclares distributed app topologyAzure cloud resourcesBicep + Azure.Provisioning SDKInfrastructure-as-code for AzureBicep provisioningAzure ARM APIDeploys Azure resources during aspire runPublish artifactsBicep files + manifest JSONOutput for azd, K8s, etc.Polyglot app hostsATS + JSON-RPCLets TypeScript/Python/etc. write app hostsLanguage SDKsCode generation (ATS → TS/Py/Go/Rust/Java)Typed wrappers for each guest languageUser: Can you generate a comprehensive overview of the code architect of this repo
GitHub Copilot: Read
Read
Read
Read
Read
Read
Read
Read
Here is a comprehensive architectural overview of the Aspire repository:

dotnet/aspire — Code Architecture Overview

1. Repository Layout
aspire/
├── src/                    # All production source code
├── tests/                  # Unit & integration tests (mirrors src/)
├── playground/             # Sample apps for manual verification
├── extension/              # VS Code extension (TypeScript)
├── docs/specs/             # Architecture specs & design documents
├── eng/                    # Build infrastructure (Arcade SDK, pipelines)
└── tools/                  # Developer tooling (QuarantineTools, etc.)

2. Major Subsystems
┌──────────────────────────────────────────────────────────────────────────┐
│                          User-Facing Entry Points                        │
│                                                                          │
│  aspire CLI (Aspire.Cli)   VS Code Extension  App Host (C# Program.cs)  │
└─────────────┬──────────────────────┬─────────────────────┬──────────────┘
              │                      │                      │
              ▼                      ▼                      ▼
┌─────────────────────┐  ┌──────────────────┐  ┌──────────────────────────┐
│  CLI Orchestration  │  │  VS Code Ext.    │  │   App Host SDK           │
│  (run/publish/add)  │  │  (debug, attach) │  │   (Aspire.Hosting)       │
└──────────┬──────────┘  └──────────────────┘  └──────────┬───────────────┘
           │                                               │
           │                               ┌──────────────▼───────────────┐
           │                               │       Application Model      │
           │                               │  IResource, Annotations, DAG │
           │                               └──────────────┬───────────────┘
           │                                              │
           │              ┌───────────────────────────────┼────────────────┐
           │              │               │               │                │
           ▼              ▼               ▼               ▼                ▼
    ┌──────────┐  ┌──────────────┐ ┌──────────┐ ┌──────────────┐ ┌──────────────┐
    │  AzDev   │  │DCP Executor  │ │Dashboard │ │  Publishing  │ │   Azure      │
    │  CLI     │  │(local run)   │ │(Blazor)  │ │  Pipeline    │ │ Provisioning │
    └──────────┘  └──────────────┘ └──────────┘ └──────────────┘ └──────────────┘

3. Core Layer: Application Model (ApplicationModel)
The app model is the central graph that everything else reads and transforms.
Key types:
TypeRole
IResource / ResourceBase of all resources (projects, containers, databases, cloud services)IResourceAnnotationMetadata attached to resources. Over 70 annotation types exist.ResourceAnnotationCollectionPer-resource bag of annotationsDistributedApplicationModelThe complete resource graph (IEnumerable<IResource>)IResourceBuilder<T>Fluent builder wrapping a resource; extension methods extend thisEndpointAnnotation / EndpointReferenceModel network endpoints and references between themReferenceExpressionLazy string expression resolving at runtime (e.g., connection strings)IValueProviderAsync-evaluated string values used in env vars, paramsKey interfaces used on resources:
InterfaceMeaning
IResourceWithConnectionStringHas a connection string expressionIResourceWithEndpointsExposes network endpointsIResourceWithEnvironmentAccepts environment variable injectionIResourceWithParametersTakes typed parameters (e.g., Bicep params)IResourceWithServiceDiscoveryParticipates in service discoveryIAzureResourceIs an Azure cloud resource
4. Orchestration Layer: DCP Executor (Dcp)
DCP (Developer Control Plane) is Aspire's local execution engine. It is a separate binary (shipped with the Aspire workload) that presents a Kubernetes-style API.
App Host Process
  └─ DcpHost (starts DCP subprocess)
      └─ DcpExecutor
          ├─ Translates IResource graph → DCP Kubernetes objects
          │   (Container, Executable, Service CRDs)
          ├─ DcpKubernetesClient → HTTP to DCP's local K8s API
          ├─ Watches resource state changes → ResourceNotificationService
          └─ Streams logs → ResourceLoggerService → Dashboard
DcpExecutor is the main orchestrator: it walks the resource graph and creates DCP custom resources via KubernetesService.
Port allocation, container pulls, environment variable injection, endpoint allocation all happen here.
Resource state changes flow back via watch loops to ResourceNotificationService, which distributes events to the Dashboard and other subscribers.

5. Dashboard (Aspire.Dashboard)
A Blazor Server web app for monitoring running resources.
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
The Dashboard subscribes to ResourceNotificationService via gRPC streaming from the App Host and renders live resource state. It also receives OTLP telemetry directly from instrumented services.

6. Publishing Pipeline (Pipelines + Publishing/)
Publishing translates the app model into deployment artifacts. It runs when aspire publish or azd invokes the app host in publish mode.
DistributedApplicationPipeline
  ├─ PipelineStep (ordered, dependency-aware)
  │   ├─ "provision-redis"        (AzureBicepResource step)
  │   ├─ "provision-keyvault"     (AzureBicepResource step)
  │   └─ "write-manifest"         (ManifestPublisher step)
  └─ PipelineExecutor (runs steps in topological order)

ManifestPublisher
  └─ Calls ManifestPublishingCallbackAnnotation per resource
      └─ Each resource writes its JSON block to aspire-manifest.json
The manifest (aspire-manifest.json) is the intermediate format consumed by azd, Kubernetes publishers, or custom tooling.

7. Azure Integration & Bicep (Aspire.Hosting.Azure)
Azure resources follow a two-layer design:
C# Integration API (e.g., AddAzureRedis, AddAzureKeyVault)
   │
   ▼
AzureProvisioningResource  ─────────────────────────────────┐
   │ (uses Azure.Provisioning SDK to declare infra in C#)   │
   │                                                        │
   ▼                                                        │
GetBicepTemplateString()                                    │
   │ → infrastructure.Build().Compile()                     │
   │ → Emits .bicep text                                    │
   │                                                        │
   ▼                ─────────────────── Publish mode ───────┘
AzurePublishingContext
   │→ Writes per-resource .bicep files to output dir
   │→ Writes main.bicep orchestrator
   │→ Writes azd-compatible manifest
   │
   ▼
BicepProvisioner (run mode)
   │→ BicepCliCompiler: calls `az bicep build` → ARM JSON
   │→ Azure ARM SDK: ArmDeploymentContent → deploys to Azure
   └→ Reads outputs back → injects into resource connection strings
Two Bicep paths:
AzureProvisioningResource – C# code (Azure.Provisioning SDK) generates Bicep at runtime. Used by all built-in Azure integrations.
AzureBicepResource – Points to a hand-authored .bicep file. For custom or advanced scenarios (builder.AddBicepTemplate(...)).
30+ Azure integrations each in their own project (src/Aspire.Hosting.Azure.*/), all building on AzureProvisioningResource.

8. Client Integration Components (Components)
These are the application-side packages — consumed by the microservices themselves (not the App Host). They handle:
Dependency injection registration (e.g., builder.AddRedisClient())
Health checks (standard .NET health checks wired to infrastructure)
Resilience (Polly pipelines for retry, circuit breaker)
Telemetry (OpenTelemetry tracing automatically configured)
Configuration binding from appsettings.json / environment
Examples: Aspire.StackExchange.Redis, Aspire.Npgsql, Aspire.Azure.Storage.Blobs, etc. (38 packages total).

9. Aspire CLI (Aspire.Cli)
A native AOT .NET CLI with these top-level commands:
CommandPurpose
aspire runLaunch app host + DCP + Dashboardaspire publishGenerate deployment artifacts (Bicep, K8s YAML)aspire addAdd NuGet integration packages to a projectaspire newScaffold from templatesaspire deployDeploy to Azure (via azd)aspire execRun a command inside the Aspire environmentaspire ps / logs / restartResource lifecycle managementaspire telemetryView OTLP traces/spans/logs from CLIaspire mcpStart MCP server for AI tool integrationaspire docsSearch documentationThe CLI communicates with the running App Host via a backchannel (gRPC/JSON-RPC) defined in Backchannel and Backchannel.

10. Polyglot AppHost — ATS System (Ats)
The newest major architectural addition: allows TypeScript, Python, Go, Rust, Java to write app hosts.
              Guest Language (e.g., TypeScript)
                     app-host.ts
              builder.addRedis("cache")          ← from generated SDK
                     │
                     │ JSON-RPC over stdio/socket
                     ▼
              .NET AppHost Server (Aspire.Hosting)
                     │
                     │ Dispatches via CapabilityDispatcher
                     ▼
              Actual C# extension method (AddRedis)
                     │
                     ▼
              Shared Application Model (same DCP, Dashboard, Bicep)
ATS (Aspire Type System):
[AspireExport("addRedis")]          ← attribute on C# method
    │
    ▼
AtsCapabilityScanner                ← reflects assemblies at build time
    │  → discovers capabilities, handle types, DTOs, enums
    ▼
AtsContext                          ← flat, portable type model
    │
    ▼
ICodeGenerator implementations:
  ├─ AtsTypeScriptCodeGenerator     → typed TS SDK classes
  ├─ Go / Python / Rust / Java generators
  └─ (one per src/Aspire.Hosting.CodeGeneration.*/ project)
Type categories in ATS:
CategoryAttributeBehavior
Handle[AspireExport] on typeOpaque reference; method calls RPC back to .NETDTO[AspireDto]Serialized as JSON, passed by valueEnum(auto-discovered)Serialized as stringPrimitive(auto-discovered)string, number, boolean, null
11. VS Code Extension (extension)
TypeScript extension providing developer experience inside VS Code:
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
    aspire-settings.schema.json ← Settings schema (currently open file)
The extension communicates with the Dashboard's REST API (Api) to show resource state, logs, and traces directly in the editor.

12. Full Dependency / Data Flow Diagram
Developer writes C# App Host
         │
         ▼
  DistributedApplicationBuilder
         │ AddRedis / AddProject / AddAzureKeyVault / ...
         ▼
  Application Model (IResource DAG with Annotations)
         │
    ─────┼─────────────────────────────────────────
    │                    │                    │
    ▼                    ▼                    ▼
DCP Executor       Dashboard (Blazor)   Publishing Pipeline
(local run)        (monitoring)         (aspire publish)
    │                    │                    │
    │              OTLP telemetry       ┌─────┴──────┐
    │              from services        │            │
    ▼                                   ▼            ▼
Containers            ┌──────────────┐  Bicep     manifest.json
Processes     ──────► │ Resource     │  files     (for azd/K8s)
Docker/Podman         │ State Events │
    │                 └──────────────┘
    ▼
Service Discovery
(OTEL, env vars, connection strings)

13. Key Architectural Principles
Annotation-driven extensibility — All resource metadata is injected as annotations. Third-party integrations extend the model without forking core types.
Context-aware behavior — DistributedApplicationExecutionContext tells every resource whether it is in Run, Publish, or Test mode, allowing the same app model to behave differently per context.
Lazy value resolution — IValueProvider, ReferenceExpression, and EndpointReference are evaluated asynchronously at startup, allowing circular-ish references and deferred config.
Pipeline as first-class concept — Provisioning, deployment, and publishing are steps in a dependency-ordered DistributedApplicationPipeline, not ad-hoc lifecycle hooks.
No IDL for polyglot — ATS uses C# reflection as the schema source; [AspireExport] attributes are the contract, making polyglot SDKs a pure output of the existing C# type system.
Separation of hosting vs. client — src/Aspire.Hosting.* is the app host (orchestration); src/Components/Aspire.* is the client library (consumed by individual microservices). They are independent NuGet packages.
User: 1. add more details about DCP executor and what resource graph information it needed? And how KubernetesService is related?
2. Add more details about C# integration API in Azure Integration & Bicep section  and what is Azure.Provisioning SDK?
GitHub Copilot: Read , lines 1 to 80
Read
Read , lines 1 to 120
Read , lines 1 to 80
Read , lines 1 to 80
Read , lines 1 to 80
Searched for regex CreateContainerAsync\|CreateExecutableAsync\|CreateServiceAsync\|CreateAsync.*Container\|CreateAsync.*Executable (**/src/Aspire.Hosting/Dcp/DcpExecutor.cs), no results
Read
Searched for regex _kubernetesService\.CreateAsync|kubernetesService\.CreateAsync (**/src/Aspire.Hosting/Dcp/DcpExecutor.cs), 4 results
Read , lines 1 to 80
Read , lines 2210 to 2300
Read , lines 80 to 200
Searched for text ConfigureRedisInfrastructure (**/src/Aspire.Hosting.Azure.Redis/AzureRedisExtensions.cs), 2 results
Read , lines 1 to 120
Read , lines 270 to 333
Read , lines 300 to 450
Read , lines 1 to 80
Searched for regex ContainerImageAnnotation|EndpointAnnotation|WaitAnnotation|ProjectResource|ExecutableResource (**/src/Aspire.Hosting/Dcp/DcpExecutor.cs), 20 results
Now I have all the details needed. Here are the two expanded sections:

4. Orchestration Layer: DCP Executor (Dcp) — Expanded
What is DCP?
DCP (Developer Control Plane) is a separate native binary shipped as part of the Aspire workload. It runs on the same machine as the app host and exposes a Kubernetes-style CRD API over a local Unix socket / named pipe. From DCP's perspective, resources are Kubernetes custom objects (CRDs), not containers or processes directly — DCP abstracts the actual container runtime (Docker, Podman) and process launcher behind that API surface.
What resource graph information does DcpExecutor consume?
DcpExecutor walks DistributedApplicationModel and reads the following annotations from each IResource:
Annotation / Resource TypeDCP CRD producedKey fields mapped
ContainerImageAnnotationContainer (CRD)image, ports, env, volumeMounts, command, args, restartPolicy, networks, persistent, start/stopDockerfileBuildAnnotationContainer.Spec.BuildDockerfile path, context, build argsEndpointAnnotationService (CRD)address, port, protocol, addressAllocationModeProjectResource / ExecutableResourceExecutable (CRD)executablePath, workingDirectory, args, env, healthProbes, ambientEnvironment, pemCertificatesContainerMountAnnotationContainer.Spec.VolumeMountsSource, target, type (bind/volume)EnvironmentAnnotation / EnvironmentCallbackAnnotationContainer.Spec.Env / Executable.Spec.EnvResolved key/value EnvVar listHealthCheckAnnotationExecutable.Spec.HealthProbesHTTP/TCP probe configWaitAnnotationOrdering between Service objectsWhich services must be ready before this resource startsProbeAnnotationContainer.Spec.HealthProbesLiveness/readiness probesThe AppResource bridge type
AppResource (in AppResource.cs) is the coupling object between the Aspire app model and the DCP model. It is never sent over the wire — it lives only in the executor's _appResources list.
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
One RenderedModelResource is created per runnable Aspire resource. One ServiceWithModelResource is created per EndpointAnnotation. The ServicesConsumed list drives startup ordering (wait conditions).
How KubernetesService fits in
DcpExecutor                               DCP Binary (local process)
    │                                           │
    │  ─── HTTP over unix socket ──────────►   │
    │                                           │
    │  CreateAsync<Container>(spec)         K8s-style CRD API
    │  CreateAsync<Executable>(spec)        (list, get, create, patch, delete, watch)
    │  CreateAsync<Service>(spec)               │
    │  WatchAsync<Container>()    ◄──────────── │ (long-poll watch stream)
    │  WatchAsync<Executable>()   ◄──────────── │
    │  GetLogStreamAsync(...)     ◄──────────── │ (stdout/stderr as HTTP stream)
    │
    ├─ kubernetesService.CreateAsync(dcpContainerResource)   // line 2267
    ├─ kubernetesService.CreateAsync(exe)                   // line 1841
    └─ kubernetesService.PatchAsync(...)                    // resource lifecycle updates
IKubernetesService (implemented by KubernetesService) is the only channel between the executor and DCP. It uses the official k8s .NET client library (the same used for real Kubernetes), pointed at DCP's local endpoint. Every DCP CRD type (Container, Executable, Service, Endpoint, ContainerExec, ContainerNetwork) implements IKubernetesStaticMetadata so the generic CreateAsync<T> / WatchAsync<T> API works uniformly.
State change propagation after creation
After creation, DcpExecutor runs three concurrent watch loops:
WatchKubernetesResourceAsync<Executable>(...)
WatchKubernetesResourceAsync<Container>(...)     →  ProcessResourceChange()
WatchKubernetesResourceAsync<Service>(...)               │
WatchKubernetesResourceAsync<Endpoint>(...)              ▼
                                               ResourceSnapshotBuilder.ToSnapshot()
                                                         │
                                                         ▼
                                               ResourceNotificationService.PublishUpdateAsync()
                                                         │
                                                         ▼
                                               Dashboard (via gRPC stream) + CLI backchannel
ResourceNotificationService is an in-process pub/sub bus. The Dashboard subscribes to it and re-renders resource state in real time.

7. Azure Integration & Bicep — Expanded
The C# Integration API
Every Azure resource type has a dedicated project (src/Aspire.Hosting.Azure.*/) that exposes a fluent C# API for declaring that Azure resource. The pattern is consistent across all 30+ integrations:
// 1. Declare the Azure resource (always returns a typed IResourceBuilder)
var cache = builder.AddAzureManagedRedis("cache");

// 2. Optionally swap in a local Docker container for dev mode
//    (context-aware: does nothing in publish mode)
var cache = builder.AddAzureManagedRedis("cache")
                   .RunAsContainer();

// 3. Configure auth, role assignments, networking
var cache = builder.AddAzureManagedRedis("cache")
                   .WithAccessKeyAuthentication();

// 4. Wire it to services like any other resource
builder.AddProject<Projects.Api>()
       .WithReference(cache);
The triple context-awareness pattern is central to all Azure integrations:
ContextBehavior
Run mode (local aspire run)RunAsContainer() — removes Azure resource, adds local Docker container insteadPublish mode (aspire publish)Emits Bicep + manifest, no containers startedAzure provisioning (run with azure provisioner)Actually deploys to Azure via ARMWhat is the Azure.Provisioning SDK?
Azure.Provisioning.* (NuGet packages by the Azure SDK team, e.g. Azure.Provisioning.Redis, Azure.Provisioning.KeyVault) is a C#-first infrastructure-as-code SDK. It lets you declare Azure infrastructure as typed C# objects that compile to Bicep text. Think of it as the Azure equivalent of CDK or Pulumi's native SDK.
Key types in the Azure.Provisioning SDK:
SDK TypeRole
ProvisionableResourceBase class for all Azure resources (e.g. RedisResource, StorageAccount)ProvisioningParameterA Bicep param declarationProvisioningOutputA Bicep output declarationBicepValue<T>A typed Bicep expression (can be a literal, a reference, or a function call)BicepFunction.Interpolate(...)Produces Bicep string interpolation ('${...}')Infrastructure.Build().Compile()Runs the entire graph and emits Bicep file contentResourceX.FromExisting(...)References an already-deployed Azure resource by nameExample — what ConfigureRedisInfrastructure does:
// From AzureRedisExtensions.cs
private static void ConfigureRedisInfrastructure(AzureResourceInfrastructure infrastructure)
{
    // 1. Declare an Azure Redis resource (Azure.Provisioning.Redis.RedisResource)
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

    // 4. Optionally store secret in Key Vault (for access-key auth path)
    var secret = new KeyVaultSecret("connectionString")
    {
        Parent = keyVault,
        Properties = new SecretProperties
        {
            Value = BicepFunction.Interpolate($"{redis.HostName},ssl=true,password={redis.GetKeys().PrimaryKey}")
        }
    };
    infrastructure.Add(secret);
}
When AzureProvisioningResource.GetBicepTemplateString() is called, it:
Creates a fresh AzureResourceInfrastructure (wrapper around Azure.Provisioning Infrastructure)
Calls ConfigureInfrastructure(infrastructure) — the callback above runs
Calls infrastructure.Build(ProvisioningBuildOptions) — Azure.Provisioning SDK computes the full Bicep module
Calls .Compile() — emits the Bicep source text as a string
Full Azure resource lifecycle mapping
C# API call (AddAzureManagedRedis)
    │
    ├─ Creates AzureManagedRedisResource (: AzureProvisioningResource)
    │   └─ Stores configureInfrastructure callback
    │
    ├─ Attaches ManifestPublishingCallbackAnnotation
    │   └─ WriteToManifest() → writes "type: azure.bicep.v0" + path to manifest.json
    │
    ├─ Attaches PipelineStepAnnotation
    │   └─ provision-{name} step → calls BicepProvisioner.GetOrCreateResourceAsync()
    │
    └─ Attaches PipelineConfigurationAnnotation
        └─ Forces Bicep template evaluation → discovers cross-resource dependencies

At run/publish time:
    GetBicepTemplateString()
        → new AzureResourceInfrastructure(this, Name)
        → ConfigureInfrastructure(infra)         ← Azure.Provisioning objects added here
        → infra.Build(options).Compile()          ← Azure.Provisioning emits Bicep text
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
The key insight: Azure.Provisioning SDK is the bridge between Aspire's C# resource model and Bicep text. Integration authors never write .bicep files by hand — they compose Azure.Provisioning typed objects in the configureInfrastructure callback, and the SDK generates correct, valid Bicep automatically.
User: How AppHost interact with guest runtime and language runtime? How to integrate it with another language like TypeScript runtime or TypeScript AppHost?
GitHub Copilot: Read , lines 100 to 300
Read
Read , lines 300 to 600
Read
Read , lines 600 to 900
Read
Read , lines 1 to 120
Read , lines 1 to 100
Read , lines 1 to 120
Read , lines 120 to 181
Here's a detailed explanation of how the AppHost interacts with guest runtimes, using TypeScript as the concrete example:

AppHost ↔ Guest Runtime Interaction
The Big Picture
The key design principle: all 40+ integrations stay in .NET. Guest language code never reimplements them. Instead, guest code calls into the .NET server via JSON-RPC and gets back opaque handles to .NET objects, which it passes forward in future calls.

1. Process Architecture
There are always three separate processes at runtime:
┌──────────────────────┐
│     Aspire CLI       │  ← orchestrator: starts both, passes socket path
└───────┬──────────────┘
        │ spawns                      spawns │
        ▼                                    ▼
┌──────────────────────┐       ┌─────────────────────────────┐
│  AppHost Server      │       │  Guest Process               │
│  (.NET)              │◄─────►│  (e.g., `npx tsx apphost.ts`)│
│                      │ RPC   │                              │
│  - ILanguageSupport  │       │  - Generated SDK             │
│  - ICodeGenerator    │       │  - User's apphost.ts         │
│  - CapabilityDispatch│       │  - ATS Client (transport.ts) │
│  - App Model & DCP   │       └─────────────────────────────┘
└──────────────────────┘
        ▲ JSON-RPC over Unix socket
        (path passed via REMOTE_APP_HOST_SOCKET_PATH env var)

2. Startup Sequence Step by Step
aspire run (in TypeScript apphost dir)
      │
      ▼
[1] CLI detects language
      │  Calls server RPC: detect(directoryPath)
      │  TypeScriptLanguageSupport.Detect() checks:
      │    - apphost.ts exists?
      │    - package.json exists?
      │  Returns: language = "typescript/nodejs"
      │
      ▼
[2] CLI requests RuntimeSpec from server
      │  Calls server RPC: getRuntimeSpec("typescript/nodejs")
      │  TypeScriptLanguageSupport.GetRuntimeSpec() returns:
      │    Execute:  { command: "npx", args: ["tsx", "{appHostFile}"] }
      │    WatchExec:{ command: "npx", args: ["nodemon", "--exec", "npx tsx {appHostFile}"] }
      │    InstallDeps: { command: "npm", args: ["install"] }
      │
      ▼
[3] CLI installs guest dependencies
      │  Runs: npm install  (in the apphost directory)
      │
      ▼
[4] CLI builds the AppHost Server .NET project
      │  dotnet build (temporary generated project in $TMPDIR/.aspire/hosts/<hash>/)
      │  Adds required hosting packages from lockfile
      │
      ▼
[5] Code generation
      │  AtsCapabilityScanner scans all loaded assemblies for [AspireExport]
      │  AtsTypeScriptCodeGenerator generates .modules/aspire.js + aspire.d.ts
      │  Emits typed builder classes with invokeCapability() under the hood
      │
      ▼
[6] CLI starts AppHost Server (.NET)
      │  Sets REMOTE_APP_HOST_SOCKET_PATH=/tmp/aspire/<session>.sock
      │  .NET server begins listening on Unix socket
      │
      ▼
[7] CLI starts Guest Process
      │  Runs: npx tsx apphost.ts
      │  Injects REMOTE_APP_HOST_SOCKET_PATH into guest's environment
      │
      ▼
[8] Guest connects and calls capabilities (see §3 below)

3. JSON-RPC Protocol: Wire Format
The transport layer uses LSP-style framing (same as VS Code extensions) over a Unix socket:
Content-Length: 127\r\n
\r\n
{"jsonrpc":"2.0","id":1,"method":"invokeCapability","params":["Aspire.Hosting/createBuilder",{}]}
The TypeScript SDK's transport.ts uses the vscode-jsonrpc npm package for this — the same package VS Code itself uses. No custom framing code needed.

4. Handle System: How Objects Cross the Boundary
Every .NET object that Guest code holds is an opaque handle — just a string ID + type ID tuple. The guest never deserializes the object's internals; it only passes the handle back for future calls.
Guest                                            .NET AppHost Server
  │                                                     │
  │── invokeCapability("Aspire.Hosting/createBuilder", {}) ──►│
  │                                                     │  Creates IDistributedApplicationBuilder
  │◄── { "$handle": "1",                                │  Stores in handle registry [1 → builder]
  │      "$type": "Aspire.Hosting/...IDistributedApplicationBuilder" }
  │
  │── invokeCapability("Aspire.Hosting.Redis/addRedis", {
  │       "builder": { "$handle": "1", "$type": "..." },  ← pass handle back
  │       "name": "cache"
  │   }) ──►│
  │         │  Looks up handle 1 → builder
  │         │  Calls RedisExtensions.AddRedis(builder, "cache")
  │         │  Stores result in handle registry [2 → IResourceBuilder<RedisResource>]
  │◄── { "$handle": "2", "$type": "Aspire.Hosting.Redis/.../RedisResource" }
  │
  │── invokeCapability("Aspire.Hosting/withEnvironment", {
  │       "builder": { "$handle": "2", ... },
  │       "name": "MY_KEY",
  │       "value": "hello"
  │   }) ──►│
The ATS type ID format is {AssemblyName}/{FullTypeName}, e.g.:
Aspire.Hosting/Aspire.Hosting.IDistributedApplicationBuilder
Aspire.Hosting.Redis/Aspire.Hosting.ApplicationModel.RedisResource

5. What the Generated TypeScript SDK Looks Like
The code generator reads [AspireExport] attributes from C# assemblies and produces a typed TypeScript module. A simplified view of what it emits:
// .modules/aspire.js  (generated — do not edit)
import { AspireClient, Handle, registerHandleWrapper } from './base.js';

// --- Handles ---
export type IDistributedApplicationBuilderHandle =
    Handle<'Aspire.Hosting/Aspire.Hosting.IDistributedApplicationBuilder'>;

export type RedisResourceBuilderHandle =
    Handle<'Aspire.Hosting.Redis/Aspire.Hosting.ApplicationModel.RedisResource'>;

// --- DistributedApplicationBuilder wrapper ---
export class DistributedApplicationBuilder {
    constructor(private readonly _handle: IDistributedApplicationBuilderHandle,
                private readonly _client: AspireClient) {}

    async addRedis(name: string, port?: number): Promise<RedisResourceBuilder> {
        const result = await this._client.invokeCapability(
            'Aspire.Hosting.Redis/addRedis',
            { builder: this._handle, name, port }
        );
        return new RedisResourceBuilder(result as RedisResourceBuilderHandle, this._client);
    }

    async build(): Promise<DistributedApplication> { ... }
}

// --- RedisResourceBuilder wrapper ---
export class RedisResourceBuilder {
    constructor(private readonly _handle: RedisResourceBuilderHandle,
                private readonly _client: AspireClient) {}

    async withEnvironment(name: string, value: string): Promise<RedisResourceBuilder> {
        const result = await this._client.invokeCapability(
            'Aspire.Hosting/withEnvironment',
            { builder: this._handle, name, value }
        );
        return new RedisResourceBuilder(result as RedisResourceBuilderHandle, this._client);
    }

    async withPersistence(interval?: number): Promise<RedisResourceBuilder> { ... }
    async withDataVolume(name?: string): Promise<RedisResourceBuilder> { ... }
}

// --- Entry point ---
export async function createBuilder(): Promise<DistributedApplicationBuilder> {
    const socketPath = process.env['REMOTE_APP_HOST_SOCKET_PATH']!;
    const client = new AspireClient(socketPath);
    await client.connect();
    const handle = await client.invokeCapability('Aspire.Hosting/createBuilder', {});
    return new DistributedApplicationBuilder(handle, client);
}
The TypeScript user's apphost.ts then just does:
import { createBuilder } from './.modules/aspire.js';

const builder = await createBuilder();
const cache = await builder.addRedis("cache");
const api = await builder.addProject("api", "../Api/Api.csproj");
await api.withReference(cache);
await builder.build().run();

6. Callbacks: Guest Code Invoked by .NET
When C# needs to call back into the guest (e.g., withEnvironmentCallback), the guest registers a callback ID:
Guest                                                  .NET AppHost Server
  │                                                           │
  │  // TypeScript registers a closure
  │  const callbackId = registerCallback(async (args, client) => {
  │      const ctx = new EnvironmentCallbackContext(args.p0, client);
  │      await ctx.environmentVariables.set("MY_KEY", "hello");
  │  });
  │
  │── invokeCapability("Aspire.Hosting/withEnvironmentCallback", {
  │       "resource": { "$handle":"2",...},
  │       "callback": "callback_1_1234567890"     ← just a string ID
  │   }) ──►│
  │         │  Stores callback string → C# delegate
  │         │  When environment is evaluated at startup:
  │         │
  │◄── invokeCallback("callback_1_1234567890", {
  │        "p0": { "$handle": "5",
  │                "$type": "Aspire.Hosting/.../EnvironmentCallbackContext" }
  │    }) ─────────────────────────────────────────────────────►│
  │    │                                                         │
  │    │  // Guest wraps handle in typed class
  │    │  // Runs user's async closure
  │    │  // ctx.environmentVariables.set("MY_KEY", "hello")
  │    │    ├── invokeCapability("set", { dict: handle, key, value }) ──►
  │    │
  │◄── result: null  ──────────────────────────────────────────►│

7. How to Add a New Guest Language (e.g., Python)
To add Python support you need exactly two things on the server side, both discovered automatically from the NuGet package:
Step 1: Implement ILanguageSupport
// In new package: Aspire.Hosting.CodeGeneration.Python
public sealed class PythonLanguageSupport : ILanguageSupport
{
    public string Language => "python/cpython";

    public Dictionary<string, string> Scaffold(ScaffoldRequest request)
    {
        return new Dictionary<string, string>
        {
            ["apphost.py"] = "from aspire import create_builder\n\nbuilder = await create_builder()\nawait builder.build().run()\n",
            ["requirements.txt"] = "aspire-hosting>=1.0.0\n",
        };
    }

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
        Language = "python/cpython",
        DisplayName = "Python",
        CodeGenLanguage = "Python",
        DetectionPatterns = ["apphost.py"],
        InstallDependencies = new CommandSpec { Command = "pip", Args = ["install", "-r", "requirements.txt"] },
        Execute = new CommandSpec { Command = "python", Args = ["{appHostFile}"] },
        WatchExecute = new CommandSpec
        {
            Command = "watchdog",
            Args = ["watchmedo", "auto-restart", "--patterns=*.py", "--recursive", "--", "python", "{appHostFile}"]
        }
    };
}
Step 2: Implement ICodeGenerator
public sealed class PythonCodeGenerator : ICodeGenerator
{
    public string Language => "Python";

    // Receives the AtsContext (all capabilities, handles, DTOs, enums)
    // Returns a dict of filename → file content
    public Dictionary<string, string> Generate(AtsContext context)
    {
        var sb = new StringBuilder();
        sb.AppendLine("# aspire.py - Generated SDK (do not edit)");
        sb.AppendLine("import json, socket");
        sb.AppendLine("from aspire_transport import AspireClient, Handle");
        sb.AppendLine();

        foreach (var capability in context.Capabilities)
        {
            // Generate typed Python method for each capability
            GenerateCapabilityMethod(sb, capability);
        }

        return new Dictionary<string, string> { [".modules/aspire.py"] = sb.ToString() };
    }
}
Step 3: Register via DI (one line in the NuGet package's hosting extension)
// In Aspire.Hosting.CodeGeneration.Python startup
services.AddSingleton<ILanguageSupport, PythonLanguageSupport>();
services.AddSingleton<ICodeGenerator, PythonCodeGenerator>();
The CLI and AppHost server automatically discover these implementations via DI from the loaded assemblies. No CLI changes needed at all — adding the NuGet package is the only integration step.

8. Summary Table
ConcernOwned byMechanism
Socket path distributionCLIREMOTE_APP_HOST_SOCKET_PATH env varLanguage detectionILanguageSupport.Detect() on serverCLI calls detect RPCRun commandILanguageSupport.GetRuntimeSpec() on serverCLI calls getRuntimeSpec RPCScaffolding new projectILanguageSupport.Scaffold() on serverCLI calls scaffoldAppHost RPCSDK code generationICodeGenerator on serverCLI calls generateCode RPCGuest → Host callsATS JSON-RPC invokeCapabilityGuest connects to Unix socketHost → Guest callbacksATS JSON-RPC invokeCallbackReverse direction on same connectionHandle lifecycleAppHost server handle registryValid until server exitsAll Aspire integrations.NET server processGuest sees them as typed SDK methodsUser: help me set up the local development mode for aspire
GitHub Copilot: Let me check the current state of the repo and relevant setup files.
Read , lines 1 to 80
Ran terminal command:  cat global.json
Ran terminal command:  ls .dotnet 2>/dev/null && echo "SDK exists" || echo "SDK not installed"
Ran terminal command:  ls -la .dotnet 2>/dev/null | head -5; echo "exit: $?"
Ran terminal command:  .dotnet/dotnet --version 2>/dev/null || echo "dotnet not found in .dotnet/"
Good — the local SDK is already installed. Let me check if the repo has been built before and what state the artifacts are in.
Ran terminal command:  ls artifacts/bin 2>/dev/null | head -10; echo "---"; ls artifacts/packages 2>/dev/null | head -5
The repo has already been built and the local SDK is installed. Here's the full local dev setup:

Local Development Setup Status
Already done:
✅ Local .NET SDK installed (dotnet → 10.0.103)
✅ Repo previously built (bin populated)
What's already available:
# Use this instead of system `dotnet` to get the local SDK
./dotnet.sh --version   # → 10.0.103

Daily Workflow
Launch VS Code correctly
Always launch VS Code from the repo root using the helper script — this sets the correct PATH so VS Code terminals automatically use the local SDK:
./start-code.sh
Don't open VS Code directly via code . — that won't have .dotnet on PATH.
Build after code changes
# Full build (restore + build, ~3-5 minutes)
./build.sh

# Build only (skip restore, faster after first run)
./build.sh --build

# Skip native AOT compilation (~1-2 minutes saved)
./build.sh --build /p:SkipNativeBuild=true
Run the sample app to verify setup
./dotnet.sh run --project playground/TestShop/TestShop.AppHost/TestShop.AppHost.csproj
This starts the TestShop distributed app and opens the Dashboard — confirms everything is wired up correctly.
Run tests for a specific project
./dotnet.sh test tests/Aspire.Hosting.Tests/Aspire.Hosting.Tests.csproj -- \
  --filter-not-trait "quarantined=true" --filter-not-trait "outerloop=true"

If You Rebuild from Scratch
# 1. Re-install local SDK (if .dotnet/ gets corrupted)
./restore.sh

# 2. Full clean build
./build.sh --rebuild

Extension Development (VS Code Extension)
If you're working on the VS Code extension under extension:
cd extension
npm install
npm run build
Then use start-code.sh from the repo root and press F5 to launch the Extension Development Host.
User: how does the Aspire Code generator work and why it needs to trigger the SDK generation when first run with aspire cli?
GitHub Copilot: Read
Read , lines 200 to 900
Read , lines 150 to 1100
Read , lines 1 to 100
Compacted conversation
Read , lines 100 to 235
Read , lines 1 to 200
Read , lines 200 to 250
Searched for regex GenerateCodeAsync|generateCode|codegen-hash|modules (**/src/Aspire.Cli/**/*.cs), 19 results
Read , lines 1 to 300
Read , lines 300 to 500
Read , lines 1010 to 1120
Searched for regex NeedsCodeGen|codegen-hash|ComputePackagesHash|\.modules|GeneratedFolder (**/src/Aspire.Cli/Projects/GuestAppHostProject.cs), 15 results
Searched for regex NeedsCodeGeneration|codegen-hash|PrepareAsync (**/src/Aspire.Cli/Projects/*.cs), 14 results
Read , lines 80 to 200
Read , lines 420 to 480
Searched for regex codegen-hash|ComputePackagesHash|GenerationHash|HashChanged|ShouldRegenerate (**/src/Aspire.Cli/**/*.cs), 5 results
Searched for regex NeedsCodeGen|codegen|hash|modules|\.modules (**/src/Aspire.Cli/Projects/DotNetBasedAppHostServerProject.cs), 20 results
Read , lines 135 to 460
Searched for regex ShouldRegenerateCode|ReadGenerationHash|\.codegen-hash|modules.*exist|Directory\.Exists.*modules (**/src/Aspire.Cli/**/*.cs), 2 results
Searched for text generateCode (**/src/Aspire.Hosting/Ats/*.cs), 1 result
Searched for regex generateCode|GenerateCode|ICodeGenerator (**/src/Aspire.Hosting/**/*.cs), 2 results
Searched for regex generateCode|GenerateCode|ICodeGenerator (**/src/Aspire.Hosting/RemoteHost/*.cs), no results
Read
Searched for files matching **/src/Aspire.Hosting/RemoteHost/*.cs, no matches
Searched for text generateCode (**/src/**/*.cs), 20 results
Read , lines 1 to 200
Searched for text AtsContextFactory (**/src/Aspire.Hosting.RemoteHost/**/*.cs), 13 results
Read
Now I have a complete end-to-end picture. Here's the full explanation:

How the Aspire Code Generator Works
The Core Problem: No Runtime Discovery
Guest languages (TypeScript, Python, etc.) can't use .NET reflection at runtime. Every capability (addRedis, withEnvironment, etc.) that comes from a NuGet package must be represented as statically-typed code in the guest language before user code can call it. That's what SDK generation produces: the .modules/aspire.ts (and base.ts, transport.ts) files that TypeScript user code imports.
The Two-Phase Split: Where Code Lives
Server side (AppHost Server process):
AssemblyLoader → AtsContextFactory → AtsCapabilityScanner → AtsContext
                                                              ↓
                                               CodeGenerationService.generateCode()
                                                              ↓
                                                    ICodeGenerator.GenerateDistributedApplication()
                                                              ↓
                                               Dictionary<string, string>  (filename → content)
AtsContextFactory.cs lazily calls AtsCapabilityScanner.ScanAssemblies() over all loaded assemblies
AtsCapabilityScanner.cs does 5-pass scanning: collect all [AspireExport] methods/types → resolve unknown types → filter invalid ones → expand interface targets to concrete types → filter overload collisions
[CodeGenerationService.cs](src/Aspire.Hosting.RemoteHost/CodeGeneration/CodeGeneration Service.cs) exposes the generateCode JSON-RPC method, which calls the registered ICodeGenerator for the requested language
ICodeGenerator.cs is the contract: GenerateDistributedApplication(AtsContext) → Dictionary<string, string>
Client side (CLI):
GuestAppHostProject.cs orchestrates everything: build the server → start it → call GenerateCodeAsync via RPC → write files to .modules/ → save .codegen-hash
AppHostRpcClient.cs sends {"method": "generateCode", "params": ["TypeScript"]} over the Unix socket
Why It Must Run on First Use
When you run aspire run against a TypeScript AppHost:
CLI scaffolds a hidden .NET project in $TMPDIR/.aspire/hosts/<hash>/ with a Program.cs that is just await RemoteHostServer.RunAsync(args), plus a .csproj referencing every Aspire NuGet package listed in aspire.json
CLI builds it — this is the PrepareAsync step in DotNetBasedAppHostServerProject.cs. On success, it returns NeedsCodeGeneration: true
CLI starts the server — the server binary loads all those Aspire assemblies into the process
CLI calls generateCode via RPC — AtsContextFactory lazily scans all loaded assemblies (including e.g. Aspire.Hosting.Redis.dll) for [AspireExport] attributes, building an AtsContext that describes every capability; then AtsTypeScriptCodeGenerator turns that into aspire.ts, base.ts, transport.ts
CLI writes the files to .modules/ and saves a .codegen-hash (SHA-256 of sorted packageId:version pairs) for future cache invalidation
CLI starts the guest (npx tsx apphost.ts) — the TypeScript code can now import { createBuilder } from './.modules/aspire.js'
The SDK cannot be committed to the repo because it is derived from whatever NuGet packages are installed — if you run aspire add redis, the Redis package is added, and on the next run aspire run detects the hash has changed and regenerates so addRedis() appears in the TypeScript SDK.
The aspire sdk generate Command
SdkGenerateCommand.cs is a stand-alone variant for integration library authors. Usage: aspire sdk generate ./MyIntegration.csproj -l typescript -o ./output
It runs the same pipeline in isolation: create a temp AppHost server project → add your .csproj as an additionalProjectReference → build → connect → call GenerateCodeAsync → write files → delete temp dir. This lets integration authors generate typed SDKs from their integration libraries for publishing to npm/PyPI.
Hash-Based Cache: When Regeneration Is Skipped
The .modules/.codegen-hash file stores the SHA-256 of packageId:version;packageId:version;... (sorted). Currently, PrepareAsync always returns NeedsCodeGeneration: true after a successful build, so generation happens every run. The hash is written after generation and is the foundation for future skip logic when the hash matches.
The 5-Pass Scanner
AtsCapabilityScanner.ScanAssemblies() runs these passes across all assemblies:
Collect — per-assembly scan for [AspireExport] on methods and types, [AspireDto] on DTOs, gathers all enum types from signatures
Resolve Unknown types — any capability parameter that references a type not in the exported set is marked Unknown
Filter invalid — capabilities with unresolved Unknown types are dropped (with diagnostics)
Expand interface targets — withEnvironment<T> where T : IResourceWithEnvironment becomes a capability entry for every concrete type that implements IResourceWithEnvironment; ExpandedTargetTypes on each AtsCapabilityInfo is pre-computed here
Filter overloads — C# allows overloaded methods; ATS capability IDs are unique, so duplicate IDs after expansion emit errors and are removed
The resulting AtsContext (capabilities, handle types, DTOs, enums) is then passed to ICodeGenerator.GenerateDistributedApplication() which templates out the guest-language SDK files.
User: how to tell what assemblies are loaded in the aspire cli?
GitHub Copilot: Searched for text AssemblyLoader (**/src/Aspire.Hosting.RemoteHost/**/*.cs), 17 results
Read
The assembly list is driven by two things:
1. AtsAssemblies in appsettings.json
When the CLI builds the hidden AppHost server project, it writes an appsettings.json into the temp project directory listing every assembly to scan. You can see this in DotNetBasedAppHostServerProject.cs — it always starts with "Aspire.Hosting" and then appends every package name from aspire.json:
{
  "AtsAssemblies": [
    "Aspire.Hosting",
    "Aspire.Hosting.Redis",
    "Aspire.Hosting.CodeGeneration.TypeScript"
  ]
}
AssemblyLoader reads this array at startup and calls Assembly.Load(name) for each — these are simple by-name loads, so the assembly must already be on the probing path (i.e., built into the server's output directory as a dependency).
2. ASPIRE_INTEGRATION_LIBS_PATH env var (prebuilt / bundled mode)
For the prebuilt CLI bundle (no local .NET SDK), PrebuiltAppHostServer restores the NuGet packages to a temp folder and passes that path via ASPIRE_INTEGRATION_LIBS_PATH. The AssemblyLoader registers a custom AssemblyLoadContext.Default.Resolving handler that probes that directory for DLLs when a name-based load fails.
How to see what's actually loaded
Look at the appsettings.json the CLI generates in $TMPDIR/.aspire/hosts/<hash>/appsettings.json — that's the exact list
Enable Debug log level for the server (set "Aspire.Hosting.RemoteHost": "Debug" in the appsettings) to see "Loaded assembly: {AssemblyName}" log lines from AssemblyLoader
Use aspire sdk dump (SdkDumpCommand.cs) which calls the getCapabilities RPC — you can infer what assemblies scanned by what capability IDs appear (e.g., Aspire.Hosting.Redis/addRedis means Aspire.Hosting.Redis was loaded)
User: why would CLI need to communicate with the AppHost server via RPC?
GitHub Copilot: Read
The RPC between CLI and AppHost server exists for a fundamental reason: the CLI is a native AOT-compiled binary that cannot dynamically load arbitrary .NET assemblies, but ALL the language-specific knowledge lives inside NuGet packages that are only present in the server process.
Here's the breakdown of why each RPC call can't be done in the CLI directly:
RPC methodWhy the CLI can't do it locally
getRuntimeSpecRuntimeSpec (how to run npx tsx, nodemon, etc.) is returned by ILanguageSupport implementations that come from NuGet packages (Aspire.Hosting.CodeGeneration.TypeScript). The CLI doesn't reference those packages.scaffoldAppHostScaffold templates (apphost.ts, package.json, etc.) are embedded resources inside those same NuGet packages, accessed via ILanguageSupport.Scaffold().generateCodeCode generation requires AtsCapabilityScanner to reflect over loaded integration assemblies (Aspire.Hosting.Redis.dll, etc.). The CLI can't load those — only the server can, because it was built with them as project/package references.getCapabilitiesSame — needs reflection over NuGet package assemblies to enumerate [AspireExport] attributes.There's also a process separation reason: the AppHost Server is the DCP orchestration runtime (it manages containers, services, the dashboard). The CLI is the user-facing shell. Keeping them in separate processes means the CLI can monitor the server, restart it, and display its exit code without being the same process.
The three-process architecture (CLI ↔ AppHost Server ↔ Guest Runtime) is intentional:
CLI (native AOT, thin orchestrator)
  │── JSON-RPC over Unix socket ──→ AppHost Server (.NET, loads all NuGet packages)
                                         │── JSON-RPC over Unix socket ──→ Guest (TypeScript/Python)
The CLI is evolving toward a "pure orchestrator" that knows nothing about specific languages or integrations — it just asks the server what to do and follows its instructions (the RuntimeSpec).
User: how is the provisioning library being used? and who generates bicep files? who deploy the bicep files?
GitHub Copilot: Searched for regex AzurePublishingContext|BicepCompiler|\.bicep|WriteBicep|GenerateBicep (**/src/**/*.cs), 20 results
Searched for files matching **/src/Aspire.Hosting.Azure/Bicep*.cs, 1 match
Searched for files matching **/src/Aspire.Hosting.Azure/AzureBicepResource.cs, 1 match
Searched for regex AzurePublishingContext|PublishAsync.*azure|DeployAsync|azd|azure developer cli (**/src/**/*.cs), 20 results
Searched for regex AzureProvisioningResource|infrastructure\.Build|GetBicepTemplateString|WriteToManifest (**/src/Aspire.Hosting.Azure/**/*.cs), 20 results
Read , lines 1 to 180
Read , lines 120 to 180
Searched for regex ProvisionAzureBicepResourceAsync|ArmDeployment|DeployBicep|azure-mgmt (**/src/Aspire.Hosting.Azure/**/*.cs), 20 results
Read , lines 150 to 285
Read , lines 1 to 300
Searched for regex AzureProvisioningResource.*GetBicepTemplateString|infrastructure\.Build\(\)|\.Compile\(\)|BicepCompiler (**/src/Aspire.Hosting.Azure/**/*.cs), 12 results
Searched for files matching **/src/Aspire.Hosting.Azure/AzureProvisioningResource.cs, 1 match
Read , lines 1 to 120
Read , lines 1 to 80
Here is the full picture of how provisioning works, from C# code all the way to deployed Azure resources:

The Three-Layer Stack
Layer 1: Azure.Provisioning SDK → Bicep text (C# code in your AppHost)
Azure.Provisioning.* NuGet packages (Azure.Provisioning.Redis, Azure.Provisioning.PostgreSql, etc.) are a C#-first IaC SDK. They let you describe Azure resources in C# using types like CdkRedisResource, RoleAssignment, BicepValue<T>, BicepFunction.Interpolate().
When AzureProvisioningResource.GetBicepTemplateFile() is called:
var infrastructure = new AzureResourceInfrastructure(this, Name);
ConfigureInfrastructure(infrastructure);      // user's callback, e.g., ConfigureRedisInfrastructure
var plan = infrastructure.Build(options);
var compilation = plan.Compile();             // Azure.Provisioning SDK → Bicep text string
File.WriteAllText(moduleSourcePath, compiledBicep.Value);
The ConfigureInfrastructure callback is where each integration (e.g., AzureRedisExtensions.cs) builds up the C# object graph — create a CdkRedisResource, configure AAD auth, add ProvisioningOutput for connection strings, etc. The SDK's Compile() call turns that into Bicep text. No Bicep files are pre-authored; they are all generated at runtime from C#.
Layer 2: Bicep text → ARM JSON (the bicep CLI)
BicepCliCompiler.CompileBicepToArmAsync() in BicepCompiler.cs shells out to bicep build --stdout (or az bicep build as fallback). This converts the generated .module.bicep file into an ARM template JSON string. The bicep binary is either the standalone Bicep CLI or Azure CLI's extension — it must be on PATH.
Layer 3: ARM template → Azure deployment (Azure Resource Manager)
BicepProvisioner.GetOrCreateResourceAsync() in BicepProvisioner.cs submits the ARM JSON to Azure using the Azure Management SDK:
var deployments = resourceGroup.GetArmDeployments();   // Azure.ResourceManager SDK
var operation = await deployments.CreateOrUpdateAsync(
    WaitUntil.Started, deploymentName,
    new ArmDeploymentContent(new(ArmDeploymentMode.Incremental)
    {
        Template = BinaryData.FromString(armTemplateContents),
        Parameters = BinaryData.FromObjectAsJson(parameters),
    }), cancellationToken);
await operation.WaitForCompletionAsync(cancellationToken);
This is a standard ARM API call — it creates a named deployment in the Azure resource group and polls until it completes. Deployment outputs (connection strings, host names, etc.) are read back and stored in resource.Outputs.

Two Different Deployment Paths
Path A: aspire run (local dev provisioning)
When you run locally with Azure resources, the provisioning pipeline runs inside the AppHost process:
AzureBicepResource constructor → adds PipelineStepAnnotation → pipeline step calls ProvisionAzureBicepResourceAsync → BicepProvisioner.GetOrCreateResourceAsync() → ARM deployment.
Deployment state is cached in ~/.aspire/deployments/<hash>.json (or similar) so re-runs skip provisioning if the checksum hasn't changed.
Path B: azd up (Azure Developer CLI deployment)
AzureBicepResource.WriteToManifest() writes to the Aspire manifest (JSON) that azd reads:
{
  "type": "azure.bicep.v0",
  "path": "redis.module.bicep",
  "params": { "location": "{azure.location}" }
}
azd reads this manifest, takes the .bicep files that were written alongside it, and runs its own ARM deployment pipeline. Aspire does not deploy in this path — it just produces the manifest and .bicep files; azd handles the actual az deployment group create.

Summary
WhoWhat
Azure.Provisioning SDKGenerates Bicep text from C# DSL (infrastructure.Build().Compile())BicepCliCompilerShells out to bicep/az bicep to compile Bicep → ARM JSONBicepProvisionerSubmits ARM JSON to Azure via Azure.ResourceManager SDK, polls for completionazdAlternative deployer — reads Aspire manifest + .bicep files, runs its own deploymentUser: how provisioning library communicate with AppHost server?
GitHub Copilot: Searched for regex PipelineStep|PipelineStepAnnotation|ResourceNotificationService|ProvisioningContext (**/src/Aspire.Hosting.Azure/**/*.cs), 20 results
Searched for files matching **/src/Aspire.Hosting/Pipelines/*.cs, 20 matches
Searched for text PipelineStepAnnotation (**/src/Aspire.Hosting/Pipelines/DistributedApplicationPipeline.cs), 1 result
Read , lines 120 to 285
Read , lines 60 to 390
Read , lines 1 to 100
The provisioning library does not communicate with the AppHost server via RPC. They run in the same process. Here's the full picture:

The Provisioning Library Is Part of the AppHost Process
There is no RPC boundary between the provisioning library and the "AppHost server." The Aspire.Hosting.Azure provisioning code is loaded as a regular .NET assembly inside the AppHost process itself. Communication is via in-process .NET calls through the dependency injection container and the Pipeline system.
How It Flows: The Pipeline
AzureBicepResource registers itself into the Pipeline via annotations in its constructor:
AzureBicepResource constructor
  ├── ManifestPublishingCallbackAnnotation → WriteToManifest() (for azd manifest generation)
  ├── PipelineStepAnnotation → creates "provision-{name}" step with Action = ProvisionAzureBicepResourceAsync
  └── PipelineConfigurationAnnotation → wires up dependency ordering between Azure resources
DistributedApplicationPipeline (in Aspire.Hosting.Pipelines) walks all resources in the app model, collects their PipelineStepAnnotations, sorts them topologically, and executes them in order.
When the "provision-redis" step runs, ProvisionAzureBicepResourceAsync resolves services directly from DI:
private static async Task ProvisionAzureBicepResourceAsync(PipelineStepContext context, AzureBicepResource resource)
{
    var bicepProvisioner = context.Services.GetRequiredService<IBicepProvisioner>();  // DI, in-process
    var azureEnvironment = context.Model.Resources.OfType<AzureEnvironmentResource>().First();
    var provisioningContext = await azureEnvironment.ProvisioningContextTask.Task;   // TaskCompletionSource

    await bicepProvisioner.GetOrCreateResourceAsync(resource, provisioningContext, ...);
}
The ProvisioningContext comes from AzureEnvironmentResource.ProvisioningContextTask — another pipeline step ("create-provisioning-context") runs first and completes that TaskCompletionSource with the Azure credentials/subscription/resource group. Individual Bicep steps DependsOn that step, so they wait for it.
Communication Channels (all in-process):
From → ToMechanism
Pipeline executor → BicepProvisionerIServiceProvider.GetRequiredService<IBicepProvisioner>()BicepProvisioner → Dashboard/UIResourceNotificationService.PublishUpdateAsync() (in-memory channel)BicepProvisioner → bicep CLIProcess.Start("bicep build ...") (subprocess, not RPC)BicepProvisioner → Azure ARM APIAzure.ResourceManager SDK over HTTPSOne Bicep resource → another (ordering)TaskCompletionSource on AzureEnvironmentResource.ProvisioningContextTask + PipelineStep.DependsOn()The only external communication provisioning does is:
Shelling out to the bicep or az CLI binary to compile Bicep → ARM JSON
HTTPS to Azure Resource Manager (management.azure.com) to submit the ARM deployment
User: if provisioning library can generate bicep and deploy via bicep , why we still need guest language runtime to communicate with AppHost dotnet server via json rpc to callback capbilities?
GitHub Copilot: They serve completely different phases. The JSON-RPC capability system is how the guest language builds the app model. The provisioning library consumes that model after it's built. One cannot replace the other.
The Sequence
TypeScript apphost.ts (running as Guest)
    │
    │ JSON-RPC: invokeCapability("Aspire.Hosting/createBuilder", {})
    ├──────────────────────────────────────> AppHost Server (.NET)
    │ <── { $handle: "1" }                   IDistributedApplicationBuilder created in memory
    │
    │ JSON-RPC: invokeCapability("Aspire.Hosting.Redis/addRedis", {builder: {$handle:"1"}, name:"cache"})
    ├──────────────────────────────────────> RedisResource added to DistributedApplicationModel
    │ <── { $handle: "2" }
    │
    │ JSON-RPC: invokeCapability("Aspire.Hosting/build", {builder: {$handle:"1"}})
    ├──────────────────────────────────────> DistributedApplication.Build() called
    │
    │ JSON-RPC: invokeCapability("Aspire.Hosting/run", {app: {$handle:"3"}})
    ├──────────────────────────────────────> app.RunAsync() starts
    │                                            │
    │                                            ├── DcpExecutor: spin up containers
    │                                            ├── Pipeline: provision Azure resources
    │                                            │     └── BicepProvisioner → bicep CLI → ARM API
    │                                            └── Dashboard: show resource states
The guest language's JSON-RPC calls are the equivalent of Program.cs in a .NET AppHost. They are what populate the DistributedApplicationModel with resources. Without them, the model is empty — there is nothing for the provisioning library or DCP executor to act on.
The Analogy
Think of it this way:
A .NET AppHost — the user writes builder.AddRedis("cache") in C#, which directly calls the in-process AddRedis() extension method, adding a RedisResource to the model
A TypeScript AppHost — the user writes await builder.addRedis("cache") in TypeScript, which sends a JSON-RPC call to the .NET server, which runs the same AddRedis() method in-process on behalf of the guest
In both cases the outcome is identical: a RedisResource lands in the DistributedApplicationModel. The provisioning pipeline doesn't know or care which language authored the model.
Why Not Generate Bicep From the Guest Directly?
The provisioning library (Bicep generation, Azure SDK, ARM deployment) has deep dependencies on:
Azure.Provisioning.* NuGet packages with .NET-only types (ProvisionableResource, BicepValue<T>, etc.)
Azure.ResourceManager SDK for ARM API calls
System.Reflection for scanning [AspireContainerImage], role assignments, etc.
A TypeScript process cannot run any of that. It can only invoke capabilities on the .NET server, which then does the actual work.
Summary: The JSON-RPC capabilities are the authoring API — the guest builds the resource graph. The provisioning library is the execution engine — it acts on that graph. They are sequential, not competing.
User: how does the guest build the resource graph via AppHost json rpc? what input does the provisioning library need in order to generate bicep files to really create the resources?
GitHub Copilot: Read , lines 1 to 100
Read , lines 1 to 120
Read , lines 120 to 300
Read , lines 100 to 260
Read , lines 200 to 300
Searched for regex RemoteAppHostService|invokeCapability|InvokeCapability (**/src/Aspire.Hosting.RemoteHost/**/*.cs), 15 results
Read , lines 140 to 380
Read
Searched for regex AzureResourcePreparer|ConfigureInfrastructure|DistributedApplicationModel.*azure (**/src/Aspire.Hosting.Azure/**/*.cs), 20 results
Read , lines 1 to 120
Now I have the full picture. Here is the end-to-end answer:

Phase 1: The Guest Builds the Resource Graph via JSON-RPC
The key insight is that JSON-RPC calls are direct reflection-based invocations of real .NET methods. There is no separate protocol translation layer — the dispatcher just calls the actual C# method.
The invocation chain
TypeScript: await builder.addRedis("cache")
    │
    │  JSON-RPC: invokeCapability("Aspire.Hosting.Redis/addRedis", {"builder": {$handle:"1"}, "name":"cache"})
    ▼
RemoteAppHostService.InvokeCapabilityAsync()
    └── CapabilityDispatcher.InvokeAsync("Aspire.Hosting.Redis/addRedis", args)
            │
            │  RegisterFromCapability() bound this capability ID to the real MethodInfo
            │  of AddRedis() at startup (via AtsCapabilityScanner reflection scan)
            │
            ├─ Unmarshal args:
            │     "builder" → HandleRegistry.TryGet("1") → returns the actual IDistributedApplicationBuilder object
            │     "name"    → JsonNode → string "cache"
            │
            ├─ Invoke via reflection:
            │     method.Invoke(null, [builder, "cache"])
            │     == RedisExtensions.AddRedis(builder, "cache")   ← the real C# extension method
            │
            └─ Marshal result:
                  IResourceBuilder<RedisResource> → HandleRegistry.Register(obj) → handle "2"
                  Return: {"$handle": "2", "$type": "Aspire.Hosting.Redis/...RedisResource"}
The HandleRegistry is a per-client ConcurrentDictionary<string, object> that stores live .NET objects by ID. When TypeScript sends {$handle:"1"} back in the next call, the dispatcher resolves it to the actual IDistributedApplicationBuilder instance.
The result: after the TypeScript guest calls createBuilder → addRedis → build → run, the AppHost process has a fully populated DistributedApplicationModel with a real RedisResource in its Resources collection — identical to what a Program.cs in a .NET AppHost would produce.

Phase 2: What the Provisioning Library Reads From the Model
AzureResourcePreparer.GetAzureResourcesFromAppModel() scans the populated model and finds each IAzureResource. For a Redis resource added by AddRedis().AsAzureManagedRedis(), the model object is an AzureManagedRedisResource which is an AzureProvisioningResource (which is an AzureBicepResource).
The provisioning library needs two things from each AzureProvisioningResource:
1. ConfigureInfrastructure callback — a Action<AzureResourceInfrastructure> set at resource creation time. When GetBicepTemplateFile() is called, it runs the callback:
ConfigureInfrastructure(infrastructure);   // → adds CdkRedisResource, ProvisioningOutput, etc.
var plan = infrastructure.Build();
var compilation = plan.Compile();          // → Bicep text string
This callback was closed over C# objects when AddRedis() ran — it carries the resource configuration directly.
2. Parameters dictionary — key/value pairs that become Bicep param declarations. These include:
location (Azure region)
principalId / principalType for managed identity role assignments
References to other resources' outputs (e.g., keyVaultUri)
These are populated by BicepUtilities.SetParametersAsync() at deploy time, resolving IManifestExpressionProvider references, ParameterResource values (from user prompts), and BicepOutputReference cross-resource references.

The Complete Picture
TypeScript apphost.ts
    │ JSON-RPC: invokeCapability("Aspire.Hosting.Redis/addRedis", ...)
    ▼
CapabilityDispatcher
    │ Reflection: RedisExtensions.AddRedis(builder, "cache", port: null)
    │             ↳ new AzureManagedRedisResource("cache", configureInfra)
    │             ↳ builder.Resources.Add(redisResource)
    ▼
DistributedApplicationModel.Resources
    └── AzureManagedRedisResource
            ├── ConfigureInfrastructure: (infra) => {
            │       var redis = new CdkRedisResource(...);
            │       infra.Add(redis);
            │       infra.Add(new ProvisioningOutput("connectionString", ...));
            │   }
            ├── Parameters: { "location": "{azure.location}", ... }
            └── PipelineStepAnnotation: → ProvisionAzureBicepResourceAsync

                            ↓ (when app.Run() is called)

AzureResourcePreparer.OnBeforeStartAsync()
    └── Scans model, builds role assignment annotations

Pipeline: "provision-redis" step
    └── BicepProvisioner.GetOrCreateResourceAsync()
            ├── resource.GetBicepTemplateFile()
            │       └── ConfigureInfrastructure(infra) → plan.Compile() → redis.module.bicep
            ├── BicepCliCompiler: "bicep build redis.module.bicep" → ARM JSON
            └── resourceGroup.GetArmDeployments().CreateOrUpdateAsync(armJson)
                    → Azure REST API → Redis Cache created in Azure
                    → Outputs (connectionString) stored in resource.Outputs["connectionString"]
User:  introduce more about DistributedApplicationModel and the architecture overview?
GitHub Copilot: Compacted conversation
Searched codebase for "DistributedApplicationModel class definition resources annotations", 24 results
Searched for files matching **/DistributedApplicationModel.cs, 1 match
Searched for files matching **/IResourceCollection.cs, 1 match
Read
Read , lines 1 to 150
Read , lines 150 to 500
Searched for regex class.*Resource.*IResource|ContainerResource|ProjectResource|ExecutableResource (**/src/Aspire.Hosting/ApplicationModel/*.cs), 20 results
Read
Read , lines 500 to 700
Read , lines 1 to 80
Read
Here's a comprehensive overview:

DistributedApplicationModel — The Application Graph
DistributedApplicationModel is the central data structure of an Aspire AppHost. It is a trivially simple object with one purpose: holding the flat list of all resources.
// src/Aspire.Hosting/ApplicationModel/DistributedApplicationModel.cs
[DebuggerDisplay("Resources = {Resources.Count}")]
public class DistributedApplicationModel(IResourceCollection resources)
{
    public IResourceCollection Resources { get; } = resources;
}
IResourceCollection is just IList<IResource> — a flat list, not a tree. The graph structure (the DAG of dependencies) is implicit, encoded in annotations and ReferenceExpression objects attached to resources. Aspire never stores an explicit edge list; edges are discovered at runtime by traversing annotations.
It's registered as a singleton in DI. Everything — DcpExecutor, AzureResourcePreparer, BicepProvisioner, manifest writers, the dashboard — gets it from DI and queries model.Resources.

IResource — The Fundamental Unit
public interface IResource
{
    string Name { get; }
    ResourceAnnotationCollection Annotations { get; }
}
That's the entire interface. Two properties. Resources are pure inert data objects — they don't start, stop, or manage themselves. Everything about their runtime behavior is carried in Annotations.
The abstract Resource base class just validates the name and provides the collection:
IResource
  └── Resource (abstract)
        ├── ContainerResource        — runs a Docker image
        ├── ProjectResource          — runs a .NET project
        ├── ExecutableResource       — runs an arbitrary binary
        ├── ParameterResource        — an external input / secret
        ├── ConnectionStringResource — a literal conn string
        └── [integration-specific]
              ├── RedisResource       : ContainerResource
              ├── PostgresServerResource : ContainerResource
              ├── AzureProvisioningResource : AzureBicepResource
              └── ...
The three primitive compute types are what DCP (DcpExecutor) knows about natively. Everything else is modeled on top of them.

Annotations — The Extensibility Mechanism
IResourceAnnotation is an empty marker interface. Any POCO implementing it can be attached to any resource:
// All of these are annotations
ContainerImageAnnotation        // → image name, tag, registry
EndpointAnnotation              // → port, scheme, name
EnvironmentCallbackAnnotation   // → lambda to inject env vars
ContainerMountAnnotation        // → volume or bind mount
HealthCheckAnnotation           // → health probe config
PipelineStepAnnotation          // → deploy pipeline step
ManifestPublishingCallbackAnnotation // → how to write to azd manifest
WaitAnnotation                  // → WaitFor dependency
Extension methods like AddRedis("cache") are really just:
new RedisResource("cache") (a ContainerResource subclass)
resource.Annotations.Add(new ContainerImageAnnotation { Image = "redis", Tag = "7.4" })
resource.Annotations.Add(new EndpointAnnotation(ProtocolType.Tcp, ..., targetPort: 6379))
Wrap in an IResourceBuilder<RedisResource> and return
The IResourceBuilder<T> is just a fluent wrapper:
IDistributedApplicationBuilder     (top-level, owns model + DI)
  └── .AddRedis("cache")
        └── IResourceBuilder<RedisResource>
              ├── .Resource          → the RedisResource instance
              ├── .WithEnvironment() → adds EnvironmentCallbackAnnotation
              ├── .WithDataVolume()  → adds ContainerMountAnnotation
              └── .WithHealthCheck() → adds HealthCheckAnnotation

Standard Capability Interfaces
Resources opt into behaviors by implementing interfaces (polymorphism over type checks):
InterfaceWhat it enables
IResourceWithEnvironment.WithEnvironment(), env var injectionIResourceWithEndpoints.WithEndpoint(), GetEndpoint(), URL generationIResourceWithConnectionString.WithReference(db) wires conn string into consumersIResourceWithServiceDiscoveryRegisters service in DNS-style discoveryIResourceWithArgsCLI argument injectionIResourceWithWaitSupportWaitFor() — blocks startup until dependency is readyIResourceWithParentLifecycle containment (child stops when parent stops)IComputeResourceMarks as a compute unit (project/container/exec)ContainerResource implements nearly all of them. ParameterResource implements almost none (it's just a value, not a runnable thing).

The Value/Reference System — How the DAG Forms
The graph is a heterogeneous DAG encoding both dependency order and value flows. Edges are encoded as structured value objects:
web ──► EndpointReference ──► api
                               │
                               └──► ConnectionStringReference ──► postgres
Key types:
TypeRun modePublish mode
EndpointReferencehttp://localhost:5000{api.bindings.http.url}ConnectionStringReferenceHost=localhost;Port=5432;...{postgres.connectionString}ParameterResourcevalue from user secrets / env${PARAM} placeholderReferenceExpressioninterpolated concrete stringinterpolated manifest expressionquot;{redis.HostName},ssl=true")
    });
    infrastructure.Add(new ProvisioningOutput("name", typeof(string))
    {
        Value = redis.Name.ToBicepExpression()
    });

    // 4. Optionally store secret in Key Vault (for access-key auth path)
    var secret = new KeyVaultSecret("connectionString")
    {
        Parent = keyVault,
        Properties = new SecretProperties
        {
            Value = BicepFunction.Interpolate(
AreaTechnologyPurpose
Local orchestrationDCP (internal), DockerStarts containers/processes during devApp modelC# resource graph, annotationsDeclares distributed app topologyAzure cloud resourcesBicep + Azure.Provisioning SDKInfrastructure-as-code for AzureBicep provisioningAzure ARM APIDeploys Azure resources during aspire runPublish artifactsBicep files + manifest JSONOutput for azd, K8s, etc.Polyglot app hostsATS + JSON-RPCLets TypeScript/Python/etc. write app hostsLanguage SDKsCode generation (ATS → TS/Py/Go/Rust/Java)Typed wrappers for each guest languageUser: Can you generate a comprehensive overview of the code architect of this repo
GitHub Copilot: Read
Read
Read
Read
Read
Read
Read
Read
Here is a comprehensive architectural overview of the Aspire repository:

dotnet/aspire — Code Architecture Overview

1. Repository Layout
aspire/
├── src/                    # All production source code
├── tests/                  # Unit & integration tests (mirrors src/)
├── playground/             # Sample apps for manual verification
├── extension/              # VS Code extension (TypeScript)
├── docs/specs/             # Architecture specs & design documents
├── eng/                    # Build infrastructure (Arcade SDK, pipelines)
└── tools/                  # Developer tooling (QuarantineTools, etc.)

2. Major Subsystems
┌──────────────────────────────────────────────────────────────────────────┐
│                          User-Facing Entry Points                        │
│                                                                          │
│  aspire CLI (Aspire.Cli)   VS Code Extension  App Host (C# Program.cs)  │
└─────────────┬──────────────────────┬─────────────────────┬──────────────┘
              │                      │                      │
              ▼                      ▼                      ▼
┌─────────────────────┐  ┌──────────────────┐  ┌──────────────────────────┐
│  CLI Orchestration  │  │  VS Code Ext.    │  │   App Host SDK           │
│  (run/publish/add)  │  │  (debug, attach) │  │   (Aspire.Hosting)       │
└──────────┬──────────┘  └──────────────────┘  └──────────┬───────────────┘
           │                                               │
           │                               ┌──────────────▼───────────────┐
           │                               │       Application Model      │
           │                               │  IResource, Annotations, DAG │
           │                               └──────────────┬───────────────┘
           │                                              │
           │              ┌───────────────────────────────┼────────────────┐
           │              │               │               │                │
           ▼              ▼               ▼               ▼                ▼
    ┌──────────┐  ┌──────────────┐ ┌──────────┐ ┌──────────────┐ ┌──────────────┐
    │  AzDev   │  │DCP Executor  │ │Dashboard │ │  Publishing  │ │   Azure      │
    │  CLI     │  │(local run)   │ │(Blazor)  │ │  Pipeline    │ │ Provisioning │
    └──────────┘  └──────────────┘ └──────────┘ └──────────────┘ └──────────────┘

3. Core Layer: Application Model (ApplicationModel)
The app model is the central graph that everything else reads and transforms.
Key types:
TypeRole
IResource / ResourceBase of all resources (projects, containers, databases, cloud services)IResourceAnnotationMetadata attached to resources. Over 70 annotation types exist.ResourceAnnotationCollectionPer-resource bag of annotationsDistributedApplicationModelThe complete resource graph (IEnumerable<IResource>)IResourceBuilder<T>Fluent builder wrapping a resource; extension methods extend thisEndpointAnnotation / EndpointReferenceModel network endpoints and references between themReferenceExpressionLazy string expression resolving at runtime (e.g., connection strings)IValueProviderAsync-evaluated string values used in env vars, paramsKey interfaces used on resources:
InterfaceMeaning
IResourceWithConnectionStringHas a connection string expressionIResourceWithEndpointsExposes network endpointsIResourceWithEnvironmentAccepts environment variable injectionIResourceWithParametersTakes typed parameters (e.g., Bicep params)IResourceWithServiceDiscoveryParticipates in service discoveryIAzureResourceIs an Azure cloud resource
4. Orchestration Layer: DCP Executor (Dcp)
DCP (Developer Control Plane) is Aspire's local execution engine. It is a separate binary (shipped with the Aspire workload) that presents a Kubernetes-style API.
App Host Process
  └─ DcpHost (starts DCP subprocess)
      └─ DcpExecutor
          ├─ Translates IResource graph → DCP Kubernetes objects
          │   (Container, Executable, Service CRDs)
          ├─ DcpKubernetesClient → HTTP to DCP's local K8s API
          ├─ Watches resource state changes → ResourceNotificationService
          └─ Streams logs → ResourceLoggerService → Dashboard
DcpExecutor is the main orchestrator: it walks the resource graph and creates DCP custom resources via KubernetesService.
Port allocation, container pulls, environment variable injection, endpoint allocation all happen here.
Resource state changes flow back via watch loops to ResourceNotificationService, which distributes events to the Dashboard and other subscribers.

5. Dashboard (Aspire.Dashboard)
A Blazor Server web app for monitoring running resources.
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
The Dashboard subscribes to ResourceNotificationService via gRPC streaming from the App Host and renders live resource state. It also receives OTLP telemetry directly from instrumented services.

6. Publishing Pipeline (Pipelines + Publishing/)
Publishing translates the app model into deployment artifacts. It runs when aspire publish or azd invokes the app host in publish mode.
DistributedApplicationPipeline
  ├─ PipelineStep (ordered, dependency-aware)
  │   ├─ "provision-redis"        (AzureBicepResource step)
  │   ├─ "provision-keyvault"     (AzureBicepResource step)
  │   └─ "write-manifest"         (ManifestPublisher step)
  └─ PipelineExecutor (runs steps in topological order)

ManifestPublisher
  └─ Calls ManifestPublishingCallbackAnnotation per resource
      └─ Each resource writes its JSON block to aspire-manifest.json
The manifest (aspire-manifest.json) is the intermediate format consumed by azd, Kubernetes publishers, or custom tooling.

7. Azure Integration & Bicep (Aspire.Hosting.Azure)
Azure resources follow a two-layer design:
C# Integration API (e.g., AddAzureRedis, AddAzureKeyVault)
   │
   ▼
AzureProvisioningResource  ─────────────────────────────────┐
   │ (uses Azure.Provisioning SDK to declare infra in C#)   │
   │                                                        │
   ▼                                                        │
GetBicepTemplateString()                                    │
   │ → infrastructure.Build().Compile()                     │
   │ → Emits .bicep text                                    │
   │                                                        │
   ▼                ─────────────────── Publish mode ───────┘
AzurePublishingContext
   │→ Writes per-resource .bicep files to output dir
   │→ Writes main.bicep orchestrator
   │→ Writes azd-compatible manifest
   │
   ▼
BicepProvisioner (run mode)
   │→ BicepCliCompiler: calls `az bicep build` → ARM JSON
   │→ Azure ARM SDK: ArmDeploymentContent → deploys to Azure
   └→ Reads outputs back → injects into resource connection strings
Two Bicep paths:
AzureProvisioningResource – C# code (Azure.Provisioning SDK) generates Bicep at runtime. Used by all built-in Azure integrations.
AzureBicepResource – Points to a hand-authored .bicep file. For custom or advanced scenarios (builder.AddBicepTemplate(...)).
30+ Azure integrations each in their own project (src/Aspire.Hosting.Azure.*/), all building on AzureProvisioningResource.

8. Client Integration Components (Components)
These are the application-side packages — consumed by the microservices themselves (not the App Host). They handle:
Dependency injection registration (e.g., builder.AddRedisClient())
Health checks (standard .NET health checks wired to infrastructure)
Resilience (Polly pipelines for retry, circuit breaker)
Telemetry (OpenTelemetry tracing automatically configured)
Configuration binding from appsettings.json / environment
Examples: Aspire.StackExchange.Redis, Aspire.Npgsql, Aspire.Azure.Storage.Blobs, etc. (38 packages total).

9. Aspire CLI (Aspire.Cli)
A native AOT .NET CLI with these top-level commands:
CommandPurpose
aspire runLaunch app host + DCP + Dashboardaspire publishGenerate deployment artifacts (Bicep, K8s YAML)aspire addAdd NuGet integration packages to a projectaspire newScaffold from templatesaspire deployDeploy to Azure (via azd)aspire execRun a command inside the Aspire environmentaspire ps / logs / restartResource lifecycle managementaspire telemetryView OTLP traces/spans/logs from CLIaspire mcpStart MCP server for AI tool integrationaspire docsSearch documentationThe CLI communicates with the running App Host via a backchannel (gRPC/JSON-RPC) defined in Backchannel and Backchannel.

10. Polyglot AppHost — ATS System (Ats)
The newest major architectural addition: allows TypeScript, Python, Go, Rust, Java to write app hosts.
              Guest Language (e.g., TypeScript)
                     app-host.ts
              builder.addRedis("cache")          ← from generated SDK
                     │
                     │ JSON-RPC over stdio/socket
                     ▼
              .NET AppHost Server (Aspire.Hosting)
                     │
                     │ Dispatches via CapabilityDispatcher
                     ▼
              Actual C# extension method (AddRedis)
                     │
                     ▼
              Shared Application Model (same DCP, Dashboard, Bicep)
ATS (Aspire Type System):
[AspireExport("addRedis")]          ← attribute on C# method
    │
    ▼
AtsCapabilityScanner                ← reflects assemblies at build time
    │  → discovers capabilities, handle types, DTOs, enums
    ▼
AtsContext                          ← flat, portable type model
    │
    ▼
ICodeGenerator implementations:
  ├─ AtsTypeScriptCodeGenerator     → typed TS SDK classes
  ├─ Go / Python / Rust / Java generators
  └─ (one per src/Aspire.Hosting.CodeGeneration.*/ project)
Type categories in ATS:
CategoryAttributeBehavior
Handle[AspireExport] on typeOpaque reference; method calls RPC back to .NETDTO[AspireDto]Serialized as JSON, passed by valueEnum(auto-discovered)Serialized as stringPrimitive(auto-discovered)string, number, boolean, null
11. VS Code Extension (extension)
TypeScript extension providing developer experience inside VS Code:
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
    aspire-settings.schema.json ← Settings schema (currently open file)
The extension communicates with the Dashboard's REST API (Api) to show resource state, logs, and traces directly in the editor.

12. Full Dependency / Data Flow Diagram
Developer writes C# App Host
         │
         ▼
  DistributedApplicationBuilder
         │ AddRedis / AddProject / AddAzureKeyVault / ...
         ▼
  Application Model (IResource DAG with Annotations)
         │
    ─────┼─────────────────────────────────────────
    │                    │                    │
    ▼                    ▼                    ▼
DCP Executor       Dashboard (Blazor)   Publishing Pipeline
(local run)        (monitoring)         (aspire publish)
    │                    │                    │
    │              OTLP telemetry       ┌─────┴──────┐
    │              from services        │            │
    ▼                                   ▼            ▼
Containers            ┌──────────────┐  Bicep     manifest.json
Processes     ──────► │ Resource     │  files     (for azd/K8s)
Docker/Podman         │ State Events │
    │                 └──────────────┘
    ▼
Service Discovery
(OTEL, env vars, connection strings)

13. Key Architectural Principles
Annotation-driven extensibility — All resource metadata is injected as annotations. Third-party integrations extend the model without forking core types.
Context-aware behavior — DistributedApplicationExecutionContext tells every resource whether it is in Run, Publish, or Test mode, allowing the same app model to behave differently per context.
Lazy value resolution — IValueProvider, ReferenceExpression, and EndpointReference are evaluated asynchronously at startup, allowing circular-ish references and deferred config.
Pipeline as first-class concept — Provisioning, deployment, and publishing are steps in a dependency-ordered DistributedApplicationPipeline, not ad-hoc lifecycle hooks.
No IDL for polyglot — ATS uses C# reflection as the schema source; [AspireExport] attributes are the contract, making polyglot SDKs a pure output of the existing C# type system.
Separation of hosting vs. client — src/Aspire.Hosting.* is the app host (orchestration); src/Components/Aspire.* is the client library (consumed by individual microservices). They are independent NuGet packages.
User: 1. add more details about DCP executor and what resource graph information it needed? And how KubernetesService is related?
2. Add more details about C# integration API in Azure Integration & Bicep section  and what is Azure.Provisioning SDK?
GitHub Copilot: Read , lines 1 to 80
Read
Read , lines 1 to 120
Read , lines 1 to 80
Read , lines 1 to 80
Read , lines 1 to 80
Searched for regex CreateContainerAsync\|CreateExecutableAsync\|CreateServiceAsync\|CreateAsync.*Container\|CreateAsync.*Executable (**/src/Aspire.Hosting/Dcp/DcpExecutor.cs), no results
Read
Searched for regex _kubernetesService\.CreateAsync|kubernetesService\.CreateAsync (**/src/Aspire.Hosting/Dcp/DcpExecutor.cs), 4 results
Read , lines 1 to 80
Read , lines 2210 to 2300
Read , lines 80 to 200
Searched for text ConfigureRedisInfrastructure (**/src/Aspire.Hosting.Azure.Redis/AzureRedisExtensions.cs), 2 results
Read , lines 1 to 120
Read , lines 270 to 333
Read , lines 300 to 450
Read , lines 1 to 80
Searched for regex ContainerImageAnnotation|EndpointAnnotation|WaitAnnotation|ProjectResource|ExecutableResource (**/src/Aspire.Hosting/Dcp/DcpExecutor.cs), 20 results
Now I have all the details needed. Here are the two expanded sections:

4. Orchestration Layer: DCP Executor (Dcp) — Expanded
What is DCP?
DCP (Developer Control Plane) is a separate native binary shipped as part of the Aspire workload. It runs on the same machine as the app host and exposes a Kubernetes-style CRD API over a local Unix socket / named pipe. From DCP's perspective, resources are Kubernetes custom objects (CRDs), not containers or processes directly — DCP abstracts the actual container runtime (Docker, Podman) and process launcher behind that API surface.
What resource graph information does DcpExecutor consume?
DcpExecutor walks DistributedApplicationModel and reads the following annotations from each IResource:
Annotation / Resource TypeDCP CRD producedKey fields mapped
ContainerImageAnnotationContainer (CRD)image, ports, env, volumeMounts, command, args, restartPolicy, networks, persistent, start/stopDockerfileBuildAnnotationContainer.Spec.BuildDockerfile path, context, build argsEndpointAnnotationService (CRD)address, port, protocol, addressAllocationModeProjectResource / ExecutableResourceExecutable (CRD)executablePath, workingDirectory, args, env, healthProbes, ambientEnvironment, pemCertificatesContainerMountAnnotationContainer.Spec.VolumeMountsSource, target, type (bind/volume)EnvironmentAnnotation / EnvironmentCallbackAnnotationContainer.Spec.Env / Executable.Spec.EnvResolved key/value EnvVar listHealthCheckAnnotationExecutable.Spec.HealthProbesHTTP/TCP probe configWaitAnnotationOrdering between Service objectsWhich services must be ready before this resource startsProbeAnnotationContainer.Spec.HealthProbesLiveness/readiness probesThe AppResource bridge type
AppResource (in AppResource.cs) is the coupling object between the Aspire app model and the DCP model. It is never sent over the wire — it lives only in the executor's _appResources list.
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
One RenderedModelResource is created per runnable Aspire resource. One ServiceWithModelResource is created per EndpointAnnotation. The ServicesConsumed list drives startup ordering (wait conditions).
How KubernetesService fits in
DcpExecutor                               DCP Binary (local process)
    │                                           │
    │  ─── HTTP over unix socket ──────────►   │
    │                                           │
    │  CreateAsync<Container>(spec)         K8s-style CRD API
    │  CreateAsync<Executable>(spec)        (list, get, create, patch, delete, watch)
    │  CreateAsync<Service>(spec)               │
    │  WatchAsync<Container>()    ◄──────────── │ (long-poll watch stream)
    │  WatchAsync<Executable>()   ◄──────────── │
    │  GetLogStreamAsync(...)     ◄──────────── │ (stdout/stderr as HTTP stream)
    │
    ├─ kubernetesService.CreateAsync(dcpContainerResource)   // line 2267
    ├─ kubernetesService.CreateAsync(exe)                   // line 1841
    └─ kubernetesService.PatchAsync(...)                    // resource lifecycle updates
IKubernetesService (implemented by KubernetesService) is the only channel between the executor and DCP. It uses the official k8s .NET client library (the same used for real Kubernetes), pointed at DCP's local endpoint. Every DCP CRD type (Container, Executable, Service, Endpoint, ContainerExec, ContainerNetwork) implements IKubernetesStaticMetadata so the generic CreateAsync<T> / WatchAsync<T> API works uniformly.
State change propagation after creation
After creation, DcpExecutor runs three concurrent watch loops:
WatchKubernetesResourceAsync<Executable>(...)
WatchKubernetesResourceAsync<Container>(...)     →  ProcessResourceChange()
WatchKubernetesResourceAsync<Service>(...)               │
WatchKubernetesResourceAsync<Endpoint>(...)              ▼
                                               ResourceSnapshotBuilder.ToSnapshot()
                                                         │
                                                         ▼
                                               ResourceNotificationService.PublishUpdateAsync()
                                                         │
                                                         ▼
                                               Dashboard (via gRPC stream) + CLI backchannel
ResourceNotificationService is an in-process pub/sub bus. The Dashboard subscribes to it and re-renders resource state in real time.

7. Azure Integration & Bicep — Expanded
The C# Integration API
Every Azure resource type has a dedicated project (src/Aspire.Hosting.Azure.*/) that exposes a fluent C# API for declaring that Azure resource. The pattern is consistent across all 30+ integrations:
// 1. Declare the Azure resource (always returns a typed IResourceBuilder)
var cache = builder.AddAzureManagedRedis("cache");

// 2. Optionally swap in a local Docker container for dev mode
//    (context-aware: does nothing in publish mode)
var cache = builder.AddAzureManagedRedis("cache")
                   .RunAsContainer();

// 3. Configure auth, role assignments, networking
var cache = builder.AddAzureManagedRedis("cache")
                   .WithAccessKeyAuthentication();

// 4. Wire it to services like any other resource
builder.AddProject<Projects.Api>()
       .WithReference(cache);
The triple context-awareness pattern is central to all Azure integrations:
ContextBehavior
Run mode (local aspire run)RunAsContainer() — removes Azure resource, adds local Docker container insteadPublish mode (aspire publish)Emits Bicep + manifest, no containers startedAzure provisioning (run with azure provisioner)Actually deploys to Azure via ARMWhat is the Azure.Provisioning SDK?
Azure.Provisioning.* (NuGet packages by the Azure SDK team, e.g. Azure.Provisioning.Redis, Azure.Provisioning.KeyVault) is a C#-first infrastructure-as-code SDK. It lets you declare Azure infrastructure as typed C# objects that compile to Bicep text. Think of it as the Azure equivalent of CDK or Pulumi's native SDK.
Key types in the Azure.Provisioning SDK:
SDK TypeRole
ProvisionableResourceBase class for all Azure resources (e.g. RedisResource, StorageAccount)ProvisioningParameterA Bicep param declarationProvisioningOutputA Bicep output declarationBicepValue<T>A typed Bicep expression (can be a literal, a reference, or a function call)BicepFunction.Interpolate(...)Produces Bicep string interpolation ('${...}')Infrastructure.Build().Compile()Runs the entire graph and emits Bicep file contentResourceX.FromExisting(...)References an already-deployed Azure resource by nameExample — what ConfigureRedisInfrastructure does:
// From AzureRedisExtensions.cs
private static void ConfigureRedisInfrastructure(AzureResourceInfrastructure infrastructure)
{
    // 1. Declare an Azure Redis resource (Azure.Provisioning.Redis.RedisResource)
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

    // 4. Optionally store secret in Key Vault (for access-key auth path)
    var secret = new KeyVaultSecret("connectionString")
    {
        Parent = keyVault,
        Properties = new SecretProperties
        {
            Value = BicepFunction.Interpolate($"{redis.HostName},ssl=true,password={redis.GetKeys().PrimaryKey}")
        }
    };
    infrastructure.Add(secret);
}
When AzureProvisioningResource.GetBicepTemplateString() is called, it:
Creates a fresh AzureResourceInfrastructure (wrapper around Azure.Provisioning Infrastructure)
Calls ConfigureInfrastructure(infrastructure) — the callback above runs
Calls infrastructure.Build(ProvisioningBuildOptions) — Azure.Provisioning SDK computes the full Bicep module
Calls .Compile() — emits the Bicep source text as a string
Full Azure resource lifecycle mapping
C# API call (AddAzureManagedRedis)
    │
    ├─ Creates AzureManagedRedisResource (: AzureProvisioningResource)
    │   └─ Stores configureInfrastructure callback
    │
    ├─ Attaches ManifestPublishingCallbackAnnotation
    │   └─ WriteToManifest() → writes "type: azure.bicep.v0" + path to manifest.json
    │
    ├─ Attaches PipelineStepAnnotation
    │   └─ provision-{name} step → calls BicepProvisioner.GetOrCreateResourceAsync()
    │
    └─ Attaches PipelineConfigurationAnnotation
        └─ Forces Bicep template evaluation → discovers cross-resource dependencies

At run/publish time:
    GetBicepTemplateString()
        → new AzureResourceInfrastructure(this, Name)
        → ConfigureInfrastructure(infra)         ← Azure.Provisioning objects added here
        → infra.Build(options).Compile()          ← Azure.Provisioning emits Bicep text
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
The key insight: Azure.Provisioning SDK is the bridge between Aspire's C# resource model and Bicep text. Integration authors never write .bicep files by hand — they compose Azure.Provisioning typed objects in the configureInfrastructure callback, and the SDK generates correct, valid Bicep automatically.
User: How AppHost interact with guest runtime and language runtime? How to integrate it with another language like TypeScript runtime or TypeScript AppHost?
GitHub Copilot: Read , lines 100 to 300
Read
Read , lines 300 to 600
Read
Read , lines 600 to 900
Read
Read , lines 1 to 120
Read , lines 1 to 100
Read , lines 1 to 120
Read , lines 120 to 181
Here's a detailed explanation of how the AppHost interacts with guest runtimes, using TypeScript as the concrete example:

AppHost ↔ Guest Runtime Interaction
The Big Picture
The key design principle: all 40+ integrations stay in .NET. Guest language code never reimplements them. Instead, guest code calls into the .NET server via JSON-RPC and gets back opaque handles to .NET objects, which it passes forward in future calls.

1. Process Architecture
There are always three separate processes at runtime:
┌──────────────────────┐
│     Aspire CLI       │  ← orchestrator: starts both, passes socket path
└───────┬──────────────┘
        │ spawns                      spawns │
        ▼                                    ▼
┌──────────────────────┐       ┌─────────────────────────────┐
│  AppHost Server      │       │  Guest Process               │
│  (.NET)              │◄─────►│  (e.g., `npx tsx apphost.ts`)│
│                      │ RPC   │                              │
│  - ILanguageSupport  │       │  - Generated SDK             │
│  - ICodeGenerator    │       │  - User's apphost.ts         │
│  - CapabilityDispatch│       │  - ATS Client (transport.ts) │
│  - App Model & DCP   │       └─────────────────────────────┘
└──────────────────────┘
        ▲ JSON-RPC over Unix socket
        (path passed via REMOTE_APP_HOST_SOCKET_PATH env var)

2. Startup Sequence Step by Step
aspire run (in TypeScript apphost dir)
      │
      ▼
[1] CLI detects language
      │  Calls server RPC: detect(directoryPath)
      │  TypeScriptLanguageSupport.Detect() checks:
      │    - apphost.ts exists?
      │    - package.json exists?
      │  Returns: language = "typescript/nodejs"
      │
      ▼
[2] CLI requests RuntimeSpec from server
      │  Calls server RPC: getRuntimeSpec("typescript/nodejs")
      │  TypeScriptLanguageSupport.GetRuntimeSpec() returns:
      │    Execute:  { command: "npx", args: ["tsx", "{appHostFile}"] }
      │    WatchExec:{ command: "npx", args: ["nodemon", "--exec", "npx tsx {appHostFile}"] }
      │    InstallDeps: { command: "npm", args: ["install"] }
      │
      ▼
[3] CLI installs guest dependencies
      │  Runs: npm install  (in the apphost directory)
      │
      ▼
[4] CLI builds the AppHost Server .NET project
      │  dotnet build (temporary generated project in $TMPDIR/.aspire/hosts/<hash>/)
      │  Adds required hosting packages from lockfile
      │
      ▼
[5] Code generation
      │  AtsCapabilityScanner scans all loaded assemblies for [AspireExport]
      │  AtsTypeScriptCodeGenerator generates .modules/aspire.js + aspire.d.ts
      │  Emits typed builder classes with invokeCapability() under the hood
      │
      ▼
[6] CLI starts AppHost Server (.NET)
      │  Sets REMOTE_APP_HOST_SOCKET_PATH=/tmp/aspire/<session>.sock
      │  .NET server begins listening on Unix socket
      │
      ▼
[7] CLI starts Guest Process
      │  Runs: npx tsx apphost.ts
      │  Injects REMOTE_APP_HOST_SOCKET_PATH into guest's environment
      │
      ▼
[8] Guest connects and calls capabilities (see §3 below)

3. JSON-RPC Protocol: Wire Format
The transport layer uses LSP-style framing (same as VS Code extensions) over a Unix socket:
Content-Length: 127\r\n
\r\n
{"jsonrpc":"2.0","id":1,"method":"invokeCapability","params":["Aspire.Hosting/createBuilder",{}]}
The TypeScript SDK's transport.ts uses the vscode-jsonrpc npm package for this — the same package VS Code itself uses. No custom framing code needed.

4. Handle System: How Objects Cross the Boundary
Every .NET object that Guest code holds is an opaque handle — just a string ID + type ID tuple. The guest never deserializes the object's internals; it only passes the handle back for future calls.
Guest                                            .NET AppHost Server
  │                                                     │
  │── invokeCapability("Aspire.Hosting/createBuilder", {}) ──►│
  │                                                     │  Creates IDistributedApplicationBuilder
  │◄── { "$handle": "1",                                │  Stores in handle registry [1 → builder]
  │      "$type": "Aspire.Hosting/...IDistributedApplicationBuilder" }
  │
  │── invokeCapability("Aspire.Hosting.Redis/addRedis", {
  │       "builder": { "$handle": "1", "$type": "..." },  ← pass handle back
  │       "name": "cache"
  │   }) ──►│
  │         │  Looks up handle 1 → builder
  │         │  Calls RedisExtensions.AddRedis(builder, "cache")
  │         │  Stores result in handle registry [2 → IResourceBuilder<RedisResource>]
  │◄── { "$handle": "2", "$type": "Aspire.Hosting.Redis/.../RedisResource" }
  │
  │── invokeCapability("Aspire.Hosting/withEnvironment", {
  │       "builder": { "$handle": "2", ... },
  │       "name": "MY_KEY",
  │       "value": "hello"
  │   }) ──►│
The ATS type ID format is {AssemblyName}/{FullTypeName}, e.g.:
Aspire.Hosting/Aspire.Hosting.IDistributedApplicationBuilder
Aspire.Hosting.Redis/Aspire.Hosting.ApplicationModel.RedisResource

5. What the Generated TypeScript SDK Looks Like
The code generator reads [AspireExport] attributes from C# assemblies and produces a typed TypeScript module. A simplified view of what it emits:
// .modules/aspire.js  (generated — do not edit)
import { AspireClient, Handle, registerHandleWrapper } from './base.js';

// --- Handles ---
export type IDistributedApplicationBuilderHandle =
    Handle<'Aspire.Hosting/Aspire.Hosting.IDistributedApplicationBuilder'>;

export type RedisResourceBuilderHandle =
    Handle<'Aspire.Hosting.Redis/Aspire.Hosting.ApplicationModel.RedisResource'>;

// --- DistributedApplicationBuilder wrapper ---
export class DistributedApplicationBuilder {
    constructor(private readonly _handle: IDistributedApplicationBuilderHandle,
                private readonly _client: AspireClient) {}

    async addRedis(name: string, port?: number): Promise<RedisResourceBuilder> {
        const result = await this._client.invokeCapability(
            'Aspire.Hosting.Redis/addRedis',
            { builder: this._handle, name, port }
        );
        return new RedisResourceBuilder(result as RedisResourceBuilderHandle, this._client);
    }

    async build(): Promise<DistributedApplication> { ... }
}

// --- RedisResourceBuilder wrapper ---
export class RedisResourceBuilder {
    constructor(private readonly _handle: RedisResourceBuilderHandle,
                private readonly _client: AspireClient) {}

    async withEnvironment(name: string, value: string): Promise<RedisResourceBuilder> {
        const result = await this._client.invokeCapability(
            'Aspire.Hosting/withEnvironment',
            { builder: this._handle, name, value }
        );
        return new RedisResourceBuilder(result as RedisResourceBuilderHandle, this._client);
    }

    async withPersistence(interval?: number): Promise<RedisResourceBuilder> { ... }
    async withDataVolume(name?: string): Promise<RedisResourceBuilder> { ... }
}

// --- Entry point ---
export async function createBuilder(): Promise<DistributedApplicationBuilder> {
    const socketPath = process.env['REMOTE_APP_HOST_SOCKET_PATH']!;
    const client = new AspireClient(socketPath);
    await client.connect();
    const handle = await client.invokeCapability('Aspire.Hosting/createBuilder', {});
    return new DistributedApplicationBuilder(handle, client);
}
The TypeScript user's apphost.ts then just does:
import { createBuilder } from './.modules/aspire.js';

const builder = await createBuilder();
const cache = await builder.addRedis("cache");
const api = await builder.addProject("api", "../Api/Api.csproj");
await api.withReference(cache);
await builder.build().run();

6. Callbacks: Guest Code Invoked by .NET
When C# needs to call back into the guest (e.g., withEnvironmentCallback), the guest registers a callback ID:
Guest                                                  .NET AppHost Server
  │                                                           │
  │  // TypeScript registers a closure
  │  const callbackId = registerCallback(async (args, client) => {
  │      const ctx = new EnvironmentCallbackContext(args.p0, client);
  │      await ctx.environmentVariables.set("MY_KEY", "hello");
  │  });
  │
  │── invokeCapability("Aspire.Hosting/withEnvironmentCallback", {
  │       "resource": { "$handle":"2",...},
  │       "callback": "callback_1_1234567890"     ← just a string ID
  │   }) ──►│
  │         │  Stores callback string → C# delegate
  │         │  When environment is evaluated at startup:
  │         │
  │◄── invokeCallback("callback_1_1234567890", {
  │        "p0": { "$handle": "5",
  │                "$type": "Aspire.Hosting/.../EnvironmentCallbackContext" }
  │    }) ─────────────────────────────────────────────────────►│
  │    │                                                         │
  │    │  // Guest wraps handle in typed class
  │    │  // Runs user's async closure
  │    │  // ctx.environmentVariables.set("MY_KEY", "hello")
  │    │    ├── invokeCapability("set", { dict: handle, key, value }) ──►
  │    │
  │◄── result: null  ──────────────────────────────────────────►│

7. How to Add a New Guest Language (e.g., Python)
To add Python support you need exactly two things on the server side, both discovered automatically from the NuGet package:
Step 1: Implement ILanguageSupport
// In new package: Aspire.Hosting.CodeGeneration.Python
public sealed class PythonLanguageSupport : ILanguageSupport
{
    public string Language => "python/cpython";

    public Dictionary<string, string> Scaffold(ScaffoldRequest request)
    {
        return new Dictionary<string, string>
        {
            ["apphost.py"] = "from aspire import create_builder\n\nbuilder = await create_builder()\nawait builder.build().run()\n",
            ["requirements.txt"] = "aspire-hosting>=1.0.0\n",
        };
    }

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
        Language = "python/cpython",
        DisplayName = "Python",
        CodeGenLanguage = "Python",
        DetectionPatterns = ["apphost.py"],
        InstallDependencies = new CommandSpec { Command = "pip", Args = ["install", "-r", "requirements.txt"] },
        Execute = new CommandSpec { Command = "python", Args = ["{appHostFile}"] },
        WatchExecute = new CommandSpec
        {
            Command = "watchdog",
            Args = ["watchmedo", "auto-restart", "--patterns=*.py", "--recursive", "--", "python", "{appHostFile}"]
        }
    };
}
Step 2: Implement ICodeGenerator
public sealed class PythonCodeGenerator : ICodeGenerator
{
    public string Language => "Python";

    // Receives the AtsContext (all capabilities, handles, DTOs, enums)
    // Returns a dict of filename → file content
    public Dictionary<string, string> Generate(AtsContext context)
    {
        var sb = new StringBuilder();
        sb.AppendLine("# aspire.py - Generated SDK (do not edit)");
        sb.AppendLine("import json, socket");
        sb.AppendLine("from aspire_transport import AspireClient, Handle");
        sb.AppendLine();

        foreach (var capability in context.Capabilities)
        {
            // Generate typed Python method for each capability
            GenerateCapabilityMethod(sb, capability);
        }

        return new Dictionary<string, string> { [".modules/aspire.py"] = sb.ToString() };
    }
}
Step 3: Register via DI (one line in the NuGet package's hosting extension)
// In Aspire.Hosting.CodeGeneration.Python startup
services.AddSingleton<ILanguageSupport, PythonLanguageSupport>();
services.AddSingleton<ICodeGenerator, PythonCodeGenerator>();
The CLI and AppHost server automatically discover these implementations via DI from the loaded assemblies. No CLI changes needed at all — adding the NuGet package is the only integration step.

8. Summary Table
ConcernOwned byMechanism
Socket path distributionCLIREMOTE_APP_HOST_SOCKET_PATH env varLanguage detectionILanguageSupport.Detect() on serverCLI calls detect RPCRun commandILanguageSupport.GetRuntimeSpec() on serverCLI calls getRuntimeSpec RPCScaffolding new projectILanguageSupport.Scaffold() on serverCLI calls scaffoldAppHost RPCSDK code generationICodeGenerator on serverCLI calls generateCode RPCGuest → Host callsATS JSON-RPC invokeCapabilityGuest connects to Unix socketHost → Guest callbacksATS JSON-RPC invokeCallbackReverse direction on same connectionHandle lifecycleAppHost server handle registryValid until server exitsAll Aspire integrations.NET server processGuest sees them as typed SDK methodsUser: help me set up the local development mode for aspire
GitHub Copilot: Let me check the current state of the repo and relevant setup files.
Read , lines 1 to 80
Ran terminal command:  cat global.json
Ran terminal command:  ls .dotnet 2>/dev/null && echo "SDK exists" || echo "SDK not installed"
Ran terminal command:  ls -la .dotnet 2>/dev/null | head -5; echo "exit: $?"
Ran terminal command:  .dotnet/dotnet --version 2>/dev/null || echo "dotnet not found in .dotnet/"
Good — the local SDK is already installed. Let me check if the repo has been built before and what state the artifacts are in.
Ran terminal command:  ls artifacts/bin 2>/dev/null | head -10; echo "---"; ls artifacts/packages 2>/dev/null | head -5
The repo has already been built and the local SDK is installed. Here's the full local dev setup:

Local Development Setup Status
Already done:
✅ Local .NET SDK installed (dotnet → 10.0.103)
✅ Repo previously built (bin populated)
What's already available:
# Use this instead of system `dotnet` to get the local SDK
./dotnet.sh --version   # → 10.0.103

Daily Workflow
Launch VS Code correctly
Always launch VS Code from the repo root using the helper script — this sets the correct PATH so VS Code terminals automatically use the local SDK:
./start-code.sh
Don't open VS Code directly via code . — that won't have .dotnet on PATH.
Build after code changes
# Full build (restore + build, ~3-5 minutes)
./build.sh

# Build only (skip restore, faster after first run)
./build.sh --build

# Skip native AOT compilation (~1-2 minutes saved)
./build.sh --build /p:SkipNativeBuild=true
Run the sample app to verify setup
./dotnet.sh run --project playground/TestShop/TestShop.AppHost/TestShop.AppHost.csproj
This starts the TestShop distributed app and opens the Dashboard — confirms everything is wired up correctly.
Run tests for a specific project
./dotnet.sh test tests/Aspire.Hosting.Tests/Aspire.Hosting.Tests.csproj -- \
  --filter-not-trait "quarantined=true" --filter-not-trait "outerloop=true"

If You Rebuild from Scratch
# 1. Re-install local SDK (if .dotnet/ gets corrupted)
./restore.sh

# 2. Full clean build
./build.sh --rebuild

Extension Development (VS Code Extension)
If you're working on the VS Code extension under extension:
cd extension
npm install
npm run build
Then use start-code.sh from the repo root and press F5 to launch the Extension Development Host.
User: how does the Aspire Code generator work and why it needs to trigger the SDK generation when first run with aspire cli?
GitHub Copilot: Read
Read , lines 200 to 900
Read , lines 150 to 1100
Read , lines 1 to 100
Compacted conversation
Read , lines 100 to 235
Read , lines 1 to 200
Read , lines 200 to 250
Searched for regex GenerateCodeAsync|generateCode|codegen-hash|modules (**/src/Aspire.Cli/**/*.cs), 19 results
Read , lines 1 to 300
Read , lines 300 to 500
Read , lines 1010 to 1120
Searched for regex NeedsCodeGen|codegen-hash|ComputePackagesHash|\.modules|GeneratedFolder (**/src/Aspire.Cli/Projects/GuestAppHostProject.cs), 15 results
Searched for regex NeedsCodeGeneration|codegen-hash|PrepareAsync (**/src/Aspire.Cli/Projects/*.cs), 14 results
Read , lines 80 to 200
Read , lines 420 to 480
Searched for regex codegen-hash|ComputePackagesHash|GenerationHash|HashChanged|ShouldRegenerate (**/src/Aspire.Cli/**/*.cs), 5 results
Searched for regex NeedsCodeGen|codegen|hash|modules|\.modules (**/src/Aspire.Cli/Projects/DotNetBasedAppHostServerProject.cs), 20 results
Read , lines 135 to 460
Searched for regex ShouldRegenerateCode|ReadGenerationHash|\.codegen-hash|modules.*exist|Directory\.Exists.*modules (**/src/Aspire.Cli/**/*.cs), 2 results
Searched for text generateCode (**/src/Aspire.Hosting/Ats/*.cs), 1 result
Searched for regex generateCode|GenerateCode|ICodeGenerator (**/src/Aspire.Hosting/**/*.cs), 2 results
Searched for regex generateCode|GenerateCode|ICodeGenerator (**/src/Aspire.Hosting/RemoteHost/*.cs), no results
Read
Searched for files matching **/src/Aspire.Hosting/RemoteHost/*.cs, no matches
Searched for text generateCode (**/src/**/*.cs), 20 results
Read , lines 1 to 200
Searched for text AtsContextFactory (**/src/Aspire.Hosting.RemoteHost/**/*.cs), 13 results
Read
Now I have a complete end-to-end picture. Here's the full explanation:

How the Aspire Code Generator Works
The Core Problem: No Runtime Discovery
Guest languages (TypeScript, Python, etc.) can't use .NET reflection at runtime. Every capability (addRedis, withEnvironment, etc.) that comes from a NuGet package must be represented as statically-typed code in the guest language before user code can call it. That's what SDK generation produces: the .modules/aspire.ts (and base.ts, transport.ts) files that TypeScript user code imports.
The Two-Phase Split: Where Code Lives
Server side (AppHost Server process):
AssemblyLoader → AtsContextFactory → AtsCapabilityScanner → AtsContext
                                                              ↓
                                               CodeGenerationService.generateCode()
                                                              ↓
                                                    ICodeGenerator.GenerateDistributedApplication()
                                                              ↓
                                               Dictionary<string, string>  (filename → content)
AtsContextFactory.cs lazily calls AtsCapabilityScanner.ScanAssemblies() over all loaded assemblies
AtsCapabilityScanner.cs does 5-pass scanning: collect all [AspireExport] methods/types → resolve unknown types → filter invalid ones → expand interface targets to concrete types → filter overload collisions
[CodeGenerationService.cs](src/Aspire.Hosting.RemoteHost/CodeGeneration/CodeGeneration Service.cs) exposes the generateCode JSON-RPC method, which calls the registered ICodeGenerator for the requested language
ICodeGenerator.cs is the contract: GenerateDistributedApplication(AtsContext) → Dictionary<string, string>
Client side (CLI):
GuestAppHostProject.cs orchestrates everything: build the server → start it → call GenerateCodeAsync via RPC → write files to .modules/ → save .codegen-hash
AppHostRpcClient.cs sends {"method": "generateCode", "params": ["TypeScript"]} over the Unix socket
Why It Must Run on First Use
When you run aspire run against a TypeScript AppHost:
CLI scaffolds a hidden .NET project in $TMPDIR/.aspire/hosts/<hash>/ with a Program.cs that is just await RemoteHostServer.RunAsync(args), plus a .csproj referencing every Aspire NuGet package listed in aspire.json
CLI builds it — this is the PrepareAsync step in DotNetBasedAppHostServerProject.cs. On success, it returns NeedsCodeGeneration: true
CLI starts the server — the server binary loads all those Aspire assemblies into the process
CLI calls generateCode via RPC — AtsContextFactory lazily scans all loaded assemblies (including e.g. Aspire.Hosting.Redis.dll) for [AspireExport] attributes, building an AtsContext that describes every capability; then AtsTypeScriptCodeGenerator turns that into aspire.ts, base.ts, transport.ts
CLI writes the files to .modules/ and saves a .codegen-hash (SHA-256 of sorted packageId:version pairs) for future cache invalidation
CLI starts the guest (npx tsx apphost.ts) — the TypeScript code can now import { createBuilder } from './.modules/aspire.js'
The SDK cannot be committed to the repo because it is derived from whatever NuGet packages are installed — if you run aspire add redis, the Redis package is added, and on the next run aspire run detects the hash has changed and regenerates so addRedis() appears in the TypeScript SDK.
The aspire sdk generate Command
SdkGenerateCommand.cs is a stand-alone variant for integration library authors. Usage: aspire sdk generate ./MyIntegration.csproj -l typescript -o ./output
It runs the same pipeline in isolation: create a temp AppHost server project → add your .csproj as an additionalProjectReference → build → connect → call GenerateCodeAsync → write files → delete temp dir. This lets integration authors generate typed SDKs from their integration libraries for publishing to npm/PyPI.
Hash-Based Cache: When Regeneration Is Skipped
The .modules/.codegen-hash file stores the SHA-256 of packageId:version;packageId:version;... (sorted). Currently, PrepareAsync always returns NeedsCodeGeneration: true after a successful build, so generation happens every run. The hash is written after generation and is the foundation for future skip logic when the hash matches.
The 5-Pass Scanner
AtsCapabilityScanner.ScanAssemblies() runs these passes across all assemblies:
Collect — per-assembly scan for [AspireExport] on methods and types, [AspireDto] on DTOs, gathers all enum types from signatures
Resolve Unknown types — any capability parameter that references a type not in the exported set is marked Unknown
Filter invalid — capabilities with unresolved Unknown types are dropped (with diagnostics)
Expand interface targets — withEnvironment<T> where T : IResourceWithEnvironment becomes a capability entry for every concrete type that implements IResourceWithEnvironment; ExpandedTargetTypes on each AtsCapabilityInfo is pre-computed here
Filter overloads — C# allows overloaded methods; ATS capability IDs are unique, so duplicate IDs after expansion emit errors and are removed
The resulting AtsContext (capabilities, handle types, DTOs, enums) is then passed to ICodeGenerator.GenerateDistributedApplication() which templates out the guest-language SDK files.
User: how to tell what assemblies are loaded in the aspire cli?
GitHub Copilot: Searched for text AssemblyLoader (**/src/Aspire.Hosting.RemoteHost/**/*.cs), 17 results
Read
The assembly list is driven by two things:
1. AtsAssemblies in appsettings.json
When the CLI builds the hidden AppHost server project, it writes an appsettings.json into the temp project directory listing every assembly to scan. You can see this in DotNetBasedAppHostServerProject.cs — it always starts with "Aspire.Hosting" and then appends every package name from aspire.json:
{
  "AtsAssemblies": [
    "Aspire.Hosting",
    "Aspire.Hosting.Redis",
    "Aspire.Hosting.CodeGeneration.TypeScript"
  ]
}
AssemblyLoader reads this array at startup and calls Assembly.Load(name) for each — these are simple by-name loads, so the assembly must already be on the probing path (i.e., built into the server's output directory as a dependency).
2. ASPIRE_INTEGRATION_LIBS_PATH env var (prebuilt / bundled mode)
For the prebuilt CLI bundle (no local .NET SDK), PrebuiltAppHostServer restores the NuGet packages to a temp folder and passes that path via ASPIRE_INTEGRATION_LIBS_PATH. The AssemblyLoader registers a custom AssemblyLoadContext.Default.Resolving handler that probes that directory for DLLs when a name-based load fails.
How to see what's actually loaded
Look at the appsettings.json the CLI generates in $TMPDIR/.aspire/hosts/<hash>/appsettings.json — that's the exact list
Enable Debug log level for the server (set "Aspire.Hosting.RemoteHost": "Debug" in the appsettings) to see "Loaded assembly: {AssemblyName}" log lines from AssemblyLoader
Use aspire sdk dump (SdkDumpCommand.cs) which calls the getCapabilities RPC — you can infer what assemblies scanned by what capability IDs appear (e.g., Aspire.Hosting.Redis/addRedis means Aspire.Hosting.Redis was loaded)
User: why would CLI need to communicate with the AppHost server via RPC?
GitHub Copilot: Read
The RPC between CLI and AppHost server exists for a fundamental reason: the CLI is a native AOT-compiled binary that cannot dynamically load arbitrary .NET assemblies, but ALL the language-specific knowledge lives inside NuGet packages that are only present in the server process.
Here's the breakdown of why each RPC call can't be done in the CLI directly:
RPC methodWhy the CLI can't do it locally
getRuntimeSpecRuntimeSpec (how to run npx tsx, nodemon, etc.) is returned by ILanguageSupport implementations that come from NuGet packages (Aspire.Hosting.CodeGeneration.TypeScript). The CLI doesn't reference those packages.scaffoldAppHostScaffold templates (apphost.ts, package.json, etc.) are embedded resources inside those same NuGet packages, accessed via ILanguageSupport.Scaffold().generateCodeCode generation requires AtsCapabilityScanner to reflect over loaded integration assemblies (Aspire.Hosting.Redis.dll, etc.). The CLI can't load those — only the server can, because it was built with them as project/package references.getCapabilitiesSame — needs reflection over NuGet package assemblies to enumerate [AspireExport] attributes.There's also a process separation reason: the AppHost Server is the DCP orchestration runtime (it manages containers, services, the dashboard). The CLI is the user-facing shell. Keeping them in separate processes means the CLI can monitor the server, restart it, and display its exit code without being the same process.
The three-process architecture (CLI ↔ AppHost Server ↔ Guest Runtime) is intentional:
CLI (native AOT, thin orchestrator)
  │── JSON-RPC over Unix socket ──→ AppHost Server (.NET, loads all NuGet packages)
                                         │── JSON-RPC over Unix socket ──→ Guest (TypeScript/Python)
The CLI is evolving toward a "pure orchestrator" that knows nothing about specific languages or integrations — it just asks the server what to do and follows its instructions (the RuntimeSpec).
User: how is the provisioning library being used? and who generates bicep files? who deploy the bicep files?
GitHub Copilot: Searched for regex AzurePublishingContext|BicepCompiler|\.bicep|WriteBicep|GenerateBicep (**/src/**/*.cs), 20 results
Searched for files matching **/src/Aspire.Hosting.Azure/Bicep*.cs, 1 match
Searched for files matching **/src/Aspire.Hosting.Azure/AzureBicepResource.cs, 1 match
Searched for regex AzurePublishingContext|PublishAsync.*azure|DeployAsync|azd|azure developer cli (**/src/**/*.cs), 20 results
Searched for regex AzureProvisioningResource|infrastructure\.Build|GetBicepTemplateString|WriteToManifest (**/src/Aspire.Hosting.Azure/**/*.cs), 20 results
Read , lines 1 to 180
Read , lines 120 to 180
Searched for regex ProvisionAzureBicepResourceAsync|ArmDeployment|DeployBicep|azure-mgmt (**/src/Aspire.Hosting.Azure/**/*.cs), 20 results
Read , lines 150 to 285
Read , lines 1 to 300
Searched for regex AzureProvisioningResource.*GetBicepTemplateString|infrastructure\.Build\(\)|\.Compile\(\)|BicepCompiler (**/src/Aspire.Hosting.Azure/**/*.cs), 12 results
Searched for files matching **/src/Aspire.Hosting.Azure/AzureProvisioningResource.cs, 1 match
Read , lines 1 to 120
Read , lines 1 to 80
Here is the full picture of how provisioning works, from C# code all the way to deployed Azure resources:

The Three-Layer Stack
Layer 1: Azure.Provisioning SDK → Bicep text (C# code in your AppHost)
Azure.Provisioning.* NuGet packages (Azure.Provisioning.Redis, Azure.Provisioning.PostgreSql, etc.) are a C#-first IaC SDK. They let you describe Azure resources in C# using types like CdkRedisResource, RoleAssignment, BicepValue<T>, BicepFunction.Interpolate().
When AzureProvisioningResource.GetBicepTemplateFile() is called:
var infrastructure = new AzureResourceInfrastructure(this, Name);
ConfigureInfrastructure(infrastructure);      // user's callback, e.g., ConfigureRedisInfrastructure
var plan = infrastructure.Build(options);
var compilation = plan.Compile();             // Azure.Provisioning SDK → Bicep text string
File.WriteAllText(moduleSourcePath, compiledBicep.Value);
The ConfigureInfrastructure callback is where each integration (e.g., AzureRedisExtensions.cs) builds up the C# object graph — create a CdkRedisResource, configure AAD auth, add ProvisioningOutput for connection strings, etc. The SDK's Compile() call turns that into Bicep text. No Bicep files are pre-authored; they are all generated at runtime from C#.
Layer 2: Bicep text → ARM JSON (the bicep CLI)
BicepCliCompiler.CompileBicepToArmAsync() in BicepCompiler.cs shells out to bicep build --stdout (or az bicep build as fallback). This converts the generated .module.bicep file into an ARM template JSON string. The bicep binary is either the standalone Bicep CLI or Azure CLI's extension — it must be on PATH.
Layer 3: ARM template → Azure deployment (Azure Resource Manager)
BicepProvisioner.GetOrCreateResourceAsync() in BicepProvisioner.cs submits the ARM JSON to Azure using the Azure Management SDK:
var deployments = resourceGroup.GetArmDeployments();   // Azure.ResourceManager SDK
var operation = await deployments.CreateOrUpdateAsync(
    WaitUntil.Started, deploymentName,
    new ArmDeploymentContent(new(ArmDeploymentMode.Incremental)
    {
        Template = BinaryData.FromString(armTemplateContents),
        Parameters = BinaryData.FromObjectAsJson(parameters),
    }), cancellationToken);
await operation.WaitForCompletionAsync(cancellationToken);
This is a standard ARM API call — it creates a named deployment in the Azure resource group and polls until it completes. Deployment outputs (connection strings, host names, etc.) are read back and stored in resource.Outputs.

Two Different Deployment Paths
Path A: aspire run (local dev provisioning)
When you run locally with Azure resources, the provisioning pipeline runs inside the AppHost process:
AzureBicepResource constructor → adds PipelineStepAnnotation → pipeline step calls ProvisionAzureBicepResourceAsync → BicepProvisioner.GetOrCreateResourceAsync() → ARM deployment.
Deployment state is cached in ~/.aspire/deployments/<hash>.json (or similar) so re-runs skip provisioning if the checksum hasn't changed.
Path B: azd up (Azure Developer CLI deployment)
AzureBicepResource.WriteToManifest() writes to the Aspire manifest (JSON) that azd reads:
{
  "type": "azure.bicep.v0",
  "path": "redis.module.bicep",
  "params": { "location": "{azure.location}" }
}
azd reads this manifest, takes the .bicep files that were written alongside it, and runs its own ARM deployment pipeline. Aspire does not deploy in this path — it just produces the manifest and .bicep files; azd handles the actual az deployment group create.

Summary
WhoWhat
Azure.Provisioning SDKGenerates Bicep text from C# DSL (infrastructure.Build().Compile())BicepCliCompilerShells out to bicep/az bicep to compile Bicep → ARM JSONBicepProvisionerSubmits ARM JSON to Azure via Azure.ResourceManager SDK, polls for completionazdAlternative deployer — reads Aspire manifest + .bicep files, runs its own deploymentUser: how provisioning library communicate with AppHost server?
GitHub Copilot: Searched for regex PipelineStep|PipelineStepAnnotation|ResourceNotificationService|ProvisioningContext (**/src/Aspire.Hosting.Azure/**/*.cs), 20 results
Searched for files matching **/src/Aspire.Hosting/Pipelines/*.cs, 20 matches
Searched for text PipelineStepAnnotation (**/src/Aspire.Hosting/Pipelines/DistributedApplicationPipeline.cs), 1 result
Read , lines 120 to 285
Read , lines 60 to 390
Read , lines 1 to 100
The provisioning library does not communicate with the AppHost server via RPC. They run in the same process. Here's the full picture:

The Provisioning Library Is Part of the AppHost Process
There is no RPC boundary between the provisioning library and the "AppHost server." The Aspire.Hosting.Azure provisioning code is loaded as a regular .NET assembly inside the AppHost process itself. Communication is via in-process .NET calls through the dependency injection container and the Pipeline system.
How It Flows: The Pipeline
AzureBicepResource registers itself into the Pipeline via annotations in its constructor:
AzureBicepResource constructor
  ├── ManifestPublishingCallbackAnnotation → WriteToManifest() (for azd manifest generation)
  ├── PipelineStepAnnotation → creates "provision-{name}" step with Action = ProvisionAzureBicepResourceAsync
  └── PipelineConfigurationAnnotation → wires up dependency ordering between Azure resources
DistributedApplicationPipeline (in Aspire.Hosting.Pipelines) walks all resources in the app model, collects their PipelineStepAnnotations, sorts them topologically, and executes them in order.
When the "provision-redis" step runs, ProvisionAzureBicepResourceAsync resolves services directly from DI:
private static async Task ProvisionAzureBicepResourceAsync(PipelineStepContext context, AzureBicepResource resource)
{
    var bicepProvisioner = context.Services.GetRequiredService<IBicepProvisioner>();  // DI, in-process
    var azureEnvironment = context.Model.Resources.OfType<AzureEnvironmentResource>().First();
    var provisioningContext = await azureEnvironment.ProvisioningContextTask.Task;   // TaskCompletionSource

    await bicepProvisioner.GetOrCreateResourceAsync(resource, provisioningContext, ...);
}
The ProvisioningContext comes from AzureEnvironmentResource.ProvisioningContextTask — another pipeline step ("create-provisioning-context") runs first and completes that TaskCompletionSource with the Azure credentials/subscription/resource group. Individual Bicep steps DependsOn that step, so they wait for it.
Communication Channels (all in-process):
From → ToMechanism
Pipeline executor → BicepProvisionerIServiceProvider.GetRequiredService<IBicepProvisioner>()BicepProvisioner → Dashboard/UIResourceNotificationService.PublishUpdateAsync() (in-memory channel)BicepProvisioner → bicep CLIProcess.Start("bicep build ...") (subprocess, not RPC)BicepProvisioner → Azure ARM APIAzure.ResourceManager SDK over HTTPSOne Bicep resource → another (ordering)TaskCompletionSource on AzureEnvironmentResource.ProvisioningContextTask + PipelineStep.DependsOn()The only external communication provisioning does is:
Shelling out to the bicep or az CLI binary to compile Bicep → ARM JSON
HTTPS to Azure Resource Manager (management.azure.com) to submit the ARM deployment
User: if provisioning library can generate bicep and deploy via bicep , why we still need guest language runtime to communicate with AppHost dotnet server via json rpc to callback capbilities?
GitHub Copilot: They serve completely different phases. The JSON-RPC capability system is how the guest language builds the app model. The provisioning library consumes that model after it's built. One cannot replace the other.
The Sequence
TypeScript apphost.ts (running as Guest)
    │
    │ JSON-RPC: invokeCapability("Aspire.Hosting/createBuilder", {})
    ├──────────────────────────────────────> AppHost Server (.NET)
    │ <── { $handle: "1" }                   IDistributedApplicationBuilder created in memory
    │
    │ JSON-RPC: invokeCapability("Aspire.Hosting.Redis/addRedis", {builder: {$handle:"1"}, name:"cache"})
    ├──────────────────────────────────────> RedisResource added to DistributedApplicationModel
    │ <── { $handle: "2" }
    │
    │ JSON-RPC: invokeCapability("Aspire.Hosting/build", {builder: {$handle:"1"}})
    ├──────────────────────────────────────> DistributedApplication.Build() called
    │
    │ JSON-RPC: invokeCapability("Aspire.Hosting/run", {app: {$handle:"3"}})
    ├──────────────────────────────────────> app.RunAsync() starts
    │                                            │
    │                                            ├── DcpExecutor: spin up containers
    │                                            ├── Pipeline: provision Azure resources
    │                                            │     └── BicepProvisioner → bicep CLI → ARM API
    │                                            └── Dashboard: show resource states
The guest language's JSON-RPC calls are the equivalent of Program.cs in a .NET AppHost. They are what populate the DistributedApplicationModel with resources. Without them, the model is empty — there is nothing for the provisioning library or DCP executor to act on.
The Analogy
Think of it this way:
A .NET AppHost — the user writes builder.AddRedis("cache") in C#, which directly calls the in-process AddRedis() extension method, adding a RedisResource to the model
A TypeScript AppHost — the user writes await builder.addRedis("cache") in TypeScript, which sends a JSON-RPC call to the .NET server, which runs the same AddRedis() method in-process on behalf of the guest
In both cases the outcome is identical: a RedisResource lands in the DistributedApplicationModel. The provisioning pipeline doesn't know or care which language authored the model.
Why Not Generate Bicep From the Guest Directly?
The provisioning library (Bicep generation, Azure SDK, ARM deployment) has deep dependencies on:
Azure.Provisioning.* NuGet packages with .NET-only types (ProvisionableResource, BicepValue<T>, etc.)
Azure.ResourceManager SDK for ARM API calls
System.Reflection for scanning [AspireContainerImage], role assignments, etc.
A TypeScript process cannot run any of that. It can only invoke capabilities on the .NET server, which then does the actual work.
Summary: The JSON-RPC capabilities are the authoring API — the guest builds the resource graph. The provisioning library is the execution engine — it acts on that graph. They are sequential, not competing.
User: how does the guest build the resource graph via AppHost json rpc? what input does the provisioning library need in order to generate bicep files to really create the resources?
GitHub Copilot: Read , lines 1 to 100
Read , lines 1 to 120
Read , lines 120 to 300
Read , lines 100 to 260
Read , lines 200 to 300
Searched for regex RemoteAppHostService|invokeCapability|InvokeCapability (**/src/Aspire.Hosting.RemoteHost/**/*.cs), 15 results
Read , lines 140 to 380
Read
Searched for regex AzureResourcePreparer|ConfigureInfrastructure|DistributedApplicationModel.*azure (**/src/Aspire.Hosting.Azure/**/*.cs), 20 results
Read , lines 1 to 120
Now I have the full picture. Here is the end-to-end answer:

Phase 1: The Guest Builds the Resource Graph via JSON-RPC
The key insight is that JSON-RPC calls are direct reflection-based invocations of real .NET methods. There is no separate protocol translation layer — the dispatcher just calls the actual C# method.
The invocation chain
TypeScript: await builder.addRedis("cache")
    │
    │  JSON-RPC: invokeCapability("Aspire.Hosting.Redis/addRedis", {"builder": {$handle:"1"}, "name":"cache"})
    ▼
RemoteAppHostService.InvokeCapabilityAsync()
    └── CapabilityDispatcher.InvokeAsync("Aspire.Hosting.Redis/addRedis", args)
            │
            │  RegisterFromCapability() bound this capability ID to the real MethodInfo
            │  of AddRedis() at startup (via AtsCapabilityScanner reflection scan)
            │
            ├─ Unmarshal args:
            │     "builder" → HandleRegistry.TryGet("1") → returns the actual IDistributedApplicationBuilder object
            │     "name"    → JsonNode → string "cache"
            │
            ├─ Invoke via reflection:
            │     method.Invoke(null, [builder, "cache"])
            │     == RedisExtensions.AddRedis(builder, "cache")   ← the real C# extension method
            │
            └─ Marshal result:
                  IResourceBuilder<RedisResource> → HandleRegistry.Register(obj) → handle "2"
                  Return: {"$handle": "2", "$type": "Aspire.Hosting.Redis/...RedisResource"}
The HandleRegistry is a per-client ConcurrentDictionary<string, object> that stores live .NET objects by ID. When TypeScript sends {$handle:"1"} back in the next call, the dispatcher resolves it to the actual IDistributedApplicationBuilder instance.
The result: after the TypeScript guest calls createBuilder → addRedis → build → run, the AppHost process has a fully populated DistributedApplicationModel with a real RedisResource in its Resources collection — identical to what a Program.cs in a .NET AppHost would produce.

Phase 2: What the Provisioning Library Reads From the Model
AzureResourcePreparer.GetAzureResourcesFromAppModel() scans the populated model and finds each IAzureResource. For a Redis resource added by AddRedis().AsAzureManagedRedis(), the model object is an AzureManagedRedisResource which is an AzureProvisioningResource (which is an AzureBicepResource).
The provisioning library needs two things from each AzureProvisioningResource:
1. ConfigureInfrastructure callback — a Action<AzureResourceInfrastructure> set at resource creation time. When GetBicepTemplateFile() is called, it runs the callback:
ConfigureInfrastructure(infrastructure);   // → adds CdkRedisResource, ProvisioningOutput, etc.
var plan = infrastructure.Build();
var compilation = plan.Compile();          // → Bicep text string
This callback was closed over C# objects when AddRedis() ran — it carries the resource configuration directly.
2. Parameters dictionary — key/value pairs that become Bicep param declarations. These include:
location (Azure region)
principalId / principalType for managed identity role assignments
References to other resources' outputs (e.g., keyVaultUri)
These are populated by BicepUtilities.SetParametersAsync() at deploy time, resolving IManifestExpressionProvider references, ParameterResource values (from user prompts), and BicepOutputReference cross-resource references.

The Complete Picture
TypeScript apphost.ts
    │ JSON-RPC: invokeCapability("Aspire.Hosting.Redis/addRedis", ...)
    ▼
CapabilityDispatcher
    │ Reflection: RedisExtensions.AddRedis(builder, "cache", port: null)
    │             ↳ new AzureManagedRedisResource("cache", configureInfra)
    │             ↳ builder.Resources.Add(redisResource)
    ▼
DistributedApplicationModel.Resources
    └── AzureManagedRedisResource
            ├── ConfigureInfrastructure: (infra) => {
            │       var redis = new CdkRedisResource(...);
            │       infra.Add(redis);
            │       infra.Add(new ProvisioningOutput("connectionString", ...));
            │   }
            ├── Parameters: { "location": "{azure.location}", ... }
            └── PipelineStepAnnotation: → ProvisionAzureBicepResourceAsync

                            ↓ (when app.Run() is called)

AzureResourcePreparer.OnBeforeStartAsync()
    └── Scans model, builds role assignment annotations

Pipeline: "provision-redis" step
    └── BicepProvisioner.GetOrCreateResourceAsync()
            ├── resource.GetBicepTemplateFile()
            │       └── ConfigureInfrastructure(infra) → plan.Compile() → redis.module.bicep
            ├── BicepCliCompiler: "bicep build redis.module.bicep" → ARM JSON
            └── resourceGroup.GetArmDeployments().CreateOrUpdateAsync(armJson)
                    → Azure REST API → Redis Cache created in Azure
                    → Outputs (connectionString) stored in resource.Outputs["connectionString"]
User:  introduce more about DistributedApplicationModel and the architecture overview?
GitHub Copilot: Compacted conversation
Searched codebase for "DistributedApplicationModel class definition resources annotations", 24 results
Searched for files matching **/DistributedApplicationModel.cs, 1 match
Searched for files matching **/IResourceCollection.cs, 1 match
Read
Read , lines 1 to 150
Read , lines 150 to 500
Searched for regex class.*Resource.*IResource|ContainerResource|ProjectResource|ExecutableResource (**/src/Aspire.Hosting/ApplicationModel/*.cs), 20 results
Read
Read , lines 500 to 700
Read , lines 1 to 80
Read
Here's a comprehensive overview:

DistributedApplicationModel — The Application Graph
DistributedApplicationModel is the central data structure of an Aspire AppHost. It is a trivially simple object with one purpose: holding the flat list of all resources.
// src/Aspire.Hosting/ApplicationModel/DistributedApplicationModel.cs
[DebuggerDisplay("Resources = {Resources.Count}")]
public class DistributedApplicationModel(IResourceCollection resources)
{
    public IResourceCollection Resources { get; } = resources;
}
IResourceCollection is just IList<IResource> — a flat list, not a tree. The graph structure (the DAG of dependencies) is implicit, encoded in annotations and ReferenceExpression objects attached to resources. Aspire never stores an explicit edge list; edges are discovered at runtime by traversing annotations.
It's registered as a singleton in DI. Everything — DcpExecutor, AzureResourcePreparer, BicepProvisioner, manifest writers, the dashboard — gets it from DI and queries model.Resources.

IResource — The Fundamental Unit
public interface IResource
{
    string Name { get; }
    ResourceAnnotationCollection Annotations { get; }
}
That's the entire interface. Two properties. Resources are pure inert data objects — they don't start, stop, or manage themselves. Everything about their runtime behavior is carried in Annotations.
The abstract Resource base class just validates the name and provides the collection:
IResource
  └── Resource (abstract)
        ├── ContainerResource        — runs a Docker image
        ├── ProjectResource          — runs a .NET project
        ├── ExecutableResource       — runs an arbitrary binary
        ├── ParameterResource        — an external input / secret
        ├── ConnectionStringResource — a literal conn string
        └── [integration-specific]
              ├── RedisResource       : ContainerResource
              ├── PostgresServerResource : ContainerResource
              ├── AzureProvisioningResource : AzureBicepResource
              └── ...
The three primitive compute types are what DCP (DcpExecutor) knows about natively. Everything else is modeled on top of them.

Annotations — The Extensibility Mechanism
IResourceAnnotation is an empty marker interface. Any POCO implementing it can be attached to any resource:
// All of these are annotations
ContainerImageAnnotation        // → image name, tag, registry
EndpointAnnotation              // → port, scheme, name
EnvironmentCallbackAnnotation   // → lambda to inject env vars
ContainerMountAnnotation        // → volume or bind mount
HealthCheckAnnotation           // → health probe config
PipelineStepAnnotation          // → deploy pipeline step
ManifestPublishingCallbackAnnotation // → how to write to azd manifest
WaitAnnotation                  // → WaitFor dependency
Extension methods like AddRedis("cache") are really just:
new RedisResource("cache") (a ContainerResource subclass)
resource.Annotations.Add(new ContainerImageAnnotation { Image = "redis", Tag = "7.4" })
resource.Annotations.Add(new EndpointAnnotation(ProtocolType.Tcp, ..., targetPort: 6379))
Wrap in an IResourceBuilder<RedisResource> and return
The IResourceBuilder<T> is just a fluent wrapper:
IDistributedApplicationBuilder     (top-level, owns model + DI)
  └── .AddRedis("cache")
        └── IResourceBuilder<RedisResource>
              ├── .Resource          → the RedisResource instance
              ├── .WithEnvironment() → adds EnvironmentCallbackAnnotation
              ├── .WithDataVolume()  → adds ContainerMountAnnotation
              └── .WithHealthCheck() → adds HealthCheckAnnotation

Standard Capability Interfaces
Resources opt into behaviors by implementing interfaces (polymorphism over type checks):
InterfaceWhat it enables
IResourceWithEnvironment.WithEnvironment(), env var injectionIResourceWithEndpoints.WithEndpoint(), GetEndpoint(), URL generationIResourceWithConnectionString.WithReference(db) wires conn string into consumersIResourceWithServiceDiscoveryRegisters service in DNS-style discoveryIResourceWithArgsCLI argument injectionIResourceWithWaitSupportWaitFor() — blocks startup until dependency is readyIResourceWithParentLifecycle containment (child stops when parent stops)IComputeResourceMarks as a compute unit (project/container/exec)ContainerResource implements nearly all of them. ParameterResource implements almost none (it's just a value, not a runnable thing).

The Value/Reference System — How the DAG Forms
The graph is a heterogeneous DAG encoding both dependency order and value flows. Edges are encoded as structured value objects:
web ──► EndpointReference ──► api
                               │
                               └──► ConnectionStringReference ──► postgres
Key types:
TypeRun modePublish mode
EndpointReferencehttp://localhost:5000{api.bindings.http.url}ConnectionStringReferenceHost=localhost;Port=5432;...{postgres.connectionString}ParameterResourcevalue from user secrets / env${PARAM} placeholderReferenceExpressioninterpolated concrete stringinterpolated manifest expressionquot;{redis.HostName},ssl=true,password={redis.GetKeys().PrimaryKey}")
        }
    };
    infrastructure.Add(secret);
}


When AzureProvisioningResource.GetBicepTemplateString() is called, it:

Creates a fresh AzureResourceInfrastructure (wrapper around Azure.Provisioning Infrastructure)
Calls ConfigureInfrastructure(infrastructure) — the callback above runs
Calls infrastructure.Build(ProvisioningBuildOptions) — Azure.Provisioning SDK computes the full Bicep module
Calls .Compile() — emits the Bicep source text as a string
Full Azure resource lifecycle mapping
C# API call (AddAzureManagedRedis)
    │
    ├─ Creates AzureManagedRedisResource (: AzureProvisioningResource)
    │   └─ Stores configureInfrastructure callback
    │
    ├─ Attaches ManifestPublishingCallbackAnnotation
    │   └─ WriteToManifest() → writes "type: azure.bicep.v0" + path to manifest.json
    │
    ├─ Attaches PipelineStepAnnotation
    │   └─ provision-{name} step → calls BicepProvisioner.GetOrCreateResourceAsync()
    │
    └─ Attaches PipelineConfigurationAnnotation
        └─ Forces Bicep template evaluation → discovers cross-resource dependencies

At run/publish time:
    GetBicepTemplateString()
        → new AzureResourceInfrastructure(this, Name)
        → ConfigureInfrastructure(infra)         ← Azure.Provisioning objects added here
        → infra.Build(options).Compile()          ← Azure.Provisioning emits Bicep text
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


The key insight: Azure.Provisioning SDK is the bridge between Aspire's C# resource model and Bicep text. Integration authors never write .bicep files by hand — they compose Azure.Provisioning typed objects in the configureInfrastructure callback, and the SDK generates correct, valid Bicep automatically.
User: How AppHost interact with guest runtime and language runtime? How to integrate it with another language like TypeScript runtime or TypeScript AppHost?
GitHub Copilot: Read , lines 100 to 300
Read
Read , lines 300 to 600
Read
Read , lines 600 to 900
Read
Read , lines 1 to 120
Read , lines 1 to 100
Read , lines 1 to 120
Read , lines 120 to 181
Here's a detailed explanation of how the AppHost interacts with guest runtimes, using TypeScript as the concrete example:


AppHost ↔ Guest Runtime Interaction
The Big Picture
The key design principle: all 40+ integrations stay in .NET. Guest language code never reimplements them. Instead, guest code calls into the .NET server via JSON-RPC and gets back opaque handles to .NET objects, which it passes forward in future calls.


1. Process Architecture
There are always three separate processes at runtime:
┌──────────────────────┐
│     Aspire CLI       │  ← orchestrator: starts both, passes socket path
└───────┬──────────────┘
        │ spawns                      spawns │
        ▼                                    ▼
┌──────────────────────┐       ┌─────────────────────────────┐
│  AppHost Server      │       │  Guest Process               │
│  (.NET)              │◄─────►│  (e.g., `npx tsx apphost.ts`)│
│                      │ RPC   │                              │
│  - ILanguageSupport  │       │  - Generated SDK             │
│  - ICodeGenerator    │       │  - User's apphost.ts         │
│  - CapabilityDispatch│       │  - ATS Client (transport.ts) │
│  - App Model & DCP   │       └─────────────────────────────┘
└──────────────────────┘
        ▲ JSON-RPC over Unix socket
        (path passed via REMOTE_APP_HOST_SOCKET_PATH env var)




2. Startup Sequence Step by Step
aspire run (in TypeScript apphost dir)
      │
      ▼
[1] CLI detects language
      │  Calls server RPC: detect(directoryPath)
      │  TypeScriptLanguageSupport.Detect() checks:
      │    - apphost.ts exists?
      │    - package.json exists?
      │  Returns: language = "typescript/nodejs"
      │
      ▼
[2] CLI requests RuntimeSpec from server
      │  Calls server RPC: getRuntimeSpec("typescript/nodejs")
      │  TypeScriptLanguageSupport.GetRuntimeSpec() returns:
      │    Execute:  { command: "npx", args: ["tsx", "{appHostFile}"] }
      │    WatchExec:{ command: "npx", args: ["nodemon", "--exec", "npx tsx {appHostFile}"] }
      │    InstallDeps: { command: "npm", args: ["install"] }
      │
      ▼
[3] CLI installs guest dependencies
      │  Runs: npm install  (in the apphost directory)
      │
      ▼
[4] CLI builds the AppHost Server .NET project
      │  dotnet build (temporary generated project in $TMPDIR/.aspire/hosts/<hash>/)
      │  Adds required hosting packages from lockfile
      │
      ▼
[5] Code generation
      │  AtsCapabilityScanner scans all loaded assemblies for [AspireExport]
      │  AtsTypeScriptCodeGenerator generates .modules/aspire.js + aspire.d.ts
      │  Emits typed builder classes with invokeCapability() under the hood
      │
      ▼
[6] CLI starts AppHost Server (.NET)
      │  Sets REMOTE_APP_HOST_SOCKET_PATH=/tmp/aspire/<session>.sock
      │  .NET server begins listening on Unix socket
      │
      ▼
[7] CLI starts Guest Process
      │  Runs: npx tsx apphost.ts
      │  Injects REMOTE_APP_HOST_SOCKET_PATH into guest's environment
      │
      ▼
[8] Guest connects and calls capabilities (see §3 below)




3. JSON-RPC Protocol: Wire Format
The transport layer uses LSP-style framing (same as VS Code extensions) over a Unix socket:
Content-Length: 127\r\n
\r\n
{"jsonrpc":"2.0","id":1,"method":"invokeCapability","params":["Aspire.Hosting/createBuilder",{}]}


The TypeScript SDK's transport.ts uses the vscode-jsonrpc npm package for this — the same package VS Code itself uses. No custom framing code needed.


4. Handle System: How Objects Cross the Boundary
Every .NET object that Guest code holds is an opaque handle — just a string ID + type ID tuple. The guest never deserializes the object's internals; it only passes the handle back for future calls.
Guest                                            .NET AppHost Server
  │                                                     │
  │── invokeCapability("Aspire.Hosting/createBuilder", {}) ──►│
  │                                                     │  Creates IDistributedApplicationBuilder
  │◄── { "$handle": "1",                                │  Stores in handle registry [1 → builder]
  │      "$type": "Aspire.Hosting/...IDistributedApplicationBuilder" }
  │
  │── invokeCapability("Aspire.Hosting.Redis/addRedis", {
  │       "builder": { "$handle": "1", "$type": "..." },  ← pass handle back
  │       "name": "cache"
  │   }) ──►│
  │         │  Looks up handle 1 → builder
  │         │  Calls RedisExtensions.AddRedis(builder, "cache")
  │         │  Stores result in handle registry [2 → IResourceBuilder<RedisResource>]
  │◄── { "$handle": "2", "$type": "Aspire.Hosting.Redis/.../RedisResource" }
  │
  │── invokeCapability("Aspire.Hosting/withEnvironment", {
  │       "builder": { "$handle": "2", ... },
  │       "name": "MY_KEY",
  │       "value": "hello"
  │   }) ──►│


The ATS type ID format is {AssemblyName}/{FullTypeName}, e.g.:

Aspire.Hosting/Aspire.Hosting.IDistributedApplicationBuilder
Aspire.Hosting.Redis/Aspire.Hosting.ApplicationModel.RedisResource


5. What the Generated TypeScript SDK Looks Like
The code generator reads [AspireExport] attributes from C# assemblies and produces a typed TypeScript module. A simplified view of what it emits:
// .modules/aspire.js  (generated — do not edit)
import { AspireClient, Handle, registerHandleWrapper } from './base.js';

// --- Handles ---
export type IDistributedApplicationBuilderHandle =
    Handle<'Aspire.Hosting/Aspire.Hosting.IDistributedApplicationBuilder'>;

export type RedisResourceBuilderHandle =
    Handle<'Aspire.Hosting.Redis/Aspire.Hosting.ApplicationModel.RedisResource'>;

// --- DistributedApplicationBuilder wrapper ---
export class DistributedApplicationBuilder {
    constructor(private readonly _handle: IDistributedApplicationBuilderHandle,
                private readonly _client: AspireClient) {}

    async addRedis(name: string, port?: number): Promise<RedisResourceBuilder> {
        const result = await this._client.invokeCapability(
            'Aspire.Hosting.Redis/addRedis',
            { builder: this._handle, name, port }
        );
        return new RedisResourceBuilder(result as RedisResourceBuilderHandle, this._client);
    }

    async build(): Promise<DistributedApplication> { ... }
}

// --- RedisResourceBuilder wrapper ---
export class RedisResourceBuilder {
    constructor(private readonly _handle: RedisResourceBuilderHandle,
                private readonly _client: AspireClient) {}

    async withEnvironment(name: string, value: string): Promise<RedisResourceBuilder> {
        const result = await this._client.invokeCapability(
            'Aspire.Hosting/withEnvironment',
            { builder: this._handle, name, value }
        );
        return new RedisResourceBuilder(result as RedisResourceBuilderHandle, this._client);
    }

    async withPersistence(interval?: number): Promise<RedisResourceBuilder> { ... }
    async withDataVolume(name?: string): Promise<RedisResourceBuilder> { ... }
}

// --- Entry point ---
export async function createBuilder(): Promise<DistributedApplicationBuilder> {
    const socketPath = process.env['REMOTE_APP_HOST_SOCKET_PATH']!;
    const client = new AspireClient(socketPath);
    await client.connect();
    const handle = await client.invokeCapability('Aspire.Hosting/createBuilder', {});
    return new DistributedApplicationBuilder(handle, client);
}


The TypeScript user's apphost.ts then just does:
import { createBuilder } from './.modules/aspire.js';

const builder = await createBuilder();
const cache = await builder.addRedis("cache");
const api = await builder.addProject("api", "../Api/Api.csproj");
await api.withReference(cache);
await builder.build().run();




6. Callbacks: Guest Code Invoked by .NET
When C# needs to call back into the guest (e.g., withEnvironmentCallback), the guest registers a callback ID:
Guest                                                  .NET AppHost Server
  │                                                           │
  │  // TypeScript registers a closure
  │  const callbackId = registerCallback(async (args, client) => {
  │      const ctx = new EnvironmentCallbackContext(args.p0, client);
  │      await ctx.environmentVariables.set("MY_KEY", "hello");
  │  });
  │
  │── invokeCapability("Aspire.Hosting/withEnvironmentCallback", {
  │       "resource": { "$handle":"2",...},
  │       "callback": "callback_1_1234567890"     ← just a string ID
  │   }) ──►│
  │         │  Stores callback string → C# delegate
  │         │  When environment is evaluated at startup:
  │         │
  │◄── invokeCallback("callback_1_1234567890", {
  │        "p0": { "$handle": "5",
  │                "$type": "Aspire.Hosting/.../EnvironmentCallbackContext" }
  │    }) ─────────────────────────────────────────────────────►│
  │    │                                                         │
  │    │  // Guest wraps handle in typed class
  │    │  // Runs user's async closure
  │    │  // ctx.environmentVariables.set("MY_KEY", "hello")
  │    │    ├── invokeCapability("set", { dict: handle, key, value }) ──►
  │    │
  │◄── result: null  ──────────────────────────────────────────►│




7. How to Add a New Guest Language (e.g., Python)
To add Python support you need exactly two things on the server side, both discovered automatically from the NuGet package:
Step 1: Implement ILanguageSupport
// In new package: Aspire.Hosting.CodeGeneration.Python
public sealed class PythonLanguageSupport : ILanguageSupport
{
    public string Language => "python/cpython";

    public Dictionary<string, string> Scaffold(ScaffoldRequest request)
    {
        return new Dictionary<string, string>
        {
            ["apphost.py"] = "from aspire import create_builder\n\nbuilder = await create_builder()\nawait builder.build().run()\n",
            ["requirements.txt"] = "aspire-hosting>=1.0.0\n",
        };
    }

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
        Language = "python/cpython",
        DisplayName = "Python",
        CodeGenLanguage = "Python",
        DetectionPatterns = ["apphost.py"],
        InstallDependencies = new CommandSpec { Command = "pip", Args = ["install", "-r", "requirements.txt"] },
        Execute = new CommandSpec { Command = "python", Args = ["{appHostFile}"] },
        WatchExecute = new CommandSpec
        {
            Command = "watchdog",
            Args = ["watchmedo", "auto-restart", "--patterns=*.py", "--recursive", "--", "python", "{appHostFile}"]
        }
    };
}


Step 2: Implement ICodeGenerator
public sealed class PythonCodeGenerator : ICodeGenerator
{
    public string Language => "Python";

    // Receives the AtsContext (all capabilities, handles, DTOs, enums)
    // Returns a dict of filename → file content
    public Dictionary<string, string> Generate(AtsContext context)
    {
        var sb = new StringBuilder();
        sb.AppendLine("# aspire.py - Generated SDK (do not edit)");
        sb.AppendLine("import json, socket");
        sb.AppendLine("from aspire_transport import AspireClient, Handle");
        sb.AppendLine();

        foreach (var capability in context.Capabilities)
        {
            // Generate typed Python method for each capability
            GenerateCapabilityMethod(sb, capability);
        }

        return new Dictionary<string, string> { [".modules/aspire.py"] = sb.ToString() };
    }
}


Step 3: Register via DI (one line in the NuGet package's hosting extension)
// In Aspire.Hosting.CodeGeneration.Python startup
services.AddSingleton<ILanguageSupport, PythonLanguageSupport>();
services.AddSingleton<ICodeGenerator, PythonCodeGenerator>();


The CLI and AppHost server automatically discover these implementations via DI from the loaded assemblies. No CLI changes needed at all — adding the NuGet package is the only integration step.


8. Summary Table

Concern	Owned by	Mechanism
Socket path distribution	CLI	REMOTE_APP_HOST_SOCKET_PATH env var
Language detection	ILanguageSupport.Detect() on server	CLI calls detect RPC
Run command	ILanguageSupport.GetRuntimeSpec() on server	CLI calls getRuntimeSpec RPC
Scaffolding new project	ILanguageSupport.Scaffold() on server	CLI calls scaffoldAppHost RPC
SDK code generation	ICodeGenerator on server	CLI calls generateCode RPC
Guest → Host calls	ATS JSON-RPC invokeCapability	Guest connects to Unix socket
Host → Guest callbacks	ATS JSON-RPC invokeCallback	Reverse direction on same connection
Handle lifecycle	AppHost server handle registry	Valid until server exits
All Aspire integrations	.NET server process	Guest sees them as typed SDK methods

User: help me set up the local development mode for aspire
GitHub Copilot: Let me check the current state of the repo and relevant setup files.
Read , lines 1 to 80
Ran terminal command: cat global.json
Ran terminal command: ls .dotnet 2>/dev/null && echo "SDK exists" || echo "SDK not installed"
Ran terminal command: ls -la .dotnet 2>/dev/null | head -5; echo "exit: $?"
Ran terminal command: .dotnet/dotnet --version 2>/dev/null || echo "dotnet not found in .dotnet/"
Good — the local SDK is already installed. Let me check if the repo has been built before and what state the artifacts are in.
Ran terminal command: ls artifacts/bin 2>/dev/null | head -10; echo "---"; ls artifacts/packages 2>/dev/null | head -5
The repo has already been built and the local SDK is installed. Here's the full local dev setup:


Local Development Setup Status
Already done:

✅ Local .NET SDK installed (dotnet → 10.0.103)
✅ Repo previously built (bin populated)
What's already available:
# Use this instead of system `dotnet` to get the local SDK
./dotnet.sh --version   # → 10.0.103




Daily Workflow
Launch VS Code correctly
Always launch VS Code from the repo root using the helper script — this sets the correct PATH so VS Code terminals automatically use the local SDK:
./start-code.sh


Don't open VS Code directly via code . — that won't have .dotnet on PATH.
Build after code changes
# Full build (restore + build, ~3-5 minutes)
./build.sh

# Build only (skip restore, faster after first run)
./build.sh --build

# Skip native AOT compilation (~1-2 minutes saved)
./build.sh --build /p:SkipNativeBuild=true


Run the sample app to verify setup
./dotnet.sh run --project playground/TestShop/TestShop.AppHost/TestShop.AppHost.csproj


This starts the TestShop distributed app and opens the Dashboard — confirms everything is wired up correctly.
Run tests for a specific project
./dotnet.sh test tests/Aspire.Hosting.Tests/Aspire.Hosting.Tests.csproj -- \
  --filter-not-trait "quarantined=true" --filter-not-trait "outerloop=true"




If You Rebuild from Scratch
# 1. Re-install local SDK (if .dotnet/ gets corrupted)
./restore.sh

# 2. Full clean build
./build.sh --rebuild




Extension Development (VS Code Extension)
If you're working on the VS Code extension under extension:
cd extension
npm install
npm run build


Then use start-code.sh from the repo root and press F5 to launch the Extension Development Host.
User: how does the Aspire Code generator work and why it needs to trigger the SDK generation when first run with aspire cli?
GitHub Copilot: Read
Read , lines 200 to 900
Read , lines 150 to 1100
Read , lines 1 to 100
Compacted conversation
Read , lines 100 to 235
Read , lines 1 to 200
Read , lines 200 to 250
Searched for regex GenerateCodeAsync|generateCode|codegen-hash|modules (**/src/Aspire.Cli/**/*.cs), 19 results
Read , lines 1 to 300
Read , lines 300 to 500
Read , lines 1010 to 1120
Searched for regex NeedsCodeGen|codegen-hash|ComputePackagesHash|\.modules|GeneratedFolder (**/src/Aspire.Cli/Projects/GuestAppHostProject.cs), 15 results
Searched for regex NeedsCodeGeneration|codegen-hash|PrepareAsync (**/src/Aspire.Cli/Projects/*.cs), 14 results
Read , lines 80 to 200
Read , lines 420 to 480
Searched for regex codegen-hash|ComputePackagesHash|GenerationHash|HashChanged|ShouldRegenerate (**/src/Aspire.Cli/**/*.cs), 5 results
Searched for regex NeedsCodeGen|codegen|hash|modules|\.modules (**/src/Aspire.Cli/Projects/DotNetBasedAppHostServerProject.cs), 20 results
Read , lines 135 to 460
Searched for regex ShouldRegenerateCode|ReadGenerationHash|\.codegen-hash|modules.*exist|Directory\.Exists.*modules (**/src/Aspire.Cli/**/*.cs), 2 results
Searched for text generateCode (**/src/Aspire.Hosting/Ats/*.cs), 1 result
Searched for regex generateCode|GenerateCode|ICodeGenerator (**/src/Aspire.Hosting/**/*.cs), 2 results
Searched for regex generateCode|GenerateCode|ICodeGenerator (**/src/Aspire.Hosting/RemoteHost/*.cs), no results
Read
Searched for files matching **/src/Aspire.Hosting/RemoteHost/*.cs, no matches
Searched for text generateCode (**/src/**/*.cs), 20 results
Read , lines 1 to 200
Searched for text AtsContextFactory (**/src/Aspire.Hosting.RemoteHost/**/*.cs), 13 results
Read
Now I have a complete end-to-end picture. Here's the full explanation:


How the Aspire Code Generator Works
The Core Problem: No Runtime Discovery
Guest languages (TypeScript, Python, etc.) can't use .NET reflection at runtime. Every capability (addRedis, withEnvironment, etc.) that comes from a NuGet package must be represented as statically-typed code in the guest language before user code can call it. That's what SDK generation produces: the .modules/aspire.ts (and base.ts, transport.ts) files that TypeScript user code imports.
The Two-Phase Split: Where Code Lives
Server side (AppHost Server process):
AssemblyLoader → AtsContextFactory → AtsCapabilityScanner → AtsContext
                                                              ↓
                                               CodeGenerationService.generateCode()
                                                              ↓
                                                    ICodeGenerator.GenerateDistributedApplication()
                                                              ↓
                                               Dictionary<string, string>  (filename → content)



AtsContextFactory.cs lazily calls AtsCapabilityScanner.ScanAssemblies() over all loaded assemblies
AtsCapabilityScanner.cs does 5-pass scanning: collect all [AspireExport] methods/types → resolve unknown types → filter invalid ones → expand interface targets to concrete types → filter overload collisions
[CodeGenerationService.cs](src/Aspire.Hosting.RemoteHost/CodeGeneration/CodeGeneration Service.cs) exposes the generateCode JSON-RPC method, which calls the registered ICodeGenerator for the requested language
ICodeGenerator.cs is the contract: GenerateDistributedApplication(AtsContext) → Dictionary<string, string>
Client side (CLI):

GuestAppHostProject.cs orchestrates everything: build the server → start it → call GenerateCodeAsync via RPC → write files to .modules/ → save .codegen-hash
AppHostRpcClient.cs sends {"method": "generateCode", "params": ["TypeScript"]} over the Unix socket
Why It Must Run on First Use
When you run aspire run against a TypeScript AppHost:

CLI scaffolds a hidden .NET project in $TMPDIR/.aspire/hosts/<hash>/ with a Program.cs that is just await RemoteHostServer.RunAsync(args), plus a .csproj referencing every Aspire NuGet package listed in aspire.json
CLI builds it — this is the PrepareAsync step in DotNetBasedAppHostServerProject.cs. On success, it returns NeedsCodeGeneration: true
CLI starts the server — the server binary loads all those Aspire assemblies into the process
CLI calls generateCode via RPC — AtsContextFactory lazily scans all loaded assemblies (including e.g. Aspire.Hosting.Redis.dll) for [AspireExport] attributes, building an AtsContext that describes every capability; then AtsTypeScriptCodeGenerator turns that into aspire.ts, base.ts, transport.ts
CLI writes the files to .modules/ and saves a .codegen-hash (SHA-256 of sorted packageId:version pairs) for future cache invalidation
CLI starts the guest (npx tsx apphost.ts) — the TypeScript code can now import { createBuilder } from './.modules/aspire.js'
The SDK cannot be committed to the repo because it is derived from whatever NuGet packages are installed — if you run aspire add redis, the Redis package is added, and on the next run aspire run detects the hash has changed and regenerates so addRedis() appears in the TypeScript SDK.
The aspire sdk generate Command
SdkGenerateCommand.cs is a stand-alone variant for integration library authors. Usage: aspire sdk generate ./MyIntegration.csproj -l typescript -o ./output
It runs the same pipeline in isolation: create a temp AppHost server project → add your .csproj as an additionalProjectReference → build → connect → call GenerateCodeAsync → write files → delete temp dir. This lets integration authors generate typed SDKs from their integration libraries for publishing to npm/PyPI.
Hash-Based Cache: When Regeneration Is Skipped
The .modules/.codegen-hash file stores the SHA-256 of packageId:version;packageId:version;... (sorted). Currently, PrepareAsync always returns NeedsCodeGeneration: true after a successful build, so generation happens every run. The hash is written after generation and is the foundation for future skip logic when the hash matches.
The 5-Pass Scanner
AtsCapabilityScanner.ScanAssemblies() runs these passes across all assemblies:

Collect — per-assembly scan for [AspireExport] on methods and types, [AspireDto] on DTOs, gathers all enum types from signatures
Resolve Unknown types — any capability parameter that references a type not in the exported set is marked Unknown
Filter invalid — capabilities with unresolved Unknown types are dropped (with diagnostics)
Expand interface targets — withEnvironment<T> where T : IResourceWithEnvironment becomes a capability entry for every concrete type that implements IResourceWithEnvironment; ExpandedTargetTypes on each AtsCapabilityInfo is pre-computed here
Filter overloads — C# allows overloaded methods; ATS capability IDs are unique, so duplicate IDs after expansion emit errors and are removed
The resulting AtsContext (capabilities, handle types, DTOs, enums) is then passed to ICodeGenerator.GenerateDistributedApplication() which templates out the guest-language SDK files.
User: how to tell what assemblies are loaded in the aspire cli?
GitHub Copilot: Searched for text AssemblyLoader (**/src/Aspire.Hosting.RemoteHost/**/*.cs), 17 results
Read
The assembly list is driven by two things:
1. AtsAssemblies in appsettings.json
When the CLI builds the hidden AppHost server project, it writes an appsettings.json into the temp project directory listing every assembly to scan. You can see this in DotNetBasedAppHostServerProject.cs — it always starts with "Aspire.Hosting" and then appends every package name from aspire.json:
{
  "AtsAssemblies": [
    "Aspire.Hosting",
    "Aspire.Hosting.Redis",
    "Aspire.Hosting.CodeGeneration.TypeScript"
  ]
}


AssemblyLoader reads this array at startup and calls Assembly.Load(name) for each — these are simple by-name loads, so the assembly must already be on the probing path (i.e., built into the server's output directory as a dependency).
2. ASPIRE_INTEGRATION_LIBS_PATH env var (prebuilt / bundled mode)
For the prebuilt CLI bundle (no local .NET SDK), PrebuiltAppHostServer restores the NuGet packages to a temp folder and passes that path via ASPIRE_INTEGRATION_LIBS_PATH. The AssemblyLoader registers a custom AssemblyLoadContext.Default.Resolving handler that probes that directory for DLLs when a name-based load fails.
How to see what's actually loaded

Look at the appsettings.json the CLI generates in $TMPDIR/.aspire/hosts/<hash>/appsettings.json — that's the exact list
Enable Debug log level for the server (set "Aspire.Hosting.RemoteHost": "Debug" in the appsettings) to see "Loaded assembly: {AssemblyName}" log lines from AssemblyLoader
Use aspire sdk dump (SdkDumpCommand.cs) which calls the getCapabilities RPC — you can infer what assemblies scanned by what capability IDs appear (e.g., Aspire.Hosting.Redis/addRedis means Aspire.Hosting.Redis was loaded)
User: why would CLI need to communicate with the AppHost server via RPC?
GitHub Copilot: Read
The RPC between CLI and AppHost server exists for a fundamental reason: the CLI is a native AOT-compiled binary that cannot dynamically load arbitrary .NET assemblies, but ALL the language-specific knowledge lives inside NuGet packages that are only present in the server process.
Here's the breakdown of why each RPC call can't be done in the CLI directly:

RPC method	Why the CLI can't do it locally
getRuntimeSpec	RuntimeSpec (how to run npx tsx, nodemon, etc.) is returned by ILanguageSupport implementations that come from NuGet packages (Aspire.Hosting.CodeGeneration.TypeScript). The CLI doesn't reference those packages.
scaffoldAppHost	Scaffold templates (apphost.ts, package.json, etc.) are embedded resources inside those same NuGet packages, accessed via ILanguageSupport.Scaffold().
generateCode	Code generation requires AtsCapabilityScanner to reflect over loaded integration assemblies (Aspire.Hosting.Redis.dll, etc.). The CLI can't load those — only the server can, because it was built with them as project/package references.
getCapabilities	Same — needs reflection over NuGet package assemblies to enumerate [AspireExport] attributes.

There's also a process separation reason: the AppHost Server is the DCP orchestration runtime (it manages containers, services, the dashboard). The CLI is the user-facing shell. Keeping them in separate processes means the CLI can monitor the server, restart it, and display its exit code without being the same process.
The three-process architecture (CLI ↔ AppHost Server ↔ Guest Runtime) is intentional:
CLI (native AOT, thin orchestrator)
  │── JSON-RPC over Unix socket ──→ AppHost Server (.NET, loads all NuGet packages)
                                         │── JSON-RPC over Unix socket ──→ Guest (TypeScript/Python)


The CLI is evolving toward a "pure orchestrator" that knows nothing about specific languages or integrations — it just asks the server what to do and follows its instructions (the RuntimeSpec).
User: how is the provisioning library being used? and who generates bicep files? who deploy the bicep files?
GitHub Copilot: Searched for regex AzurePublishingContext|BicepCompiler|\.bicep|WriteBicep|GenerateBicep (**/src/**/*.cs), 20 results
Searched for files matching **/src/Aspire.Hosting.Azure/Bicep*.cs, 1 match
Searched for files matching **/src/Aspire.Hosting.Azure/AzureBicepResource.cs, 1 match
Searched for regex AzurePublishingContext|PublishAsync.*azure|DeployAsync|azd|azure developer cli (**/src/**/*.cs), 20 results
Searched for regex AzureProvisioningResource|infrastructure\.Build|GetBicepTemplateString|WriteToManifest (**/src/Aspire.Hosting.Azure/**/*.cs), 20 results
Read , lines 1 to 180
Read , lines 120 to 180
Searched for regex ProvisionAzureBicepResourceAsync|ArmDeployment|DeployBicep|azure-mgmt (**/src/Aspire.Hosting.Azure/**/*.cs), 20 results
Read , lines 150 to 285
Read , lines 1 to 300
Searched for regex AzureProvisioningResource.*GetBicepTemplateString|infrastructure\.Build\(\)|\.Compile\(\)|BicepCompiler (**/src/Aspire.Hosting.Azure/**/*.cs), 12 results
Searched for files matching **/src/Aspire.Hosting.Azure/AzureProvisioningResource.cs, 1 match
Read , lines 1 to 120
Read , lines 1 to 80
Here is the full picture of how provisioning works, from C# code all the way to deployed Azure resources:


The Three-Layer Stack
Layer 1: Azure.Provisioning SDK → Bicep text (C# code in your AppHost)
Azure.Provisioning.* NuGet packages (Azure.Provisioning.Redis, Azure.Provisioning.PostgreSql, etc.) are a C#-first IaC SDK. They let you describe Azure resources in C# using types like CdkRedisResource, RoleAssignment, BicepValue<T>, BicepFunction.Interpolate().
When AzureProvisioningResource.GetBicepTemplateFile() is called:
var infrastructure = new AzureResourceInfrastructure(this, Name);
ConfigureInfrastructure(infrastructure);      // user's callback, e.g., ConfigureRedisInfrastructure
var plan = infrastructure.Build(options);
var compilation = plan.Compile();             // Azure.Provisioning SDK → Bicep text string
File.WriteAllText(moduleSourcePath, compiledBicep.Value);


The ConfigureInfrastructure callback is where each integration (e.g., AzureRedisExtensions.cs) builds up the C# object graph — create a CdkRedisResource, configure AAD auth, add ProvisioningOutput for connection strings, etc. The SDK's Compile() call turns that into Bicep text. No Bicep files are pre-authored; they are all generated at runtime from C#.
Layer 2: Bicep text → ARM JSON (the bicep CLI)
BicepCliCompiler.CompileBicepToArmAsync() in BicepCompiler.cs shells out to bicep build --stdout (or az bicep build as fallback). This converts the generated .module.bicep file into an ARM template JSON string. The bicep binary is either the standalone Bicep CLI or Azure CLI's extension — it must be on PATH.
Layer 3: ARM template → Azure deployment (Azure Resource Manager)
BicepProvisioner.GetOrCreateResourceAsync() in BicepProvisioner.cs submits the ARM JSON to Azure using the Azure Management SDK:
var deployments = resourceGroup.GetArmDeployments();   // Azure.ResourceManager SDK
var operation = await deployments.CreateOrUpdateAsync(
    WaitUntil.Started, deploymentName,
    new ArmDeploymentContent(new(ArmDeploymentMode.Incremental)
    {
        Template = BinaryData.FromString(armTemplateContents),
        Parameters = BinaryData.FromObjectAsJson(parameters),
    }), cancellationToken);
await operation.WaitForCompletionAsync(cancellationToken);


This is a standard ARM API call — it creates a named deployment in the Azure resource group and polls until it completes. Deployment outputs (connection strings, host names, etc.) are read back and stored in resource.Outputs.


Two Different Deployment Paths
Path A: aspire run (local dev provisioning)
When you run locally with Azure resources, the provisioning pipeline runs inside the AppHost process:
AzureBicepResource constructor → adds PipelineStepAnnotation → pipeline step calls ProvisionAzureBicepResourceAsync → BicepProvisioner.GetOrCreateResourceAsync() → ARM deployment.
Deployment state is cached in ~/.aspire/deployments/<hash>.json (or similar) so re-runs skip provisioning if the checksum hasn't changed.
Path B: azd up (Azure Developer CLI deployment)
AzureBicepResource.WriteToManifest() writes to the Aspire manifest (JSON) that azd reads:
{
  "type": "azure.bicep.v0",
  "path": "redis.module.bicep",
  "params": { "location": "{azure.location}" }
}


azd reads this manifest, takes the .bicep files that were written alongside it, and runs its own ARM deployment pipeline. Aspire does not deploy in this path — it just produces the manifest and .bicep files; azd handles the actual az deployment group create.


Summary

Who	What
Azure.Provisioning SDK	Generates Bicep text from C# DSL (infrastructure.Build().Compile())
BicepCliCompiler	Shells out to bicep/az bicep to compile Bicep → ARM JSON
BicepProvisioner	Submits ARM JSON to Azure via Azure.ResourceManager SDK, polls for completion
azd	Alternative deployer — reads Aspire manifest + .bicep files, runs its own deployment

User: how provisioning library communicate with AppHost server?
GitHub Copilot: Searched for regex PipelineStep|PipelineStepAnnotation|ResourceNotificationService|ProvisioningContext (**/src/Aspire.Hosting.Azure/**/*.cs), 20 results
Searched for files matching **/src/Aspire.Hosting/Pipelines/*.cs, 20 matches
Searched for text PipelineStepAnnotation (**/src/Aspire.Hosting/Pipelines/DistributedApplicationPipeline.cs), 1 result
Read , lines 120 to 285
Read , lines 60 to 390
Read , lines 1 to 100
The provisioning library does not communicate with the AppHost server via RPC. They run in the same process. Here's the full picture:


The Provisioning Library Is Part of the AppHost Process
There is no RPC boundary between the provisioning library and the "AppHost server." The Aspire.Hosting.Azure provisioning code is loaded as a regular .NET assembly inside the AppHost process itself. Communication is via in-process .NET calls through the dependency injection container and the Pipeline system.
How It Flows: The Pipeline
AzureBicepResource registers itself into the Pipeline via annotations in its constructor:
AzureBicepResource constructor
  ├── ManifestPublishingCallbackAnnotation → WriteToManifest() (for azd manifest generation)
  ├── PipelineStepAnnotation → creates "provision-{name}" step with Action = ProvisionAzureBicepResourceAsync
  └── PipelineConfigurationAnnotation → wires up dependency ordering between Azure resources


DistributedApplicationPipeline (in Aspire.Hosting.Pipelines) walks all resources in the app model, collects their PipelineStepAnnotations, sorts them topologically, and executes them in order.
When the "provision-redis" step runs, ProvisionAzureBicepResourceAsync resolves services directly from DI:
private static async Task ProvisionAzureBicepResourceAsync(PipelineStepContext context, AzureBicepResource resource)
{
    var bicepProvisioner = context.Services.GetRequiredService<IBicepProvisioner>();  // DI, in-process
    var azureEnvironment = context.Model.Resources.OfType<AzureEnvironmentResource>().First();
    var provisioningContext = await azureEnvironment.ProvisioningContextTask.Task;   // TaskCompletionSource

    await bicepProvisioner.GetOrCreateResourceAsync(resource, provisioningContext, ...);
}


The ProvisioningContext comes from AzureEnvironmentResource.ProvisioningContextTask — another pipeline step ("create-provisioning-context") runs first and completes that TaskCompletionSource with the Azure credentials/subscription/resource group. Individual Bicep steps DependsOn that step, so they wait for it.
Communication Channels (all in-process):

From → To	Mechanism
Pipeline executor → BicepProvisioner	IServiceProvider.GetRequiredService<IBicepProvisioner>()
BicepProvisioner → Dashboard/UI	ResourceNotificationService.PublishUpdateAsync() (in-memory channel)
BicepProvisioner → bicep CLI	Process.Start("bicep build ...") (subprocess, not RPC)
BicepProvisioner → Azure ARM API	Azure.ResourceManager SDK over HTTPS
One Bicep resource → another (ordering)	TaskCompletionSource on AzureEnvironmentResource.ProvisioningContextTask + PipelineStep.DependsOn()

The only external communication provisioning does is:

Shelling out to the bicep or az CLI binary to compile Bicep → ARM JSON
HTTPS to Azure Resource Manager (management.azure.com) to submit the ARM deployment
User: if provisioning library can generate bicep and deploy via bicep , why we still need guest language runtime to communicate with AppHost dotnet server via json rpc to callback capbilities?
GitHub Copilot: They serve completely different phases. The JSON-RPC capability system is how the guest language builds the app model. The provisioning library consumes that model after it's built. One cannot replace the other.
The Sequence
TypeScript apphost.ts (running as Guest)
    │
    │ JSON-RPC: invokeCapability("Aspire.Hosting/createBuilder", {})
    ├──────────────────────────────────────> AppHost Server (.NET)
    │ <── { $handle: "1" }                   IDistributedApplicationBuilder created in memory
    │
    │ JSON-RPC: invokeCapability("Aspire.Hosting.Redis/addRedis", {builder: {$handle:"1"}, name:"cache"})
    ├──────────────────────────────────────> RedisResource added to DistributedApplicationModel
    │ <── { $handle: "2" }
    │
    │ JSON-RPC: invokeCapability("Aspire.Hosting/build", {builder: {$handle:"1"}})
    ├──────────────────────────────────────> DistributedApplication.Build() called
    │
    │ JSON-RPC: invokeCapability("Aspire.Hosting/run", {app: {$handle:"3"}})
    ├──────────────────────────────────────> app.RunAsync() starts
    │                                            │
    │                                            ├── DcpExecutor: spin up containers
    │                                            ├── Pipeline: provision Azure resources
    │                                            │     └── BicepProvisioner → bicep CLI → ARM API
    │                                            └── Dashboard: show resource states


The guest language's JSON-RPC calls are the equivalent of Program.cs in a .NET AppHost. They are what populate the DistributedApplicationModel with resources. Without them, the model is empty — there is nothing for the provisioning library or DCP executor to act on.
The Analogy
Think of it this way:

A .NET AppHost — the user writes builder.AddRedis("cache") in C#, which directly calls the in-process AddRedis() extension method, adding a RedisResource to the model
A TypeScript AppHost — the user writes await builder.addRedis("cache") in TypeScript, which sends a JSON-RPC call to the .NET server, which runs the same AddRedis() method in-process on behalf of the guest
In both cases the outcome is identical: a RedisResource lands in the DistributedApplicationModel. The provisioning pipeline doesn't know or care which language authored the model.
Why Not Generate Bicep From the Guest Directly?
The provisioning library (Bicep generation, Azure SDK, ARM deployment) has deep dependencies on:

Azure.Provisioning.* NuGet packages with .NET-only types (ProvisionableResource, BicepValue<T>, etc.)
Azure.ResourceManager SDK for ARM API calls
System.Reflection for scanning [AspireContainerImage], role assignments, etc.
A TypeScript process cannot run any of that. It can only invoke capabilities on the .NET server, which then does the actual work.
Summary: The JSON-RPC capabilities are the authoring API — the guest builds the resource graph. The provisioning library is the execution engine — it acts on that graph. They are sequential, not competing.
User: how does the guest build the resource graph via AppHost json rpc? what input does the provisioning library need in order to generate bicep files to really create the resources?
GitHub Copilot: Read , lines 1 to 100
Read , lines 1 to 120
Read , lines 120 to 300
Read , lines 100 to 260
Read , lines 200 to 300
Searched for regex RemoteAppHostService|invokeCapability|InvokeCapability (**/src/Aspire.Hosting.RemoteHost/**/*.cs), 15 results
Read , lines 140 to 380
Read
Searched for regex AzureResourcePreparer|ConfigureInfrastructure|DistributedApplicationModel.*azure (**/src/Aspire.Hosting.Azure/**/*.cs), 20 results
Read , lines 1 to 120
Now I have the full picture. Here is the end-to-end answer:


Phase 1: The Guest Builds the Resource Graph via JSON-RPC
The key insight is that JSON-RPC calls are direct reflection-based invocations of real .NET methods. There is no separate protocol translation layer — the dispatcher just calls the actual C# method.
The invocation chain
TypeScript: await builder.addRedis("cache")
    │
    │  JSON-RPC: invokeCapability("Aspire.Hosting.Redis/addRedis", {"builder": {$handle:"1"}, "name":"cache"})
    ▼
RemoteAppHostService.InvokeCapabilityAsync()
    └── CapabilityDispatcher.InvokeAsync("Aspire.Hosting.Redis/addRedis", args)
            │
            │  RegisterFromCapability() bound this capability ID to the real MethodInfo
            │  of AddRedis() at startup (via AtsCapabilityScanner reflection scan)
            │
            ├─ Unmarshal args:
            │     "builder" → HandleRegistry.TryGet("1") → returns the actual IDistributedApplicationBuilder object
            │     "name"    → JsonNode → string "cache"
            │
            ├─ Invoke via reflection:
            │     method.Invoke(null, [builder, "cache"])
            │     == RedisExtensions.AddRedis(builder, "cache")   ← the real C# extension method
            │
            └─ Marshal result:
                  IResourceBuilder<RedisResource> → HandleRegistry.Register(obj) → handle "2"
                  Return: {"$handle": "2", "$type": "Aspire.Hosting.Redis/...RedisResource"}


The HandleRegistry is a per-client ConcurrentDictionary<string, object> that stores live .NET objects by ID. When TypeScript sends {$handle:"1"} back in the next call, the dispatcher resolves it to the actual IDistributedApplicationBuilder instance.
The result: after the TypeScript guest calls createBuilder → addRedis → build → run, the AppHost process has a fully populated DistributedApplicationModel with a real RedisResource in its Resources collection — identical to what a Program.cs in a .NET AppHost would produce.


Phase 2: What the Provisioning Library Reads From the Model
AzureResourcePreparer.GetAzureResourcesFromAppModel() scans the populated model and finds each IAzureResource. For a Redis resource added by AddRedis().AsAzureManagedRedis(), the model object is an AzureManagedRedisResource which is an AzureProvisioningResource (which is an AzureBicepResource).
The provisioning library needs two things from each AzureProvisioningResource:
1. ConfigureInfrastructure callback — a Action<AzureResourceInfrastructure> set at resource creation time. When GetBicepTemplateFile() is called, it runs the callback:
ConfigureInfrastructure(infrastructure);   // → adds CdkRedisResource, ProvisioningOutput, etc.
var plan = infrastructure.Build();
var compilation = plan.Compile();          // → Bicep text string


This callback was closed over C# objects when AddRedis() ran — it carries the resource configuration directly.
2. Parameters dictionary — key/value pairs that become Bicep param declarations. These include:

location (Azure region)
principalId / principalType for managed identity role assignments
References to other resources' outputs (e.g., keyVaultUri)
These are populated by BicepUtilities.SetParametersAsync() at deploy time, resolving IManifestExpressionProvider references, ParameterResource values (from user prompts), and BicepOutputReference cross-resource references.


The Complete Picture
TypeScript apphost.ts
    │ JSON-RPC: invokeCapability("Aspire.Hosting.Redis/addRedis", ...)
    ▼
CapabilityDispatcher
    │ Reflection: RedisExtensions.AddRedis(builder, "cache", port: null)
    │             ↳ new AzureManagedRedisResource("cache", configureInfra)
    │             ↳ builder.Resources.Add(redisResource)
    ▼
DistributedApplicationModel.Resources
    └── AzureManagedRedisResource
            ├── ConfigureInfrastructure: (infra) => {
            │       var redis = new CdkRedisResource(...);
            │       infra.Add(redis);
            │       infra.Add(new ProvisioningOutput("connectionString", ...));
            │   }
            ├── Parameters: { "location": "{azure.location}", ... }
            └── PipelineStepAnnotation: → ProvisionAzureBicepResourceAsync

                            ↓ (when app.Run() is called)

AzureResourcePreparer.OnBeforeStartAsync()
    └── Scans model, builds role assignment annotations

Pipeline: "provision-redis" step
    └── BicepProvisioner.GetOrCreateResourceAsync()
            ├── resource.GetBicepTemplateFile()
            │       └── ConfigureInfrastructure(infra) → plan.Compile() → redis.module.bicep
            ├── BicepCliCompiler: "bicep build redis.module.bicep" → ARM JSON
            └── resourceGroup.GetArmDeployments().CreateOrUpdateAsync(armJson)
                    → Azure REST API → Redis Cache created in Azure
                    → Outputs (connectionString) stored in resource.Outputs["connectionString"]


User: introduce more about DistributedApplicationModel and the architecture overview?
GitHub Copilot: Compacted conversation
Searched codebase for "DistributedApplicationModel class definition resources annotations", 24 results
Searched for files matching **/DistributedApplicationModel.cs, 1 match
Searched for files matching **/IResourceCollection.cs, 1 match
Read
Read , lines 1 to 150
Read , lines 150 to 500
Searched for regex class.*Resource.*IResource|ContainerResource|ProjectResource|ExecutableResource (**/src/Aspire.Hosting/ApplicationModel/*.cs), 20 results
Read
Read , lines 500 to 700
Read , lines 1 to 80
Read
Here's a comprehensive overview:


DistributedApplicationModel — The Application Graph
DistributedApplicationModel is the central data structure of an Aspire AppHost. It is a trivially simple object with one purpose: holding the flat list of all resources.
// src/Aspire.Hosting/ApplicationModel/DistributedApplicationModel.cs
[DebuggerDisplay("Resources = {Resources.Count}")]
public class DistributedApplicationModel(IResourceCollection resources)
{
    public IResourceCollection Resources { get; } = resources;
}


IResourceCollection is just IList<IResource> — a flat list, not a tree. The graph structure (the DAG of dependencies) is implicit, encoded in annotations and ReferenceExpression objects attached to resources. Aspire never stores an explicit edge list; edges are discovered at runtime by traversing annotations.
It's registered as a singleton in DI. Everything — DcpExecutor, AzureResourcePreparer, BicepProvisioner, manifest writers, the dashboard — gets it from DI and queries model.Resources.


IResource — The Fundamental Unit
public interface IResource
{
    string Name { get; }
    ResourceAnnotationCollection Annotations { get; }
}


That's the entire interface. Two properties. Resources are pure inert data objects — they don't start, stop, or manage themselves. Everything about their runtime behavior is carried in Annotations.
The abstract Resource base class just validates the name and provides the collection:
IResource
  └── Resource (abstract)
        ├── ContainerResource        — runs a Docker image
        ├── ProjectResource          — runs a .NET project
        ├── ExecutableResource       — runs an arbitrary binary
        ├── ParameterResource        — an external input / secret
        ├── ConnectionStringResource — a literal conn string
        └── [integration-specific]
              ├── RedisResource       : ContainerResource
              ├── PostgresServerResource : ContainerResource
              ├── AzureProvisioningResource : AzureBicepResource
              └── ...


The three primitive compute types are what DCP (DcpExecutor) knows about natively. Everything else is modeled on top of them.


Annotations — The Extensibility Mechanism
IResourceAnnotation is an empty marker interface. Any POCO implementing it can be attached to any resource:
// All of these are annotations
ContainerImageAnnotation        // → image name, tag, registry
EndpointAnnotation              // → port, scheme, name
EnvironmentCallbackAnnotation   // → lambda to inject env vars
ContainerMountAnnotation        // → volume or bind mount
HealthCheckAnnotation           // → health probe config
PipelineStepAnnotation          // → deploy pipeline step
ManifestPublishingCallbackAnnotation // → how to write to azd manifest
WaitAnnotation                  // → WaitFor dependency


Extension methods like AddRedis("cache") are really just:

new RedisResource("cache") (a ContainerResource subclass)
resource.Annotations.Add(new ContainerImageAnnotation { Image = "redis", Tag = "7.4" })
resource.Annotations.Add(new EndpointAnnotation(ProtocolType.Tcp, ..., targetPort: 6379))
Wrap in an IResourceBuilder<RedisResource> and return
The IResourceBuilder<T> is just a fluent wrapper:
IDistributedApplicationBuilder     (top-level, owns model + DI)
  └── .AddRedis("cache")
        └── IResourceBuilder<RedisResource>
              ├── .Resource          → the RedisResource instance
              ├── .WithEnvironment() → adds EnvironmentCallbackAnnotation
              ├── .WithDataVolume()  → adds ContainerMountAnnotation
              └── .WithHealthCheck() → adds HealthCheckAnnotation




Standard Capability Interfaces
Resources opt into behaviors by implementing interfaces (polymorphism over type checks):

Interface	What it enables
IResourceWithEnvironment	.WithEnvironment(), env var injection
IResourceWithEndpoints	.WithEndpoint(), GetEndpoint(), URL generation
IResourceWithConnectionString	.WithReference(db) wires conn string into consumers
IResourceWithServiceDiscovery	Registers service in DNS-style discovery
IResourceWithArgs	CLI argument injection
IResourceWithWaitSupport	WaitFor() — blocks startup until dependency is ready
IResourceWithParent	Lifecycle containment (child stops when parent stops)
IComputeResource	Marks as a compute unit (project/container/exec)

ContainerResource implements nearly all of them. ParameterResource implements almost none (it's just a value, not a runnable thing).


The Value/Reference System — How the DAG Forms
The graph is a heterogeneous DAG encoding both dependency order and value flows. Edges are encoded as structured value objects:
web ──► EndpointReference ──► api
                               │
                               └──► ConnectionStringReference ──► postgres


Key types:

Type	Run mode	Publish mode
EndpointReference	http://localhost:5000	{api.bindings.http.url}
ConnectionStringReference	Host=localhost;Port=5432;...	{postgres.connectionString}
ParameterResource	value from user secrets / env	${PARAM} placeholder
ReferenceExpression	interpolated concrete string	interpolated manifest expression

ReferenceExpression is the key glue type. It wraps an interpolated string handler that captures structured value objects, not their resolved values:
// This does NOT resolve "localhost:5432" at build time.
// It stores the reference to pg.PrimaryEndpoint for deferred evaluation.
public ReferenceExpression ConnectionStringExpression =>
    ReferenceExpression.Create(
        $"Host={pg.PrimaryEndpoint.Property(EndpointProperty.Host)};" +
        $"Port={pg.PrimaryEndpoint.Property(EndpointProperty.Port)}");


At run time: IValueProvider.GetValueAsync() resolves all references to concrete strings.
At publish time: IManifestExpressionProvider.ValueExpression emits {pg.bindings.tcp.host} placeholders into the azd manifest.


The Full Architecture Overview
┌─────────────────────────────────────────────────────┐
│                   AppHost Process                   │
│                                                     │
│  Program.cs                                         │
│    var builder = DistributedApplication             │
│                    .CreateBuilder(args);            │
│    builder.AddPostgres("pg");        ─────────────► IResourceCollection
│    builder.AddProject("api")                        │  [PostgresServerResource]
│           .WithReference(pg);        ─────────────► │  [ProjectResource]
│    builder.Build().Run();                           │     └─Annotations:
│                                                     │       EnvironmentCallback(
│  DistributedApplicationModel ◄──────────────────────┤         "ConnectionStrings__pg"
│    .Resources: IResourceCollection                  │          = pg.ConnectionStringExpr)
│                                                     │
│  ─────────── Lifecycle/Orchestration ────────────── │
│                                                     │
│  DcpExecutor                                        │
│    scans Resources for annotations                  │
│    → creates DCP CRDs (Container, Exe, AppExec...)  │
│    → KubernetesService.CreateAsync(crd)             │
│    → watches for state changes                      │
│                                                     │
│  AzureResourcePreparer (OnBeforeStartAsync)         │
│    scans Resources for IAzureResource               │
│    → BicepProvisioner.GetOrCreateResourceAsync()    │
│      → resource.ConfigureInfrastructure(infra)      │
│      → infra.Build().Compile() → Bicep text         │
│      → az deployment create → ARM API               │
│      → reads Outputs back into resource             │
│                                                     │
│  ResourceNotificationService                        │
│    receives state updates from DCP watcher          │
│    → publishes snapshots to Dashboard (gRPC stream) │
│                                                     │
│  ─────────── Publish Mode (aspire publish) ──────── │
│                                                     │
│  ManifestPublisher                                  │
│    scans Resources for ManifestPublishingCallback   │
│    → each resource writes its own JSON section      │
│    → ReferenceExpressions emit {placeholder} text   │
│                                                     │
│  ─────────── Remote Guest AppHost (polyglot) ─────  │
│  JsonRpcServer (Unix socket)                        │
│    CapabilityDispatcher                             │
│      ← invokeCapability "IDistributedApplicationBuilder/AddRedis"
│      → HandleRegistry.TryGet(builderHandle)         │
│      → RedisExtensions.AddRedis(builder, "cache")   │
│      → stores new RedisResource in IResourceCollection
│      → returns handle to new IResourceBuilder       │
└─────────────────────────────────────────────────────┘




Key Insight: Inert Data + External Orchestration
The design philosophy is:

DistributedApplicationModel is a passive data store — just the resource list
Resources are pure data objects — annotation bags with a name
All behavior (starting, stopping, deploying, manifest-writing, logging) lives in external services that scan the model and react to annotations
This means: anything that needs to do something with resources just takes DistributedApplicationModel from DI, iterates model.Resources, checks for specific annotations or interfaces, and acts accordingly. New capabilities (like a new deployment target or a new dashboard widget) can be added by creating a new annotation type + a new service that scans for it — zero changes to the core model.
User: how is the resource graph being used in generating the bicep files? Or they didn't use the resource graph?
GitHub Copilot: Optimizing tool selection...The resource graph (the DistributedApplicationModel.Resources list) is directly used to generate Bicep files for Azure provisioning.
How it works:

The provisioning system (e.g., AzureResourcePreparer, BicepProvisioner) scans the resource graph for resources that represent Azure infrastructure (those implementing IAzureResource or having AzureBicepResourceAnnotation).
For each such resource, it calls the resource’s ConfigureInfrastructure callback, which describes how to build the Bicep model using the resource’s configuration and relationships.
The infrastructure builder (AzureResourceInfrastructure) accumulates all these resource definitions and their dependencies.
Finally, infra.Build().Compile() emits the Bicep text, which is then deployed to Azure.
Summary:
The resource graph is the source of truth for what gets included in the Bicep file. Only resources present in the graph and recognized as Azure resources are translated into Bicep definitions. The graph is not just used—it is essential for Bicep generation.
User: how is the resource graph being used in generating the bicep files? Or they didn't use the resource graph?
GitHub Copilot: Searched codebase for "resource graph bicep generation", 20 results
Searched for regex ResourceGraph|resource graph (**/*.cs), 20 results
Searched for regex resource graph|ResourceGraph|GetResources|application model (**/src/Aspire.Hosting.Azure/**/*.cs), 20 results
Read , lines 1 to 150
Read , lines 1 to 200
Read , lines 1 to 150
Read , lines 150 to 350
Searched for text AzureResourceInfrastructure (**/src/Aspire.Hosting.Azure/**/*.cs), 20 results
The resource graph (the dashboard's UI visualization component) is not used in Bicep generation. Here's how Bicep files are actually produced:
Per-resource, self-contained generation (AzureProvisioningResource.GetBicepTemplateFile)
Each AzureProvisioningResource independently compiles its own Bicep module by invoking ConfigureInfrastructure callback on an AzureResourceInfrastructure instance, then using the Azure.Provisioning SDK to build and compile it:
ConfigureInfrastructure(infrastructure)  →  infrastructure.Build()  →  plan.Compile()  →  .bicep file


This is entirely local to each resource — no graph traversal.
Cross-resource wiring (AzurePublishingContext.WriteModelAsync)
When publishing (e.g., aspire publish), AzurePublishingContext.cs iterates the flat list of AzureBicepResources in the application model and resolves cross-resource dependencies by chasing BicepOutputReference → moduleMap[b.Resource] lookups. Outputs from one module are wired as parameters to another via module.Parameters.Add(...). This is resolved during publishing but doesn't use a graph algorithm — it's just dictionary lookups after flattening.
The one place "graph walking" does happen
In AzureResourcePreparer.cs, BuildRoleAssignmentAnnotations does walk the resource graph (appModel.GetComputeResources() → GetAzureReferences(resource)) to discover which compute resources reference which Azure resources. But this is only for setting up role assignments (IAM/RBAC) — not for generating Bicep content.
Summary: Bicep content is generated per-resource using the Azure.Provisioning SDK, and cross-resource parameter/output wiring is handled through a flat moduleMap dictionary in AzurePublishingContext. The resource graph visualization in the dashboard is entirely separate and unrelated.
User: How is App Host related to provisioning SDK? why generating bicep needs to have a AppHost server and communicate with dotnet AppHost server?
GitHub Copilot: Searched codebase for "AppHost server publish bicep communicate DCP", 24 results
Searched for regex publish.*apphost|apphost.*publish|WriteModelAsync|PublishingContext (**/src/**/*.cs), 20 results
Read , lines 1 to 100
Read , lines 120 to 350
Read , lines 100 to 190
Searched for regex backchannel|BackChannel|IAppHostCliBackchannel (**/src/Aspire.Hosting/**/*.cs), 20 results
Searched for regex publish.*pipeline|pipeline.*publish|PipelineStep|PublishAsync (**/src/Aspire.Cli/**/*.cs), 11 results
Searched for text pipeline (**/src/Aspire.Hosting/Pipelines/**/*.cs), 20 results
Searched for files matching **/src/Aspire.Hosting/Pipelines/*.cs, 20 matches
Read , lines 100 to 220
Searched for regex DistributedApplicationOperation|publish.*mode|PublishMode|IsPublishMode (**/src/Aspire.Hosting/**/*.cs), 20 results
Searched for regex --operation|operation.*publish|DOTNET_ASPIRE_OPERATION (**/src/Aspire.Hosting/**/*.cs), 20 results
Read , lines 200 to 420
Read , lines 120 to 160
This is a nuanced architectural point. The App Host is where Bicep generation actually happens — the CLI just launches it.
Here's the full picture:
Why the AppHost must run
The App Host is a C# program that imperatively builds the resource model at runtime. Things like:
builder.AddAzureKeyVault("kv")
    .ConfigureInfrastructure(infra => {
        // This is a C# lambda/closure — not serializable
        var kv = infra.GetProvisionableResources().OfType<KeyVault>().Single();
        kv.Properties.EnablePurgeProtection = true;
    });


The ConfigureInfrastructure callback is a live C# delegate. There's no way to statically analyze it or execute it without actually running the App Host process. The full DistributedApplicationModel only exists in memory after the App Host starts up.
The publish flow end-to-end
aspire publish (CLI)
    │
    ├─► launches: dotnet run AppHost.csproj --operation publish --publisher azure
    │       ↓
    │   AppHost starts in PublishMode (DcpHost skipped, no containers, no dashboard)
    │   ↓
    │   Pipeline is assembled from PipelineStepAnnotations registered by each resource
    │   ↓
    │   AzureEnvironmentResource.PublishAsync() fires
    │   ↓
    │   AzurePublishingContext.WriteModelAsync(model, environment)
    │       ├─ iterates model.Resources (the in-memory, fully materialized resource list)
    │       ├─ calls resource.GetBicepTemplateFile() → runs ConfigureInfrastructure()
    │       └─ writes .bicep files to disk
    │
    └─► backchannel (Unix socket, JSON-RPC)
            ← App Host sends progress updates back to the CLI for display


The backchannel (in Backchannel) is a Unix socket JSON-RPC channel for progress reporting only — the CLI streams task completion events from it and displays them. The actual Bicep generation all happens inside the App Host process.
What the CLI does
In PipelineCommandBase.cs, the CLI:

Launches the App Host with --operation publish
Waits for the backchannel to be established
Streams GetPublishingActivitiesAsync() → prints progress/status
Waits for the App Host process to exit
Why not just have the SDK generate Bicep directly?
Because the Azure provisioning SDK (Azure.Provisioning.*) only knows about Bicep primitives — it has no concept of Aspire parameters, cross-resource BicepOutputReferences, or connection string expressions. The AzurePublishingContext in AzurePublishingContext.cs is what glues them together, and it requires the live DistributedApplicationModel to do so. That model only exists after the App Host has run its startup code.
Short answer: The App Host's Program.cs code is the configuration. You need to execute it to know what resources exist and how they're wired together, so Bicep generation runs inside the App Host process, with the CLI just orchestrating launch and displaying progress.
User: how to debug the aspire run for a guest AppHost server and how to debug the bicep generation
GitHub Copilot: Searched codebase for "guest AppHost debug WaitForDebugger attach process", 21 results
Searched for regex debug|WaitForDebugger|--debug|debugMode (**/src/Aspire.Cli/**/*.cs), 20 results
Read
Read , lines 120 to 340
Searched for regex debug|Debug|wait-for-debugger (**/src/Aspire.Cli/Projects/PrebuiltAppHostServer.cs), 9 results
Searched for regex ASPIRE_APPHOST_DEBUG|DOTNET_WAIT_FOR_DEBUGGER|UnixSocketPath|socketPath (**/src/Aspire.Cli/**/*.cs), 20 results
Searched for regex GetBicepTemplateFile|GetBicepTemplateString|ConfigureInfrastructure (**/src/Aspire.Hosting.Azure/**/*.cs), 20 results
Read , lines 210 to 320
Here's a practical guide to debugging each layer:


Debugging the Guest AppHost Server
The two processes involved:

AppHost Server — the prebuilt .NET process (Aspire.Hosting.RemoteHost) that runs as the JSON-RPC server
Guest AppHost — the TypeScript/Python/etc process that executes the user's code via GuestRuntime
1. Enable verbose CLI output
aspire run --debug --project apphost.ts


--debug sets Logging__LogLevel__Default=Debug on the AppHost server (PrebuiltAppHostServer.cs) and enables debug-level CLI logging. All stdout/stderr from both processes is forwarded to the CLI log.
2. Attach a debugger to the AppHost Server (.NET process)
Use --wait-for-debugger to pause the process at startup:
aspire run --wait-for-debugger --project apphost.ts


This sets ASPIRE_WAIT_FOR_DEBUGGER=true, which causes DistributedApplication.WaitForDebugger() to spin in a loop printing the PID. You then attach to that PID in Visual Studio / VS Code with "Attach to Process". There's a 30-second timeout (configurable via ASPIRE_WAIT_FOR_DEBUGGER_TIMEOUT).
3. Attach a debugger to the Guest process
The guest process (node, python, etc.) is started by GuestRuntime.ExecuteCommandAsync() — it's a plain child process with no special wait loop. To debug it:

Add a language-level debugger break at the start of your AppHost code (e.g., debugger; in TypeScript, breakpoint() in Python)
Then attach the language-specific debugger to the process ID printed in the CLI output (use --debug to see it)
4. Log file
All AppHost server stdout/stderr is written to:
<log-dir>/apphost.log


(The log dir is printed by the CLI on startup when --debug is used)


Debugging Bicep Generation
Bicep generation happens inside the AppHost process during PublishAsync → WriteModelAsync. There are several approaches:
1. Run publish with --debug and inspect the temp files
aspire publish --debug --output-path ./infra


The --debug flag makes the CLI print all activity details. The generated .bicep files are written to the output path so you can inspect the final result.
2. Set a breakpoint directly in GetBicepTemplateFile
Since Bicep generation is in-process (inside the running AppHost), attach the .NET debugger and set a breakpoint in AzureProvisioningResource.cs:
GetBicepTemplateFile()
  → ConfigureInfrastructure(infrastructure)   ← your lambda runs here
  → infrastructure.Build(ProvisioningBuildOptions)  ← Azure.Provisioning SDK
  → plan.Compile()                              ← produces Bicep text


Use --wait-for-debugger with aspire publish:
aspire publish --wait-for-debugger --project AppHost.csproj


3. Unit test Bicep generation directly
The easiest approach — you don't need to start any server:
var builder = DistributedApplication.CreateBuilder(
    new DistributedApplicationOptions { Args = ["--operation", "publish"] });

var myResource = builder.AddAzureKeyVault("kv");
var app = builder.Build();

// Force Bicep generation and inspect the output
var resource = (AzureProvisioningResource)myResource.Resource;
var bicep = resource.GetBicepTemplateString();
Console.WriteLine(bicep);


This is what the test projects in Aspire.Hosting.Azure.Tests do — they call GetBicepTemplateString() directly without any CLI or server involvement and compare the output with snapshots via Verify.
4. Inspect intermediate ProvisioningPlan
The plan variable in GetBicepTemplateFile (line 90) is the ProvisioningPlan from Azure.Provisioning. You can inspect it under the debugger — it exposes the full list of ProvisionableResource objects, their properties, and the module structure before compilation.
