# Your Service Mesh Is Not Your Network: Understanding Consul vs Submariner

A practical look at where service mesh ends and multi-cluster networking begins in enterprise Kubernetes.

Kubernetes platforms have evolved far beyond simply running containers.
Most enterprise environments today operate across:
• multiple clusters
• multiple regions
• hybrid cloud
• on-prem infrastructure
• regulated network zones
• and increasingly complex application ecosystems

As these environments scale, platform engineering also becomes layered:
• service meshes
• multi-cluster networking
• zero trust platforms
• API gateways
• overlay networking
• service discovery
• cross-cluster routing

Eventually, a question shows up in almost every architecture discussion:
“If we already have Consul or Istio, do we still need something like Submariner?”

At a distance, the confusion makes sense. All of these platforms appear to solve cross-cluster communication in some form.
But underneath, they solve very different problems.

Recently, I was part of a discussion where a consulting engineering team strongly pushed the idea that Consul could fully replace Submariner. The proposal sounded attractive on paper: fewer moving parts, centralized governance, and a cleaner cloud-native architecture model.

The conversation itself was valuable because it exposed one of the most common misunderstandings in modern Kubernetes platforms:
“Service mesh and network connectivity are not the same thing.”

And when those layers get blurred together, architecture decisions start becoming oversimplified very quickly.

# The Real Problem Enterprises Are Solving

Most organizations do not stay with a single Kubernetes cluster forever.
Over time, environments naturally expand into:
• production clusters
• non-production clusters
• isolated regulatory zones
• DR regions
• shared platform clusters
• edge clusters
• hybrid cloud
• air-gapped environments

Eventually, workloads in one cluster need to communicate with workloads in another.
That communication may involve:
• microservices
• databases
• Kafka
• replication traffic
• legacy middleware
• shared infrastructure services
• observability systems
• internal APIs

That communication requirement introduces two completely different architectural concerns:
1. Network Connectivity
Can the clusters themselves communicate?
2. Service Connectivity
How should applications securely communicate?

This is where tools like Submariner, Consul, and Istio start entering architecture conversations.
The mistake happens when teams assume they solve the same problem.
They do not.

# Understanding What Submariner Actually Does

Submariner is fundamentally a multi-cluster networking solution.
Its purpose is straightforward:
Extend network connectivity between Kubernetes clusters.

Think of Submariner as:
• network extension
• cross-cluster routing
• secure tunnels
• pod-to-pod connectivity

Submariner primarily operates at:
• Layer 3 (routing)
• Layer 4 (TCP/UDP connectivity)

Think of it as infrastructure plumbing between clusters.

# What Submariner Provides

## Cross-Cluster Pod Connectivity

A pod running in Cluster A can directly communicate with a pod running in Cluster B without the application itself understanding anything about service meshes.
From the application’s perspective, it behaves like routable network infrastructure.

## Secure Network and Cross-Cluster Tunnels

Submariner creates:
• IPSec tunnels
• WireGuard tunnels
• encrypted routing paths
between clusters.

This is infrastructure networking.
Not service discovery.
Not application governance.
Not Layer 7 traffic management.

## Protocol-Agnostic Connectivity

This becomes extremely important in enterprise environments.
Not every workload:
• runs HTTP
• supports sidecars
• participates in a mesh
• or was designed for cloud-native patterns

Examples include:
• databases
• Kafka
• LDAP
• storage systems
• replication traffic
• middleware
• internal TCP services
• operational tooling

Submariner works well because these applications simply see normal network connectivity without requiring:
• sidecars
• service mesh participation
• HTTP awareness
• or application modernization

## Globalnet Support

One of the most important enterprise capabilities Submariner provides is solving overlapping CIDR problems.
Example:
Cluster A → 10.0.0.0/16
Cluster B → 10.0.0.0/16

Normally this breaks routing.
Submariner’s Globalnet capability solves this through address translation and network abstraction.

