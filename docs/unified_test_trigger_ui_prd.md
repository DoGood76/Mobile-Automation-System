# 🧪 Executive Summary: Unified Test Trigger UI

## 📌 What Are We Building?

We're building a **Unified Test Trigger UI** to allow teams to manually initiate mobile automation tests via a clean, structured interface — starting with Jenkins as the first delivery vehicle.

This is not just a new form. It’s the **first official mechanism** for test execution and will serve as the **execution contract** for all future trigger systems, including Bitbucket, Xray, and APIs.

---

## 😱 Why Now?

- Teams use **manual test flows** or **fragmented homegrown automation**
- No **standard trigger method**, leading to coordination bottlenecks
- DevOps and QA are overwhelmed supporting multiple approaches
- System and device resources are under strain from uncoordinated usage

If we don’t standardize now, we risk:
- ❌ Each team building its own automation system
- ❌ Increased ops overhead and DevOps burnout
- ❌ Inconsistent test results and no shared visibility

---

## ✅ What It Enables

- QA, developers, and leads can run targeted test suites via UI
- Consistent selection of app version, device groups, and test scope
- Dry-run preview of test/device matrix before execution
- Clean JSON contract for backend execution — decouples UI from Jenkins

---

## 🔗 Strategic Impact

- **Unblocks future trigger interfaces** — Bitbucket, Jira/Xray, and APIs
- Aligns all teams to a **single execution framework**
- Reduces tool sprawl and technical debt
- Makes shared automation infra manageable and scalable

---

## 🧭 Initial Scope (Phase 1)

- Jenkins UI trigger (Git branch, Artifactory path, or pre-installed)
- Test tag selection and expressions
- Device group filtering (model, OS, capabilities)
- Global execution options (parallelism, retries, dry-run)
- JSON output to Trigger Listener

---

## 📊 Success Metrics

| Metric | Goal |
|--------|------|
| % of test runs triggered via UI | > 60% in 30 days |
| Avg time to configure a run | < 2 minutes |
| Dry run prevents execution errors | > 90% |
| Trigger job misconfigs | ↓ 50% |

---

## 📣 Call to Action

- ✅ Approve scope and Jenkins-based MVP
- 🤝 Align backend teams on trigger contract
- 🔜 Begin UI development and backend simulation APIs

---

# 📘 Full PRD

# 🧪 Unified Test Trigger UI – PRD

## TL;DR

This PRD proposes a rich, flexible **Unified Test Trigger UI** for initiating mobile automation tests manually. Initially implemented in Jenkins, this interface will empower QA, developers, and team leads to define test scope, app versions, and execution environments — without needing CLI access or brittle scripting. This is the first official trigger in the automation ecosystem and lays the foundation for future trigger capabilities across Bitbucket, Xray, and APIs.

---

## ❗Problem Statement

Today, there is **no unified, reliable way** to trigger automated tests across teams. While several automation systems currently exist, many teams still rely entirely on manual testing. This fragmented landscape creates several critical problems:

* ❌ No official or consistent method for triggering automation
* ❌ Teams risk building **siloed automation workflows**, each with their own tools, formats, and interfaces
* ❌ DevOps and QA must support multiple systems and inconsistent pipelines
* ❌ Difficult to manage shared compute resources (devices, runners, environments)
* ❌ No centralized visibility into test execution across projects

If each team continues developing its own test infrastructure, the result will be operational chaos — fractured tooling, redundant infrastructure, and bottlenecks for shared teams.

This PRD introduces a **Unified Test Trigger UI** — implemented first via Jenkins — to establish a **standard interface and contract for executing mobile automation tests**. It’s the foundation for building a scalable, team-agnostic automation ecosystem.

In the initial rollout, the Jenkins UI will serve as the **primary interface** for triggering tests across QA, developers, and team leads. Until other mechanisms are implemented (Bitbucket annotations, Jira/Xray triggers, API access), Jenkins will be the entry point for automation workflows.

