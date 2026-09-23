# 🧠 AB-620 Cheat Sheet: Microsoft Certified AI Agent Builder Associate
*Microsoft Copilot Studio · Power Platform · Azure AI*

> ⚠️ **Check before you rely on this:** Copilot Studio changes often. Features leave preview, get renamed, and limits change. Compare this sheet with the official **Study Guide** on Microsoft Learn and the <a href="https://github.com/MicrosoftLearning/AB-620T00_build_integrated_ai_agents_in_copilot_studio">AB-620T00 labs</a>. Numbers marked "≈" are approximate.

---

## 1️⃣ Agent Architecture & Orchestration

### 1.1 RAG: Knowledge Sources

📌 **Key concept:** With Retrieval-Augmented Generation (RAG), the agent first finds relevant passages in your data. It then writes an answer from them and cites the sources, so it doesn't rely only on what the LLM already knows.

⚙️ **Configuration:** *Agent → Knowledge → Add knowledge*

| Source | Auth | Typical use | Key points |
|---|---|---|---|
| Public website | None | Public FAQ, product docs | Uses Bing. URL depth is limited (≈ 2 levels). |
| SharePoint / OneDrive | Entra ID (user) | HR policies, internal docs | Uses **the signed-in user's permissions** |
| Uploaded files | None | Static PDFs, DOCX | A **stored copy**. It doesn't sync with the original. |
| Dataverse | Entra ID | Structured business data | Add synonyms and a glossary |
| Copilot / Graph connectors | Entra ID | ServiceNow, Salesforce, etc. | Needs M365 setup |
| Azure AI Search | Key / Entra ID | Large custom vector index | You control chunking and embeddings |

**Settings to know (Generative AI):**
- **Allow the AI to use its own general knowledge:** turn off to keep answers grounded in your sources.
- **Content moderation** (Low/Medium/High): higher means stricter filtering and fewer answers. Some protections (prompt-injection defense, certain sensitive-data blocking) are **always on** and can't be disabled at any level.
- **Generative answers node** (in a topic): limits the search to specific sources.

⚙️ **Azure AI Search + Foundry model catalog:** Add it as a **data connection** (not a manual endpoint/key wire-up — that can create a broken connection). Only **one vector index per connection**. Beyond RAG, **Azure AI Foundry's model catalog** ("bring your own model", 1,800+ models) lets you swap the model used by **prompts** or the agent's **primary orchestration model** — separate settings from each other.

🚨 **Exam traps:**
- *"No answers from SharePoint."* → The agent is set to **No authentication**. SharePoint needs Entra ID sign-in.
- *"Reduce hallucinations."* → Turn off general knowledge and raise moderation. Adding trigger phrases doesn't help.
- *"The document changes weekly."* → Use SharePoint, not an upload.
- *"Need multiple indexes."* → One Azure AI Search connection = **one index**; add another connection.
- Experimental/preview Foundry models aren't recommended for production and may process data outside your org's geographic boundary.

---

### 1.2 Classic vs Generative Orchestration

📌 **Key concept:**
- **Classic:** picks one topic by matching **trigger phrases**.
- **Generative:** the LLM builds a **multi-step plan** across topics, tools, knowledge and agents. It chooses them by **name and description**.

| Criterion | Classic | Generative |
|---|---|---|
| How it chooses | Trigger phrases | **Descriptions** |
| Several tools per turn | ❌ | ✅ |
| Collecting inputs | Question nodes | **Automatic slot filling** from the conversation |
| Predictability | High | Medium |
| Best for | Scripted, regulated flows | Open-ended assistants |

⚙️ **Configuration:** *Settings → Generative AI → Orchestration: Generative*. Write clear **Agent instructions** covering scope, tone and which tools to prefer. Write descriptions that say when to use the item, for example: *"Use when the user wants the status of an existing order by order number."*

🚨 **Exam traps:**
- *"A topic is never chosen under generative orchestration."* → **Fix its description**, not its trigger phrases.
- *"The agent re-asks for info the user already gave."* → Define **topic inputs** so slot filling can reuse it.
- *"The process must be identical every time."* → Use explicit topic logic or classic orchestration.

