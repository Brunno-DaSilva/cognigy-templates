# Cognigy.AI API Call Templates — Best Practices Library

> A collection of reusable, production-ready Cognigy.AI Flow templates for external API integrations. Designed for Solutions Architects and Flow developers who want to build reliable, observable, and maintainable API integrations from day one.

---

## 🏗️ Template Architecture Overview

Every template in this library follows the **same structural pattern**, ensuring consistency across integrations, predictable error handling, and unified analytics tracking via Cognigy Insights.

```
Start
 └── Task: API - [METHOD_NAME] - Start
      └── Code: StartLatencyTimer
           └── POST Signal → <API_URL>
                └── Code: DiffLatencyTimer
                     └── Lookup: StatusCode
                          ├── 202 Ok
                          │    ├── Task: API - [METHOD_NAME] - Success
                          │    ├── 400: Next Action (2)
                          │    └── 200: Next Action
                          ├── 400 Bad Request
                          │    ├── Task: API - [METHOD_NAME] - Failure
                          │    └── 200: Next Action
                          └── Default (500+)
                               └── If: statusCode >= 500
                                    ├── Then → Once
                                    │         ├── On First Time
                                    │         │    ├── Sleep: 1s
                                    │         │    ├── Task: API - [METHOD_NAME] - Failure
                                    │         │    └── Execute: 9.1 - API [METHOD_NAME] (retry)
                                    │         └── Afterwards
                                    │              ├── Email Notification
                                    │              └── Task: API - [METHOD_NAME] - Failure
                                    └── Else
                                         ├── Email Notification
                                         └── Task: API - [METHOD_NAME] - Failure
End
```

---

## 🔍 Node-by-Node Breakdown

### ▶️ `Start`

The entry point of the Flow. In production use, this is triggered by a calling Flow or as part of a larger orchestration.

---

### ✅ `Task: API - [METHOD_NAME] - Start`

**Type:** Complete Task Node

Marks the **beginning** of the API call lifecycle for analytics purposes. This fires a goal event into **Cognigy Insights / OData**, enabling you to track:

- How often this API integration is invoked
- Drop-off points if the flow never reaches Success or Failure tasks
- Funnel analysis across the full API call journey

> 💡 **Best Practice:** Always pair a Start task with corresponding Success and Failure tasks to get complete funnel visibility in Insights.

---

### ⏱️ `Code: StartLatencyTimer`

**Type:** Code Node

Captures a timestamp at the start of the API call using `Date.now()` or equivalent, and stores it in the Cognigy context (e.g., `context.latencyStart`).

This is the **first half** of the latency measurement pair.

```javascript
// Example implementation
context.latencyStart = Date.now();
```

---

### 📡 `POST Signal → <API_URL>`

**Type:** HTTP Request Node

The **core API call**. Sends the HTTP request to the target external system.

**Configurable fields:**

- `URL` — Replace `<API_URL>` with the target endpoint
- `Method` — Set to `GET`, `POST`, or `PATCH` depending on the template
- `Headers` — Authorization tokens, Content-Type, etc.
- `Body` — JSON payload (for POST/PATCH templates)
- `Response Storage` — Result is stored in `input.httprequest`

> 💡 **Best Practice:** Store the API URL in a **Cognigy Connection** or **Config Store** rather than hardcoding it. This makes environment promotion (dev → staging → prod) seamless.

---

### ⏱️ `Code: DiffLatencyTimer`

**Type:** Code Node

Calculates the **elapsed time** since `StartLatencyTimer` fired and stores the result in context for logging or Insights reporting.

```javascript
// Example implementation
context.latencyMs = Date.now() - context.latencyStart;
```

> 💡 **Best Practice:** Log this value to an analytics endpoint or include it in your task metadata to identify slow API dependencies over time.

---

### 🔀 `Lookup: StatusCode`

**Type:** Lookup Node

Branches the flow based on `input.httprequest.statusCode`. This is the **central routing mechanism** of the template.

| Branch            | Condition                          | Meaning                                   |
| ----------------- | ---------------------------------- | ----------------------------------------- |
| `202 Ok`          | Status = `202`                     | Request accepted / async processing       |
| `400 Bad Request` | Status = `400`                     | Client-side error (bad input/payload)     |
| `Default`         | Everything else (including `500+`) | Server errors, timeouts, unexpected codes |

> 💡 **Best Practice:** Expand this Lookup with additional cases as needed (e.g., `401 Unauthorized`, `404 Not Found`) to handle your specific API's response contract.

---

### ✅ `Task: API - [METHOD_NAME] - Success`

**Type:** Complete Task Node

Fires on successful API responses. Records a **Success goal** in Cognigy Insights, enabling:

- Success rate tracking over time
- Cross-flow comparison of API reliability
- OData exports for BI/reporting tools

---

### ❌ `Task: API - [METHOD_NAME] - Failure`

**Type:** Complete Task Node

Fires on any failure path (400, 500, or exhausted retries). Records a **Failure goal** in Cognigy Insights.

> 💡 **Best Practice:** Use consistent task naming conventions (`API - [METHOD_NAME] - Failure`) so your Insights dashboards can aggregate failure rates across all API templates with a single filter.

---

### ➡️ `200: Next Action` / `400: Next Action (2)`

**Type:** Comment / Placeholder Nodes

These are **handoff points** — where the template ends and your custom Flow logic begins. Replace these with whatever should happen after a successful or failed API call (e.g., Say Node, set context variables, trigger another sub-flow).