**Critically**, this Jenkins trigger will also define the **execution contract and user interaction model** that all future triggers must follow — making this the foundational pattern for a scalable, unified automation ecosystem.

---

## 🌟 Goals

* Enable manual triggering of test runs through a user-friendly UI
* Provide a clean, flexible interface for test scope, version, and device selection
* Support validation and simulation of test plans before execution
* Integrate cleanly with the underlying test orchestration backend
* Lay the foundation for a unified, multi-trigger test execution architecture

---

## 👥 Target Users

* QA Engineers
* Developers
* Team Leads
* Release Engineers

---

## ✨ User Flow

Imagine a QA lead validating a hotfix on Friday:

1. Opens the UI, clicks "Run Mobile Tests"
2. Selects `@smoke` + `@login` tests on `release/v1.2.4`
3. Picks Samsung & Pixel devices with Android 13+
4. Runs a Dry Simulation – sees 12 tests across 3 devices
5. Confirms and clicks "Trigger Job"

No CLI, no YAML editing. Just clarity and control.

---

## 🧹 Core Functional Sections

### 📦 1. Source / Version Under Test

**🌟 Purpose:**
Define how the app version and artifacts (APK, JAR, AAR, etc.) are linked to the test run.

The user selects one of the following **three mutually exclusive options**:

#### 🔘 Option 1: Build from Git Branch

*(Used for daily CI automation and feature validation)*

* **User Input:**
  Dropdown list of branches (e.g., `main`, `develop`, `release/v1.2.4`, `feature/1234-2FA`)

* **Behavior:**

  * Jenkins clones the selected Git branch
  * Builds the required artifacts
  * Uploads them to Artifactory and validates presence
  * Generates JSON for the Trigger Listener

* **JSON Output:**

```json
{
  "source": {
    "type": "build_from_branch",
    "git_branch": "release/v1.2.4",
    "commit": "HEAD",
    "artifact_path": "artifactory/releases/calendar-v1.2.4/",
    "artifacts": [
      "calendar.apk",
      "test-lib.jar"
    ]
  }
}
```

📜 *If no artifacts are specified, the Trigger Listener may infer default filenames (e.g., `calendar.apk`) or retrieve all files under the provided path.*

---

#### 🔘 Option 2: Use Existing Artifacts from Artifactory

*(Used for retesting known app versions)*

* **User Input:**
  Dropdown or free-text field (e.g., `releases/calendar-v1.3.1`)

* **Behavior:**

  * Validates that the artifact path exists
  * Optionally lists specific artifacts
  * Generates JSON for the Trigger Listener

* **JSON Output:**

```json
{
  "source": {
    "type": "existing_artifacts",
    "artifact_path": "artifactory/releases/calendar-v1.3.1/",
    "artifacts": [
      "calendar.apk",
      "test-lib.jar"
    ]
  }
}
```

📜 *If no artifacts are specified, the Trigger Listener may infer default filenames (e.g., `calendar.apk`) or retrieve all files under the provided path.*

---

#### 🔘 Option 3: No Build / No Artifact Download

*(Used when the app is already installed on the target devices)*

* **Behavior:**

  * No build, no validation
  * App is assumed to be pre-installed
  * JSON still generated for test execution

* **JSON Output:**

```json
{
  "source": {
    "type": "pre_installed"
  }
}
```

---

### 🛠️ UI Behavior

* One option must be selected:
  `Build from Git Branch` | `Use Existing Artifacts` | `No Build / Pre-installed`
* Relevant inputs are shown dynamically

---

### ✅ Summary Table

| Mode                         | Jenkins Builds? | Validates Artifacts? | Downloads? | App Pre-installed? | JSON Fields Included                                |
| ---------------------------- | --------------- | -------------------- | ---------- | ------------------ | --------------------------------------------------- |
| Build from Branch            | ✅ Yes           | ✅ Yes (after upload) | ❌ No       | ❌ No               | `git_branch`, `artifact_path`, optional `artifacts` |
| Use Existing Artifacts       | ❌ No            | ✅ Yes                | ❌ No       | ❌ No               | `artifact_path`, optional `artifacts`               |
| No Build / Already Installed | ❌ No            | ❌ No                 | ❌ No       | ✅ Yes              | `type: pre_installed`                               |

