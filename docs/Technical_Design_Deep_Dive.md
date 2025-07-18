# Technical Design Deep Dive

To extend the **Component Architecture** with deeper technical details, the following sections cover internal
communication patterns, deployment topology, runtime behaviors, interface contracts, and key technology
trade-offs. Each topic is explored with real-world considerations and concrete examples to guide
implementation.

**Internal Communication & Messaging Design**

All microservices communicate asynchronously via a message broker (RabbitMQ or Kafka) to decouple
components and provide backpressure. **Topics/Queues Structure:** Each stage in the workflow can use
distinct topics/queues with clear naming conventions. For example, a **trigger.events** topic might carry
commit or Jira trigger messages from the _Trigger Listener_ , a **test.plan** queue carries manifests from the
_Test Planner_ , and an **execution.results** queue carries test outcomes from _Executor Agents_. Using a topic
exchange in RabbitMQ, messages can be published with routing keys like planner.testPlan.created
or executor.testCase.finished, allowing each service’s queue to bind to only the events it needs

. In a Kafka-based design, separate topics (e.g. TestPlanCreated, TestResult) would serve a
similar role, logically partitioning the event stream.

**Event Payloads & Schemas:** Every message follows a defined schema (e.g. JSON schema or Avro) to serve
as a **data contract** between services. This ensures that producers and consumers share an understanding
of each event’s structure. For instance, a **Trigger Event** might be a JSON payload like:

##

```json
{
  "event": "TestTriggerReceived",
  "source": "Bitbucket",
  "repo": "mobile-app",
  "branch": "feature/login-tests",
  "commit": "a1b2c3d",
  "triggerType": "commit_annotation",
  "timestamp": "2025-06-24T11:52:00Z"
}
```
The _Trigger Router_ would enrich this and emit a **Test Plan event** (e.g. TestPlanCreated) containing a
manifest. A simplified **Test Manifest** example:

```json

{
  "event": "TestPlanCreated",
  "planId": "plan-1234",
  "tests": [
    {"id": "TC_001", "name": "LoginTest", "priority": "High"},
    {"id": "TC_002", "name": "PurchaseTest", "priority": "Medium"}
  ],
  "environment": { "target": "Staging", "browser": "Chrome" },
  "createdBy": "TestPlanner",
  "timestamp": "2025-06-24T11:53:00Z"
}
```

This manifest is consumed by the Execution Coordinator. After execution, an Execution Result event might look like:
```json

{
  "event": "TestPlanCreated",
  "planId": "plan-1234",
  "tests": [
    {"id": "TC_001", "name": "LoginTest", "priority": "High"},
    {"id": "TC_002", "name": "PurchaseTest", "priority": "Medium"}
  ],
  "environment": { "target": "Staging", "browser": "Chrome" },
  "createdBy": "TestPlanner",
  "timestamp": "2025-06-24T11:53:00Z"
}
```


This manifest is consumed by the _Execution Coordinator_. After execution, an **Execution Result** event might
look like:



```json
{
  "event": "TestResult",
  "planId": "plan-1234",
  "testId": "TC_001",
  "status": "PASSED",
  "durationSeconds": 45,
  "agent": "executor-3",
  "logUrl": "http://elk/logs/plan-1234/TC_001.log",
  "timestamp": "2025-06-24T11:55:30Z"
}
```
Such structured messages enable each service to process data without tight coupling, and they simplify
routing logic (e.g. the _Result Router_ can inspect "event": "TestResult" and forward to Xray or Jenkins
accordingly).

**Reliable Delivery & Retries:** The messaging layer will ensure at-least-once delivery. In RabbitMQ,
producers publish to an exchange and messages persist in queues until acknowledged by consumers.
Consumers should **acknowledge** messages only after successful processing. If a consumer fails or a
processing error occurs, the message is _not_ acknowledged – the broker will re-queue and redeliver it (or
another consumer can pick it up), ensuring no event is lost. To handle transient failures, implement a retry
mechanism with backoff at the message-processing level. One proven approach is using **delayed
requeueing** : e.g. configure a **delay exchange/queue** for retries. When a message fails, route it to a delay
queue that re-delivers after a short interval (e.g. 10 seconds), and increase the delay on subsequent failures