---

### 🔁 `If: statusCode >= 500`

**Type:** If Node

Handles server-side errors by checking if the status code is `>= 500`. Separates transient server errors (worth retrying) from confirmed failures.

**Condition:**

```
{{ input.httprequest.statusCode >= 500 }}
```

---

### 🔂 `Once` (Retry Gate)

**Type:** Once Node

Controls the **retry logic** — ensures the retry only fires **once**, preventing infinite retry loops.

| Branch          | Behavior                                                                      |
| --------------- | ----------------------------------------------------------------------------- |
| `On First Time` | Executes the retry path — sleeps 1 second, then re-executes the Flow          |
| `Afterwards`    | On any subsequent call, skips retry and goes straight to failure notification |

> 💡 **Best Practice:** The `Once` node uses Cognigy's built-in session state. This pattern gives you exactly **one automatic retry** on 500 errors before escalating — a safe, controlled approach to transient failures.

---

### 💤 `Sleep: 1s (1000ms)`

**Type:** Sleep Node

Introduces a **1-second delay** before the retry attempt. This prevents hammering a struggling API endpoint immediately after a 500 error.

> 💡 **Best Practice:** For APIs with known rate limits or cold-start behavior, consider increasing this delay (e.g., 3s–5s) or making it a configurable context variable.

---

### 🔄 `Execute: 9.1 - API [METHOD_NAME]`

**Type:** Execute Flow Node

**Triggers the retry** by re-executing the current Flow from the top. Combined with the `Once` node, this creates a single, clean retry cycle.

---

### 📧 `Email Notification`

**Type:** Send Email Node (appears in both `Afterwards` and `Else` branches)

Sends an **alert email** when the API fails and retries are exhausted. This is your escalation path for operational awareness.

**Recommended email content:**

- Flow name and template ID
- Timestamp of failure
- Status code received
- Session/user ID for traceability

> 💡 **Best Practice:** Route these notifications to a shared team inbox or integrate with a ticketing system (e.g., via a secondary API call to ServiceNow/Jira) rather than a personal email.

---

### 🏁 `End`

The terminal node of the Flow. All branches — success, failure, and retry — converge here.

---

## 🧩 How to Use These Templates

### 1. Import the Template

Download the desired `.json` package and import it into your Cognigy.AI environment via **Manage → Packages**.

### 2. Replace Placeholders

| Placeholder                   | Replace With                                                |
| ----------------------------- | ----------------------------------------------------------- |
| `[METHOD_NAME]`               | Your descriptive method name (e.g., `GET Employee Details`) |
| `<API_URL>`                   | Your target API endpoint (use a Connection for credentials) |
| `200: Next Action`            | Your success path logic                                     |
| `400: Next Action (2)`        | Your failure/fallback logic                                 |
| Email Notification recipients | Your team or ticketing system                               |

### 3. Configure Task Names

Rename all Task nodes to match your method name. Consistent naming is critical for Insights dashboards to work correctly.

### 4. Set Up Cognigy Connections

Store API credentials in **Cognigy Connections** (not hardcoded). Reference them in the HTTP Request node.

---

## 📊 Insights & Analytics

This template is built with **full Insights observability** out of the box. Once deployed, you can track the following goals in your Insights dashboards:

| Goal               | Task Node   | What It Measures               |
| ------------------ | ----------- | ------------------------------ |
| API Call Initiated | `- Start`   | Total invocation volume        |
| API Call Succeeded | `- Success` | Success rate                   |
| API Call Failed    | `- Failure` | Failure rate, retry exhaustion |

Use **OData** to export this data for external BI tools or SLA reporting.

---

## ⚙️ Latency Monitoring

Every template captures API response time via the `StartLatencyTimer` and `DiffLatencyTimer` code nodes. Use this data to:

- Set SLA baselines per endpoint
- Alert on latency regressions
- Compare response times across environments

---

## 🛡️ Error Handling Philosophy

| Error Type         | Template Response                                                               |
| ------------------ | ------------------------------------------------------------------------------- |
| `400 Bad Request`  | Log failure task → hand off to next action (no retry — client must fix payload) |
| `500 Server Error` | Retry once after 1s delay → if still failing, email alert + failure task        |
| Other / Default    | Same as 500 path                                                                |

---

## 🗂️ Repository Structure

```
cognigy-api-templates/
├── README.md                        ← You are here
├── 9.1-GET-Employee-Details/
│   ├── flow.json
│   └── README.md
├── 9.2-POST-Incidents/
│   ├── flow.json
│   └── README.md
├── 9.3-POST-Notes/
│   ├── flow.json
│   └── README.md
├── 9.4-PATCH-Incident-Details/
│   ├── flow.json
│   └── README.md
└── shared/
    └── latency-code-snippets.js
```

---

## 🤝 Contributing

When adding a new API template to this library, ensure it follows the standard architecture above and includes:

- [ ] Start, Success, and Failure Task nodes with consistent naming
- [ ] Latency timer code nodes
- [ ] StatusCode Lookup with at minimum `2xx`, `400`, and `Default` branches
- [ ] Retry logic using `Once` + `Sleep` + `Execute`
- [ ] Email notification on exhausted retries
- [ ] A template-specific `README.md`

---

_Maintained by the Solutions Architecture team. For questions or contributions, open an issue or pull request._
