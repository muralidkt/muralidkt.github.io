---
title: "How to Choose an Agent Runtime"
author: Murali Kotharamban
pubDatetime: 2026-10-11T00:00:00.000Z
featured: false
draft: false
tags:
  - AI
  - Agents
  - Platform Engineering
  - Agent Runtime
  - Security
description: "A practical guide to agent hosting models, eight focused selection criteria, and the tradeoffs between managed runtimes and general-purpose compute."
---

After building agent systems on several runtimes, I’ve learned that getting an agent to run is only the beginning. The harder question—and one worth asking before the first deployment—is whether that runtime fits how the agent will actually behave in production.

The first version often looks like an ordinary API: receive a request, call a model, invoke a few tools, and return a response. Then a larger task outlasts the request. A user closes the browser halfway through. A deployment replaces the process. An approval arrives the following morning.

These are the moments when the choice of runtime starts to matter.

By **agent runtime**, I mean the hosting environment in which the agent application executes: the compute, lifecycle, and platform services that keep it operating. Choosing that environment begins with understanding how the work needs to live.

## Start With the Shape of the Work

Consider an **invoice-handling agent for a small plumbing business**. The owner uploads a supplier invoice and asks: “Does this match what we ordered and received?” The agent can search order confirmations, supplier quotes, delivery slips, and credit notes, then prepare a bill for the owner to approve and record in the accounting system.

The interesting cases involve incomplete or conflicting information. An invoice lists twelve fittings, but the delivery slip shows ten. The agent might look for a second delivery. If it finds none, it may check for a credit note covering the missing items. A price discrepancy could send it to the agreed quote. If the records cannot explain the difference, it asks the owner a specific question.

Each result determines what the agent checks next:

```text
invoice + available evidence + remaining budget
                     |
             choose the next step
                     |
       validate permissions and limits
                     |
        +------------+-------------+
        |                          |
  fetch a document          prepare recommendation
  or ask a question         or report missing evidence
        |
  record the result -> reassess -> next iteration
```

The model chooses where to look and whether another check is useful. Application rules validate amounts, enforce permissions and limits, and require approval before recording a bill. The agent can leave a discrepancy unresolved instead of guessing. This combination of a flexible investigation and explicit controls is the behaviour the runtime must support.

There are three broad ways to host that application:

| Hosting model            | How execution works                                                                              | Main tradeoff                                                                                          |
| ------------------------ | ------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------ |
| Invocation-based compute | A request or event starts a bounded unit of work. Longer tasks can be dispatched asynchronously. | Convenient scaling, with execution limits and restart behaviour that the application must accommodate. |
| Managed agent sessions   | A service provisions an environment for related agent interactions and manages its lifecycle.    | Session and platform integration, within the service’s invocation and lifecycle contracts.             |
| Application services     | Your agent runs in a service or worker process on containers or VMs.                             | Control over process behaviour and deployment, with more application and operational responsibilities. |

These models can be combined. An API might accept an uploaded invoice, a worker might host the agent, and the owner’s reply might resume it after a missing delivery detail is supplied.

For this agent, checking documents and waiting for the owner have different lifetimes. One invoice may need two checks; another may require several searches and a question answered the next morning. Measure representative cases, and preserve the evidence and open questions across waits. The process need not remain alive while the owner is out on a job.

With that distinction established, the runtime comparison becomes more concrete.

## Eight Questions That Narrow the Choice

These are eight decision areas, not a universal ranking. Start with hard requirements: the application must run, reach its dependencies, meet its execution deadlines, and satisfy its access and isolation needs. Then compare viable designs on performance and operating effort. A private-network requirement may settle one team’s choice; startup latency may settle another’s.

For each question, distinguish what the runtime provides from what your application must implement. Storage and retries are platform mechanisms. Recovering the correct business action is a property of the complete system.

### 1. How Long Must Execution Stay Alive?

Separate the lifetime of the client connection from the lifetime of the task. An HTTP request can time out while background work continues, or a platform may terminate work when the request ends. Streaming a response does not establish which behaviour applies.

For the invoice agent, measure processing time across straightforward invoices and cases needing several document searches. Compare that with the runtime’s request deadline, maximum execution time, and idle shutdown policy. An asynchronous design needs a durable way to retrieve progress and the recommendation after the original connection closes.

The requirement to write down is precise: **an accepted invoice check must continue independently of the owner’s browser connection**, within explicit wall-clock, tool-call, and cost budgets.

