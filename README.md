### Unity Gateway for coding agents standard configuration and rollout guide



This guide sets up Claude Code and Codex to run through Unity Gateway, with governed models, team skills, MCP servers, tracing, rate limits and per-developer budgets.

You do it in three steps:
1. Complete the prerequisites below (once per workspace).
2. Fill in three placeholders in [`coding-agent-config.json`](coding-agent-config.json) and publish it (command :  _databricks api post /api/ai-gateway/v2/coding-agent-configs --json @coding-agent-config.json_)
3. Developers install the Unity Gateway CLI (`ug`) and run `ug claude` or `ug codex`.

Every workspace uses the same file. Only the catalog, schema and budget ID change.

## How it fits together

| What | Where it's set | Who sets it |
|---|---|---|
| Agents, models, defaults, MCP servers, skills, tracing | `coding-agent-config.json` | Workspace admin |
| Model services, rate limits, tags, payload logging, guardrails | Unity Catalog (your catalog and schema) | Workspace admin |
| Who can use which models | Unity Catalog grants to account-level groups | Workspace admin |
| Spend limits | Unity Gateway budgets | Account admin |

The configuration file is a menu, not access control. Listing a model in the file doesn't give anyone access; grants do. Removing it from the file doesn't revoke access either.

---

## 1. Prerequisites

### 1.1 Turn on previews
| Preview | Where | Why |
|---|---|---|
| Unity Gateway Managed Configuration | Workspace > Previews | Publish the configuration |
| Enhanced Unity Gateway | Account console > Previews | Budget blocking, spend-based model recommendations, `ug usage` |

### 1.2 Create groups
Use **account-level** groups (synced from your identity provider). Workspace-local groups don't work for Unity Catalog grants or rate limits.

| Group                | Can use                                        |
| -------------------- | ---------------------------------------------- |
| `grp-coding-all`     | Default kit for all developers / users         |
| `grp-coding-premium` | Expensive models                               |
| `grp-non-dev`        | Open-source models only for non critical users |

### 1.3 Create model services
Choose a catalog and schema for coding agents, for example `main.coding_agents`. The catalog needs its own storage location, because skills are stored there.

For this example, we use reasoning / fast / oss prefix for model services which makes it easier for end users to identify functional use cases for each users.

In **AI Gateway**, create one model service per model in that schema:

| Model service | Points to | Used by | Access | Rate limit (per user) |
|---|---|---|---|---|
| `reasoning_claude_opus` | Claude Opus | Claude Code | `grp-coding-premium` | 60 requests/min |
| `reasoning_claude_sonnet` | Claude Sonnet | Claude Code | `grp-coding-all` | 120 requests/min |
| `fast_claude_haiku` | Claude Haiku | Claude Code | `grp-coding-all` | 120 requests/min |
| `reasoning_gpt` | GPT | Codex | `grp-coding-all` | 120 requests/min |
| `fast_gpt_mini` | GPT mini | Codex | `grp-coding-all` | 120 requests/min |
| `oss_glm` | GLM | Both | `grp-coding-all`, `grp-non-dev` | 120 requests/min |
| `oss_kimi` | Kimi | Both | `grp-coding-all`, `grp-non-dev` | 120 requests/min |
| `oss_deepseek` | DeepSeek | Both | `grp-coding-all`, `grp-non-dev` | 120 requests/min |

Point each service at the latest version of its model in `system.ai`. Name services by model family, not version. When a new version ships, you repoint the service, and nobody's setup changes.

On each service, set:
- **Rate limit:** per user, per minute, as in the table.
- **Inference table:** turn it on to log requests and responses to your schema.
- **Guardrails:** PII (input and output), unsafe content (input), jailbreak (input). Start in **log-only** mode. Blocking can reject later turns of a coding session, so switch only once you've reviewed what gets flagged.
- **Tags:**

| Tag | Value |
|---|---|
| `workload` | `ai-coding` |
| `cost_center` | Your cost center |
| `owner` | Owning team |
| `model_category` | `reasoning`, `fast` or `oss` |

Tags show up in the usage and billing system tables, and budgets can filter on them. We recommend a tag policy for `cost_center` and `owner` so only approved values are accepted. Tag policies apply account-wide, so check with other teams that use the same keys.

