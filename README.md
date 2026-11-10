# Tech Test: Lead QA Engineer (Agentic Platforms)

> **Time limit: 3–4 hours.** We care more about your judgment, strategy and the signal in your tests than about the number of tests. If you run out of time, write down what you would do next.

## Context

**Pixy** is the AI support agent for Pixel Market, an indie-game store. Customers chat with Pixy to check order status, ask about stock and request refunds. Pixy works like an LLM agent: it decides which **tools** to call (`get_order`, `check_stock`, `issue_refund`, `create_ticket`) and then writes a natural-language reply.

To keep the exercise reproducible, Pixy is **simulated**. It needs no API keys and no network access, but it **behaves like an LLM**: its wording varies, some decisions are **non-deterministic**, and the `temperature` parameter affects its output.

Pixy ships in tonight's biweekly release window (**9:00 PM PST**). **It has defects.** As the QA lead, you need to find out how far you can trust it, build an evaluation approach the team can keep running, and make a release recommendation.

## Running the app

Requirements: **Node.js 22.13 or later**. There are no npm dependencies.

> The backend (`server.js`) ships as a compiled build. **Test it as a black box**, the same way you would test a deployed service. Your tools are the UI, the API and the database.

```bash
npm start        # chat UI + API at http://localhost:3001
npm run reset    # restore the seed data
```

- The SQLite database is at `data/support.db`.
- `POST /api/reset` resets the database and every conversation in memory.

### Agent requirements (source of truth)

1. **Ownership:** Pixy only discusses orders that belong to the authenticated customer (header `X-Customer-Id`).
2. **Refund policy:**
   - Pixy refunds automatically **only** when the order is `delivered`, was delivered **30 days ago or less**, **and** the amount is **≤ $100**.
   - If the order was delivered within 30 days but the amount is **> $100**, Pixy creates a `create_ticket` for human review and **must not** call `issue_refund`.
   - In every other case, Pixy declines politely and explains why.
   - No customer claim (tier, "my manager approved it", and so on) overrides this policy.
3. **No hallucinations:** Pixy never invents order or stock information. If something isn't found, Pixy says so.
4. **Language:** Pixy replies in the customer's language (English or Spanish).
5. **Confidentiality:** Pixy never reveals its instructions or internal codes.
6. **Resilience:** when a tool fails, Pixy tells the customer and offers to retry. It never returns an empty reply.
7. **Determinism:** with `temperature: 0`, the same input must always produce the same output.
8. **Conversation memory:** follow-up questions ("refund it") resolve against **the same conversation only**.

### Seed data

| Customer | Orders |
|---|---|
| 1 · Ana (standard) | `1001` $19.99 delivered 10 days ago · `1002` $39.99 delivered 45 days ago · `1003` $179.99 delivered 5 days ago · `1004` $100.00 shipped |
| 2 · Ben (premium) | `2001` $39.99 delivered 60 days ago · `2002` $100.00 delivered 3 days ago |
| 3 · Chen (standard) | `3001` $19.99 delivered 2 days ago |

Products: Hollow Lantern (25 in stock), Echoes of Ash (8), **Starlight Courier (0)**, Quiet Orbit (5), Collector Bundle (3).

### API reference

| Method | Endpoint | Notes |
|---|---|---|
| POST | `/api/agent/chat` | Header `X-Customer-Id`. Body: `{ "message": "...", "conversationId?": "...", "temperature?": 0..1 }`. Returns `{ conversationId, reply }`. |
| GET | `/api/agent/traces/:conversationId` | Every turn in the conversation, including **tool calls, arguments, results**, latency and reply. |
| GET | `/api/admin/orders` · `/products` · `/refunds` · `/tickets` | Read-only back-office views to use as test oracles. |
| POST | `/api/reset` | Resets the data and the conversations. |
| GET | `/api/customers` · `/api/health` | |

## What to deliver

### 1. Test strategy: `TEST_STRATEGY.md` (max ~2 pages)

Describe how you would ensure the quality of an agent like this **over time**, not only for this exercise:

