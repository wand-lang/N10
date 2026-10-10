# N10 — Implementation plan (draft)

Status: draft, 2026-10-10. Written in ASD-STE100 Simplified Technical English.

## Sources

- "N10 in Millwright" (doc, 2026-10-09):
  https://claude.ai/artifact/7JkRvpaKNyEW3MLKo5uyUf
- "Proposal: Typed AI Triage with Wand, Jev, and Claude" (doc, 2026-10-09):
  https://claude.ai/artifact/3H3y1q7qoMAkmcyUnRqEue
- Wand 0.104.0, `../wand/docs/reference.md` (written "R" below).

## Scope

This repo is the Wand library. It is not Millwright. The library gives a
workflow author these parts:

1. `Prob`: "probably one of these": a typed answer from System 1, with a probability for each option.
2. A System 1 client for the `/v1/systemone` API (Jev, or Lev self-hosted).
3. A System N client for Claude.
4. Steps and workflows: send an event to the cheapest level that is sure.
5. Record and replay of model calls, for tests and for audit.
6. A spend cap on System N calls.
7. Bounds on actions, and habits: turn repeated System N cases into checked Wand actions.
8. Level-share counts for each workflow.

Millwright parts (event intake, durable runs, approval queue as a service,
gateway as a service) are out of scope. The library must not block them.

## Use cases

N10 fits a process with high volume, repeated cases, a small set of
decisions, and bounded actions. Most triage and back-office work has that
shape. The use cases below come from the two source docs.

### Triage (from the proposal)

| Use case | System 1 decides | System 0 does | System N handles | Bound (example hosts) | Data, System 1 backend |
| --- | --- | --- | --- | --- | --- |
| CI failures (pilot) | kind, known issue, blocking score | rerun, comment, ticket, notify | explain new failures | `ci.internal`, `tracker.internal`, `chat.internal` | build logs, scrubbed; Jev |
| On-call ops | alert kind, known incident, customer impact now | runbook steps: restart, scale, roll back (approval at first) | investigate new alerts, draft incident notes | `deploy.internal`, `metrics.internal` | metrics and logs; Jev |
| Support routing | category, team, urgency, known fix | route, tag, known fixes (resets, refunds under a limit) | write replies, unusual tickets | `helpdesk.internal`, `billing.internal` | customer data; Lev, self-hosted |
| Security triage | alert kind, likely false positive, severity | suppress known noise, enrich, open case | investigate, write case summary | `siem.internal`, `cases.internal` | sensitive; Lev, self-hosted |

### Back office (from the Millwright doc)

| Use case | System 1 decides | System 0 does | System N handles | Bound (example hosts) |
| --- | --- | --- | --- | --- |
| Invoice intake | duplicate, known vendor, which approver | match to PO, post to ERP, route for approval | new vendors, odd formats, disputes | `erp.internal` |
| Expense review | in policy, needs receipt, risk level | approve under limit, request receipt | exceptions, explanations to the employee | `expenses.internal`, `mail.internal` |
| Access requests | standard for role, needs manager, risk | grant standard access, open ticket | unusual requests, the justification | `iam.internal`, `tracker.internal` |
| Vendor email | category, urgency, needs reply | file, tag, forward, template replies | non-template replies | `mail.internal` |
| Employee onboarding | role type, location, equipment set | create accounts, order equipment, schedule | special cases | `iam.internal`, `procurement.internal`, `calendar.internal` |

The docs do not set a System 1 backend for the back-office use cases.

### Where N10 does not fit at first: extraction