This becomes critical in:
• acquisitions
• hybrid cloud
• large enterprises
• independently managed Kubernetes environments

# Understanding What Consul Actually Does

Consul solves a very different problem.
Consul primarily operates at:
• the service layer
• the application communication layer
• the security and policy layer

Its focus is:
• service discovery
• service governance
• traffic control
• zero trust communication

Not how cluster networks connect.

# What Consul Provides

## Service Discovery

Applications communicate using logical service names instead of directly managing IP addresses.

## mTLS Between Services

Consul automatically encrypts service-to-service communication.
This is especially important in:
• financial institutions
• regulated environments
• zero trust architectures

## Traffic Management

Consul provides Layer 7 capabilities such as:
• retries
• failover
• canary deployments
• traffic splitting
• service intentions
• observability

These are service governance features.
Not network routing features.

## Identity-Based Security

Consul enables workload identity and policy-driven communication between services.
Again, this is service-layer governance.
Not foundational cluster networking.

# Where Architecture Conversations Usually Go Wrong

This is the pattern I often see during platform modernization discussions.
A consulting or vendor engineering team deploys:
• Consul
• Istio
• or another service mesh

They demonstrate:
• cross-cluster service communication
• encrypted service traffic
• service discovery

Immediately, the assumption becomes:
“We no longer need Submariner.”

Most proof-of-concepts focus on:
• HTTP services
• sidecar-enabled applications
• mesh-participating workloads

But real enterprise environments also contain:
• databases
• replication traffic
• operational tooling
• legacy middleware
• stateful systems
• raw TCP services

That distinction matters a lot once platforms move beyond demo environments and into production operations.

# The Simplest Way to Think About It

Submariner answers:
“Can these cluster networks communicate?”

Consul answers:
“How should services securely communicate?”

# Why This Matters in Real Enterprise Platforms

In real enterprise platforms, especially regulated environments, infrastructure is rarely greenfield.
Reality usually includes:
• modern APIs
• legacy applications
• shared services
• middleware
• stateful systems
• batch workloads
• hybrid connectivity
• compliance segmentation

A pure “everything goes through the mesh” strategy sounds elegant until:
• a database requires replication
• Kafka needs low-level TCP connectivity
• a storage system depends on flat networking
• a legacy application cannot support sidecars
• an air-gapped environment limits mesh federation
• operational tooling needs direct routing semantics

At that point, organizations rediscover why foundational networking still matters.

# The Architecture Most Enterprises Eventually Land On

The most stable enterprise pattern is usually layered architecture.

## Network Layer

Use:
• Submariner
• or another multi-cluster networking solution
for:
• cluster connectivity
• pod routing
• infrastructure traffic
• raw TCP/UDP communication

## Service Layer

Use:
• Consul
• Istio
• or another service mesh
for:
• mTLS
• traffic policy
• observability
• zero trust governance
• service-level routing

# The Bigger Lesson

One thing platform engineering teaches over time is this:
Every infrastructure layer eventually tries to absorb adjacent responsibilities.

Service meshes try to become networking platforms.
Ingress controllers try to become API gateways.
CI/CD systems try to become developer portals.
Observability platforms try to become AIOps engines.

The challenge for enterprise architects is understanding where those boundaries should remain intentionally separated.
Because operational clarity matters more than architectural hype.

# Final Thoughts

This is not really a “Consul vs Submariner” discussion.
Both platforms solve important problems. They simply operate at different layers of the Kubernetes stack.

Submariner focuses on:
• cluster connectivity
• routing
• infrastructure communication

Consul focuses on:
• service discovery
• secure service-to-service communication
• traffic governance

In many enterprise environments, both layers still matter.
The most resilient Kubernetes platforms are rarely built by forcing every problem into a single abstraction layer. They are built by understanding where each layer belongs, where responsibilities begin, and where they should intentionally stop.

Because in large-scale platform engineering, architectural clarity almost always outlives architectural hype.