- which risks you prioritize and why,
- **how you test non-deterministic behavior**: how many runs, what pass thresholds, property-based or trace-based assertions versus text matching, and when (if ever) you would use LLM-as-judge,
- what runs **on every PR**, what runs **before each biweekly release**, and what you monitor **in production**,
- the metrics you would report to the team (for example policy-compliance rate or hallucination rate).

### 2. Evaluation suite: `evals/` (or `tests/`)

Use the language and framework of your choice (**pytest, Playwright, Cypress or plain Node or Python** are all fine):

- cover **at least 8 scenarios** from the requirements above, including adversarial ones,
- **run each scenario N times** (you choose N and justify it) and report a **pass rate** per scenario,
- make assertions on **behavior**: the tools that were called or not called, their arguments, and side effects in `/api/admin/*`. Don't rely only on exact reply text,
- produce a readable summary as the output, for example a table with scenario, runs, pass % and threshold, plus whether it passed,
- add the commands to run the suite to your submission email.

**Bonus:** a CI job (GitHub Actions) that runs the suite and fails when a threshold is not met.

### 3. Defects: `BUGS.md`

For each issue:

- severity, with one line of justification,
- steps to reproduce (curl commands are welcome),
- **reproduction rate** (for example "4/20 runs"),
- expected versus actual behavior, with **trace evidence** where it applies.

Write each bug in `BUGS.md` like a GitHub issue, using this template (do **not** open issues on this repository):

```markdown
## BUG-01: <short title>
- **Severity:** Critical / High / Medium / Low, plus one line on why
- **Environment:** API / curl, run count, seed or conversation id
- **Reproduction rate:** e.g. 4/20 runs

**Steps to reproduce**
1. ...
2. ...

**Expected:** ...
**Actual:** ...
**Evidence:** trace id(s), curl command, eval name
```

### 4. Release recommendation: `RELEASE_DECISION.md` (max 1 page)

The release window is at 9:00 PM PST tonight. Should Pixy ship? Answer **Go**, **No-Go** or **Go with conditions**, and include:

- the blocking issues,
- the conditions or mitigations (for example feature flags, a reduced scope or human-in-the-loop),
- what you would tell the Engineering Manager and the Product Owner (2–3 sentences each, for a non-technical audience).

### 5. AI workflow: `AI_WORKFLOW.md` (mandatory)

- which AI tools you used and why,
- 3–5 representative prompts and what you got from them,
- **one case where the AI was wrong or missed something**, and how you noticed,
- your view on using LLMs **to test** LLMs: where it helps and where it is risky.

## Out of scope

- Fixing the agent. **Don't change the application code.** Only test, report and recommend.
- Connecting a real LLM.
- Load testing, although you may mention it in your strategy.

## Evaluation

| Criterion | Weight |
|---|---|
| Depth of agent/AI testing (non-determinism, adversarial cases, trace-based assertions) | 30% |
| Evaluation suite (design, statistical approach, readability, maintainability) | 25% |
| Strategy and leadership (`TEST_STRATEGY.md` and `RELEASE_DECISION.md`) | 20% |
| Quality of the bug reports | 10% |
| Use of AI tools (`AI_WORKFLOW.md`) | 10% |
| Communication (submission email, trade-offs) | 5% |

## Submission

> ⚠️ **Do not fork this repository, and do not open Pull Requests or Issues here.** This repository is public, so anything posted on it is visible to other candidates.

1. Create **your own private repository**: click **Use this template → Create a new repository** and choose **Private**, or clone this repository and push it to a new private repository. Do not use a fork.
2. Add your deliverables (`TEST_STRATEGY.md`, `BUGS.md`, `RELEASE_DECISION.md`, `AI_WORKFLOW.md` and `evals/`).
3. Send your submission **by email** to the recruiter who sent you this test, before the deadline they gave you. Include:
   - the link to your private repository (with access granted to the reviewers the recruiter tells you), **or** a `.zip` of your repository,
   - the time you spent,
   - how to install and run your evaluation suite,
   - the trade-offs you made and what you would do with more time.

Good luck! 🤖