. After a maximum retry count or total delay, messages can be sent to a **Dead Letter Queue** (DLQ) for
manual inspection or automated cleanup. This pattern prevents endless re-processing of poison
messages while giving transient issues time to recover. For example, the _Execution Coordinator_ applying a
retry policy might catch a failure to schedule a test (e.g. no available agent), then publish the manifest to a
delay queue with a retry count increment. Meanwhile, all event payloads should include identifiers (like
planId or testId) to ensure idempotency – if the same message is delivered twice, the processing
service can detect duplicates (using a cache or database of processed IDs) to avoid double-processing in
rare edge cases.

**Deployment Topology (Docker & Kubernetes)**

Each component will be packaged as a Docker container for consistency and isolation. Using Docker
ensures that tests and services run in identical environments, eliminating “works on my machine” issues.
The overhead of containerization is minimal – containers share the host OS kernel and incur negligible performance cost compared to bare metal execution. For instance, an _Executor Agent_ container can
include all necessary test libraries and device drivers without polluting the host system, and it can be
replaced/upgraded independently.

For orchestration and scaling, **Kubernetes (K8s)** is recommended. All services (Trigger Listener, Router,
Planner, etc., along with the message broker itself) would be deployed as Kubernetes **Deployments** or
**StatefulSets**. This provides robust scheduling, self-healing, and scaling out of the box. Each microservice
runs in its own Pods (with one container per Pod for simplicity), and can be replicated as needed.
Kubernetes will monitor each container’s health via **liveness and readiness probes** – for example, the
_Execution Coordinator_ could expose a /health endpoint. If a probe fails, Kubernetes will restart the
container or remove it from service until it’s healthy. This ensures fault recovery: a crashed component
is automatically brought back.