Much back-office work starts by pulling fields out of a document. System 1
cannot return text, so extraction starts at System N. It moves down as
habits form: after System N reads a few invoices from one vendor, it can
write a parser for that vendor's format, and that vendor's invoices move to
System 0. System 1 can then check each parse ("Do these fields match the
document?") before the action runs.

### Order

CI failures come first: no customer data, low-risk actions, high volume,
and every decision can be checked (a rerun passes or it does not). Then, in
order of rising risk: on-call ops, support routing, security triage. Each
one reuses the steps, the trace and the review process from CI.

On-call actions touch production, so they start behind approval and become
automatic one at a time. Support and security hold customer and security
data, so they use Lev on our own hardware.

## Terms

- **Level.** Where a decision or an action runs. System 0 is code. System 1
  is a model that picks one option from a closed set, with a probability for
  each option. System N is a frontier model. A level belongs to a call, not
  to a part of a step.
- **Workflow.** The steps for one kind of event. It names only its start
  step. Its steps are the steps that N10 finds from the start.
- **Step.** One decision. It has a name, an `ask`, an `on` map, an optional
  `on_error`, a `fallback` and a `bound`.
- **`ask`.** Reads the event and returns a `Prob`. Usually System 1. It can
  also be System N or code. It must not change anything outside the run.
- **`Prob`.** "Probably one of these": a probability for each option. If the
  top option is at or above the threshold, N10 uses it. If not, the step
  goes to `fallback`.
- **`on`.** Maps each option to an action, or to another step (`N10.go`).
- **`fallback`.** Runs when `ask` is unsure or `on` has no entry for the
  option. Usually System N.
- **Action.** Code that does the work at a step and returns where to go
  next. A **written action** is written by a person. A **learned action** is
  written by System N and added to `on` after review.
- **Bound.** The limit on every action at a step: its input, its result,
  and what it may reach (hosts, files, commands). A person writes it and a
  reviewer approves it.
  - For learned actions, the bound is the main control. N10 refuses an
    action that does not fit, before a person reviews it.
  - For written actions, the bound is a second check. It catches what a
    code review misses, such as a call to a host that nobody intended.
  - A bound limits where an action reaches, not how much it does there. A
    rule like "refunds under $50" goes in the action, or in the host.
- **Habit.** A System N case that repeats, turned into a learned action.
  When a habit is active, that case runs at System 0 with no model call.
  Habits are how a workflow gets cheaper. They are safe only because of
  bounds.

## Interface sketch (work in progress)

The unit is a **step**. A workflow is one or more steps. There are two
versions below: a tree (each event goes through once) and a workflow that
can wait for a signal and loop back. This is pseudo code: `N10.*`,
`System1.*`, `SystemN.*` and the actions do not exist yet.

### Tree version

A workflow with no waits and no loops. Each step refers to the steps it
can go to, and the workflow names only the start. N10 derives the set of
steps from the start.

```
job ─> scrub ─> [triage] ──Flaky/Infra──> rerun_once ──> stop
                   │ ├──Dependency──> [dependency] ──Floating──> open_pin_pr ──> stop
                   │ │                     └──unsure/Pinned──> SystemN.explain ──> stop
                   │ └──unsure/CodeBug/Config/Other──> SystemN.explain ──> stop
                   │
              action fails
                   └──> [recover] ──Timeout──> rerun_once ──> stop
                           ├──RateLimited──> wait_then_rerun ──> stop
                           └──unsure/Other──> SystemN.investigate ──> stop
```

```ocaml
uses {Clock, Net.Listen(:8080), Net(ci.internal, tracker.internal, api.typesafe.ai, api.anthropic.com)}

let N10 = import github.com/wand-lang/N10

type Job(id: String, repo: String, commit: String, test: String, log: String)

-- What System 1 may answer
type FailureKind = Flaky | Infra | Dependency | Config | CodeBug | Other
type Pin = Pinned | Floating
type Fault = Timeout | RateLimited | Other

-- Bound: what an action at any step may do.
-- An action returns where to go next:
--   N10.stop result   end here
--   N10.go step       go to another step now
type Action = N10.Run Job -> Result N10.Next Failure ! {Net(ci.internal, tracker.internal)}

-- System 0: remove secrets from the log before any model sees it
let secrets =
  [ r/ghp_[A-Za-z0-9]{36}/                      -- GitHub tokens
  , r/AKIA[0-9A-Z]{16}/                         -- AWS access keys
  , r/Bearer [A-Za-z0-9._\-]+/                  -- bearer tokens
  , r/(password|secret|token)=[^\s]+/ ]         -- key=value pairs

let scrub (job: Job) =
  let clean = List.fold_left (fn log r -> Regex.replace_all r "[REDACTED]" log) job.log secrets;
  Job(job, log = clean)

-- Actions: rerun_once, open_pin_pr, wait_then_rerun (not shown).
-- Each one fits Action.

-- Steps
let recover = N10.step "recover"
  ( bound = Action
  , ask = fn run -> System1.choose Fault.all run      -- run.failure holds the error
  , on = [ (Timeout, rerun_once)
         , (RateLimited, wait_then_rerun) ]
  , fallback = SystemN.investigate )

let dependency = N10.step "dependency"
  ( bound = Action
  , ask = fn run -> System1.choose Pin.all run
  , on = [ (Floating, open_pin_pr) ]
  , on_error = recover
  , fallback = SystemN.explain )

let triage = N10.step "triage"
  ( bound = Action
  , ask = fn run -> System1.choose FailureKind.all run
  , on = [ (Flaky, rerun_once)
         , (Infra, rerun_once)
         , (Dependency, N10.go dependency) ]
  , on_error = recover
  , fallback = SystemN.explain )

-- The workflow: only the start
let on_failure = N10.workflow
  ( event = Job.decoder
  , before = scrub
  , start = triage )

-- Kick-off: CI posts each failed job here
let route (req: HTTP.Incoming) =
  match (req.method, HTTP.segments req) with
  | (HTTP.POST, ["ci", "failed"]) -> (
    match JSON.decode Job.decoder req.body with
    | Error e -> HTTP.reply 400 e
    | Ok job -> (
      N10.run! on_failure job;
      HTTP.reply 202 ""
    )
  )
  | _ -> HTTP.reply 404 ""

HTTP.serve! HTTP.Server(port = :8080, deadline = 5m) route
```

- `N10.Run` holds the event and the last failure (`run.event`, `run.failure`).
- `on` maps each option to an action, or to `N10.go step`. The bound
  applies to actions, not to `N10.go`. Learned actions are added to `on`,
  but never as `N10.go`.
- `on_error` names the step that a failed action goes to. With no
  `on_error`, a failure ends the run.
- With no `wait`, `N10.run!` runs the event to the end and stores nothing.

### Workflow version

A workflow that can wait for a signal from outside and loop back. It uses
the same parts as the tree version, plus `N10.wait` and signal steps.

```
            ┌────────────── Floating: open_pin_pr ──> stop
            │
       [dependency] ── unsure/Pinned ──> SystemN.explain ──> stop
            ^
            │ Dependency
job ──> [triage] ── Flaky/Infra: rerun ──> [rerunning] ── passed ──> stop
            │                                  ^   │
            │ unsure/CodeBug/Config/Other      │   │ failed
            v                                  │   v
      SystemN.explain ──> stop                 └─ [recover] ── unsure/Other ──> SystemN.investigate ──> stop
                                     Timeout: rerun   │
                                     (max 2 reruns)   └─ RateLimited: wait, rerun ──┘
```

```ocaml
uses {Clock, Net.Listen(:8080), Net(ci.internal, tracker.internal, api.typesafe.ai, api.anthropic.com)}

let N10 = import github.com/wand-lang/N10

type Job(id: String, repo: String, commit: String, test: String, log: String)

-- What System 1 may answer
type FailureKind = Flaky | Infra | Dependency | Config | CodeBug | Other
type Pin = Pinned | Floating
type Fault = Timeout | RateLimited | Other

-- The signals that come from outside
type Signal = RerunPassed | RerunFailed

-- Bound: what an action at any step may do.
-- An action returns where to go next:
--   N10.stop result   end here
--   N10.go step       go to another step now
--   N10.wait step     stop, and go to step when a signal comes
type Action = N10.Run Job -> Result N10.Next Failure ! {Net(ci.internal, tracker.internal)}

let scrub (job: Job) = ...                       -- as in the tree version

-- Actions: rerun_once, open_pin_pr, comment_flaky (not shown).
-- Each one fits Action.

-- Steps
let rerunning = N10.on_signal "rerunning"
  ( limit = 2                                    -- at most 2 reruns, then fallback
  , on = [ (RerunPassed, comment_flaky)
         , (RerunFailed, N10.go_name "recover") ]
  , fallback = SystemN.investigate )

let rerun_then_wait run = ...                    -- rerun_once, then N10.wait rerunning
let wait_then_rerun run = ...                    -- sleep, rerun_once, then N10.wait rerunning

let recover = N10.step "recover"
  ( bound = Action
  , ask = fn run -> System1.choose Fault.all run      -- run.failure holds the error
  , on = [ (Timeout, rerun_then_wait)
         , (RateLimited, wait_then_rerun) ]
  , fallback = SystemN.investigate )

let dependency = N10.step "dependency"
  ( bound = Action
  , ask = fn run -> System1.choose Pin.all run
  , on = [ (Floating, open_pin_pr) ]
  , on_error = recover
  , fallback = SystemN.explain )

let triage = N10.step "triage"
  ( bound = Action
  , ask = fn run -> System1.choose FailureKind.all run
  , on = [ (Flaky, rerun_then_wait)
         , (Infra, rerun_then_wait)
         , (Dependency, N10.go dependency) ]
  , on_error = recover
  , fallback = SystemN.explain )

-- The workflow: only the start
let on_failure = N10.workflow
  ( event = Job.decoder
  , before = scrub
  , start = triage )

-- Kick-off: one route starts a run, one sends a signal to it
let route (req: HTTP.Incoming) =
  match (req.method, HTTP.segments req) with
  | (HTTP.POST, ["ci", "failed"]) -> (
    match JSON.decode Job.decoder req.body with
    | Error e -> HTTP.reply 400 e
    | Ok job -> (
      N10.start! on_failure job.id job;
      HTTP.reply 202 ""
    )
  )
  | (HTTP.POST, ["ci", "rerun", id, "passed"]) -> (
    N10.signal! on_failure id RerunPassed;
    HTTP.reply 202 ""
  )
  | (HTTP.POST, ["ci", "rerun", id, "failed"]) -> (
    N10.signal! on_failure id RerunFailed;
    HTTP.reply 202 ""
  )
  | _ -> HTTP.reply 404 ""

HTTP.serve! HTTP.Server(port = :8080) route
```

Differences from the tree version:

- `N10.wait step` stops the run and stores it. `N10.signal!` resumes it at
  that step. A workflow with a wait uses `N10.start!`, not `N10.run!`.
- `N10.on_signal name (...)` is a step with no `ask`. The signal picks the
  entry in `on`. `limit` is how many times a run may enter the step.
- A loop needs one reference by name: `rerunning` and `recover` refer to
  each other, so `rerunning` uses `N10.go_name "recover"`. A name must
  resolve to a step that N10 finds through direct references.
- Open: a learned action that returns `go` or `wait` adds an edge. Learned
  actions could be allowed to return only `stop`.

### Parts of a step

| Part | Type | What it is |
| --- | --- | --- |
| name | `String` | Unique in the workflow. Used for traces, metrics and stored runs. |
| `bound` | `Run 'a -> Result Next Failure ! {...}` | The type every action at the step must match. A person writes and reviews it. |
| `ask` | `Run 'a -> Prob 'k` | Picks one option from a closed set. Usually System 1; can be System N or plain code. It must not change anything outside the run. |
| `on` | `List ('k, action or go)` | Each option goes to an action or to another step. Learned actions are added here, never as `go`. |
| `on_error` | step | Where a failed action goes. With none, a failure ends the run. |
| `fallback` | `Run 'a -> Result Next Failure` | Used when `ask` is unsure or `on` has no entry for the option. Usually System N. |

A signal step (`N10.on_signal`) has a name, `on` (keyed by signal), `limit`
and `fallback`.

The workflow (`N10.workflow`) has `event` (the decoder, for replay),
`before` (a System 0 function run first) and `start`. Its steps are the
steps reachable from `start`.

### Decisions so far

- `Prob 'k` = "probably one of these": a probability for each option.
  `decide` takes the top option if it is at or above the threshold.
- Thresholds are settings, per backend and per step.
- A level belongs to a call, not to a part. The event level is the highest
  level of any call.
- A workflow names only its start. Its steps are derived from the start.
- Learned actions are leaves. They must fit `bound`, and `go` does not fit
  `bound`, so System N cannot add structure.
- N10 is generic over the event type. `JSON.of` encodes any value, so only
  the decoder is passed in.
- Every part of a step takes `N10.Run`: the event and the last failure.
- `N10.run!` is the one entry point for an event with no waits: trace,
  tape, spend cap. `N10.start!` and `N10.signal!` are for workflows that
  wait.
- `before` is a plain function and counts as System 0.

## Bounds

A bound is the limit on every action at a step. A person writes it, and a
reviewer approves it, before any learned action exists.

### What a bound is for

- **Learned actions:** the bound is the main control. When a person writes
  a step, the actions that System N will write later do not exist yet. The
  bound is the rule that they must follow. N10 refuses an action that does
  not fit, before a person reviews it.
- **Written actions:** the bound is a second check. A person already
  reviews the code. The bound catches what the review misses, such as a call
  to a host that nobody intended.

### Parts

| Part | Example |
| --- | --- |
| Input | `Run Job` |
| Result | `Next` or `Failure` |
| Reach | hosts (`ci.internal`), files, commands |

Each step has its own bound, because risk is different at each step. The
bound applies to the actions in `on`. It does not apply to `N10.go`, `ask`
or `fallback`.

### Enforcement

| Where | Input and result | Reach |
| --- | --- | --- |
| Wand 0.104.0 | checked at load | effect labels checked at load (`Shell` refused); hosts checked only at run time, by the guard handler |
| Wand with hosts in types | checked at load | checked at load |
| Python, TypeScript | checked at run time | a sandbox (network and file rules) |

The **guard handler** runs each action inside a `Net!http` handler that
refuses every host outside the bound. It works today, but it catches a
wrong host only when the action runs.

### Limits

- **A bound limits reach, not amount.** "Refunds under $50" cannot be a
  bound. Put that rule in the action, or in the host that the action calls.
- **`Shell` gets around `Net`.** `Shell(curl)` reaches any host, so a bound
  must not allow `Shell`.
- **Model hosts and action hosts must be different.** `ask` may call model
  hosts. If a model host were also an action host, `ask` could act.
- **`Raise` counts as an effect.** A learned action must return its errors
  as `Failure`, so that `on_error` can send them on.

### Test findings (Wand 0.104.0)

Tested with a learned-action interface and `Wand.load`:

| Action | Result |
| --- | --- |
| POST to `ci.internal` | loaded |
| `curl` the log to `paste.example.com` | refused at load: "the type allows {Net}, but the body performs Raise, Shell" |
| POST the log to `paste.example.com`, with that host in its own `uses` line | loaded; the guard blocked the request at run time |
| Call `HTTP.post!` (raises) | refused at load: "the body performs Raise" |

Other findings:

- A type's effect row does not accept a host (`! {Net(ci.internal)}` does
  not parse). Only `uses` accepts hosts.