---

### 1.3 Topics & Triggers

📌 **Key concept:** A topic is a reusable piece of conversation. Topics are either **Custom** or **System**.

| Trigger / system topic | Purpose |
|---|---|
| Phrases (recognized intent) | Standard intent trigger |
| Conversation Start | Greeting |
| Fallback | No intent matched |
| Multiple Topics Matched | "Did you mean…?" |
| Escalate | Hand off to a human |
| On Error | Global error handling |
| End of Conversation | CSAT survey |
| Sign in | Forces authentication |
| Reset Conversation | Clears variables |
| Event / Activity / Message received, Inactivity | Channel events and timeouts |

**Autonomous triggers** (for example "When a new email arrives" or "When a Dataverse row is added") run through Power Automate with **no user present**. They use the **maker's connections**.

**Key nodes:** Send message, Ask a question, Condition, Set / Parse value / Clear variable, **Redirect**, **End current topic**, **End all topics**, **Transfer conversation**, Call tool, **HTTP request**, Generative answers, Adaptive Card.

🚨 **Exam traps:**
- **Redirect** runs the target topic and then *returns* to the caller. **End all topics** clears the whole stack.
- Autonomous triggers run as the **maker**, so apply least privilege.
- *"React when the web page loads."* → Use an **Event received** trigger, not phrases.

---

### 1.4 Multi-Agent & A2A

| Type | Description | When to use |
|---|---|---|
| **Child (inline) agent** | Lives inside the parent agent | Split up a big agent. Shares the parent's lifecycle. |
| **Connected Copilot Studio agent** | A separate published agent | Reuse across teams, with **its own lifecycle and owners** |
| **Foundry / Fabric data agent** | External Microsoft agents | Pro-code reasoning, analytics |
| **A2A agent** | Open Agent-to-Agent protocol (Linux Foundation) | Agents from other vendors or platforms |

📌 **A2A basics:** Each agent publishes an **Agent Card**, a JSON file describing its skills, endpoint and auth. It is usually at `/.well-known/agent.json`; newer spec versions use `agent-card.json`. Agents exchange **tasks and messages** over HTTPS using JSON-RPC, with SSE for streaming.

⚙️ **Configuration:** *Agents → Add an agent → Copilot Studio / Foundry / A2A*. Enter the endpoint and auth, and write a clear **description**, because the parent agent uses it to route requests.

🚨 **Exam traps:**
- *"Teams need separate release cycles."* → Use **connected agents**, not child agents.
- Multi-agent routing **requires generative orchestration**.
- **A2A** connects agent ↔ agent. **MCP** connects agent ↔ tools and data.

---

### 1.5 Adaptive Cards

📌 **Key concept:** Platform-agnostic JSON UI, rendered natively by the host channel. Two patterns: **Ask with Adaptive Card** node (*interactive* — needs ≥ 1 submit button, captures input into variables) vs. a card attached to a **Message/Question** node (*non-interactive*, display only).

⚙️ **Configuration:** *Topic → Add node → Ask with Adaptive Card* → **Edit adaptive card** (visual designer or raw JSON payload editor). Output variables auto-generate from card inputs. Bind dynamic data into the card with **Power Fx**.

| Host | Max schema version |
|---|---|
| Web Chat (default website) | 1.6 |
| Copilot Studio test chat | 1.6 |
| Microsoft Teams | 1.5 |
| Omnichannel live chat widget | 1.5 |

🚨 **Exam traps:**
- **Teams and Omnichannel cap at schema v1.5** — a v1.6-only feature (e.g. `Action.Execute`) can silently fail there.
- Use the Adaptive Card **node** only when you need a submit button/input; for pure display, attach the card to a **Message** node instead.
- Sending several cards in a row? Give each `Action.Submit` a **unique ID** in its `data` payload — otherwise a user can click a stale button from an earlier card.

---