**Service Discovery:** Within the cluster, services find each other using K8s DNS service names. For instance,
the RabbitMQ broker can be accessed at a consistent hostname (from a Kubernetes Service) like amqp://
message-broker:5672, and internal REST/gRPC calls (if any) can use service names (e.g. [http://http://execution-coordinator:8080)](http://execution-coordinator:8080). Kubernetes’ built-in **CoreDNS** and service registry allow any component
to locate others by service name without manual configuration. In practice, much communication is via
the broker, so direct service-to-service calls are minimal, but discovery is still useful for external integrations
(e.g., the _Trigger Listener_ calling Jenkins, or the _Telemetry Collector_ sending data to ELK inside the cluster).

**Scaling Patterns:** Each microservice can be scaled horizontally by increasing its replica count (each replica
consuming from the same queue/topic). For example, if test volume grows, you might run multiple _Executor
Agent_ instances and multiple _Execution Coordinator_ instances. The message broker and Kubernetes will
distribute load: with RabbitMQ, multiple consumers on a queue will automatically load-balance the
messages (round-robin delivery to consumers) , and with Kafka, a consumer group ensures each
message partition is handled by one consumer. Kubernetes **Horizontal Pod Autoscaler (HPA)** can be
employed to scale pods based on CPU/memory or even custom metrics like queue length. A real-world
pattern could be to monitor the length of the execution.tasks queue; if it grows beyond a threshold,
automatically scale out more _Executor Agent_ pods to catch up on the backlog. Conversely, scale in when idle.
This dynamic scaling keeps throughput high and latency low.

**Fault Tolerance & Recovery:** The combination of a broker and K8s yields a robust system. If a service
instance dies mid-processing a message, the broker will detect the lost connection and requeue any
unacknowledged message for another instance to consume. Kubernetes will then respawn the failed
pod. Design services to be stateless or use external state (like a database) for long-lived context so that a
new instance can resume work without loss. For example, the _Execution Coordinator_ could store ongoing
test run states in a Redis or database, so if it restarts, it can reload state and avoid “orphaned” test
executions. Additionally, use **readiness probes** : an instance will only pull messages from the queue when
it’s marked ready. This prevents sending events to an uninitialized service (e.g., a cold _Executor Agent_ won’t
receive a test to run until it has finished startup checks). In Kubernetes, pod disruption budgets and multi-
zone deployment can further ensure high availability – the system remains operational even during node
upgrades or outages.


**Runtime Behavior: Retries, Backoff & Load Balancing**

At runtime, the system must intelligently handle retries, task distribution, and failures in a way that
maintains throughput without overwhelming any component.

**Exponential Backoff with Jitter:** When retrying operations (whether requeueing messages or
reattempting an external API call), use an **exponential backoff** strategy to gradually increase the wait time
between attempts. For example, if contacting a device or external service fails, first wait 1 second, then 2
seconds, then 4, etc., up to a cap. Crucially, add a bit of **jitter** (randomized delay) to each wait interval to
avoid synchronized retry storms. This prevents many instances from retrying failed operations in
lockstep, which can create load spikes. The goal is to spread out retries to an approximately constant rate

. Many robust libraries (e.g. Polly for .NET) implement this pattern; in fact, _eShopOnContainers_ uses a
Polly retry policy to wait and retry when RabbitMQ isn’t immediately available on startup. Applying
backoff+jitter at multiple levels – for message processing retries and for any HTTP calls (to Jira, Jenkins, etc.)
- greatly improves resilience under high load or momentary outages.

**Job Splitting & Parallelism:** The _Test Planner_ produces a manifest of tests which the _Execution Coordinator_
can break into independent jobs. Rather than treating a whole test suite as one monolithic task, the
coordinator can **split the manifest** into discrete test execution jobs. For instance, if a manifest lists 50 tests,
the coordinator could enqueue each test (or logical batch of tests) as a separate message on the execution
queue. This enables parallel execution across many _Executor Agents_. Real-world testing frameworks use
similar sharding techniques – dividing tests by class, feature, or dynamic weighting – to maximize parallel
throughput. The _Execution Coordinator_ may also apply **job grouping** logic (e.g., group tests that require a
specific device together to run on the same agent, or split heavy tests to different agents). This strategy
should be tunable: for quick running tests, dispatching one test per message yields optimal scaling; for
heavier tests or limited resources, bundling a few tests per job can reduce overhead.

**Dynamic Load Balancing:** With multiple executor instances, distributing tests evenly is key. If using a
message queue, balancing is handled inherently – each agent pulls a new task when it’s free, which
naturally balances load. No single coordinator needs to micromanage which agent gets which test, since
the queue acts as a **work queue** and idle consumers will automatically retrieve pending tasks. This pull-
based load balancing is effective: a slow agent will consume fewer tasks, a fast agent will consume more. If
a more directed approach is needed (for example, to utilize particular hardware or to account for agent
capabilities), the _Execution Coordinator_ can maintain a registry of available agents and their current load. It
could then publish jobs to specific per-agent queues or include an agent identifier in the message header.
However, in most cases the simpler design is to let any agent pick up any task, and rely on **idempotency** (if
a task times out or fails on one agent, it can be retried on another). A **watchdog** mechanism in the
coordinator should track each test job’s status – if a test has been running too long or an agent became
unresponsive, the coordinator can mark that job as failed and re-queue it to be picked up by a different
agent. This ensures no single point of failure in execution.

Additionally, implement **circuit breakers** for repeated failures: e.g., if a particular test case fails on all retry
attempts due to an environmental issue, the system might stop retrying it further and flag the run as
unstable rather than cycling indefinitely. All such runtime patterns (backoff, retries, watchdog timeouts,
circuit breakers) contribute to a robust execution flow that can self-correct under stress.


**Service Interface Contracts (Manifests & Results)**

Designing clear interfaces between services is critical for a maintainable system. Each microservice should
expose well-defined **data contracts** for the events or APIs it produces and consumes:

  - Test Manifest Contract: The output of Test Planner – the manifest – should have a versioned
schema. For example, v1 of the manifest might include fields for test identifiers, repository info, and
environment config. If later extended (say to include test priority or estimated duration), a version
bump or backward-compatible change should be managed. A sample manifest JSON was shown
above; in practice this could be documented via a JSON Schema or an AsyncAPI specification for the
TestPlanCreated event, so that all consumers (especially the Execution Coordinator ) know the
expected structure. Key fields likely include a unique plan ID , list of tests (each with an ID or
name, and perhaps a reference to the code or Xray IDs), target environment details, and metadata
like who/what triggered the run.

  - Execution Request Contract: There may be an internal contract for how an execution job is
described when the Execution Coordinator dispatches it. For instance, a message to an Executor Agent
might include the plan ID , test ID , and any needed setup info (environment variables, device ID,
etc.). Standardizing this ensures any new type of executor (e.g., a different platform runner) can plug
in easily.

  - Result Message Contract: The Executor Agent emits results with a known schema. At minimum this
includes the test ID , plan ID/run ID , execution status (PASSED, FAILED, ERROR, SKIPPED),
timestamps , and perhaps a link or blob of the test output (logs, screenshots, etc.). Additional fields
like duration , failure reason or stack trace, and environment info (which device or browser was
used) provide valuable context. This result schema should be consistent whether the origin was a
Bitbucket trigger or a Jira trigger – upstream systems only care that a test result event contains the
data to report back. The Result Router will use this contract to map results to the correct reporting
channels (for example, constructing a JUnit report for Jenkins or an Xray result payload for Jira). An
example result message was given above; in practice, a real manifest ID and test identifier tying back
to test definitions will be used so that the Notifier and Router services can enrich the reports.
  - Manifest & Result Versioning: Over time, if new fields are added (say, a screenshots array for
visual evidence, or a field for retriedCount ), services should ignore fields they don’t recognize (for
forward compatibility) and ideally include a version number in the payload. For instance,
"manifestVersion": 2 could be included in the manifest. This way, the Execution Coordinator
can adjust behavior if needed (e.g., if manifestVersion 1 doesn’t have test priorities but v2 does,
the coordinator might use that for scheduling). Keeping interface changes backward-compatible or
rolling them out in tandem with producers/consumers prevents crashes due to schema mismatches.
  - API Interfaces (if any): Aside from asynchronous messages, some services may offer APIs (for
example, a Cancel Test Run API in the Execution Coordinator , or a health check endpoint in each
service). These APIs should be documented (using OpenAPI/Swagger for REST, or protos for gRPC)
and follow consistent authentication/authorization as required by the on-prem security guidelines.
Any data exchanged via API should be treated as part of the contract – e.g., a GET /status/
{planId} could return a JSON with fields like
    ```json
    {
        "planId": "...",
        "status": "Running",
        "passed": 10, 
        "failed": 2
    }
    ```
 for a dashboard to consume. Ensuring these responses are structured and versioned makes integration with external tools (like a custom dashboard or the ELK
stack) more straightforward.

In summary, treating all inter-service communications – whether message payloads or API calls – as formal
interfaces with defined schemas will reduce ambiguity. Teams can develop and test each microservice
against these contracts. Tools like AsyncAPI can be used to document the message formats for the event
bus , similar to how one would use Swagger for RESTful APIs.

**Technology Choices & Trade-offs**

Finally, the design should be informed by the trade-offs of key technologies:

  - RabbitMQ vs. Kafka for Event Handling: Both RabbitMQ and Apache Kafka are viable brokers, but
they excel in different scenarios. RabbitMQ is a lightweight, general-purpose message broker that
uses a push model – it ensures messages are delivered to consumers via smart routing (exchanges,
bindings, routing keys) with support for complex routing patterns (fan-out, topics, direct).
RabbitMQ prioritizes reliable delivery (acknowledgments, persistence, and even message priority
ordering) and is ideal for workflows that require immediate action on events, dynamic routing, or
fine-grained control of individual messages. For example, RabbitMQ can easily implement a priority
queue so that urgent test runs (say from a release branch) jump ahead of routine nightly runs. It
also shines in request/reply scenarios or where message TTL (time-to-live) and DLQs are needed out-
of-the-box.
  - **Kafka** , on the other hand, is designed as a distributed **event streaming platform**. It uses a pull model –
producers append messages to a log (topic) and consumers read at their own pace, tracking offsets.
Kafka excels at high-throughput, horizontal scalability and event replay. It provides durability by persisting
events for a configured retention period, allowing new consumers to replay past events (e.g. to recompute
test analytics) and making it easy to handle sporadic consumers. The trade-off is that Kafka has higher
complexity: managing a Kafka cluster (with ZooKeeper or KRaft consensus) and handling partitioning
requires more operational effort. Kafka is well-suited if the system anticipates very high event volumes
(thousands of test events per second, for instance) or needs to integrate with big-data pipelines. It’s also
useful if one event (like a _Test Completed_ ) should be consumed by many independent systems (analytics,
reporting, etc.) as it handles fan-out via consumer groups without burdening the producer.

In summary, use RabbitMQ if you need **low-latency, guaranteed delivery with complex routing** for each
message (typical in task processing systems), and use Kafka if you need **scalable, replayable event
streams** and plan to treat the event log as a source of truth for later consumption. For an on-prem,
moderate-scale test automation system, RabbitMQ is often simpler to adopt and can meet reliability needs
(with clustering for HA), whereas Kafka might be justified if you foresee expanding into big-data analytics or
requiring long-term retention of all events.

  - Docker Containers vs. Bare Metal Execution: Using Docker containers for the Executor and other
services brings significant advantages in consistency and isolation. Containers package the
application and its dependencies and share the host OS kernel, which means they have minimal
performance overhead compared to running directly on the host. In fact, properly configured
containers can achieve near bare-metal speed for CPU and I/O-bound tasks , while offering
benefits like dependency isolation and easy cleanup of test environments. With Docker, each test run can start from a clean state (a fresh container or snapshot) ensuring no leftover processes or
memory from previous runs – this is crucial for accurate test results.
Bare metal (running processes directly on host or VMs without containers) might offer slight simplification
in some cases (no container runtime layer), but it comes at the cost of environment drift and more complex
scaling. In a bare-metal approach, one might have a fixed pool of machines (or VMs) that run tests. Over
time, these machines could accumulate differences (different library versions, uncleaned temp files, etc.),
leading to flaky tests. Containers solve this by using immutable images for test environments. Moreover,
Docker enables **on-demand scaling** : new containers can be launched in seconds to accommodate surge in
tests, whereas provisioning new bare-metal executors could be much slower. Tools like Kubernetes are built
around container orchestration; leveraging them is far easier than attempting to schedule tasks on raw
VMs.

A trade-off to note: if certain tests require access to special hardware (USB devices, GPUs, etc.), running in
containers may require extra configuration (device drivers, privileged mode or device passthrough).
However, Kubernetes and Docker do support these (for example, Docker can pass USB devices into a
container, and Kubernetes device plugins allow use of GPUs, etc.). Therefore, even hardware-intensive test
scenarios (like mobile device testing with OpenSTF) can be containerized with planning. Bare metal
execution would only be a consideration if container overhead or compatibility proved problematic in a
specific case, but given modern container capabilities, those cases are rare. In practice, the consistency and
ease of **snapshotting environment configurations** with Docker far outweigh any negligible performance
differences.

In conclusion, the system should favor **RabbitMQ** (or a similar lightweight broker) for its internal event
pipeline unless scaling requirements clearly demand Kafka’s throughput and replay features. Similarly, it
should utilize **Docker containers orchestrated by Kubernetes** for all components – this provides isolation,
ease of deployment, and scalability, while still achieving near-native performance and enabling robust
management of the microservices at runtime. These choices align with on-premises constraints (no external
dependencies) and ensure the architecture is extensible, maintainable, and capable of handling both
current needs and future growth.

**Sources:**
 - Messaging | Axinom Mosaic
https://docs.axinom.com/concepts/messaging/

- RabbitMQ vs Kafka - Difference Between Message Queue Systems - AWS
https://aws.amazon.com/compare/the-difference-between-rabbitmq-and-kafka/


- 10 Docker Myths Debunked | Docker
https://www.docker.com/blog/docker-myths-debunked/

- DeepStream 6.1.1 docker container vs. bare metal performance - DeepStream SDK - NVIDIA Developer
Forums
https://forums.developer.nvidia.com/t/deepstream-6-1-1-docker-container-vs-bare-metal-performance/

- Kubernetes for Microservices: Best Practices and Patterns - DEV Community
https://dev.to/rubixkube/kubernetes-for-microservices-best-practices-and-patterns-

- Exponential Backoff And Jitter | AWS Architecture Blog
https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/

- Implementing an event bus with RabbitMQ for the development or test environment - .NET | Microsoft
Learn
https://learn.microsoft.com/en-us/dotnet/architecture/microservices/multi-container-microservice-net-applications/rabbitmq-event-bus-development-test-environment