- `Wand.check_file` reports `Net` without the hosts.
- `wand t` does not check a literal URL against `uses`. A wrong host is
  caught when the request is sent.
- `uses {Net(...)}` accepts host names and IPv4. It does not accept IPv4
  patterns, IPv6 or ports.

### Examples

**CI triage.** `Net(ci.internal, tracker.internal)`

| Action | Result |
| --- | --- |
| Rerun the job on `ci.internal` | fits |
| Comment on a ticket on `tracker.internal` | fits |
| Post the log to `paste.example.com` | refused: host not in bound |

**Invoice intake.** `Net(erp.internal)`

| Action | Result |
| --- | --- |
| Post a known vendor's invoice to the ERP | fits |
| Email the vendor about a duplicate | refused: no mail host in bound |
| Save the invoice PDF to disk | refused: no `FS.Write` in bound |

**Support routing.** `Net(helpdesk.internal, billing.internal)`

| Action | Result |
| --- | --- |
| Tag the ticket and route it to a team | fits |
| Refund $20 through `billing.internal` | fits |
| Refund $5,000 through `billing.internal` | **fits**: the bound limits reach, not amount |

## Forming a habit

A habit is a fixed series of tool calls that System N made for a repeated
case, saved as one learned action. After review, the case runs at System 0
with no model call.