## 2️⃣ Connectivity & Extensions

### 2.1 Custom Connectors (REST)

📌 **Key concept:** A reusable wrapper around a REST API, defined in **OpenAPI 2.0 (Swagger)**. It works in Copilot Studio, Power Automate and Power Apps.

⚙️ **Steps:**
1. **Create:** start blank, from an OpenAPI file or URL, or from a Postman collection.
2. **General:** set the host and base URL.
3. **Security:** None, API Key, Basic or OAuth 2.0.
4. **Definition:** add operations (`operationId`, summary, description) and schemas.
5. **Code (optional):** C# to transform requests or responses.
6. **Test**, then add the connector to the agent as a **Tool**.

```yaml
swagger: '2.0'
info: { title: Orders API, version: '1.0' }
host: api.contoso.com
basePath: /v1
schemes: [https]
securityDefinitions:
  api_key: { type: apiKey, in: header, name: x-api-key }
paths:
  /orders/{orderId}:
    get:
      operationId: GetOrder
      summary: Get order status
      description: Returns status and ETA for an order ID
      parameters:
        - { name: orderId, in: path, required: true, type: string }
      responses:
        '200':
          description: OK
          schema:
            type: object
            properties:
              status: { type: string }
              eta: { type: string, format: date-time }
```

| Option | When to use |
|---|---|
| Custom connector | Reusable, **governed by DLP**, can go in solutions |
| HTTP Request node | Quick one-off call |
| Power Automate flow | Multi-step logic, loops, transformations |

🚨 **Exam traps:**
- The spec must be **OpenAPI 2.0**. Convert 3.0 files first.
- The **summary and description** decide whether generative orchestration picks the tool.
- New custom connectors fall into the DLP **Non-business** group by default and can be blocked.
- *"The API is on-premises."* → Use the **On-premises data gateway**.

---

### 2.2 MCP (Model Context Protocol)

📌 **Key concept:** MCP is an open standard, created by Anthropic, for exposing **tools, resources and prompts**. The agent **discovers the tools automatically**, so when the server changes, the agent picks up the changes.

⚙️ **Configuration:** *Tools → Add a tool → Model Context Protocol* (wizard), or a custom connector with `x-ms-agentic-protocol: mcp-streamable-1.0`.
- Transport: **Streamable HTTP**. SSE is deprecated.
- Auth: None, API key or OAuth 2.0.
- You can turn individual tools on or off.

```yaml
paths:
  /mcp:
    post:
      summary: Contoso MCP Server
      x-ms-agentic-protocol: mcp-streamable-1.0
      operationId: InvokeMCP
      responses: { '200': { description: OK } }
```

| | REST connector | MCP |
|---|---|---|
| Operations | Fixed in the OpenAPI file | **Discovered dynamically** |
| When the API changes | Edit the connector | Update the server only |
| Orchestration | Any | **Generative required** |

🚨 **Exam traps:**
- *"Tools must update without editing the agent."* → Use MCP.
- MCP needs **generative orchestration**.
- MCP connections are still subject to **DLP**.

---

### 2.3 Power Automate Cloud Flows (as a Tool)

📌 **Key concept:** The trigger is **"When an agent calls the flow"** (formerly "Run a flow from Copilot"). The flow returns data with the **"Respond to the agent"** action.

| Supported inputs/outputs | Not supported natively |
|---|---|
| Text, Number, Boolean, Date (plus File input) | Tables and arrays |

➡️ To return a list, **serialize it to a JSON string** in the flow and parse it in the topic:

```powerfx
Set(Topic.Orders,
  ForAll(Table(ParseJSON(Topic.OrdersJson)),
    { Id: Text(ThisRecord.Value.id), Status: Text(ThisRecord.Value.status) }))
```

⚙️ **Constraints:**
- **Timeout ≈ 100 seconds** for the response. Put "Respond to the agent" early and let slow work continue after it.
- The flow must be in the **same environment**, and ideally the **same solution**, as the agent.
- Choose the connection mode: the **maker's connection** or the **end user's credentials**.

