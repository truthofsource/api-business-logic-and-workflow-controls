# API Business Logic and Reliable Workforce Processes

**Make sure a business change is allowed, accurate and based on the latest information.**

[Business problem](#the-business-problem) · [Solution](#the-solution) · [Strengths](#engineering-strengths) · [Proof](#proof-it-works) · [Commands](docs/COMMANDS.md) · [Step by step](#step-by-step)

![Local implementation reference](docs/images/01-overview.png)

## The Business Problem

Payroll and employee administration depend on more than a form being filled in. A wage change needs a valid amount, an authorized reviewer and the current record. A payroll draft may contain meaningful earlier work even if its latest visible version appears empty. Staff need those distinctions to remain true when several people use the system.

Two reviewers can open the same record before either saves. Without a current-version check, the later action can overwrite or approve information the reviewer never saw. A delete operation that looks only at the newest draft can remove work whose history still matters. Hiding controls in the browser does not enforce those rules on a direct request.

The business need was dependable server-side rules for approval, change and deletion, so the stored result reflects an allowed action on the right version of the record.

## The Solution

Put the business rules on the server, where users cannot bypass them by changing the page. Check the requested change, the person’s permissions and the record’s version before saving. This project uses real wage and payroll-draft rules with fictional examples.

**Business outcome:** The example blocks unauthorized or out-of-date approvals and avoids treating meaningful payroll work as an empty draft.

## Tools Used and What They Do

| Tool or service | What it does | Where it fits |
| --- | --- | --- |
| Node.js and TypeScript | Express the pay, approval, version and discard rules. | Actual selected rule implementation and source tests. |
| Azure Functions | Receive the protected business request in the platform. | Reference route context; the local example is not a deployed handler. |
| Microsoft Entra ID | Supply roles and verified staff identity for allowed actions. | Actual access boundary; fixture roles do not verify real tokens. |
| Azure SQL and T-SQL | Apply guarded changes and preserve related records together. | Actual store context; pure-rule tests do not exercise live transactions. |
| Node test runner and Python | Check extracted decisions and show fictional outcomes. | Local validation and preview utilities. |
| Terraform and GitHub Actions | Provide related infrastructure and manual reference checks. | No business operation or infrastructure change occurs through the demo. |

The rule, API route, database permission and user interface are separate parts of one process. Passing a local rule test does not establish a completed approval, payment or saved record.

## Step by Step

A short walkthrough of the process, with technical instructions and evidence limits in the sections below.

1. **Receive the requested change** — The server receives an action such as a wage update or payroll decision. It checks who is making the request and which record they want to change.

2. **Check the business rules (TypeScript)** — The application checks the values, permissions and current record status. An outdated or unauthorized request is rejected before anything is saved.

3. **Save a consistent result (Azure SQL)** — An allowed change is saved together with its audit record. Checks inside the database help prevent two competing requests from applying conflicting decisions.

4. **Explain the outcome** — The application returns a useful success, validation or conflict message. The local examples demonstrate these decisions with fictional records.

## Engineering Strengths

| Strength | Practical value |
| --- | --- |
| **Rules stay testable** | Pure rules can be checked without an HTTP server, Azure login or live database. |
| **Conflicts are explicit** | A out-of-date record version or decision that is no longer awaiting approval returns a conflict instead of silently overwriting newer work. |
| **Data meaning is preserved** | Draft classification examines every saved version and treats unknown shapes conservatively. |

## Architecture

```mermaid
flowchart LR
  UI[User action / JSON] --> Route[HTTP route]
  Route --> Auth[Verified identity / route permissions]
  Auth --> Rules[Input / role / state / revision rules]
  Rules --> Store[Parameterized SQL transaction]
  Store --> Audit[Business change + audit]
  Audit --> Response[Useful success / validation / conflict response]
```

This project is derived from actual workforce-platform implementation. Identifying labels/targets use examples. The local viewer and demonstration are new reference code; [source mapping](docs/SOURCE-MAP.md) identifies the copied/extracted files and limitations.

## What Was Built

| Component | Purpose | Implementation |
| --- | --- | --- |
| Actual extracted wage rules | Validate money/date/action fields, role combination and version still awaiting a decision | [Rules](src/wageRules.ts) |
| Actual draft classifier | Examine every payroll snapshot; refuse unknown/incorrectly structured shape | [Source](src/payrollPacketDiscard.ts) |
| Actual route reference | Authorization before store, JSON/body limits and useful failures | [Route](reference/wagechanges-route.ts) |
| Actual SQL store reference | Serializable transaction, locking, parameterized queries and audit | [Store](reference/wageChanges.ts) |
| Local demonstration | Exercise actual rule/evaluation code using fictional data | [Demo](scripts/demo.py) |
| Dated context | Keep earlier platform results separate from the new local example | [Context](data/history.json) |

## Proof It Works

### 1. Run focused checks

```bash
node --experimental-strip-types --test src/payrollPacketDiscard.test.ts src/wageRules.test.ts
node --experimental-strip-types src/example.mjs
python -B scripts/demo.py
```

**Observed:** Ten Node tests passed: two original discard tests and eight new focused tests over the actual extracted wage rules. [Validation](docs/VALIDATION.md) records the exact scope and distinguishes pure/mock checks from testing with the hosted services.

### 2. Inspect the decision output

![Working reference example](docs/images/02-example.png)

The output is explicitly fictional and exercises the included logic. A successful rule test or simulation is not a completed real-world operation. The public example runs standalone rules and reports errors. It does not verify login credentials, connect to SQL, test competing requests across servers or complete payroll.

### 3. Open the reference viewer

```bash
python -B scripts/preview.py --port 8139
```

Open [the local reference](http://127.0.0.1:8139). Switch between Overview, Working example and Implementation context. Ctrl+C stops it. Screenshots are genuine local captures, not hosted application/Azure Portal screenshots.

## What Business Logic Means Here

Business logic is the set of rules that gives a request meaning: who can approve a wage change, what a valid rate/date looks like, which revision may still be changed and whether a saved payroll draft contains work worth preserving. An API route exposes that action over HTTP; SQL persists the accepted result; the frontend explains and requests the action.

The actual wage route authenticates/authorizes before opening the store, separates self-service current pay from staff operations, restricts content type/body size/fields and maps known errors to useful responses. Its Functions authLevel=anonymous means the app's own verified role middleware is responsible for authorization; it does not mean the business action is public.

The extracted rules require a positive decimal rate with at most two decimal places and a real date. Approval requires Admin or HR together with Payroll. The row must still be pending at the record version supplied in the request. Invalid input is 400, wrong permission is 403 and stale state is 409. An unexpected store failure asks the user to reload history before retrying because a save result may be uncertain.

The full database code checks the saved history again before changing it. It uses SQL queries that keep user values separate from query instructions, and database locks that prevent competing changes. It then saves an audit record. Record IDs and version checks help prevent duplicates and outdated changes. The standalone rule tests do not test these database and retry steps.

The draft classifier checks all versions. Earlier lines, payment changes, secure details or meaningful notes make the packet a draft even when the latest version has no hours. Unknown/incorrectly structured snapshots return unknown rather than being classified safe to discard.


## Operations, Limits and Growth

State clearly which system is being checked, who owns it and what the result actually proves. Test the cases that should be denied or rejected as well as the successful ones. Keep incomplete results clearly marked. Preserve private inputs and unrelated changes. Before expanding the design, measure what is slow or limited, review costs, and check access restrictions and recovery again. [Operations and scaling](docs/OPERATIONS-AND-SCALING.md).

## Troubleshooting Lessons

A draft-discard operation failed because the runtime identity lacked one required column read. Tests with database-owner authority had missed the gap. [Read the incident, evidence and verification](docs/LESSONS-LEARNED.md).

## Project Guides

| Guide | Purpose |
| --- | --- |
| [Implementation](docs/IMPLEMENTATION.md) | Rules, boundaries and actual source design |
| [Commands](docs/COMMANDS.md) | Local tests, expected output and inspection |
| [Operations and scaling](docs/OPERATIONS-AND-SCALING.md) | Strengths, weaknesses, costs and growth |
| [Source map](docs/SOURCE-MAP.md) | Actual files and public adaptations |
| [Validation](docs/VALIDATION.md) | What was tested locally and what still needs real-service testing |

No Azure service, business record or live state was changed. The public validation workflow is manual-only and was not dispatched, conserving hosted Actions minutes.