### Requirement: System N acts through tools

In a `fallback`, System N acts only through tools, and the step's bound
limits those tools (for example: rerun, comment, open ticket). N10 records
each tool call. If System N only writes text, there is nothing to turn
into an action.

### Writing

```
System N case ─> record ─> group ─> write ─> check ─> replay ─> PR ─> merge
```

1. **Record.** Each time a step goes to System N, N10 logs a case: the
   step, the `Prob` from `ask`, the event, and the tool calls that
   System N made.
2. **Group.** N10 groups the cases by step and top option (for example
   `recover` / `Other`). When a group reaches the group size (3 by
   default), it becomes a candidate.
3. **Write.** System N gets the group, the step's bound, the step's
   options, and the written actions as examples. It returns one file with
   one action, and a short reason.
4. **Check.** N10 checks the action against the bound. A check error goes
   back to System N, with a retry limit. An action that does not fit is
   never shown to a person.
5. **Replay.** N10 runs the group's recorded cases against the action and
   compares the tool calls with what System N did.
6. **Review.** N10 opens a PR with the action, its `uses` line, the
   replay results and the reason. A person merges it or closes it.

### Recall

The recall key is the option. A merged habit is an entry in `on`, so
recall is the same lookup as for a written action:

```
ask ─> Prob ─> top option ─> entry in on ─> learned action
```