---

### ✅ Jenkins UX

<p align="center">
  <img src="./images/jenkins-branch-artifacts-selection.png" alt="UI mockup" width="400" />
</p>




---
### 🧪 2. Test Source

> 📝 **Note:** This section is **optional** if the test sources are already bundled with the source artifacts in the *Source / Version Under Test* section.

If test definitions are **independent from the app version**, users must specify where the tests are located.

Defines the origin of the test definitions to be executed.

* **Option 1: Git Branch(es)**

  * **Purpose:** Used when test definitions are in source control and may be versioned independently. The source can be in the same or a different repository than the app.
  * **User Input:** One or more branches (e.g., `main`, `develop`, `release/v1.2.4`) with optional repository URL override if not the current project
  * **Behavior:** Jenkins fetches test definitions from the selected branches or repositories and generates the relevant test artifact paths.
  * **JSON Output:**

```json
{
  "test_sources": {
    "test_source_1": {
      "type": "git_branch",
      "branches": ["main"],
      "repository": "https://example.com/test-repo.git"
    },
    "test_source_2": {
      "type": "git_branch",
      "branches": ["main"],
      "repository": "https://example.com/test-behave-repo.git"
    }
  }
}
```

* **Option 2: Artifacts from Artifactory**

  * **Purpose:** Use pre-built test libraries or packages from Artifactory.
  * **User Input:** Path to test artifacts (e.g., `test-bundles/v1.3.1`)
  * **Behavior:** Jenkins validates the artifact path exists and references it in the test trigger payload.
  * **JSON Output:**

```json
{
  "test_source": {
    "type": "artifactory_path",
    "path": "test-bundles/v1.3.1"
  }
}
```

* **Option 3: None**

  * **Purpose:** Assumes tests are already included in the app version's artifact and no additional test path is needed.
  * **User Input:** None
  * **Behavior:** No test-specific source is added; tests are assumed bundled in app source.
  * **JSON Output:**

```json
{
  "test_source": {
    "type": "none"
  }
}
```
### 🛠️ UI Behavior

* User can choose one or more test sources.
* Test sources are grouped in a list format with ability to add/remove items.
* Each source is labeled and validated individually.
* Repository field is optional when source is same as app repository.

---
### ✅ Jenkins UX

<p align="center">
  <img src="./images/jenkins-branch-artifacts-selection.png" alt="UI mockup" width="400" />
</p>
---

### ✅ Summary Table

| Mode                    | Source Type       | Requires Repo? | Jenkins Validates? | Can Be Multiple? | JSON Format         |
| ----------------------- | ----------------- | -------------- | ------------------ | ---------------- | ------------------- |
| Git Branch              | git\_branch       | ✅ Optional     | ✅ Yes              | ✅ Yes            | test\_sources array |
| Artifactory Test Bundle | artifactory\_path | ❌ No           | ✅ Yes              | ✅ Yes            | test\_source object |
| No Additional Source    | none              | ❌ No           | ❌ No               | ❌ No             | test\_source object |

---

### 🧾 Combined JSON Output Example

```json
{
  "test_sources": {
    "test_source_1": {
      "type": "git_branch",
      "branches": ["main"],
      "repository": "https://example.com/test-repo.git"
    },
    "test_source_2": {
      "type": "artifactory_path",
      "path": "test-bundles/v1.3.1"
    }
  }
}
```

---
### 3. Test Selection

Select the specific tests to execute from the defined sources.

* **Multi-select predefined test groups** (e.g., `@sanity`, `@regression`, `@smoke`)

  * **Purpose:** Run common sets of tests by category

