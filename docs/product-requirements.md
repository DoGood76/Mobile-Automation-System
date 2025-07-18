# 📱 Mobile Automation Testing System – Product Requirements Document (PRD)

---

## ✅ TL;DR

We're building a scalable, fault-tolerant Android test automation system that supports cross-project, multi-branch, and scheduled execution. It integrates with Bitbucket, Jenkins, Jira/Xray, and OpenSTF. Features include test orchestration, retry logic, logging, and Notification Channel & Alerting — empowering faster releases with higher confidence.

---

## 🔍 Problem Statement

Our current mobile testing process relies heavily on **manual validation**, which is time-consuming, inconsistent, and difficult to scale. As product complexity and test coverage grow, manual efforts become a bottleneck — **slowing development** and reducing **confidence in release quality**.

We lack a centralized and automated **test orchestration system** to manage, execute, and track tests across **multiple projects, branches, and environments**.

### This leads to:

- ❌ Inconsistent test quality
- ❌ Limited visibility into test health and coverage
- ❌ Delayed feedback loops for developers
- ❌ High QA overhead
- ❌ Fragmented tooling and workflows
- ❌ Increased risk of regressions

### ✅ We Need a Test Automation Solution That:

- Supports multi-project, multi-branch, and scheduled test execution
- Runs on real devices (OpenSTF) and AVD
- Offloads repetitive manual QA work
- Provides consistent test execution
- Tracks flaky tests, retry stats, and trends
- Integrates with Jenkins, Bitbucket, Jira/Xray, and Notification Channel
- Improves delivery speed and developer confidence
- **Is fully deployable on-premises and does not rely on external cloud services**

---

## 🌟 Goals

### ✅ Business Goals

- Reduce QA cycle time by 50%
- Cover 80%+ of regression tests via automation
- Reduce post-release bugs by 40%
- Improve developer velocity and merge confidence

### 👥 User Goals

- QA can trigger and view test results from Xray
- Developers get pass/fail feedback within 10 min
- Scheduled tests run nightly without manual effort
- All test data is centralized and traceable

### 📈 Success Metrics & KPIs

| KPI                              | Target / Goal                    | Purpose                                          |
| -------------------------------- | -------------------------------- | ------------------------------------------------ |
| 🔁 Avg. PR validation time       | ≤ 10 minutes                     | Shows speed of feedback from automation          |
| 📥 Time from commit to merge     | Reduced by 30–50%                | Indicates faster delivery cycles                 |
| ✅ % of PRs auto-approved         | ≥ 70% post automation            | Reflects trust in automated quality gates        |
| 🚫 Code review rejection rate    | Reduced by 25%                   | Indicates early bug detection by automated tests |
| 🚀 Deploy frequency (pre/post)   | Increased after test infra setup | More confidence leads to more frequent releases  |
| 🛠️ Manual QA effort per release | Reduced by ≥ 50%                 | Demonstrates resource efficiency                 |
| 🔍 Time to detect regression     | Reduced by ≥ 50%                 | Catches bugs before merge or release             |
| 📊 Test-to-code ratio trend      | Rising trend (coverage growth)   | Encourages better test culture                   |

### ❌ Non-Goals

- iOS support
- Cloud device farms (local STF only)


---

## 🧑‍💻 User Stories

- As a QA engineer, I want to trigger regression tests from Xray and view results in Xray test Cycle.
- As a developer, I want pull requests tests to run automatically and notify me on failure prior of being review by co-worker.
- As a team lead, I want to monitor test coverage per project or feature, so I can ensure critical areas are not missed.
- As a team lead, I want to track flaky tests and retry stats over time.
- As a release manager, I want to cancel test runs to prioritize hotfix validation.
- As a product manager, I want to know which tests failed before a release, so I can assess risk and make informed decisions.

---

## 🔧 Functional Requirements

### 🦢 Test Execution