🚨 **Exam traps:**
- *"The flow times out."* → Move the **Respond** step earlier.
- *"Return a list of records."* → Send JSON text and use **Parse value** or `ParseJSON`.
- *"The flow doesn't appear in the agent."* → Check the trigger type, environment and solution.
- Don't confuse this with a native **Agent flow** (below) — both can use "When an agent calls the flow," but a Power Automate flow is authored in Power Automate; an Agent flow is authored inside Copilot Studio's own **Flows** designer.

---

### 2.4 Agent Flows (Native Flow Builder)

📌 **Key concept:** Copilot Studio's **own** low-code flow builder (separate page from Power Automate, though it runs on the same engine). A flow can run standalone or be attached to an agent as a **tool**, in which case it needs the same **"When an agent calls the flow"** trigger + **"Respond to the agent"** action.

⚙️ **Configuration:** *Copilot Studio → Flows (left nav) → New flow → Agent flow* → add input parameters on the trigger, add actions, add output parameters on "Respond to the agent" → **Publish** → attach via *Agent → Tools → Add tool*.

**Human-in-the-loop actions:**

| Action | Purpose | Status |
|---|---|---|
| **Request information (RFI)** | Pauses the flow, notifies a human reviewer, resumes with their input | GA |
| **Multistage approvals** | Mixes manual and AI approval stages with conditions between them | **Preview** |

⚙️ **Key limits:** The **"Respond to the agent"** action must reply within **≈ 100 seconds** (real-time; toggle **Asynchronous response = Off**). Actions placed *after* the response can keep running for up to **30 days** ("express mode"). Runs are visible in the agent's **Activity** page (Activity map / Transcript).

🚨 **Exam traps:**
- The 100-second limit is on the **response action**, not the whole flow.
- **Multistage/AI approvals is Preview** — don't treat it as GA on the exam.
- An **Agent flow** lives natively in Copilot Studio; a **Power Automate cloud flow** is a separate canvas you switch to — pick the one the scenario names.

---

### 2.5 Computer Use

📌 **Key concept:** A **tool** (not a channel, not an orchestration mode) that lets the agent operate a Windows virtual desktop/browser via simulated mouse, keyboard and screen reasoning — for apps/sites with no API. **Requires generative orchestration.**

⚙️ **Configuration:** *Agent → Tools → Add tool → New tool → Computer use* → describe the task in natural language.

| "Machine" option | Description | Status |
|---|---|---|
| **Hosted browser** (Windows 365 for Agents) | Microsoft-managed, zero setup; **not** Entra-joined/Intune-managed; not for production | Preview |
| **Cloud PC pool** | Windows 365 Cloud PCs, Entra-joined, Intune-managed; production-ready | Preview |
| **Bring-your-own-machine** | A Windows device you register/manage | — |

**Other settings:** **Credentials to use** — *maker-provided* (default; shared access risk) vs. *end-user credentials*. **Stored credentials** live in Power Platform storage or your own **Azure Key Vault**. **Human supervision** — an email reviewer is notified if the agent flags a potentially harmful instruction.

⚙️ **Monitoring:** Agent → **Activity** page → Activity map / Transcript (step-by-step reasoning + screenshots). Detailed logs can be stored in **Dataverse** (toggle in *Power Platform admin center → Environments → Settings → Products → Features*).

🚨 **Exam traps:**
- Computer use needs **generative orchestration** — classic-orchestration agents can't use it.
- Only **Cloud PC pool** and **bring-your-own-machine** are Entra-joined/Intune-managed; the **hosted browser** is not.
- **Maker-provided credentials** mean anyone the agent is shared with inherits the maker's access on that machine — a common "what's the security risk" scenario.

---

### 2.6 Custom Prompts (Prompt Builder) as a Tool

📌 **Key concept:** A **prompt** is a reusable, general-purpose instruction-following action (summarize, classify, extract, rewrite…) with defined **inputs** and an **output**, built in the **Prompt builder**. This is **not** the same as the **Generative answers** node: a prompt is instructions you author yourself; generative answers is automatic RAG grounding over knowledge sources with citations.