An option can be too wide. `Other` can hold "disk full" and "new linter
rule", and one key cannot recall two habits. There are two fixes:

- **A. A new option.** The PR also adds an option (for example `DiskFull`)
  to the step's options. A person approves it with the action. Recall stays
  a plain lookup. Use this in stage B.
- **B. A match test.** Each learned action has a check: code, or a System 1
  yes/no question ("Is this like these cases?"). Recall becomes a second
  `ask`. Use this in stage C, where habits are added with no code change.

### A habit is also a tool

A merged habit is added to System N's tools. When a case almost matches,
System N can call the habit as one tool. This makes System N cheaper, and
its tool calls simpler to turn into the next habit.

### By stage

| Stage | Who finds the group | Who merges | Recall |
| --- | --- | --- | --- |
| B, manual | a person, or N10's case log | a person (PR) | a new option (A) |
| C, automatic | N10 | a person approves, then shadow and promote | a match test (B) |

## Differences between the docs and Wand 0.104.0

The docs assume some Wand features that do not exist. The plan below uses
what Wand has now. Each item is a decision to confirm.

| Doc assumes | Wand 0.104.0 has | Plan |
| --- | --- | --- |
| A `Model(jev)` effect label | Twelve fixed labels. A library cannot add one (R:1530). | Use `Net(<model host>)`. The manifest line still shows which models a file can call. Ask for a `Model` label in Wand later, as a separate change. |
| `Eval.check` against a type and an effect bound | `Wand.check` (no expected type) and `Wand.load!` with an `interface` (R:2309, R:7169) | Write the action to a file. Load it with `Wand.load!` against an `Action` interface. The interface sets the type and the effects. |
| A record/replay handler for model calls | `Test.with_http` and `Test.http_calls` only. No tapes. | Write a `Net!http` handler that records to a JSON-lines tape and replays from it. |
| `wand --dry-run` shows all actions | Dry run withholds every POST and answers `202` with an empty body (runner.ml:1148) | Model calls are POST, so they fail to decode in a dry run. N10 must replay from a tape in a dry run, or route to "unsure". Decide in phase 4. |
| A `Dist 'a` type (named `Prob` here) | No probability type. `Float` has no `log` or `exp`. | Write `Prob` in the library over a sum of words, using `T.all`. |
| Question schema from a sum type | `T.all` lists the constructors of a sum of words (commit c32a3de) | Use `T.all` to send the allowed answers. |

