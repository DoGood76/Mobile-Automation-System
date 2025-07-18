# 🧪 Unified Test Trigger UI – PRD

## TL;DR
This PRD proposes a rich, flexible **Unified Test Trigger UI** for initiating mobile automation tests manually. Initially implemented in Jenkins, this interface will empower QA, developers, and team leads to define test scope, app versions, and execution environments — without needing CLI access or brittle scripting. This is the first official trigger in the automation ecosystem and lays the foundation for future trigger capabilities across Bitbucket, Xray, and APIs.

---

## ❗ Problem Statement

Today, there is **no unified, reliable way** to trigger automated tests across teams. While several automation systems currently exist, many teams still rely entirely on manual testing. This fragmented landscape creates several critical problems:

- ❌ No official or consistent method for triggering automation  
- ❌ Teams risk building **siloed automation workflows**, each with their own tools, formats, and interfaces  
- ❌ DevOps and QA must support multiple systems and inconsistent pipelines  
- ❌ Difficult to manage shared compute resources (devices, runners, environments)  
- ❌ No centralized visibility into test execution across projects  

This PRD introduces a **Unified Test Trigger UI** — implemented first via Jenkins — to establish a **standard interface and contract for executing mobile automation tests**. It’s the foundation for building a scalable, team-agnostic automation ecosystem.

---

## 🎯 Goals

- Enable manual triggering of test runs through a user-friendly UI  
- Provide a clean, flexible interface for test scope, version, and device selection  
- Support validation and simulation of test plans before execution  
- Integrate cleanly with the underlying test orchestration backend  
- Lay the foundation for a unified, multi-trigger test execution architecture

---

## 👥 Target Users

- QA Engineers  
- Developers  
- Team Leads  
- Release Engineers

---

## ✨ User Flow

Example:

1. Open UI  
2. Select test tags: `@smoke` + `@login`  
3. Select source: `release/v1.2.4`  
4. Choose devices: Samsung + Pixel with Android 13+  
5. Dry run simulation  
6. Click "Trigger Job"

---

## 🧩 Core Functional Sections

### 📦 1. Source / Version Under Test

**Purpose:** Define where the application under test comes from. The user selects one of the following options:

---

#### 🔘 Option 1: Build from Git Branch

**Purpose:** Daily use for feature and CI flows.

- **User Input:** Dropdown list of branches (`main`, `develop`, `release/x.y.z`, `feature/1234-2FA`)
- **Behavior:**
  - Jenkins clones repo, builds artifacts (APK, JAR, AAR, etc.)
  - Uploads to Artifactory
  - Validates that upload succeeded
  - Does **not** download artifacts — instead, generates JSON with:
    - branch
    - artifact path
    - optionally: file names

- **Failure:** If branch doesn't exist or build/upload fails → job terminates.

**JSON Example:**
```json
{
  "source": {
    "type": "build_from_branch",
    "git_branch": "release/v1.2.4",
    "commit": "HEAD",
    "artifact_path": "artifactory/releases/calendar-v1.2.4/",
    "artifacts": ["calendar.apk", "test-lib.jar"]
  }
}
```

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