⚙️ **Configuration — three places to add a prompt:**
- **Agent tool:** *Agent → Tools → New tool → Prompt*
- **Topic node:** *Topic → Add node → Add a tool → New prompt*
- **Agent flow node:** *Flow → Insert action → AI capabilities → Run a prompt*

A prompt = **Instruction** (what to do) + **Context** (data to act on). Choose the **model** from a dropdown (default is a lightweight GPT model; more models — GPT, Claude, Grok, etc. — are available via the **Azure AI Foundry model catalog**, "bring your own model"). Set **temperature** for creativity vs. consistency.

🚨 **Exam traps:**
- *"Summarize this ticket in a fixed format"* → **Prompt**. *"Answer using our SharePoint docs"* → **Generative answers**.
- Prompts run in **Copilot Studio** consume **Copilot Credits**; the same prompt reused in **Power Apps/Power Automate** consumes **AI Builder / prompt builder credits** instead — separate billing pools.
- **No-authentication (anonymous)** agents can't use **Dataverse** as a knowledge source inside a prompt.

---

## 3️⃣ Logic & Data Manipulation

### 3.1 Variable Scopes

| Scope | Prefix | Lifetime | Use |
|---|---|---|---|
| Topic | `Topic.` | Current topic (can be an input or output) | Local data, passing values between topics |
| Global | `Global.` | **The whole conversation (session)** | Customer ID, language |
| System | `System.` | Read-only | `System.User.DisplayName`, `System.Activity.Text`, `System.Conversation.Id` |
| Environment | `Env.` | Per environment (from the solution) | URLs, config |
| Auth | `System.User.*` | Signed-in user | `System.User.Email`, `System.User.AccessToken` (manual auth) |

⚙️ **Global variable options:** *"External sources can set values"* lets a channel set the value, for example through a URL parameter or the Web Chat store. Global variables are cleared by the **Reset Conversation** topic.

🚨 **Exam traps:**
- Global ≠ persistent. Values are lost after the session ends. To keep data across sessions, store it in **Dataverse**.
- Pass data between topics with **topic input/output variables**. It's the cleaner design.
- `System.*` variables are **read-only**.

### 3.2 Power Fx Essentials

```powerfx
// Strings
$"Order {Topic.OrderId} is {Topic.Status}"
Upper() Lower() Proper() Trim() Len()
Left(s,3) Right(s,2) Mid(s,2,4)
Substitute(Topic.Phone, " ", "")
Find("@", Topic.Email)
IsMatch(Topic.Email, Match.Email)
IsMatch(Topic.OrderId, "^ORD-\d{6}$")
Split("a;b;c", ";")                 // → table with a "Value" column
Text(Now(), "yyyy-mm-dd")  Value("42")

// Tables
CountRows(t)  First(t)  Last(t)  Index(t, 2)
Filter(t, Status = "Open")
LookUp(t, Id = Topic.OrderId).Status
Sort(t, Date, SortOrder.Descending)
ForAll(t, ThisRecord.Id)
Concat(t, Id & " - " & Status, Char(10))   // table → single string

// Logic & JSON
If(c, a, b)  Switch(x, 1, "A", "B")
IsBlank(x)  Coalesce(x, "default")
ParseJSON(s)  JSON(record)
```

⚙️ **Where you can use Power Fx:** the *Formula* option in Set variable, Condition nodes (formula mode), node inputs, and inside messages with `{ }`.

🚨 **Exam traps:**
- `Split` returns a **table**, not an array of strings.
- `ParseJSON` returns an **untyped object**. Wrap values in `Text()`, `Value()` or `Table()`.
- `Concat` joins the rows of a table. `Concatenate` joins individual strings.
- Canvas-app functions such as `Collect`, `Patch` and `Navigate` are **not available**.

---

## 4️⃣ ALM & Governance

### 4.1 Environments