- Trigger tests via CLI, Xray, Bitbucket PR, or cron
- Select tests by tag, suite, project, branch, or Jira/Xray test ID
- Run on physical and virtual Android devices
- Support parallel execution
- Cancel/pause/resume in-flight runs
- Support retry of failed tests with configurable logic


### 📊 Test Reporting

- Real-time status of running jobs
- Allure-compatible visual reporting
- Logs, screenshots, stack traces
- Sync results with Jira/Xray test plans
- Push alerts (failures, flaky tests) via Notification Channel or webhook based on thresholds
- Export all test execution logs and metrics to ELK stack
- Kibana dashboards allow users to:
    - Filter test results by branch, commit, test suite, retry count, and environment
    - Visualize flaky tests and retry trends over time
    - Investigate individual test logs, stack traces, and failures
- All logs and metrics include structured fields (e.g., `test_id`, `run_id`, `retry`, `commit_hash`) for 

### ⏰ Scheduling & Queuing

- Nightly test plans via cron
- Priority queue with retry logic
- Concurrency limits based on available devices

### 🔐 Access Control

- Role-based permissions (Dev, QA, Admin)
- Action audit logs (who triggered what)

---

## ⚙️ Non-Functional Requirements

### 🧩 Scalability

- Support hundreds of tests concurrently
- Use Docker/K8s for container execution
- Executor agents must scale based on available physical/virtual devices (STF/AVD)

### 💥 Fault Tolerance

- Detect and recover from executor crashes
- Retry failed tests due to infra instability
- Maintain test state in orchestrator
- Include watchdog monitoring for unresponsive containers and auto-recovery

### 🔐 Security

- Encrypted storage for logs/results
- Secure access to code/devices
- Each test execution must run in an isolated container to prevent test data leakage

### ⚡ Performance

- Test trigger-to-start time < 30 sec
- PR test results within 10 min
- Retry delay: 45 seconds

### ⚒️ Maintainability

- Plugin architecture for tools
- Minimal manual ops required
- All services must produce traceable structured logs

### ☁️ Deployment

- **On-premises deployment required** (Kubernetes-based)
- **Must not depend on cloud-based services or SaaS tools**

---

## 💥 Fault Tolerance & Crash Recovery

### Crash Detection & Retry

- Detect crashes using liveness and readiness probes
- Mark test as crashed and log error details
- Automatically re-queue crashed test for retry based on policy

### Stateful Tracking

- Maintain test state within the Execution Coordinator
- Ensure test resumption after partial failure
- Maintain unique run IDs to correlate retries with original executions

### Resilient Logging

- Use sidecar or stream-based logging to ensure no log loss
- Logs are shipped immediately to ELK
- Each log entry tagged with run ID and retry metadata

### Watchdog Monitoring

- Periodically probe executor health and responsiveness
- Automatically kill and reschedule jobs stuck in unresponsive containers
- Alert via Notifier Service on repeated watchdog triggers

---

## ⟳ Retry Policy

| Failure Condition      | Retry? | Notes                           |
| ---------------------- | ------ | ------------------------------- |
| Device disconnect      | ✅      | Infra-related                   |
| Timeout during setup   | ✅      | Infra slowness or crash         |
| App install failure    | ✅      | Device-specific issue           |
| Container crash        | ✅      | System failure                  |
| Test assertion failure | ❌      | Product bug — not retried       |
| Flaky test (tagged)    | ✅      | Allow up to 3 retries if tagged |

### Defaults

- Max retries: `1` (or `3` for `@flaky`)
- Retry delay: `45s`
- Clean environment between retries
- Retry data reported and logged

---

## 📊 Retry Metrics

| Metric                                   | Description                                     |
| ---------------------------------------- | ----------------------------------------------- |
| retry\_rate\_overall                     | % of tests that retried at least once           |
| avg\_retries\_per\_test                  | Includes 0-retry tests in average               |
| flaky\_test\_count                       | Tests that passed only after retry              |
| infra\_failure\_retries                  | Retries triggered by infra (e.g., crash/device) |
| retry\_success\_rate                     | % of retries that passed                        |
| max\_retry\_depth\_reached               | Tests that failed after max retry               |
| added\_execution\_time\_due\_to\_retries | Total time cost of retries                      |

