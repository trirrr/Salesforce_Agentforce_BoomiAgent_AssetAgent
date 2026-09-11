# Aprimo SalesForce Account Lookup — Agentforce Tool

> **Built by Rob Rojas, Technical Engagement Manager & Solutions Architect at Aprimo**
> *Salesforce Agentforce · Boomi iPaaS · Salesforce CRM · Enterprise AI Integration*

---

## What This Does (Plain English)

Imagine a customer calls in and says, *"Hi, I'm from Smart Health Systems."* Normally, a support agent would need to pause the conversation, open Salesforce in a separate window, search for that company, scan the account record, and then come back to the conversation — all while the customer waits.

**This tool eliminates that pause entirely.**

When a Salesforce AI agent (Agentforce) hears a customer mention their company, it automatically looks up that account in Salesforce CRM *in the background* — in 3–4 seconds — and comes back with a full picture: the company's industry, size, revenue tier, account rating, sales region, and more. The AI agent then uses all of that context to respond intelligently, personalize the conversation, and make smart decisions — without any human having to lift a finger.

**The result:** AI agents that behave like experienced sales reps who already know the customer — not like bots reading from a script.

---

## Business Value at a Glance

| Without This Tool | With This Tool |
|---|---|
| Agent asks customer to identify themselves | Agent already knows who they are |
| Rep manually searches CRM mid-conversation | Account data loads automatically in background |
| Generic responses for all customers | Personalized by industry, tier, company size |
| Human routes calls based on account value | AI routes intelligently based on real CRM data |
| Salesforce data siloed from AI layer | Salesforce and Agentforce work as one system |

---

## What Gets Built Here

This repository contains a **production-ready integration** that connects three enterprise systems:

```
Customer Message
      ↓
Salesforce Agentforce (AI Agent)
      ↓  [invokes this tool]
Boomi iPaaS (middleware layer)
      ↓  [authenticated SOQL query]
Salesforce CRM (account data)
      ↓  [structured JSON response]
Agentforce (uses context to respond)
```

The middleware layer (Boomi) handles all the hard parts: authentication, query building, filtering, and returning clean structured data the AI can actually use.

---

## Example: What This Looks Like in Action

**Customer says:**
> "Hey, I'm from Smart Health Systems. Can you walk me through your DAM features?"

**What happens behind the scenes (invisible to customer):**
1. Agentforce detects a company mention and invokes this tool
2. Tool sends the company name to the Boomi endpoint
3. Boomi authenticates with Salesforce and runs the account lookup
4. Salesforce returns the full account record
5. Boomi returns structured data to Agentforce

**What the AI agent responds:**
> "Welcome! I can see Smart Health Systems is one of our Hot-rated Healthcare prospects with around 500 employees. Let me tailor this to what matters most for healthcare organizations your size…"

The customer gets a personalized, informed response. The sales team gets a better-qualified interaction. Nobody had to do anything manually.

---

## Architecture

### The 5-Shape Boomi Process

```
[1] WSS Listener
    Receives the POST request from Agentforce with accountName (and optional filters)

[2] Groovy Parser
    Extracts field values from the JSON body
    Builds a dynamic SOQL WHERE clause with wildcards for fuzzy matching
    Uses ExecutionUtil.setDynamicProcessProperty() — critical (see Gotchas)

[3] Salesforce Query
    OAuth-authenticated connection to Salesforce
    Executes the SOQL against the Account object
    Returns up to 30+ fields per account record

[4] Data Mapper
    Transforms Salesforce field names into clean JSON keys
    Handles null fields gracefully

[5] Response Handler
    Returns structured JSON to the calling Agentforce agent
    Agent receives data and incorporates it into its next response
```

### Agentforce Side

The tool runs as one of three specialized subagents in the Agentforce architecture:

- **Salesforce Account Integration** — this tool (account lookups)
- **Automated Asset Surfacing** — DAM content retrieval
- **Content Relevance Matching** — industry-to-collection mapping

Each subagent owns exactly one action, with mutually exclusive classification descriptions so Agentforce's Atlas reasoning engine routes cleanly without ambiguity.

---

## API Reference

### Endpoint

```
POST https://aprimo-prod.boomi.cloud/ws/simple/executeAprimo_SF_AccountLookup
```

### Authentication

Basic Auth using the `Salesforce-aprimo-master` credential configured in Boomi Shared Web Server.

> ⚠️ **Note:** This is Boomi's Shared Web Server auth — separate from the Boomi Platform API credentials. Do not confuse the two. They authenticate against entirely different surfaces.