### 2. What Survives a Restart?

A **run** is the business task; a **worker** is the process currently executing it. One run may need several workers over its lifetime. Keeping conversation history in memory or routing requests to the same instance does not make that run recoverable.

The invoice agent needs its documents, completed checks, open questions, and remaining budget after a restart. A particularly difficult window is the accounting system accepting an approved bill just before the worker dies. If the external bill ID was not recorded, blindly retrying may create a duplicate. Persist the approved action first, then check the accounting system after an uncertain response; request help when the outcome cannot be determined safely.

Compare persistent storage options, retry behaviour, shutdown signals, and the time allowed to drain work during replacement. Those are runtime selection criteria. Your application uses them to preserve progress and resume safely; it still owns the idempotency or reconciliation needed for external actions.

### 3. How Does Identity Reach the Agent?

An authenticated caller, a workload credential, and permission to access a particular business’s records are three different things. The runtime may verify the caller’s token and provide credentials to the process. Your application must still decide which resources the caller may use.

For the invoice agent, reading a supplier quote and creating a bill are different permissions. Scope credentials to the business and permitted actions. The application must validate the owner’s approval and recheck access when execution resumes. A model’s choice of tool does not grant permission to use it.

Managed identity integration can save substantial engineering work. Its value depends on whether it fits the permissions your agent actually needs, including the relationship between a user and their runs.

### 4. What Is the Isolation Boundary?

Decide whether users share an application process or receive separate execution environments. A shared service can use resources efficiently, but mutable context and user credentials must remain scoped to each run. A dedicated environment provides more separation while adding allocation and lifecycle costs.

Also examine resource isolation. A large batch of scanned invoices or repeated document searches might consume excessive memory or occupy every available API connection. Per-run concurrency limits and bounded resource usage help prevent it from degrading other users’ work.

If the service supports several businesses, keep each business’s documents and credentials within its access boundary, and specify how much shared capacity one invoice check may consume. A session identifier alone does not establish those boundaries.

### 5. Can It Run the Application You Need?

Check the real application’s language version, native dependencies, CPU architecture, memory footprint, and process requirements. A small agent that mostly waits for remote APIs has different needs from a service that loads substantial libraries or maintains background processes.

This is where hosting abstractions become concrete. An isolate-based environment and a Linux container offer different capabilities. Even a service that accepts your container image may require a particular architecture, port, health endpoint, or invocation protocol.

Deploy the actual dependency set early. Measure startup and memory usage with realistic context sizes. The relevant compute requirement comes from the hosted application; calling a large remote model does not itself require a GPU in the agent host.

### 6. Can It Reach the Required Systems?

Map the connections from the runtime to model APIs, private tools, databases, and result storage. Private routing, regional placement, outbound restrictions, and credential endpoints can eliminate an otherwise attractive hosting option.

For the invoice agent, trace access to the document store, accounting API, and any document-reading service. Sensitive business records may be available through authenticated public endpoints or require private connectivity. Completion time depends on these calls, as well as startup and local processing.

Test the complete route, including gateways and proxies. A host may support streaming while an intermediary buffers the response. A reachable tool may still stall indefinitely unless the application sets a deadline.

### 7. Can It Meet Your Latency Target Under Load?

Measure time to first progress and completion latency for both cold and warm execution. Under load, queueing and slow scale-out can dominate those numbers. Adding workers also increases pressure on model quotas and internal APIs, potentially causing failures and more retries.

A useful starting estimate is:

```text
average active runs ≈ runs entering execution per second
                    × average active duration in seconds
```

As an illustrative calculation, six runs starting per minute and remaining active for three minutes means roughly 18 active runs in steady state. Count time spent waiting on model and tool responses while execution remains allocated; a wait for the owner’s reply persisted with its worker released does not occupy that capacity. Queueing and tool concurrency budgets should be designed around this workload, then tested under bursts.

Compare p95 latency at the expected burst size, along with capacity limits and warm-instance controls. The runtime determines how compute becomes available; your application must admit work at a rate its dependencies can sustain.

### 8. What Will It Cost to Run and Maintain?

Agents often spend much of their execution time waiting for models and tools. Inspect whether billing follows allocated resources, active CPU, elapsed invocation time, or another unit. The same workload can have very different economics under those models.

Include warm capacity, storage, networking, telemetry, retries, and engineering effort. Make that effort concrete: determine how your team will inspect a failed run, cancel active work, deploy safely, and roll back. A service with these capabilities may justify a higher compute price. An existing platform may be economical because the team already operates it well.