## Package layout

Follow plimsoll and `../wand/CLAUDE.md` Part B.

```
wand.pkg            { package = github.com/wand-lang/N10, wand = 0.104.0 }
N10.wand            public API: step, go, stop, workflow, run!
prob.wand           Prob type and its functions (pure)
system1.wand        System 1 client
systemn.wand        System N client
tape.wand           record and replay handlers
budget.wand         spend cap handler
bound.wand          guard handler and bound checks
habit.wand          case log, candidates, check, shadow, promote, retire
metrics.wand        level-share counts
_lib/               helpers that are not API
examples/_ci.wand   the CI triage pilot as an example workflow
test_*.wand         tests, run with `wand s`
testdata/           recorded tapes
CLAUDE.md, CHANGELOG.md
```

## Phases

There are three stages. Each stage is useful by itself. Each phase ends with
`wand t`, `wand f` and `wand s` clean, and one release.

### Stage A — Routing, with bounds on written actions

#### Phase 0 — Set up

- [ ] Get the request and response shapes of `/v1/systemone` (Choice, Score,
      yes/no). Record a real answer to `testdata/`.
- [ ] Get the Claude Messages API shape that N10 uses (tool use for typed
      output, or plain text).
- [ ] Add `wand.pkg`, `CLAUDE.md`, `CHANGELOG.md` and an empty `N10.wand`.
- [ ] Find out if the `Net` bound accepts a host that comes from config
      (Lev runs on a host we do not know at compile time).

#### Phase 1 — `Prob` (pure, no effects)

