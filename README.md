# V3S: What, Why, How

> **V3S** is a full-stack Software Composition Analysis (SCA) and vulnerability tracking platform. It reads dependency manifest files from many languages, checks them against a CVE database, and sends alerts when a new zero-day risk appears.

---

## 1. Tech Stack

| Tier | Technology | Job |
|------|-----------|-----|
| **Client** | React.js, Tailwind CSS / UI library, Axios / Fetch | Dashboard, file upload, security score, patch advice |
| **Backend** | Java (Spring Boot or Servlet container), Jackson (JSON/XML) | Routing, parsing, version matching |
| **Database** | MongoDB (Spring Data MongoDB or Mongo Java Driver) | Users, projects, dependencies, CVE cache |
| **Auth + Push** | Firebase Authentication, Firebase Cloud Messaging (FCM) | Login, tenant isolation, push alerts |
| **API style** | REST over HTTP (JSON and multipart file upload) | Client talks to backend |

---

## 2. What is it?

V3S is a **multi-language software supply chain scanner**.

- A developer uploads a manifest file (`pom.xml`, `package.json`, `Cargo.toml`, etc.) or a zip of the whole source tree.
- V3S extracts every library and version from it.
- It checks each library against known vulnerabilities (CVEs).
- It shows a **health score**, the list of vulnerable libraries, and the **patched version** to upgrade to.
- If a new zero-day CVE appears later, it sends a push notification to the affected project owners.

**Simple example:** You upload a `pom.xml` that uses `log4j-core 2.14.1`. V3S finds that this version is inside the vulnerable range of **CVE-2021-44228 (Log4Shell, CRITICAL, CVSS 10.0)** and tells you to upgrade to `2.15.0`.

---

## 3. Why does it matter?

### The complexity trap
Modern applications rarely use one language. One platform can have a React frontend, a Node.js gateway, a Java microservice, and Rust components. Each one has its own manifest format and its own package ecosystem.

### The blind spot
Most security tools lock you into **one ecosystem**. Without one common layer, a security team cannot see the total risk across all services. (Alag-alag tools, alag-alag reports, poori picture kisi ko nahi dikhti.)

### The real-world threat
When a library like log4j is found vulnerable, teams must find which of their projects use it. Doing this by hand takes days. V3S does it in seconds because every dependency is already stored in one normalized format.

### Why it is actionable
V3S does not only say "you are vulnerable". It gives the **severity**, the **CVE id**, and the **patched version**, so the developer knows the exact fix.

---

## 4. How does it work?

### 4.1 High-level architecture

```
+-------------------------------------------------------------------------+
|                           CLIENT TIER (React)                           |
|  - Multi-language Dashboard    - Security Score Card                    |
|  - Project Workspace           - Vulnerability & Patch Recommendations  |
+------------------------------------+------------------------------------+
                                     |
                                     | HTTP REST APIs (JSON / Multipart)
                                     v
+-------------------------------------------------------------------------+
|                          BACKEND TIER (Java)                            |
|  +-------------------------------------------------------------------+  |
|  | Manifest Router & Dispatcher                                      |  |
|  +-------------------+-------------------+---------------------------+  |
|                      |                   |                              |
|                      v                   v                              |
|           +------------------+   +------------------+                   |
|           | Maven XML Parser |   | NPM JSON Parser  |   [More Parsers]  |
|           +--------+---------+   +--------+---------+                   |
|                    +----------+-----------+                             |
|                               v                                         |
|  +-------------------------------------------------------------------+  |
|  | Vulnerability Matching Engine (Range Checks vs. CVE DB)           |  |
|  +-------------------------------------------------------------------+  |
+------------------------------------+------------------------------------+
                                     |
                       +-------------+-------------+
                       v                           v
+-------------------------------+   +------------------------------------+
|     DATA TIER (MongoDB)       |   |       REAL-TIME TIER (Firebase)    |
| - Users Collection            |   | - Firebase Auth (Session Tokens)   |
| - Projects & Dependencies     |   | - FCM (Push Notifications for      |
| - CVE Cache Database          |   |   Critical Zero-Day Alerts)        |
+-------------------------------+   +------------------------------------+
```

### 4.2 Step-by-step scan lifecycle

**Step 1: Ingestion**
The user uploads a manifest file or a zip through the React client. The user is already logged in through Firebase Auth.

**Step 2: Dispatch and parsing**
The Java backend receives the file. A class called `ManifestDispatcher` looks at the file type and hands it to the correct parser:

- `pom.xml` goes to `MavenPomParser`
- `package.json` goes to `NpmPackageJsonParser`
- `Cargo.toml` and other formats are supported by design (the "More Parsers" slot), so new parsers can be added without changing the dispatcher logic.

If a zip is uploaded, the backend scans the extracted files for known manifest names and sends each one to its parser.

**Step 3: Normalization**
Whatever the original format was (XML, JSON, TOML), every parser returns the same simple object:

```json
{ "name": "...", "version": "...", "ecosystem": "maven | npm | ..." }
```

This is the key idea. After this step, the rest of the system does not care which language the project used.

**Step 4: Range matching (the core of SCA)**
For each dependency, the backend:

1. Queries the `vulnerabilities` collection in MongoDB by `packageName` and `ecosystem`.
2. If a record exists, compares the **installed version** against the `vulnerableVersionRange` string.

Example: `vulnerableVersionRange = ">=2.0-beta9 <2.15.0"`. Installed `2.14.1` falls inside this range, so it is **vulnerable**. Installed `2.15.0` is outside it, so it is **safe**.

Matching only the name is not enough. The version range is what makes the result correct.

**Step 5: Report and storage**
The backend calculates the project `healthScore`, marks each dependency with `isVulnerable`, `cveId`, `patchedVersion` and `severity`, and saves this in the `projects` document. The JSON report goes back to React, which shows the score card and the fix suggestions.

**Step 6: Background alerting**
When a new zero-day CVE is imported later:

1. The system checks the cached dependencies of all existing projects.
2. It finds projects whose installed version falls inside the new vulnerable range.
3. Firebase Cloud Messaging sends a push notification **only to those owners**.

This is why "real-time" matters here. Users get warned without re-uploading anything.

---

## 5. Design Patterns Used

| Pattern | Where | Why |
|---------|-------|-----|
| **Strategy** | Each parser (`MavenPomParser`, `NpmPackageJsonParser`) implements one common parsing interface | Different file formats, same contract |
| **Dispatcher** | `ManifestDispatcher` picks the right parser by file type | One entry point for all manifests |
| **Normalization (common model)** | The `Dependency` object `{name, version, ecosystem}` | Matching engine stays language-independent |

**Adding a new language:** write one new parser class that returns `Dependency` objects, and register it with the dispatcher. Nothing else changes.

---

## 6. Component Responsibilities

### Frontend (React)
- Shows security scores (Critical, High, Medium, Low) per project.
- Uploads manifests or zips.
- Filters dependencies by ecosystem.
- Shows remediation advice such as the target patch version.

### Backend (Java)
- File routing and dispatching.
- Converting manifest entries to the `Dependency` model.
- Matching dependencies against CVE version ranges.

### Database (MongoDB)
- Stores dependency trees with a flexible schema, so projects with many stacks fit easily.
- Keeps a local CVE cache indexed by package name and ecosystem for fast lookups.

### Auth and Push (Firebase)
Firebase does **two separate jobs**:
1. **Firebase Authentication:** logs users in and enforces **tenant isolation**, so each developer sees only their own workspace.
2. **FCM:** delivers push notifications for newly found vulnerabilities.