### 1.4 Grant access
| On | Grant | To |
|---|---|---|
| Your catalog and schema | `USE CATALOG`, `USE SCHEMA` | All three groups |
| Each model service | `EXECUTE` | The groups in the 1.3 table |
| MCP servers | `EXECUTE`, plus access to the data and tools behind them | `grp-coding-all` |
| Skills | `READ VOLUME` | `grp-coding-all` |

**Direct model access:** by default, all users can call the built-in `system.ai` models directly. That skips your model services, tags and rate limits. The backstop budget (1.5) caps that spend. Removing the default access affects other features (Playground, AI functions, Genie), so test that carefully before changing it.

### 1.5 Set up budgets (account admin)
Create two monthly budgets per workspace:

| Budget                                              | Scope                                                | Per-user limit            | Purpose                                                                        |
| --------------------------------------------------- | ---------------------------------------------------- | ------------------------- | ------------------------------------------------------------------------------ |
| Main, e.g. `coding-agents-<cost center>`            | The workspace + tag `cost_center` = your cost center | e.g. $200/month           | Limits spend on your model services. **Its ID goes in the configuration file** |
| Backstop, e.g. `coding-agents-backstop-<workspace>` | The workspace only                                   | About 1.5x the main limit | Catches direct `system.ai` use, which has no tags                              |

For each budget:
1. In the account console, create a budget. Set the period to **Monthly** and the resource type to **Unity Gateway**.
2. Set the scope: select the workspace. For the main budget, also add the tag filter `cost_center`, matching the tag on your model services.
3. Add a **Per-user** threshold with the limit from the table.
4. Choose the actions **Send alert** (add admin recipients) and **Block usage**. During the pilot, use **Send alert** only.
5. Optional: add a **Per-user override** with a higher limit for `grp-coding-premium`.
6. Save. For the main budget, copy the budget ID. You'll need it in step 2.

Blocking happens in near real time. Requests already in progress finish, so spend can go slightly over the limit.

### 1.6 Skills and tracing
- **Skills:** add your team skills to the same schema. Every skill there is shared with developers automatically.
- **Traces:** in AI Gateway > Govern > Traces, choose where traces are stored, for example `main.coding_agents_traces`.
- **Usage data:** have an account admin enable the `ai_gateway` system schema. Usage data then appears in `system.ai_gateway.usage`.

---

## 2. Configure and publish

Replace three placeholders in `coding-agent-config.json`:

| Placeholder | Replace with |
|---|---|
| `${CATALOG}` | Your catalog, e.g. `main` |
| `${SCHEMA}` | Your schema, e.g. `coding_agents` |
| `${BUDGET_ID}` | The main budget's ID from 1.5 |

Then publish it, as a workspace admin:
```bash
databricks api post /api/ai-gateway/v2/coding-agent-configs --json @coding-agent-config.json
```
Or paste it into **Unity Gateway > Govern > Agent configuration > Edit configuration** and click **Save**.