---

## 🔔 Notification Channel & Alerting

The system must support flexible integration with messaging platforms (e.g., Slack) to deliver real-time alerts and status notifications for a wide range of events.

### Notification Types

- **Infrastructure Issues**: Alert on executor crashes, device unavailability, or long queue delays
- **Flaky Test Warnings**: Highlight repeated flaky test patterns
- **TBD**



## 🧭 Monitoring & Debugging

### 🧑‍💼 User Experience Flow

This section describes how users interact with the system through integrations and observability tools.

#### 1. Triggering a Test

- **Option A: Commit Annotation Trigger**

  - Developer pushes a commit with annotations like `@automation @regression @TEST-1234`
  - A version control hook detects the annotations, parses them, and triggers the test run

- **Option B: Jenkins Pipeline Trigger**

  - User triggers the test manually through a Jenkins job (UI button or API call)
  - Parameters include test suite, branch, tags, and environment

- **Option C: Xray-Driven Trigger**

  - From Jira/Xray, user clicks a "Run Automated Test" button
  - Test plan is used to trigger execution via API

- **Option D: Scheduled Cron Trigger**

  - Tests are executed automatically based on nightly or scheduled cron jobs

#### 2. Cancel or Retry a Test Run

- Users can cancel an active test run via Jenkins
- Retrying a test is possible through:
  - Rerun button in Jenkins
  - Xray test execution rerun
  - Re-pushing annotated commit

#### 3. Viewing and Investigating Results

- Logs and telemetry data are stored and visualized via **ELK (Kibana)**
- Results include:
  - Logs, stack traces, and retry history
  - Test metadata (PR number, commit, environment)
  - Traces and metrics via **OpenTelemetry/APM standards**
- Users filter and explore test execution data in Kibana dashboards

### 📊 ELK Observability

All logs and test execution metrics are exported to the ELK stack using **OpenTelemetry** or APM-compatible standards.

- **Kibana** is used for:

  - Viewing real-time and historical test logs
  - Investigating flaky failures or environment issues
  - Filtering by test, run ID, environment, branch, or commit

- **Metrics Tracked** include:

  - Retry counts
  - Execution duration
  - Device usage
  - Failure trends
  - **TBD**

- **Dashboards** can be created for:

  - QA health monitoring
  - Flaky test heatmaps
  - Execution time trends by project or team



---

## 🧰 Component Architecture

### 🔄 Message-Driven Microservices Architecture

This architecture is fully designed for **on-premises deployment**. All components communicate via local message brokers (e.g., RabbitMQ or Kafka), and no component depends on external cloud services. Infrastructure such as ELK, OpenTelemetry, and integration endpoints (e.g., Jira, Jenkins, Bitbucket) must all run within the secured internal network.

This system adopts a **Jenkins-centric coordination model**, where all peripheral trigger sources (such as Xray, Bitbucket, or scheduled jobs) are funneled through Jenkins as a unified execution control point. Jenkins acts as the central entry for triggering test workflows, ensuring standardized authentication, auditing, and traceability across teams.

All remaining processing—including planning, execution, observability, and result reporting—is handled by a set of independently scalable microservices. These services are designed to ensure high availability, isolation of responsibilities, and robust recovery from faults.

### Core Components:

| Component                 | Description |
|--------------------------|-------------|
| **Trigger Listener** | Captures test initiation requests from Bitbucket, Jenkins, Xray, or a scheduler (cron) and pushes standardized events into a message queue. It also proxies external trigger sources through Jenkins when required, acting as a centralized gateway. <br> *Example*: Detects a commit annotated with `@automation` and emits a trigger message via Jenkins webhook. |
| **Trigger Router** | Validates and enriches the message with metadata (e.g., branch, commit, tags, environment) before routing it to the Test Planner. *Example*: Adds test suite and target device profile to the trigger event. |
| **Test Planner** | Parses the repository and test framework to extract the test manifest based on the metadata. The `test manifest` is the formal contract between planning and execution, including test files, required devices, env vars, and timeouts. *Example*: Generates a list of `@smoke` tests from a given commit in a Behave framework. |
| **Execution Coordinator** | Manages the lifecycle of test jobs, including scheduling from the manifest, applying retry logic, balancing across agents, tracking job state, and performing watchdog monitoring for unresponsive containers. |
| **Executor Agent** | Stateless service that executes the test jobs in isolated Docker/K8s containers. Communicates with OpenSTF and connected test hardware. Streams structured logs and telemetry to ELK and sends results to the Result Router. |
| **Result Router** | Collects and enriches result events (e.g., duration, retries, outcomes) and delegates them to specialized publisher microservices, each responsible for reporting to a specific external system (e.g., Xray, Bitbucket, Jenkins). Ensures delivery confirmation and fault isolation per integration. Example: **1.** The Xray Publisher updates test results in Jira **2.** The Bitbucket Publisher updates the PR status **3.** The Jenkins Publisher marks the pipeline build as success/failure, archives Allure results, and updates execution counters in the Jenkins UI |
| **Notifier Service** | Sends alerts and summaries to Notification Channel or other endpoints based on policy thresholds. |
| **Telemetry Collector** | Aggregates logs and performance metrics from all services and exports them to ELK using OpenTelemetry. Focuses on continuous observability and debugging.  Example: Streams real-time test duration, device CPU usage, and container health to Kibana dashboards. |                                   |

                                                                                                                      

#### Advantages:

- Clear separation of concerns between input, planning, execution, and reporting
- Scalable and fault-tolerant using message queues and backpressure
- Extensible design allows future integration with platforms like GitHub Actions, MS Teams, or custom dashboards
- Supports isolated development and monitoring of each service for easier debugging and maintenance

---

### 🔁 Example Scenario

#### Trigger Scenario:

A developer pushes to the `release/v1.3` branch with the following commit message:

```
feat(login): Add 2FA tests @automation @smoke @XRAY-124 @XRAY-127
```

#### End-to-End Flow:

1. **Trigger Listener** receives webhook from Bitbucket and calls Jenkins job `Run-Mobile-Automation`. Jenkins emits an event:

```json
{
  "source": "bitbucket",
  "branch": "release/v1.3",
  "commit": "a9f31b6",
  "trigger_type": "PR",
  "tags": ["@smoke", "@automation"],
  "xray_ids": ["XRAY-124", "XRAY-127"]
}
```

2. **Trigger Router** adds:

```json
{
  "repo": "android-auth",
  "branch": "release/v1.3",
  "commit": "a9f31b6",
  "tags": ["@smoke", "@automation"],
  "xray_ids": ["XRAY-124", "XRAY-127"],
  "env": "staging",
  "jenkins_build": "#143"
}
```

3. **Test Planner**:

- Clones `commit a9f31b6`
- Scans Behave `features/` for matching tags
- Matches 12 tests
- Detects framework `Behave + Python 3.11`
- Emits manifest:

```json
{
  "framework": "behave",
  "tests": [
    "features/login.feature:10",
    "features/two_factor.feature:22"
  ],
  "required_devices": ["android-12", "pixel-5"],
  "timeout": 600
}
```

4. **Execution Coordinator**:

- Splits jobs by device
- Applies retry rules
- Dispatches to two Executor Agents

5. **Executor Agent**:

- Runs test
- Streams logs to ELK
- Pushes result to Result Router

6. **Result Router**:

- Forwards to:
  - Xray Publisher → Jira
  - Bitbucket Publisher → commit status
  - Jenkins Publisher → updates Jenkins task status, uploads reports, archives Allure results

7. **Notifier Service**:

- Notifies via Slack: "Smoke Suite for release/v1.3 completed – 10 passed, 2 failed"

8. **Telemetry Collector**:

- Streams logs, retry counts, and execution time to ELK for dashboards

---