| Environment | Role | Solution type |
|---|---|---|
| Default | Everyone is a maker | ❌ Avoid for production agents |
| Developer / Dev | Building | **Unmanaged** |
| Sandbox (Test/UAT) | Validation | **Managed** |
| Production | Live | **Managed**, restricted access |

**Managed Environments** add sharing limits, solution checker enforcement and usage insights. They are **required for Power Platform Pipelines**.

### 4.2 Solutions

| | Unmanaged | Managed |
|---|---|---|
| Where | Dev | Test / Prod |
| Editable | ✅ | ❌ |
| Deleting the solution | Components remain | **Components are removed** |

⚙️ **Best practices:**
- Build the agent **inside a custom solution** and set it as the **preferred solution**.
- Use a custom **publisher** with its own prefix (for example `cts_`).
- Use **connection references** instead of fixed connections.
- Use *Add required objects* to include flows, connectors and environment variables.

### 4.3 Pipelines & CI/CD

| Tool | Use |
|---|---|
| **Power Platform Pipelines** | Built-in, low-code Dev → Test → Prod. Needs a **host environment** and Managed Environments. |
| Azure DevOps / GitHub Actions | Pro-code CI/CD with source control and approvals |
| PAC CLI | Scripting |

```bash
pac auth create --environment https://contoso-dev.crm.dynamics.com
pac solution export --name ContosoAgent --path ./ContosoAgent.zip --managed
pac solution create-settings --solution-zip ./ContosoAgent.zip --settings-file settings.json
pac solution import --path ./ContosoAgent.zip --settings-file settings.json
```

```yaml
- uses: microsoft/powerplatform-actions/export-solution@v1
  with:
    environment-url: ${{ secrets.DEV_URL }}
    app-id: ${{ secrets.CLIENT_ID }}
    client-secret: ${{ secrets.CLIENT_SECRET }}
    tenant-id: ${{ secrets.TENANT_ID }}
    solution-name: ContosoAgent
    managed: true
```

### 4.4 Environment Variables

📌 **Key concept:** Configuration values that differ in each environment, such as API URLs, SharePoint sites or IDs.

| Type | Example |
|---|---|
| Text / Number / Boolean / JSON | API base URL, feature flag |
| Data source | SharePoint site or list |
| **Secret** | Stored in **Azure Key Vault** (you store a reference to it) |

⚙️ In an agent, reference them as `Env.cts_ApiBaseUrl`. At import time, supply values through the **deployment settings file** or the pipeline prompt.

🚨 **Exam traps:**
- *"A different URL in each environment without editing the agent."* → Use an environment variable.
- *"Store an API key securely."* → Use a **Secret** environment variable backed by Key Vault. Don't use Text.
- A **current value** stored in the solution overrides the value you set at import time. Remove the current value before exporting.
- *"Connections break after import."* → Use **connection references**, mapped in the settings file.

### 4.5 Governance

- **DLP policies** group connectors as Business, Non-business or Blocked. They can also restrict Copilot Studio features such as knowledge sources, channels and HTTP calls.
- **Analytics** covers sessions, resolution rate, escalation rate, CSAT and unrecognized topics. Audit logs go to **Microsoft Purview**.
- **Sharing:** editor vs viewer roles. Security groups control who can use the agent.

🚨 **Trap:** *"Block makers from publishing agents without authentication."* → Use **DLP** or tenant/environment settings in the admin center, not a per-agent setting.

### 4.6 Testing & Evaluation (Agent Evaluation)

📌 **Key concept:** Distinct from ad-hoc **Test chat** (one conversation at a time). **Agent evaluation** runs a **test set** (up to **100 test cases**) automatically against the agent to measure accuracy, relevancy and safety **at scale and repeatably**.