For the invoice agent, estimate cost per processed invoice using measured document reads, tool calls, active compute, and model usage. Include unresolved cases and work stopped at a budget limit. A cheap execution that repeatedly searches the same documents without resolving or clearly explaining a discrepancy is still wasted spend.

## How the Options Compare

The questions above describe requirements. The products below package different subsets of the capabilities needed to meet them. AgentCore Harness is included as a managed layer on AgentCore Runtime: choosing it delegates more of the agent’s execution behaviour to AWS. This is an architectural comparison, with provider details checked in October 2026.

| Option                                  | Where it fits                                                  | Main advantage                                                         | Main tradeoff                                                                                                      |
| --------------------------------------- | -------------------------------------------------------------- | ---------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| AWS AgentCore Runtime                   | Agents that benefit from managed sessions and AWS integration  | Session hosting with authentication and private-network integration    | You adopt its deployment and lifecycle contracts and still own application recovery and user permissions.          |
| AWS AgentCore Harness on Runtime        | Agents that fit a configurable managed loop                    | Managed loop and configured integrations on the Runtime infrastructure | You adopt the harness’s execution model; custom environment support does not mean control of the agent process.    |
| Modal Functions or Servers              | Asynchronous agent work or a custom agent service              | Managed compute with configurable environments                         | The hosting primitive determines invocation, timeout, retry, and concurrency behaviour.                            |
| Cloudflare Workers with Durable Objects | Agents organised around events and persistent per-entity state | Routing and coordination within the application platform               | Isolate limits and object lifecycle shape the application; persistent storage does not preserve a running process. |
| Cloudflare Containers                   | Agent services needing a Linux environment within Cloudflare   | Container processes coordinated through Durable Objects                | Readiness, instance lifetime, and persistence need explicit handling.                                              |
| Container services such as ECS/Fargate  | Teams with an established container platform                   | Familiar deployment, networking, and workload identity                 | Session routing, run admission, and recovery remain application or platform responsibilities.                      |
| Team-operated Kubernetes or VMs         | Workloads requiring greater infrastructure control             | Control over placement, scheduling, and topology                       | More responsibility for capacity, upgrades, deployment, and host or cluster operations.                            |