### Request Body

```json
{
  "accountName": "Smart Health Systems",
  "industry": "Healthcare",
  "type": "Prospect",
  "billingCountry": "United States"
}
```

All fields except `accountName` are optional filters. Omit them to broaden the search.

### Response

```json
{
  "accounts": [
    {
      "id": "001Hs00005uiMzZIAU",
      "name": "Smart Health Systems",
      "industry": "Healthcare",
      "type": "Prospect",
      "billingCountry": "France",
      "billingState": "Ile-de-France",
      "billingCity": "Paris",
      "annualRevenue": 50000000,
      "numberOfEmployees": 500,
      "rating": "Hot",
      "accountOwnerName": "Jane Rivera",
      "lastActivityDate": "2026-08-15",
      "sdoPartnershipStatus": "Active",
      "tier": "Enterprise",
      "sdoSalesRegion": "EMEA",
      "opportunityWinRate": 0.62,
      "numberClosedOpportunities": 4
    }
  ]
}
```

Full response includes 30+ fields across text, numeric, status, date, and location categories.

---

## How Agentforce Agents Use This Data

Once the agent has account context, it can:

**Personalize responses**
Reference company size, industry, and account rating in real-time ("I see you're a Hot-rated Enterprise prospect in Healthcare...")

**Route intelligently**
Direct high-value accounts (Hot rating, high revenue) to senior reps or specialized teams automatically

**Tailor messaging**
Adjust tone, recommendations, and product emphasis based on industry type and account tier

**Surface account history**
Reference last activity date, closed opportunities, win rate, and partnership status to inform next steps

**Fill data gaps**
When Salesforce records are incomplete, the integration supports enrichment branching — an AI inference step can infer missing fields (industry, country) from the company name alone before proceeding

---

## Configuration

### Boomi Process Settings

| Setting | Value |
|---|---|
| Process Name | `Aprimo_SF_AccountLookup` |
| Listener Type | Web Services Server (WSS) |
| Atom Tier | Intermediate or Basic (not Advanced — see Gotchas) |
| Auth Type | Basic Auth |
| Salesforce Connection | OAuth via Named Credential |

### Supported Query Filters

| Filter Field | SOQL Behavior |
|---|---|
| `accountName` | LIKE with % wildcard (case-insensitive fuzzy match) |
| `industry` | LIKE with % wildcard |
| `type` | LIKE with % wildcard |
| `billingCountry` | LIKE with % wildcard |

---

## Deployment Sequence

Every deployment must follow this exact order. Skipping any step leaves the old version active.

```
1. BUILD
   Make changes in Boomi Integration → Save

2. DEPLOY
   Deploy → Create Packaged Component
   Select Production Environment → Next: Review → Deploy

3. RESTART LISTENERS
   Manage → Atom Management → [Production Atom]
   Listeners → Restart All

4. VERIFY
   Listener list should show Aprimo_SF_AccountLookup as Active
   If missing: confirm the Start Shape is configured as a WSS Listener

5. TEST
   POST a sample payload to the endpoint
   Confirm account data returns in response body
   Check Boomi Process Reporting to verify the SOQL built correctly
```

---

## Agentforce Setup

1. In **Salesforce Setup → Agent Actions**, create a new External Service action pointing to the Boomi endpoint
2. Define input variable: `accountName` (Text, required)
3. Define output: structured JSON (the agent receives this and incorporates it into its response)
4. In **Agentforce Builder**, add the action to the `Salesforce Account Integration` subagent only
5. Write a classification description that makes this subagent the exclusive owner of account lookup requests
6. **Save → Commit → Activate** (all three steps — Save alone does not deploy)

> ⚠️ **Critical:** In Agentforce Builder, saving does not activate the agent. You must Commit and then Activate separately. Missing this step means the agent continues running the prior version.

---

## Gotchas & Lessons Learned

These are hard-won discoveries from the production build — none of them are in the official documentation.

### 1. ExecutionUtil vs. props for Dynamic Process Properties
**Problem:** Setting document-scoped dynamic properties using `props.setProperty()` silently fails with no error.
**Fix:** Always use `ExecutionUtil.setDynamicProcessProperty()` for the `document.dynamic.userdefined.*` scope. The process appears to run successfully, which is what makes this so hard to find.