- [ ] `type Prob 'k` holds pairs of an option and its probability.
- [ ] `top`, `p`, `sure`, and `decide threshold prob : Option 'k`.
- [ ] Tests with `>>` examples and `test_prob.wand`.

#### Phase 2 — System 1 client

- [ ] `System1.choose`, `score`, `yes_no`, each `! {Env, Net(<host>)}`.
- [ ] Backend record: base URL, key name, threshold. Jev and Lev use the
      same code.
- [ ] Send the allowed answers from `T.all`. Refuse an answer outside the set.
- [ ] Tests with `Test.with_http` and the recorded answers from phase 0.

#### Phase 3 — System N client and spend cap

- [ ] `SystemN.explain` and `SystemN.investigate` (text for a person).
- [ ] `Budget.capped! limit thunk`: a `Net!http` handler that counts spend
      for the System N host. At the limit, it stops calls and returns
      `Error OverBudget`. The caller sends the event to a person.
- [ ] Tests for the cap with fake responses.

#### Phase 4 — Steps and workflows

- [ ] `N10.step`, `N10.go`, `N10.stop`, `N10.workflow`, `N10.run!`, `N10.Run`.
- [ ] Find the steps from the start. Check that names are unique.
- [ ] `on_error`: a failed action goes to the named step. With none, the run
      ends.
- [ ] Trace record for each event: steps, level of each call, model answers,
      action taken.
- [ ] `Tape.recording!` and `Tape.replaying!` handlers on `Net!http`.
- [ ] Decide the dry-run rule (replay from a tape, or treat as unsure).

#### Phase 5 — Bounds on written actions

- [ ] A bound for each step: input, result, and allowed reach.
- [ ] Guard handler: run each action inside a `Net!http` handler that
      refuses hosts outside the bound.
- [ ] Check that each written action's type fits the bound.
- [ ] Tests: an action that reaches an extra host is blocked.

#### Phase 6 — Metrics and the pilot example

- [ ] Level-share counts for each workflow and each step, from the trace.
- [ ] `examples/_ci.wand`: the CI triage workflow, with its written actions.
- [ ] Secret scrub as a `before` function, with tests.

### Stage B — Manual habits

System N proposes an action. People review it and merge it as code. N10
does not promote or retire anything.

#### Phase 7 — Propose a learned action

- [ ] Case log on disk (JSON lines, `FS.lock`): each System N case, with the
      step, the option, the event and what System N did.
- [ ] System N acts through tools in `fallback`, and N10 records each tool
      call. See "Forming a habit".
- [ ] `SystemN.write_action`: writes an action for a group of cases.
- [ ] Check the action against the step's bound, with `Wand.load!` and the
      guard. Check errors go back to System N, with a retry limit.
- [ ] Replay the group's recorded cases against the action. Compare with
      what System N did.
- [ ] Open a PR with the action, its `uses` line, the replay results and,
      if needed, a new option for the step.
- [ ] Add each merged habit to System N's tools.

### Stage C — Automatic habits

#### Phase 8 — Find, shadow, promote, retire

- [ ] Group System N cases. A group of the set size (default 3) is a
      candidate.
- [ ] Match test on each learned action, for recall without a new option.
- [ ] Shadow: after approval, run the action next to System N for a set
      number of cases. Log the results, do not apply them.
- [ ] Promote when the shadow results agree. Retire when cases sent to the
      action later fail.
- [ ] Registry: one record for each learned action, with state
      (candidate, review, shadow, active, retired), owner and history.
- [ ] Version the routing table, so a run sees one table from start to end.

## Open questions

- [ ] **Forming a habit:** how does N10 compare tool calls in replay (same
      calls in the same order, or the same end result)?
- [ ] Which parts does a host runtime (Temporal, Millwright) own: runs,
      waits and signals? If the host owns them, `N10.wait`, `on_signal`,
      `start!` and `signal!` leave N10.
- [ ] Who owns and defines `Failure`?
- [ ] Is a `Model` effect label in Wand still wanted, after N10 works on
      `Net(host)`?
- [ ] Ask Wand for hosts in type effect rows, so a bound can refuse a host
      at load time.
- [ ] Which dry-run rule does N10 use for model calls?
- [ ] Where does the habit registry live: in the workflow repo (files and
      PRs) or in Millwright?
- [ ] What group size and shadow length are the defaults?
- [ ] Who approves a learned action?
- [ ] Which CI system, issue tracker and chat tool does the pilot use?