⚙️ **Configuration:** *Agent → Evaluation → New evaluation* → choose **Single responses** or **Conversational** (multi-turn) → populate cases via a **Quick question set** (auto-generated from the agent's instructions), manual entry, spreadsheet import, or by converting saved test-chat conversations.

| Method | Measures | Scoring |
|---|---|---|
| General quality | Overall response quality | Scored % |
| Content safety | Hate, violence, self-harm, sexual content | Pass/fail |
| Compare meaning | Semantic match to an expected answer | Scored % |
| Tool use | Whether the expected tool/resource was used | Pass/fail |
| Keyword / exact / text-similarity match | Literal match to expected text | Pass/fail or scored % |
| Custom | Your own evaluation instructions | Pass/fail per label |

🚨 **Exam traps:**
- **100 test cases max** per test set.
- Results are retained **89 days only** — export to CSV to keep them longer.
- Only **one evaluation run at a time** per agent.
- *"Test a single scripted conversation."* → Use **Test chat**. *"Score accuracy across many scenarios automatically."* → Use **Agent evaluation**.
- Safety evaluators **complement**, they don't replace, Responsible AI content filters.

### 4.7 Application Insights Monitoring

📌 **Key concept:** The built-in **Analytics** tab gives dashboards out of the box (see 4.5). **Application Insights** is an *additional*, deeper telemetry pipeline into **Azure Monitor** for custom KQL queries, alerting and long-term retention — none of which the built-in Analytics tab provides.

⚙️ **Configuration:** *Agent → Settings → Advanced → Application Insights* → paste the **Connection string** from your Azure Application Insights resource. Toggles: **Enable logging**, **Log conversation details** (user ID/name, message text), **Log sensitive Activity properties**, **Node execution events** (one event per topic node run).

🚨 **Exam traps:**
- Requires an **Azure subscription** — it isn't "free" like the built-in Monitor tab.
- Turning on **Log conversation details** / **sensitive Activity properties** has privacy implications — a likely compliance-scenario question.
- Analyze data in the **Azure Portal → Logs (KQL)** against the `customEvents` table, not inside Copilot Studio.

---

## 5️⃣ Security & Channels

### 5.1 Authentication Options

| Option | Who signs in | Channels | Key facts |
|---|---|---|---|
| **No authentication** | Nobody | Public websites, demo site | ❌ No SharePoint knowledge, no user context |
| **Authenticate with Microsoft** (Entra ID, automatic) | Entra ID user, through SSO | **Teams, M365 Copilot, Power Apps** | No setup needed. **Doesn't work** on a custom website. |
| **Authenticate manually** | Entra ID v2 or any **OAuth 2.0** identity provider | Any channel, including a **custom website** | You register the app yourself. You get `System.User.AccessToken`. |

⚙️ **Manual Entra ID setup:**
1. Create an app registration and add the **redirect URI** `https://token.botframework.com/.auth/web/redirect`.
2. Create a client secret or certificate and add **API permissions** (for example `User.Read`, `Sites.Read.All`).
3. In Copilot Studio, enter the Client ID, Secret, Tenant, **Token exchange URL** (for SSO) and **Scopes** (for example `profile openid`).
4. For web SSO, **expose an API** (`api://<id>/...`) and set the token exchange URL to that scope.
5. Optional: turn on *"Require users to sign in"*.

🚨 **Exam traps:**
- *"SSO on a custom website."* → Use **manual** authentication with a token exchange URL. Automatic auth only works in Microsoft channels.
- *"Call a user-scoped API."* → Pass `System.User.AccessToken` in the Authorization header. This only works with manual auth.
- *"Sign-in loop / redirect error."* → The redirect URI is missing or wrong.
- Changing auth settings requires you to **republish** the agent.

### 5.2 Human-in-the-Loop (Handoff)

📌 **Key concept:** The **Escalate** system topic or a **Transfer conversation** node hands the conversation to a live agent. The **full conversation transcript and variables** go along with it.

| Target | Notes |
|---|---|
| **Dynamics 365 Contact Center / Omnichannel for Customer Service** | Native integration. Context is passed automatically. |
| Other engagement hubs (Salesforce, ServiceNow, Genesys, LivePerson…) | Through the Bot Framework or partner adapters |

⚙️ **Setup:** *Settings → Customer engagement hub → Dynamics 365 Omnichannel → Connect*. Workstreams and routing rules are configured in Contact Center. Map context variables (for example `Global.CustomerId`) for routing.

**Other human-in-the-loop patterns:**
- **Request for information** / approval steps in agent flows, used by autonomous agents.
- **Tool confirmation**: require the user to confirm before an action runs.

🚨 **Exam traps:**
- *"The live agent must see what the user already said."* → Handoff to Omnichannel **passes the transcript automatically**. You don't need to build anything.
- *"Handoff doesn't work."* → Check that the Omnichannel channel is connected **and** the agent is republished.
- *"An autonomous agent must get a manager's approval."* → Add an **approval / human step** in the agent flow, not Escalate.

### 5.3 Multi-Channel Deployment

| Channel | Auth | Notes |
|---|---|---|
| **Teams & Microsoft 365 Copilot** | Microsoft (SSO) | Publish, then **submit to the admin** for org-wide availability, or share it |
| **Demo website** | Any | For testing only |
| **Custom website** | None or Manual | Iframe embed code or **Web Chat (Direct Line)**. **Web channel security** uses a secret or token. |
| Mobile / custom app | Manual | Direct Line API |
| Facebook, WhatsApp, SMS | Manual | WhatsApp and SMS go through **Azure Communication Services** or Omnichannel |
| SharePoint, Power Pages | Varies | Embedded experiences |

⚙️ **Web security:** turn on **"Require secured access"** under *Settings → Security → Web channel security*. Your backend exchanges the **secret** for a **Direct Line token**. Never put the secret in client-side code.

```javascript
// Backend: exchange secret → token (never expose the secret in the browser)
const res = await fetch("https://directline.botframework.com/v3/directline/tokens/generate",
  { method: "POST", headers: { "Authorization": "Bearer " + DIRECT_LINE_SECRET } });
const { token } = await res.json();
```

🚨 **Exam traps:**
- Publishing is **required after every change** before any channel shows the update.
- *"The Teams agent isn't visible to all users."* → A **Teams admin must approve it** in the Teams admin center.
- *"Protect the web agent from unauthorized embedding."* → Use **secured access** with **token exchange on the backend**.
- Adaptive Card rendering **differs by channel**. Test in each target channel.

---

## 🎯 Final "Which Option?" Quick Reference

| Scenario | Answer |
|---|---|
| Answers from internal documents, respecting permissions | SharePoint knowledge + Entra ID auth |
| Generative orchestration ignores a tool | Improve its **description** |
| Call an agent from another vendor | **A2A** |
| Tools that update themselves dynamically | **MCP** |
| Reusable, governed REST API | **Custom connector** (OpenAPI 2.0) |
| Return a list from a flow | Serialize to JSON, then `ParseJSON` |
| Data shared across topics in one session | `Global.` variable |
| Remember data across sessions | Dataverse |
| Different API URL per environment | **Environment variable** |
| Secret API key | Secret env var + **Key Vault** |
| Low-code Dev → Test → Prod | **Power Platform Pipelines** (Managed Environments) |
| SSO on a custom website | **Manual auth** + token exchange URL |
| Hand off to a live agent with context | Escalate → **D365 Omnichannel** |
| Secure web embed | **Web channel security** + backend token exchange |
| Automate a legacy app/site with no API | **Computer use** tool (needs generative orchestration) |
| Human approval inside an automated flow | **Agent flow** with Request-for-Information / approval action |
| Score accuracy across many scenarios before go-live | **Agent evaluation** (test set, up to 100 cases) |
| Custom telemetry, alerting, long-term retention | **Application Insights** (beyond built-in Analytics) |
| Summarize/classify/rewrite with your own instructions | **Custom prompt** (Prompt builder), not Generative answers |
| Card feature not working in Teams | Teams caps Adaptive Card schema at **v1.5** |

**Good luck on AB-620! 🚀** Practise every scenario in a Developer environment. Hands-on experience helps most with the "which option?" questions.