### 2. The Atom Tier Requirement
**Problem:** Advanced tier atoms do not support the WSS Listener pattern used here.
**Fix:** This process must deploy to an Intermediate or Basic tier atom. Advanced atoms require the API Service wrapper pattern — a fundamentally different architecture.

### 3. SOQL Wildcard Bug (Agent API Tool vs. Direct POST)
**Problem:** When called via Boomi Agent Control Tower's API tool, `accountName` was passed as `%` (a wildcard) instead of the actual company name, causing the SOQL to return all accounts. When called via direct POST, it worked correctly.
**Root cause:** The Boomi Agent API tool sends parameters in a different payload structure than a direct POST. The Groovy parser was reading from the wrong JSON path.
**Fix:** Inspect the exact payload in Boomi Process Reporting shape execution logs, and update the Groovy extraction logic to match the Agent tool's body format.

### 4. Agentforce Permission Failures
**Problem:** The agent could invoke the action but received permission errors on execution.
**Root cause:** The Einstein Agent User was not a member of the `AgentforceServiceAgentUserPsg` permission set group. Admin debug context and live agent execution context are separate security surfaces — something that works when you test as an admin may silently fail when the agent runs it.
**Fix:** Add the Einstein Agent User to the `AgentforceServiceAgentUserPsg` group, and validate in live agent context, not admin context.

### 5. Agentforce Action Output Cannot Be Modified Post-Creation
**Problem:** Once an agent action's output definition is saved in Agentforce, it cannot be edited.
**Fix:** Delete and recreate the action from scratch if the output schema needs to change. Plan your output structure carefully before creating the action.

### 6. One Action Per Subagent
**Problem:** Assigning multiple actions to a single subagent caused Agentforce's routing to behave unpredictably — actions intended for one subagent contaminated another.
**Fix:** Each subagent owns exactly one action. Classification descriptions must be mutually exclusive, written to clearly fence each subagent's domain.

---

## Troubleshooting

**The endpoint returns no accounts**
Check Boomi Process Reporting for the execution. View the Groovy shape's document log to see the exact SOQL that was built. Confirm the `accountName` field was extracted correctly — not being passed as a `%` wildcard (see Gotcha #3).

**The listener isn't responding**
Confirm the process is deployed to the correct atom (not an Advanced tier). Go to Manage → Atom Management → Listeners and confirm the process appears as Active. If not, restart all listeners and redeploy.

**Permission errors in Agentforce**
Do not test only in admin context. Run a live conversation in the Agentforce preview panel using the actual agent persona. Check that the Einstein Agent User has the `AgentforceServiceAgentUserPsg` group membership (see Gotcha #4).

**Agent isn't routing to this subagent**
Review the classification description on the `Salesforce Account Integration` subagent. It must unambiguously describe the account lookup use case and not overlap in wording with the other two subagents. The Atlas reasoning engine routes based on intent matching against classification descriptions.

---

## File Structure

```
/
├── README.md                              <- You are here
├── docs/
│   ├── FINAL_CONFIG.md                   <- Credentials and endpoint details
│   └── LESSONS_LEARNED.md                <- Extended technical discovery log
├── boomi/
│   └── processes/
│       └── Aprimo_SF_AccountLookup.xml   <- Boomi process definition (exportable)
└── agentforce/
    └── tool-definitions/                 <- Agentforce agent tool configuration files
```

---

## About the Builder

This integration was designed and built end-to-end by **Rob Rojas**, Technical Engagement Manager and Solutions Architect at Aprimo. The build spans Salesforce Agentforce configuration, custom Apex development (`@InvocableMethod` wrappers, Named Credentials, External Credential Principal Access), Boomi iPaaS process design, Groovy scripting, SOQL query construction, and full production deployment.

The same architecture underpins a broader three-tier agentic system connecting Agentforce → Boomi → Aprimo DAM, enabling AI agents to surface brand-approved digital assets in real time during customer conversations.

**Key skills demonstrated:**
- Salesforce Agentforce — subagent architecture, Atlas routing engine, permission model, deployment lifecycle
- Boomi iPaaS — WSS Listener, Groovy scripting, Data Mapper, Salesforce connector, process reporting
- Enterprise integration design — OAuth, Named Credentials, error handling, deployment sequencing
- AI agent tooling — tool definition, input/output schema design, agent instruction writing
- Production debugging — permission model root-causing, SOQL diagnostics, cross-platform auth distinction

---

*Questions about this integration or the broader Agentforce + Boomi + Aprimo DAM architecture?*
*Reach out via LinkedIn or robert.trirrr@gmail.com*