* **Free-text input for specific tags or test IDs** (e.g., `@login`, `xray-123`)

  * **Purpose:** Target precise test scopes for debug or retests

* **Tag expressions** (e.g., `(@checkout or @login) and not @slow`)

  * **Purpose:** Logical selection of tests by condition

* **Exclusion tags** (e.g., `not @unstable`)

  * **Purpose:** Filter out flaky or long-running tests

* **Toggle: Dry run**

  * **Purpose:** Preview test-device matrix before execution by triggering simulation API call

* **Output preview**

  * **Purpose:** Shows number of matching tests and affected devices after Dry Run

##### ⚙️ Selection Logic Priority

1. **Tag Expression** (if provided) overrides all other selection inputs
2. If no expression is provided:

   * Union of selected **test groups** and **free-text tags/IDs**
   * Then apply **exclusion tags** to filter final test set

> ⚠️ UI should clearly warn when Tag Expression is overriding all other selections

**Example Behavior:**

* Selected: `@sanity`, `@smoke` + `@login`, `xray-123`
* Tag Expression: `(@checkout or @login) and not @slow`
* ✅ Final Selection: Only tests matching tag expression

---

### 4. Device Selection

* Create device groups (executed in parallel):

  * Filter by device brand or model (`Samsung`, `Pixel 6`, `Xiaomi`)
  * Filter by OS version (`13`, `14`, `15`)
  * Filter by capability (`Camera`, `Biometric`, `NFC`, `5G`)
* Option to add/remove multiple device groups
* Label and save device group presets

---

### 5. Execution Options (Global)

* Toggle: Parallel Execution
* Retry failed tests: Toggle + Count
* Toggle: Fail Fast
* Timeout (in minutes)
* Toggle: Dry Run only (no execution, just validation)
* Toggle + Field: Save configuration as reusable preset

---

## 📊 Success Metrics

| Metric                                      | Target               | Rationale                                 |
| ------------------------------------------- | -------------------- | ----------------------------------------- |
| % of test runs triggered via UI             | > 60% within 1 month | Shows adoption by QA/release teams        |
| Average time to configure run               | < 2 minutes          | Confirms improved efficiency              |
| % of dry runs that prevent execution errors | > 90%                | Proves simulation is reducing failed runs |
| Trigger job error rate                      | Reduced by 50%       | Fewer misconfigured jobs                  |

---

## 🚰 Technical Considerations

* **Form Validation:** Tag syntax, required fields, APK existence
* **Data Sources:**

  * Git: for branch lists
  * Artifactory: for APK/JAR paths
  * STF or farm service: for real-time device availability and specs
* **Dry Run Simulation:**

  * Sends request to `/simulate` endpoint in Test Planner
  * Returns matched tests and device execution matrix
* **Security:**

  * Role-based UI access
  * Audit trail of who ran what, when
* **Presets:**

  * Stored in config backend
  * Shared across teams or roles

---

## 🩱 Influence on Future Trigger Architecture

This UI and interaction model defines the **canonical contract** for future triggers, including:

* Bitbucket commit triggers with `@automation` annotations
* Xray test-case-level buttons (“Run Tests”)
* REST API triggers via external systems
* Trigger Listener contract standardization
* JSON-first execution abstraction decoupled from Jenkins

---

## 🚀 Optional Enhancements

* **Template Sharing:** Save/load/share full test configurations
* **Run Again:** Re-run previous job with same config
* **Live Error UX:** Real-time validation of missing APKs, bad tags, no devices
* **Bitbucket/Xray Integration:** Trigger runs via Git or Jira

---

## ✅ Acceptance Criteria

* [ ] Users can select test scope using tags and groups
* [ ] Users can define the app source using Git, Artifactory, or skip
* [ ] Device groups can be created and filtered
* [ ] Global execution options are configurable
* [ ] Dry run returns valid preview of test/device matrix
* [ ] JSON is always sent to Trigger Listener on successful job start
* [ ] Errors are handled clearly (missing branches, missing APKs, no test match)