For **Modal**, the choice between Functions and Servers matters. Functions document retries after certain container failures; Servers do not automatically retry failed requests. Evaluate the primitive that matches your application’s execution shape. [Modal Functions](https://modal.com/docs/guide/functions), [Modal Servers](https://modal.com/docs/guide/servers).

For **Cloudflare**, distinguish application state from live compute. Durable Objects can be evicted or hibernate while their persistent storage remains. Containers have a separate start, readiness, and stop lifecycle. Those behaviours determine how an agent reconnects and resumes. [Durable Object lifecycle](https://developers.cloudflare.com/durable-objects/concepts/durable-object-lifecycle/), [Container lifecycle](https://developers.cloudflare.com/containers/concepts/architecture/), [Workers limits](https://developers.cloudflare.com/workers/platform/limits/).

**AgentCore Runtime** illustrates the tradeoff between managed integration and control. It offers IAM SigV4 or configured JWT authentication, outbound identity capabilities, and VPC connectivity. That can reduce integration work for an AWS-based platform. It also accepts custom application containers, so flexibility needs to be assessed against its specific deployment and lifecycle requirements. [Authentication](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-oauth.html), [VPC connectivity](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/agentcore-vpc.html), [Container requirements](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/getting-started-custom.html).

**AgentCore Harness** adds a managed agent loop on top of that runtime. You configure the model, instructions, tools, and execution limits; AWS runs the loop. Runtime hosting gives you control of your application’s loop, while Harness takes on that responsibility. That makes Harness worth evaluating when its supported behaviour fits the agent you want to operate. [AWS’s Harness and Runtime comparison](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/harness-vs-runtime.html).

There is a concrete deployment consequence: Harness supports custom container environments, but overrides their `ENTRYPOINT` and `CMD`. Your installed dependencies are available; your existing application startup command is not what launches the agent. This is a useful example of environment flexibility and execution control being separate choices. [Harness environment contract](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/harness-environment.html).

Managed integration still leaves application permission decisions to you. AgentCore explicitly leaves the mapping between users and runtime sessions to the backend. Conventional ECS applications can also use task roles for AWS access. The practical advantage of managed agent hosting is the integration work it takes on; your application’s permission decisions remain essential. [AgentCore sessions](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-sessions.html), [ECS task roles](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-iam-roles.html).

## Open-Source Options to Try

There are also open-source projects you can explore locally or deploy on your own infrastructure. Some provide agent hosting; others supply a harness or the controls around execution. Knowing which layer a project covers helps you choose a useful starting point.

| Project                                                       | Licence         | What you can explore                                                                                                                                                                                                       |
| ------------------------------------------------------------- | --------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [Agyn](https://github.com/agynio/platform)                    | AGPL-3.0        | A Kubernetes-based agent runtime platform that runs agent containers and manages their lifecycle. A candidate when you want to operate your own agent hosting platform.                                                    |
| [AgentScope 2.0](https://github.com/agentscope-ai/agentscope) | Apache-2.0      | A self-hosted agent service with APIs, sessions, and persistence. Your application integrates user authentication. [Service documentation](https://docs.agentscope.io/latest/en/deploy/agent-service#user-authentication). |
| [TrueForge](https://github.com/truefoundry/trueforge)         | MIT             | A deployable agent harness and server with an API, UI, sessions, and approvals. Useful for trying a configurable agent loop; hosted deployments require supporting infrastructure such as PostgreSQL and Redis.            |
| [Deep Agents](https://github.com/langchain-ai/deepagents)     | MIT             | An agent harness with planning, context management, and tool integration. It supplies agent behaviour; you choose the hosting environment separately.                                                                      |
| [Preloop](https://github.com/preloop/preloop)                 | Apache-2.0 core | Governance through model and tool gateways, approvals, and observability, plus event-driven flows and runners. Explore it for controlling agent execution and its supported deployment paths.                              |

If you encounter the standalone **AgentScope Runtime** repository, its maintainers now direct users to AgentScope 2.0 and plan to archive the older project. Start with the current project when evaluating it. [Migration notice](https://github.com/agentscope-ai/agentscope-runtime#archive-notice).

These projects offer different starting points, so apply the same eight questions to the complete setup. For example, trying a harness locally demonstrates its agent behaviour; you still need to evaluate deployment, identity, recovery, and operating cost. Preloop likewise describes itself as a control plane with governed agent flows rather than a general-purpose hosting runtime. [Preloop’s runtime distinction](https://preloop.ai/vs/aws-agentcore).

For the invoice agent, use a small set of representative documents to try the relevant options. Self-hosting gives your team control over deployment, along with responsibility for the infrastructure, upgrades, and operational integrations it requires.

## Making the Decision for the Invoice Agent

Return to the workload: an agent chooses which business records to inspect, follows discrepancies, asks targeted questions, and prepares a recommendation for approval. Suppose the team building this service already operates Python applications in AWS and wants to reuse its identity and deployment practices.

That gives us a useful baseline: an API backed by workers on the existing platform. Compare it with AgentCore Runtime if managed sessions and identity integration would remove work the team currently owns. Bounded invocations also merit a trial if they can host the loop within the required limits, or resume it from persisted state. Waiting for the owner’s reply alone does not justify a continuously running process.

Harness is another path if its managed loop supports the tools, limits, approval, and resumption behaviour we require. Test that fit with real invoice cases. The shortlist should follow the workload and the team’s existing capabilities.

Whichever host we choose, a replacement worker needs the invoice, evidence references, completed checks, open questions, and remaining budget. When the owner supplies a missing delivery detail, resume with that answer as new evidence. Approval should identify the reviewed version of the proposed bill, so later changes require a fresh decision.

Try a matching invoice, a discrepancy explained by another delivery, and a case with missing evidence. Then test the difficult failure window: terminate a worker immediately after the accounting API accepts a bill. Recovery should find the existing record or flag uncertainty rather than create a duplicate. Also test cancellation and a deployment while a case awaits the owner’s reply. Correlate business, invoice, tool action, and worker attempt in the logs.

Keep the model, instructions, and tool access comparable across candidates. Measure processing latency, recovery, and cost when several invoices arrive together; track human waiting time separately. Check recommendations against the source documents. Reject designs that fail mandatory requirements, then compare the remaining integration and operating effort.

The decision should be explainable in a few sentences: this runtime meets our execution and connectivity requirements; it fits our identity model; we have demonstrated recovery; and we understand the responsibilities we still own.

That is what I now look for when choosing an agent runtime: a clear fit between the way the agent works and the way the platform keeps it running.
