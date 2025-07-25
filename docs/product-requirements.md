# 📱 Mobile Automation Testing System – Product Requirements Document (PRD)

---

## ✅ TL;DR

We're building a scalable, fault-tolerant Android test automation system that supports cross-project, multi-branch, and scheduled execution. It integrates with Bitbucket, Jenkins, Jira/Xray, and OpenSTF. Features include test orchestration, retry logic, logging, and Slack-based alerting — empowering faster releases with higher confidence.

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
- Runs on real devices (OpenSTF + AVD)
- Offloads repetitive manual QA work
- Provides consistent test execution
- Tracks flaky tests, retry stats, and trends
- Integrates with Jenkins, Bitbucket, Jira/Xray, and Slack
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

| KPI                             | Target / Goal                    | Purpose                                          |
| ------------------------------- | -------------------------------- | ------------------------------------------------ |
| 🖁 Avg. PR validation time      | ≤ 10 minutes                     | Shows speed of feedback from automation          |
| 🗅️ Time from commit to merge   | Reduced by 30–50%                | Indicates faster delivery cycles                 |
| ✅ % of PRs auto-approved        | ≥ 70% post automation            | Reflects trust in automated quality gates        |
| ❌ Code review rejection rate    | Reduced by 25%                   | Indicates early bug detection by automated tests |
| 🚀 Deploy frequency (pre/post)  | Increased after test infra setup | More confidence leads to more frequent releases  |
| 🛠️ Manual QA effort per release | Reduced by ≥ 50%                 | Demonstrates resource efficiency                 |
| 🔍 Time to detect regression    | Reduced by ≥ 50%                 | Catches bugs before merge or release             |
| 📊 Test-to-code ratio trend     | Rising trend (coverage growth)   | Encourages better test culture                   |

### ❌ Non-Goals

- iOS support
- Cloud device farms (local STF only)
- Replacing test frameworks (pytest/behave stay)
- Providing hosted SaaS solution
- Managing test cases manually through UI

---

## 🧑‍💻 User Stories

- As a QA engineer, I want to trigger regression tests from Xray and view results in Slack.
- As a developer, I want PR tests to run automatically and notify me on failure.
- As a developer, I want retry info and flaky test tracking available in commit or PR context.
- As a product owner, I want to ensure regressions are caught before release.
- As a team lead, I want to track flaky tests and retry stats over time.
- As a team lead, I want a dashboard to review overall test stability per project.
- As a release manager, I want to cancel test runs to prioritize hotfix validation.

---

## 🔧 Functional Requirements

### 🥚 Test Execution

- Trigger tests via CLI, Xray, Bitbucket PR, or cron
- Select tests by tag, suite, project, branch, or Jira/Xray test ID
- Run on physical and virtual Android devices
- Support parallel execution
- Cancel/pause/resume in-flight runs
- Support retry of failed tests with configurable logic
- Include retry metadata in result payload

### 📊 Test Reporting

- Real-time status of running jobs
- Allure-compatible visual reporting
- Logs, screenshots, stack traces
- Sync results with Jira/Xray test plans
- Push alerts (failures, flaky tests) via Notification Channel (e.g., Slack, email, webhook) based on thresholds
- Export all test execution logs and metrics to ELK stack (e.g., via Filebeat or OpenTelemetry)
- **Kibana dashboards** allow users to:
  - Filter test results by branch, commit, test suite, retry count, and environment
  - Visualize flaky tests and retry trends over time
  - Investigate individual test logs, stack traces, and failures
- All logs and metrics include structured fields (e.g., `test_id`, `run_id`, `retry`, `commit_hash`) for traceability and analysis

### ⏰ Scheduling & Queuing

- Nightly test plans via cron
- Priority queue with retry logic
- Concurrency limits based on devices

### 🔐 Access Control

- Role-based permissions (Dev, QA, Admin)
- Action audit logs (who triggered what)

---

## ⚙️ Non-Functional Requirements

### 🧹 Scalability

- Support hundreds of tests concurrently
- Use Docker/K8s for container execution
- Executor agents must scale based on available physical/virtual devices (STF/AVD)

### 💥 Fault Tolerance

- Detect and recover from executor crashes
- Retry failed tests due to infra instability
- Maintain test state in orchestrator
- All services must emit structured logs and metrics via OpenTelemetry
- Include watchdog monitoring for unresponsive containers and auto-recovery

### 🔐 Security

- Encrypted storage for logs/results
- Secure access to code/devices
- Optional SSO support
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

## 🧰 Component Architecture

### 🔄 Message-Driven Microservices Architecture with Jenkins-Centric Triggering

This architecture is fully designed for **on-premises deployment**. All components communicate via a local message broker (e.g., RabbitMQ or Kafka). No component depends on external cloud services. Infrastructure such as ELK, OpenTelemetry, and integration endpoints (e.g., Jira, Jenkins, Bitbucket) must all run within the secured internal network.

This system adopts a **Jenkins-centric triggering model**, where Jenkins serves as the unified control point for initiating test workflows. However, **Jenkins is not the central orchestrator of the entire system**. It only standardizes execution requests from peripheral systems like Bitbucket, Xray, or scheduled jobs.

All remaining processing—including planning, execution, observability, and result reporting—is handled by a set of independently scalable microservices. These services are designed to ensure high availability, isolation of responsibilities, and robust recovery from faults.

### Core Components

| Component                 | Description                                                                                                                                                                                                                                                                                                                            |
| ------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Trigger Listener**      | Captures test initiation requests from Bitbucket, Jenkins, Xray, or a scheduler (cron). Forwards them through Jenkins where applicable. Converts the event into a standard message and places it on the message queue. *Example*: Triggers a test run from Xray using Jenkins API and emits a queue event.                             |
| **Trigger Router**        | Validates and enriches the message with metadata (e.g., branch, commit, tags, environment) before routing it to the Test Planner. *Example*: Adds test suite and target device profile to the trigger event.                                                                                                                           |
| **Test Planner**          | Parses the repository and test framework to extract the test manifest based on the metadata. The `test manifest` is the formal contract between planning and execution, including test files, required devices, env vars, and timeouts. *Example*: Generates a list of `@smoke` tests from a given commit in a Behave framework.       |
| **Execution Coordinator** | Accepts the test manifest and schedules test executions across available Executor Agents. Applies retry policies and runs watchdog monitoring to detect silent or stuck jobs. *Example*: Retries a test job after detecting a crash, rerouting it to a different executor.                                                             |
| **Executor Agent**        | Executes test jobs in isolated environments (e.g., Docker or K8s containers). Communicates with OpenSTF and any attached test hardware. Sends logs to ELK and results to the Result Router. *Example*: Runs a test on a physical Android device and streams real-time logs.                                                            |
| **Result Router**         | Enriches results with contextual metadata and routes them to dedicated microservices that integrate with external systems like Xray, Jenkins, or Bitbucket. Each system has its own publisher service for robustness. *Example*: The Xray Publisher updates test results in Jira, while the Bitbucket Publisher updates the PR status. |
| **Notifier Service**      | Monitors for test state changes and policy breaches. Sends alerts to Slack, email, or webhook based on configured thresholds. *Example*: Notifies QA when flaky test count exceeds five in a suite.                                                                                                                                    |
| **Telemetry Collector**   | Collects logs and execution metrics from all components and streams them to ELK using OpenTelemetry standards. *Example*: Exports container resource usage and retry trends to Kibana dashboards.                                                                                                                                      |

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