### What the file sets
| Setting | Value |
|---|---|
| Default agent | Claude Code |
| Claude Code models | Opus, Sonnet, Haiku, GLM, Kimi, DeepSeek. Default: Sonnet |
| Codex models | GPT, GPT mini, GLM, Kimi, DeepSeek. Default: GPT |
| MCP servers | Databricks SQL, GitHub |
| Skills | Everything in your schema |
| Tracing | On for both agents |
| MCP lockdown | Developers can only use the MCP servers in the file (see section 6 if developers don't have admin rights on their machines) |
| Spend-based recommendations | At 80% of a developer's budget, new sessions start on Haiku; at 95%, on GLM. The block at 100% is the hard stop |

Keep the file in version control. One configuration is published per workspace. To change it later, edit the file and update the published version with `PATCH /api/ai-gateway/v2/coding-agent-configs/{id}`.

---

## 3. Developer setup
Send developers this:

One time installation for Unity Gateway CLI
```bash
#One time installation for unity gateway cli
uv tool install git+https://github.com/databricks/unity-gateway
ug --version
```


```bash
# One-time configure to sign in to databricks workspace and get agent config
ug configure        # syncs models, MCP servers and skills

# daily usage pattern
ug claude           # start Claude Code (or: ug codex)
ug usage            # your spend and limit this month
ug doctor           # if something isn't working
```
- Run `ug configure` without `--agent`, so MCP servers and skills sync too.
- GitHub asks each developer to sign in the first time they use it.
- Users in `grp-non-dev` don't have access to the default model (Sonnet), so they start with `ug claude --model <catalog>.<schema>.oss_glm`.

| Problem                           | Fix                                                                                                   |
| --------------------------------- | ----------------------------------------------------------------------------------------------------- |
| 401 errors                        | Remove `ANTHROPIC_API_KEY` or `ANTHROPIC_AUTH_TOKEN` from your shell profile and open a new terminal  |
| "OS-managed settings … conflict"  | A machine-wide Claude Code configuration already exists. Run `ug revert`, or ask IT if they manage it |
| "Permission to use model service" | That model isn't included in your access. Ask your admin                                              |
| Rate limit                        | Wait a minute, or switch to a lighter model with `/model`                                             |
| Budget reached                    | `ug usage` shows your limit. Ask your admin for an increase                                           |

---

## 4. Rollout plan

| Phase | Who | Settings | Move on when |
|---|---|---|---|
| **1. Prepare** | Admins | Prerequisites done in a test workspace | Configuration published and tested |
| **2. Pilot** | One team, 5–15 developers | Budgets alert only; guardrails log-only | 2 weeks with no blocking issues; most of the team's usage goes through the gateway; per-developer cost baseline known |
| **3. Expand** | A department | Budgets block; limits set from the pilot baseline | Little direct `system.ai` use; budget blocks only hit expected users |
| **4. Standardize** | All workspaces | Same file everywhere, changes reviewed in version control | Every workspace runs the standard file |

**Decide before the pilot:**
- Who gets Opus, and who should be limited to open-source models.
- Cost centers and owners for tags, and the monthly limit per developer.
- Whether developers have admin rights on their machines, or whether you use MDM (Jamf, Intune).
- Whether your Claude Code managed settings use `forceLoginMethod` or `forceLoginOrgUUID`. These block gateway sign-in.

Before the pilot, also check that each model works in each agent you list it for, especially the open-source models in Claude Code.

---

## 5. Operate

**Monitor**
- **Spend by cost center, user and model:** `system.ai_gateway.usage`, grouped by `endpoint_tags['cost_center']`, `requester` and `destination_model`. It includes cached-token detail.
- **What you were billed:** `system.billing.usage`, by `custom_tags` and `identity_metadata.run_as`.
- **Direct use:** coding-agent traffic to `system.ai.*` instead of your schema.

**Make changes**

| Change | How |
|---|---|
| New model version | Repoint the model service. The configuration stays the same |
| New model | Create a model service with tags and grants, add it to the agent's list in the file, publish |
| Who has access | Change group membership in your identity provider |
| Configuration | Edit the file, review it, publish. Developers get the change on their next launch or `ug configure` |

---

## 6. Options
Each option changes only the settings listed.

| Option | When | Change |
|---|---|---|
| **Smart Routing** | You want the lowest cost per task, and per-model tags, rate limits and guardrails matter less | List the built-in `system.ai` Claude models instead of your services, and set `smart_routing.enabled` to `true`. Smart Routing only works with `system.ai` models. Budgets then filter by workspace instead of tag |
| **Your own provider** | You already have an Anthropic, Amazon Bedrock or OpenAI contract | Add the provider in Unity Gateway > Providers, then use it as the agent's model source. Turn on the External Model Spend in Budgets preview |
| **No admin rights on laptops** | Developers can't use sudo and there's no MDM | Remove `agent_native_settings` from the file. It writes a machine-wide settings file |

---

## 7. Known limitations
- One configuration per workspace. Give teams different access through groups and budget overrides.
- Spend-based recommendations are suggestions. Developers can still pick another model; the budget block is the hard limit.
- Guardrails and the trace table are set up in the UI only.
- Slack, web sessions and Remote Control don't go through the gateway.
- Each account can have up to 1,000 active budgets.

## Reference
- [Configure coding agents centrally](https://docs.databricks.com/aws/en/ai-gateway/coding-agent-configure-govern)
- [Coding agent configuration schema](https://docs.databricks.com/aws/en/ai-gateway/coding-agent-config-schema)
- [Control coding agent spending](https://docs.databricks.com/aws/en/ai-gateway/coding-agent-costs)
- [Set up model capacity and providers](https://docs.databricks.com/aws/en/ai-gateway/coding-agent-bring-your-own-providers)
