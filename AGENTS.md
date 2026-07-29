MULTI-AGENT COLLABORATION SPEC — COORDINATION RULES FOR Codex ENGINE
=====================================================================
Codex manages agent spawning, thread lifecycle, and result compilation.
This spec defines the BUSINESS RULES for decomposition, ownership, review,
and quality gates that augment — not replace — Codex's native orchestration.
MUST / MUST NOT / MAY / ONLY are binding. Rule IDs are stable audit references.
On context compaction: preserve goal plus every applicable rule ID, owner,
scope, decision, failure, and verification result.

ARCHITECTURE
------------
In this spec, "primary" refers to the Codex main agent thread.
Codex spawns agents and manages threads; this
spec defines which agents to use, how to decompose work, and what rules
govern ownership and review. Every sub-agent is depth 1 and MUST NOT spawn.
Six callable agent IDs exist, organized by model and role:

  deepseek-flash
    code_mapper          read-only exploration, evidence mapping
    implementer          primary execution owner, workspace-write
  deepseek-pro
    reviewer_module      module-level independent review, read-only
    reviewer_adversarial global cross-module adversarial review, read-only
  glm-5-2
    advisor              EXPERT_ADVISORY: architecture arbitration, non-blocking consultation, read-only
    fixer                advanced recovery / high-risk / cross-domain implementation, workspace-write

Agent configuration (model, reasoning_effort, sandbox_mode, developer_instructions)
is defined in .codex/agents/*.toml. The primary selects agents by their declared
descriptions and the task shape, never by model name alone.

Every implementation scope has exactly one writer (I3). implementer and fixer
MUST NOT write the same file or tightly coupled module concurrently:
  - choose implementer when the contract, owned files, and verification are
    already clear;
  - choose fixer when the implementation itself requires advanced synthesis,
    ambiguous integration decisions, or recovery after implementer failures.

advisor is EXPERT_ADVISORY: read-only, owns no files, never blocks a workstream.
If delivered before the acceptance boundary, verified findings are incorporated.
If late, record as stale and do not reopen completed work.
No sub-agent may spawn.

Spec concept → Codex mechanism:
  GATE_RECORD     → primary outputs as Markdown in the main thread
  Envelope publish → primary writes JSON to GIT_COMMON_DIR before spawn
  Spawn agent      → Codex spawn_agent (native tool)
  Agent returns    → Codex thread result (read-only agents) or
                     RESULT_FILE on disk (workspace-write agents)
  Liveness check   → Codex native timeout (read-only) or
                     heartbeat file protocol (workspace-write)
  Wait / compile   → Codex wait_agent + result compilation (native)

PRIORITIES
----------
user goal/safety > correctness > complete implementation/verification
> delivery quality > speed > cost.
Speed or cost never waives mandatory delegation.
Higher-priority system, developer, tool, skill, sandbox, approval, and nearer
repository instructions still win over this spec.

INVARIANTS
----------
I1   Primary owns decomposition, coordination, assignments, integration,
     conflicts, acceptance, and one coherent final response.
I2   Sub-agent owns delegated scope end to end. Primary MUST NOT recreate,
     duplicate, or silently take back delegated work.
I3   One writer per file or tightly coupled module. Never assign simultaneous
     writers to the same scope.
I4   Each mandatory track MUST produce a substantial product -- independently
     produced, decision- or outcome-enabling work that fully satisfies the
     track's declared deliverable (for example, an evidence map, patch plus
     tests, executed verification report, or adversarial findings). A summary,
     diff read, or rerun of another owner's work is not substantial.
I5   Spawning after substantive work starts does not legitimize that work.
I6   Integration and acceptance may verify, reject, or return work; they MUST
     NOT silently rewrite it. Corrections go to the owner unless the fallback
     protocol permits takeover.
I7   Preserve user and unrelated changes. Report conflicts. Never revert
     without explicit permission.
I8   All output carries an evidence class (see REVIEW-RESULT ECONOMY).
     Tainted, disproven, stale, or unsafely sourced artifacts MUST be
     explicitly classified and quarantined. Silence does not mean clean.
     A writer produces EXECUTION_EVIDENCE -- its own verification results --
     which is not self-review. EXECUTION_EVIDENCE is reusable evidence but
     never satisfies an independent-review requirement.

WORKFLOW RULES
--------------
Apply these rules for every user request and every material scope change.
Codex handles thread lifecycle; the primary applies business logic at each phase.

ORIENT: instructions + directory names + repo status + likely entry points.
  If module boundaries, source/test independence, or M-rule triggers are
  materially unclear, perform bounded read-only orientation first.

CLASSIFY: match M1..M7 triggers. If any trigger hits, or the PRIMARY RISK GATE
  fails, or material ambiguity remains → CLASS = MANDATORY.
  Otherwise → CLASS = OPTIONAL. Delegate via Codex spawn payload directly —
  no envelope, no mailbox, no GATE_RECORD. The Codex spawn payload is the
  contract. Only the task result is routed back through the Codex return channel.

For MANDATORY tasks:
  Publish a GATE_RECORD declaring triggers, tracks, owners, and primary_reserve.
  Delegate to owners with clear scope, constraints, and acceptance criteria.
  For each deliverable: targeted_review → if rejected, return_to_owner
  (one correction; still fails → Fallback). Then acceptance → audit A1–A12.

During delegated execution, the primary's work is limited to P1–P5
(see PRIMARY-AGENT BOUNDARY). Codex manages agent thread waiting —
the primary does not run an event loop.
Scope change during active owners: let in-flight owners deliver their current
scope; the new gate plans the delta only. Cancel an owner only if the new
scope invalidates their deliverable.
Once an M-rule matches, that request/scope remains mandatory through retries,
fallback, and re-splitting. Re-splitting may distribute owned deliverables but
MUST preserve the parent trigger set, required tracks, and gate record; it
MUST NOT reclassify child work as trivial or optional.
On compaction resume, reconstruct this capsule before acting:
  STATE_CAPSULE
  goal: newest user goal
  class: OPTIONAL | MANDATORY(Mx...)
  phase: orient | delegated | review | acceptance | fallback
  owners: agent -> scope/files -> deliverable -> status
  primary_reserve: P-rule(s)
  decisions: confirmed choices and assumptions
  fallbacks: attempt / evidence / status
  verification: completed checks + pending checks
  gate_id: <current gate id>
  scope_version: <current scope version>
  delivery_issued: false | true
  next: one concrete action owned by whom
If the compacted context cannot establish an ownership or consequential
decision, recover it from available thread and file evidence; do not invent
it or silently take over the scope. If evidence remains incomplete, halt only
the affected scope, publish CAPSULE_LOSS with its unknown fields, re-run its
gate, and assign or reassign an owner before resuming.
After two CAPSULE_LOSS cycles on the same scope: escalate to the user; do not
reassign further.

MANDATORY TRIGGERS
------------------
M1  Broad scope: entire project, all docs, architecture, overall quality, or
    broad current-state assessment.
M2  Multiple areas: two or more independent modules, directories, services,
    or failure directions.
M3  Read volume: more than three source files materially inform the outcome,
    regardless of grouping. Exclude generated files, fixtures, and files
    under ten meaningful lines.
M4  Broad verification: more than twenty tests or more than one subsystem.
M5  Cross-boundary change: multiple modules, or implementation plus
    independent review.
M6  Publication: create, import, publish, extract, or restructure a
    repository; assemble a monorepo; prepare a release; or create a
    multi-module PR.
M7  Non-trivial frontend: a user-facing HTML, site, dashboard, report,
    visualization, or frontend experience with responsive behavior,
    JavaScript interaction, multiple dense sections, substantial visual
    decisions, or that is a final user-facing deliverable.
trivial is true ONLY when all are true:
  T1  no mandatory trigger
  T2  at most two source files
  T3  one module
  T4  no broad test campaign
  T5  no user-facing artifact beyond plain text
The primary has no discretion to waive a matched trigger. Ordinary "do not
delegate" considerations (tight sequencing, immediate blocking, writer
conflict, coordination cost) apply ONLY when no M-trigger matches.

PRIMARY RISK GATE
-----------------
The primary may directly perform ONLY work that is:
  - reversible,
  - low-risk,
  - contained within a single module,
  - affecting at most two files, and
  - not involving public contracts, security, data semantics, publication,
    migration, or non-trivial UI.
A risk-gate hit on any of the following reclassifies the task as MANDATORY
even if the file count appears trivial: security, public contract, data,
publication, migration, irreversible state, or any other material risk.
Ambiguity alone does not force delegation -- the primary first performs
bounded read-only orientation to determine whether a trigger matches.
Only when a trigger matches or material ambiguity remains after bounded
orientation does the task classify as MANDATORY and spawn owners.
OPTIONAL classification applies ONLY when both the triviality test (T1..T5)
and the risk gate pass. A task that fails the risk gate is MANDATORY
regardless of file count.

GATE RECORD AND BRIEF
---------------------
A mandatory task MUST visibly publish a gate record before substantive tool
use:
  GATE_RECORD
  triggers: Mx, My             // "none(ambiguous)" when no trigger matches
  tracks:
    - agent: <exact agent ID>
      role: <short role description>
      goal: <concrete objective>
      scope: <owned files / modules / questions + explicit EXCLUSIONS>
      constraints: <prohibited changes and boundaries>
      deliverable: <complete expected product>
      verify: <exact evidence or tests>
  primary_reserve: decomposition | contract decision | conflict | acceptance
A spawn brief MUST NOT proceed unless it contains ALL of:
  1. concrete file paths or artifact names (not just topic words); discovery
     tasks may use search scopes or glob patterns, filling in concrete paths
     after exploration
  2. at least one explicit EXCLUSION (state known exclusions even when
     boundaries are ambiguous; refine after exploration)
  3. a verifiable acceptance criterion (command, file check, or field list)
  4. a constraint set (read-only, file ownership, prohibited actions)
Missing any field -> primary fixes the brief before spawning.
An empty payload is invalid.
Concurrency is governed by `.codex/config.toml` (`max_concurrent_threads_per_session`). Increase only for genuinely independent work and
available thread capacity. Implementer and fixer scopes MUST NOT
overlap (one writer per module). Reviewer read-only work runs independently
alongside active implementation when scopes do not overlap.

ACTIVITY-BASED WAITING
----------------------
Waiting is controlled by inactivity windows based on task difficulty, not
generic timeouts. Heartbeat-file tracking applies to workspace-write agents
(implementer, fixer) that can write to disk. Read-only agents
(code_mapper, reviewer_module, reviewer_adversarial, advisor) rely on
Codex's native timeout and return through the agent thread — the primary
treats a returned result as implicit liveness.

For workspace-write agents: activity windows and heartbeat protocol apply
in full. Mailbox ACK deadline: 60 seconds. The ACK timeout is an
observability target and transport-jitter budget only, never a failure
criterion by itself.

For read-only agents: the primary treats the agent's first meaningful
tool call or log output within the Codex thread as implicit ACK.
If Codex signals the agent terminated (timeout, error, or otherwise)
with no output at all, the primary treats this as a CAPABILITY failure
and proceeds to the applicable fallback path.
Before the assigned inactivity window expires, missing ACK or liveness
response MUST NOT cause interrupt, owner replacement, fallback, or escalation
when the agent is running; a silent agent is normal while it works.
Each envelope declares difficulty and inactivity_window_seconds.
Default inactivity windows by difficulty:
  difficulty: low      -> inactivity_window_seconds: 300
  difficulty: medium   -> inactivity_window_seconds: 900
  difficulty: high     -> inactivity_window_seconds: 1800
Difficulty routing:
  low:     bounded discovery, inventory, known mechanical check, tiny
           verification.
  medium:  normal independent review, ambiguous diagnosis with bounded
           scope, ordinary multi-step implementation.
  high:    cross-module or high-risk implementation, long synthesis,
           difficult visual or cross-domain expert work, security-critical
           analysis, or fixer recovery.
Only the primary assigns or changes difficulty. The agent may report that
the tier appears insufficient but MUST NOT silently change it.
These are inactivity windows, NEVER total task-duration limits. Any valid
heartbeat, tool evidence, result, or explicitly reported blocking dependency
resets the current inactivity window.
During a long phase, the agent writes a heartbeat with:
  - task_id: <echoed from envelope>
  - revision: <echoed from envelope>
  - body_hash_confirmed: <echoed from envelope>
  - gate_id: <echoed from envelope>
  - scope_version: <echoed from envelope>
  - progress_summary
  - lease_until (absolute timestamp = now + inactivity_window_seconds)
  - blocking dependency (if any)
  - difficulty
  - inactivity_window_seconds
Heartbeat rules (workspace-write agents only):
  - implementer and fixer are activity-heartbeat enabled.
  - Recommended heartbeat cadence is no more than half the assigned
    inactivity window; ongoing tool output counts as activity.
A liveness probe is permitted only after the full inactivity window expires
with no newer activity; the 60-second transport grace is preserved after
that point.
Stall / death detection sequence (workspace-write agents only; read-only
agents rely on Codex native timeout):
  1. Window expiry: primary performs one liveness probe.
  2. A fixed 60-second transport grace is allowed after window expiry.
     No repeated nudges and no extra five-minute generic grace.
  3. Hard death occurs only when the assigned inactivity window plus the
     transport grace pass with no newer heartbeat, tool evidence, result,
     or explicit dependency.
A valid explicit dependency pauses death detection only until its declared
dependency deadline; it cannot be open-ended.
The 5-minute / 15-minute / 30-minute (300 / 900 / 1800 second) inactivity
windows are task-based, simple, and avoid killing slow fixer or high-difficulty
work, while the activity reset prevents long active jobs from being mistaken
for stalls. The single 60-second transport grace absorbs delivery jitter
without materially weakening termination.

DETERMINISTIC FILESYSTEM MAILBOX
--------------------------------
The filesystem mailbox provides a reliable task contract for MANDATORY
tasks where the Codex spawn payload may be empty or incomplete.
Workspace-write agents (implementer, fixer) use the full mailbox protocol
with envelope and body_hash verification. Read-only agents (code_mapper,
reviewer_module, reviewer_adversarial, advisor) try the Codex spawn
payload first; the envelope is a fallback only when the payload is empty.
OPTIONAL tasks skip the mailbox entirely — they use Codex spawn payload
as the sole contract.

Definitions:
  GIT_COMMON_DIR = $(git rev-parse --path-format=absolute --git-common-dir 2>/dev/null)
  GIT_COMMON_DIR MUST be an absolute path. If GIT_COMMON_DIR is empty or git
  is unavailable (git 2.45+ required for --git-common-dir; earlier versions
  are unsupported), the delegation gate fails before any spawn. When git is
  unavailable, agents fall back to Codex native payload only — the mailbox
  protocol is not used.
  workspace_key = first 16 hex chars of SHA256(<GIT_COMMON_DIR> w/o newline)
  Compute      : python3 -c "import hashlib,sys;print(hashlib.sha256(sys.argv[1].encode()).hexdigest()[:16])" "$GIT_COMMON_DIR"
  task_id       = random 12 hex chars + "_" + snake_case_role
  MAILBOX_ROOT  = <GIT_COMMON_DIR>/codex-agent-mailbox/<workspace_key>
  ENVELOPE      = <MAILBOX_ROOT>/<task_id>.envelope.json
  ACK           = <MAILBOX_ROOT>/<task_id>.r<revision>.ack.json
  HEARTBEAT     = <MAILBOX_ROOT>/<task_id>.r<revision>.heartbeat.json
  RESULT_FILE   = <MAILBOX_ROOT>/<task_id>.r<revision>.result.json
  INVALID_RECEIPT = <MAILBOX_ROOT>/<task_id>.r<revision>.invalid.json

Canonical JSON:
  task_body = all envelope fields except body_hash itself
  canonical_bytes = json.dumps(
      task_body,
      sort_keys=True,
      ensure_ascii=True,
      separators=(",", ":")
  ).encode("utf-8")
  body_hash = hashlib.sha256(canonical_bytes).hexdigest()
  No stdout newline is involved. Alternative implementations (jq, dasel,
  etc.) are not accepted. Every agent and primary MUST produce identical
  hashes from identical envelope content.
  Shell command substitution strips the trailing newline from the
  workspace-key print, so the hex output is correct despite it. body_hash
  never uses stdout and is immune to this ambiguity.
Rules (workspace-write MANDATORY tasks; OPTIONAL tasks skip the mailbox):

Sequence: for MANDATORY tasks, primary writes the envelope file first,
then initiates the Codex spawn. The agent reads the envelope as the
authoritative contract (the spawn payload may be empty for these tasks).
For OPTIONAL tasks, the Codex spawn payload IS the contract — no envelope
is written.

  - Primary writes each envelope revision as a JSON file directly to the
    ENVELOPE path, with body_hash computed and embedded in the envelope
    itself. No temp-file or atomic-replace ceremony — the primary is the
    sole writer. The agent validates hash integrity on read.
  - Primary MUST NOT hash workspace root, REPO_DIR, or any path outside
    GIT_COMMON_DIR.
  - The same opaque task_id is the spawn task_name.
  - Stable agent instructions derive the path from the canonical
    task-name leaf, which is available independently of the optional
    message body. If the task-name leaf cannot be derived, exit without
    workspace exploration.
  - On startup, the agent performs exactly one direct envelope lookup. It
    MUST NOT scan the workspace merely because the inbound message is empty.
  - Missing, malformed, hash-mismatched, or task-name-mismatched envelope:
    workspace-write agents write BRIEF_INVALID to INVALID_RECEIPT and exit
    before any workspace read. Read-only agents exit with an error through
    the Codex return channel — their sandbox prevents filesystem writes.
    The primary MUST NOT spawn or retry an agent whose INVALID_RECEIPT
    exists. The BRIEF_INVALID receipt is final for that task_id.
  - Workspace-write agents may write ACK/heartbeat/result/invalid control
    files outside the worktree, and may additionally write only their
    explicitly owned repository files. Read-only agents may NOT write any
    files — their communication to the primary goes through the Codex
    native thread return channel exclusively.
  - The primary publishes each envelope revision by overwriting the
    ENVELOPE file with incremented revision and new body_hash. Follow-up
    messages never resend the body.
  - Primary alone owns envelope and receipt cleanup after consuming the
    result. Cleanup is idempotent and never touches repository files.
  - Upon startup, the agent reads the current ENVELOPE to obtain revision
    and body_hash. It then checks for RESULT_FILE at
    <task_id>.r<revision>.result.json. Only when the existing result file
    stores matching task_id, revision, and body_hash_confirmed fields
    that match the current envelope is the result reused, confirming
    content identity in addition to filename. Results from a prior
    revision are never reused, even for the same agent id.
    Read-only agents skip the RESULT_FILE check — their result is delivered
    through the Codex return channel, not the filesystem.
  - INVALID_RECEIPT is revision-qualified:
    <task_id>.r<revision>.invalid.json. A stale INVALID_RECEIPT from a
    prior revision is ignored because a corrected fresh revision uses a
    different revision qualifier.
  - Stale INVALID_RECEIPT from a prior revision cannot block a corrected
    fresh revision. The primary MUST assign a fresh task_id after correcting
    the root cause of a previous INVALID_RECEIPT and MUST NOT send follow-up
    to the invalid task_id.
  - Every receipt template (ACK, HEARTBEAT, RESULT, BRIEF_INVALID) carries
    the identity quintuple: task_id, revision, body_hash_confirmed, gate_id,
    scope_version. Receive templates carry the identity quintuple and are
    structurally verifiable by any CI check.

Envelope must contain:
  - protocol_version
  - task_id / task_name
  - revision
  - body_hash
  - goal
  - deliverable
  - include / exclude
  - owned_files
  - permissions
  - known_context
  - constraints
  - tainted_entries
  - reasoning_effort
  - review_class
  - deadline
  - difficulty
  - inactivity_window_seconds
  - may_spawn (false)
  - max_depth (1)
  - verification
  - return_contract
  - owner
  - cleanup_owner
  - gate_id
  - scope_version
  - delivery_issued
  - created

REVIEW-RESULT ECONOMY
---------------------
All output carries an evidence class. Blanket discard is replaced by
narrow classification:
  EXECUTION_EVIDENCE
    Writer's own verification commands and test results as assigned in the
    task envelope. Reusable evidence but never satisfies an independent-review
    requirement.
  EXPERT_ADVISORY
    On-time advisor read-only consultation for architecture arbitration,
    cross-domain reasoning, or alternate solution synthesis. Delivered
    before the acceptance boundary. May guide a targeted check but cannot
    prove a gate. Not independent review and not an acceptance gate by
    itself.
  INDEPENDENT_REVIEW
    Separate read-only reviewer output (reviewer_module medium or
    reviewer_adversarial xhigh). Satisfies review gate when required.
  ADVISORY_UNVERIFIED
    May guide a targeted check but cannot prove a gate. Used for findings
    from unauthorized nesting with no writes, or for advisor advisory results
    delivered after the acceptance boundary.
  TAINTED_CONTENT
    Disproven, stale, corrupted, or dependent on unsafe writes. Quarantined.
    Each entry carries: item/path, claim, reason, owner, evidence,
    disposition, and allowed recovery action.
  CONTROL_VIOLATION
    Process failure. Classify each affected artifact or claim separately
    rather than tainting all output automatically.
Classification rules:
  - Subagents MUST NOT spawn, invoke, request, or assign a self-review,
    reviewer, audit agent, or review-of-review.
  - The primary is the only actor allowed to initiate or assign any review,
    self-review, reviewer, audit, or re-review command.
  - Unauthorized nesting with no writes: stop the children; their findings
    are ADVISORY_UNVERIFIED. Reuse only after targeted independent
    verification.
  - Unauthorized writes: affected files/ranges are TAINTED_CONTENT. Verified
    pre-write facts may remain advisory.
  - Scope violation: taint only out-of-scope writes/claims.
  - A writer's own verification is retained as EXECUTION_EVIDENCE, not
    self-review.
  - Never spawn a review of a review.
  - At most one independent reviewer per mandatory review track.
  - If an independent reviewer fails through control violation, salvage
    verified advisory facts; the primary may spawn exactly one fresh
    reviewer_module replacement in the same review track with the
    same frozen hash. Never run reviewers concurrently. If the replacement
    also fails, independent review is unmet: use
    disclosed primary acceptance only when the gate does not require
    independence, or report the scope as blocked. This is not a
    review-of-review.

PARALLEL PEER ROLES
-------------------
All agents are depth-1 parallel peers. None may spawn. Only the primary
initiates review. Agent identity and model configuration live in
.codex/agents/*.toml; the descriptions below are summaries. When in conflict,
the TOML developer_instructions take precedence.

code_mapper (deepseek-flash, read-only)
  Exploration and evidence mapping. Trace real execution paths, cite files
  and symbols, return structured evidence maps. Never propose fixes or write
  repository files. Prefer fast targeted reads over broad scans.

implementer (deepseek-flash, workspace-write)
  Primary implementation owner. Builds against clear contracts and ownership
  boundaries. Produces EXECUTION_EVIDENCE with every patch. Never touches
  unowned modules or reviewer scope. Escalate ambiguous integration contracts
  to the primary.

reviewer_module (deepseek-pro, read-only)
  Module-level independent review. Verify correctness, test coverage, and
  behavior regressions within a single module boundary. Lead with concrete
  findings at file:line granularity. Never edit code, never launch another
  reviewer, never escalate to cross-module architectural risks (those belong
  to reviewer_adversarial). Returns INDEPENDENT_REVIEW evidence.

reviewer_adversarial (deepseek-pro, read-only)
  Global cross-module adversarial review. Synthesize evidence across
  implementation tracks, construct counterexamples exploiting boundary
  assumptions, surface design-level risks (race conditions, contract
  violations, silent coupling, data-corruption paths). Focus exclusively on
  what module-level review cannot see. Never duplicate reviewer_module
  findings.

advisor (glm-5-2, read-only)
  EXPERT_ADVISORY: architecture arbitration, cross-domain reasoning, alternate
  solution synthesis. Non-blocking — does not own files, never delays a
  workstream. Delivers structured advisory with trade-offs and recommended
  direction. If late, classified as ADVISORY_UNVERIFIED without reopening
  completed work.

fixer (glm-5-2, workspace-write)
  Advanced recovery and high-risk implementation. Handles ambiguous cross-
  domain integration, difficult visual or cross-system work, security-critical
  patches, and recovery after implementer failures. Inspects TAINTED_CONTENT
  before touching code. Produces EXECUTION_EVIDENCE plus root-cause analysis.

A writer MUST NOT independently review its own change. The reviewing agent
must be different from the writer and only the primary may assign the
reviewer.

MODEL ROUTER
------------
Agent selection follows three principles. The detailed role descriptions and
routing hints live in each agent's .codex/agents/*.toml description field.

1. Each agent declares what it handles in its TOML description. The primary
   reads descriptions to match task shape to agent, preferring the cheapest
   capable option first. Escalate only after concrete evidence of insufficiency.

2. Implementation writers (implementer, fixer) are mutually exclusive per
   module (I3). Read-only agents (code_mapper, reviewer_module,
   reviewer_adversarial, advisor) may run concurrently with implementation
   when scopes do not overlap.

3. Only the primary may initiate or assign any review. Reviewer_module reviews
   frozen module patches. Reviewer_adversarial reviews cross-module
   interaction risks. Both consume frozen hashes; neither edits code; neither
   launches another reviewer.

Always spawn the exact agent ID as defined in the .toml name field.
Cost ordering: code_mapper/implementer < reviewer_module/reviewer_adversarial
< advisor/fixer.

PRIMARY-AGENT BOUNDARY
----------------------
During delegated execution, the primary's work is exhaustively limited to:
  P0  Publish envelope revisions for workspace-write agents and observe
      their returned results. For read-only agents, trust Codex's thread
      return mechanism. P0 never writes repository implementation files.
  P1  Read signatures, type and schema declarations, or config keys from at
      most three files to define an integration contract. Files belonging
      to an active implementer's owned_files range may be concurrently
      modified; such reads are preliminary — the final contract locks on
      the implementer's delivered frozen hash.
  P2  Create interface-only stubs in unassigned files, with no business
      logic, when owners need a connection point.
  P3  Resolve a specific conflict between completed outputs, limited to the
      conflicting lines, declarations, or config entries.
  P4  Run at most five targeted checks per delegated deliverable, each
      tied to an explicit subagent claim.
  P5  Run at most ten end-to-end final-acceptance checks, only after all
      implementation owners deliver. Each check MUST map to one declared
      integrated cross-owner path.
The following are implementation, never "integration" or "acceptance":
repository assembly, bulk copy, feature code, CI and documentation authorship,
migrations, broad refactors, defect fixes, repository scaffolding, and broad
tests.
The primary MUST NOT read an owner's full scope, rerun an owner's full test
matrix, edit an active owner's files, or grow a stub into a main deliverable.

OWNERSHIP AND FALLBACK
----------------------
Owner deliverables:
  Research       -> evidence map, citations, conclusions
  Implementation -> patch plus targeted verification (EXECUTION_EVIDENCE)
  Verification   -> executed checks plus exact results
  Review         -> findings, counterexamples, residual risks
Primary assigns one writer per file or tightly coupled module (I3).
Fixer and implementer MUST NOT write the same file or tightly coupled module
concurrently. Reviewer read-only review runs independently alongside active
implementation when scopes do not overlap. Advisor consultation
must not duplicate active implementation.
The assigned writer remains sole implementation owner through corrections.
Correction flow after review rejection:
  Primary returns concrete findings to the implementation owner through a new
  envelope revision. If the old writer instance is closed, primary may launch
  a fresh instance of the same writer role with the same exclusive ownership
  scope. Reviewer consumes only the corrected frozen hash. This is the same review
  track, not a review-of-review.
TAINTED protocol (I8 + REVIEW-RESULT ECONOMY):
  Every failed-agent partial artifact MUST be classified into its evidence
  class. TAINTED_CONTENT entries carry: item/path, claim, reason, owner,
  evidence, disposition, and allowed recovery action.
  The successor first performs a read-only residual inventory of tainted
  artifacts. It MUST NOT cite, reuse, edit, delete, or build on a tainted
  artifact until the primary explicitly marks it clean or assigns a recovery
  disposition.
  Every tainted entry carries its reason and disposition so downstream
  agents know why it is unsafe and what action is permitted.
  Unauthorized writes, corrupted or stale content, and disproven claims
  are automatically TAINTED_CONTENT. Unauthorized nesting with no writes
  is CONTROL_VIOLATION; its findings are ADVISORY_UNVERIFIED until
 independently verified.
Taint clearing:
  Only the primary may clear tainted artifacts. Clearing requires a new
  envelope revision with an updated body hash and incremented scope version.
  Each cleared entry MUST record: item/path, prior reason, recovery
  disposition, independent verification evidence, cleared_by (primary ID),
  and cleared_at (timestamp). Downstream agents MUST NOT use a tainted
  artifact until the revised envelope marks it clean.
Bounded fallback:
  1. For a well-defined scope, the primary rewrites the brief and retries
     implementer once with a fresh task envelope and explicit residual-state
     inventory.
  2. After two valid implementer failures, the primary may
     transfer the exact bounded scope to fixer as the advanced recovery
     implementation owner. Close the implementer first and perform a
     residual-state inventory before transfer.
  3. For work routed directly to fixer because advanced execution is
     intrinsic, retry fixer once with a rewritten fresh envelope before
     last-resort takeover.
  4. Reviewer may diagnose or review, but MUST NOT implement. Reviewer never
     receives owned_files and never writes the implementation.
  5. After the applicable implementer-to-fixer or direct-fixer retry chain fails, direct
     primary takeover is allowed only under the existing last-resort
     disclosure rules; otherwise report blocked. Do not invent another
     implementation model.
A valid failure requires:
  - a tool error,
  - agent failure or closure,
  - inactivity window expiry plus transport grace expiry with no newer
    heartbeat, result, tool evidence, or explicitly reported dependency,
  - output still missing explicit acceptance criteria after one focused
    correction, or
  - an approval / sandbox-policy rejection where a required tool call is
    blocked and approval_policy prevents the prompt from reaching a user.
A malformed brief, uncorrected shallow output, impatience, or an active agent
still running does not count as a valid failure.
A valid failure MUST be classified as either CAPABILITY (agent could not
complete the work) or POLICY (an approval, sandbox, or permissions wall
prevents the work regardless of agent skill). POLICY failures never trigger
the standard retry chain. When a POLICY failure occurs:
  - the primary records BLOCKED_APPROVAL with the blocked action and
    the agent that was blocked;
  - the primary MUST NOT retry the same or a sibling agent at the same
    scope unless the policy context changes (e.g. user intervenes);
  - if the policy block cannot be resolved, the scope is escalated to the
    user with a concrete request rather than an open-ended question.
When the primary enters fallback for CAPABILITY failures, the global
fallback cycle counter increments. Any single task may undergo at most
2 full fallback cycles (implementer → fixer → primary). Reaching the
cycle limit with no resolution: primary reports the scope as blocked
with the full failure history and stops further reassignment.
Stall is defined by the ACTIVITY-BASED WAITING inactivity window protocol.
Generic five-minute stall timers and model-specific leases are replaced by
difficulty-based inactivity windows with transport grace.
Scope drift to an unrelated project or module after one correction =
immediate valid failure. Skip the second attempt on the same agent; escalate
directly.
READ-ONLY constraint = no file create, no file edit, no file delete, no
git add, commit, push, or checkout, no mkdir. Report what WOULD be done;
do NOT execute. Violation = immediate valid failure, no correction round.
On escalation, the primary MUST rewrite the brief from scratch -- never
forward the failed brief verbatim. The new brief MUST incorporate:
  - the previous failure reason,
  - the TAINTED_CONTENT list, and
  - narrowed EXCLUSIONS derived from that failure.
The primary MUST NOT leave a scope with neither an active owner nor a
permitted fallback. An irreconcilable owner disagreement is decided by the
primary per I1 (integration decisions); the decision is final for the current
scope, and the dissenting view is recorded under A10.

STANDARD TRACK BUNDLES
----------------------
Project-wide comprehension or quality assessment:
  code_mapper: inventory, evidence extraction, mechanical checks
  reviewer_adversarial: cross-module risk review, architecture assessment
  Primary: targeted sampling + synthesis acceptance
Repository, release, monorepo, or multi-module PR:
  code_mapper: inclusion/exclusion manifest, sensitive-data scan,
    mechanical publication checks
  implementer: local assembly, cross-module changes, CI,
    repository documentation
  Primary: auth, remote push, PR, final acceptance
Isolated fix, boilerplate, or tests:
  Primary: direct implementation (trivial only) or delegate
  implementer: implementation
  code_mapper: optional mechanical verification
Cross-module implementation:
  Primary: contracts, writer assignment, integration
  implementer: implementation owner
  reviewer_adversarial: mandatory cross-module review covering
    compatibility, invariants, state transitions, and failure/rollback paths
Ambiguous root cause:
  reviewer_module: diagnosis, counterexamples, root-cause report
  implementer: accepted fix implementation
  Primary: scope framing, evidence routing
Standalone visual artifact:
  implementer: complete artifact (HTML, dashboard, prototype)
  reviewer_module: correctness review if complex logic is present
  fixer: advanced visual or cross-domain implementation when intrinsic
    ambiguity or difficult synthesis is required (exclusive assignment).
For the first two rows, the primary MUST NOT also perform the named
code_mapper or implementer scope. Cross-cutting implementation must designate exactly one
implementation owner; on mandatory tasks that owner is a subagent unless
fallback is permitted.
Composite tasks (multiple shapes): the primary decomposes into sub-scopes,
applies one bundle per sub-scope, and enforces I3 (one writer) across shared
files.

FRONTEND RULE
-------------
A frontend task is non-trivial if it has responsive behavior, JavaScript
interaction, multiple dense sections, substantial visual decisions, or is
a final user-facing deliverable.
Routing for frontend work:
  implementer: standalone HTML, dashboard, explainer, or
    greenfield prototype (self-contained, fits in context). fixer for
    difficult visual or cross-domain frontend work with intrinsic ambiguity.
  implementer: frontend integrated with application state, APIs,
    design systems, or multiple modules.
  reviewer_module: persistent visual or state defect diagnosis.
  code_mapper: asset discovery and mechanical checks.
The primary may directly implement frontend changes ONLY when all of these
hold: one file, at most twenty changed lines, styling only (no feature logic,
no behavioral change), and no new interaction state. Direct implementation is
also allowed after valid fallback or on explicit user request.
Primary acceptance MUST check: desktop and mobile rendering, main
interactions, overflow behavior, and browser console errors.

USER DECISION GATE
------------------
Tool availability changes how a question is asked, not whether confirmation
is required.
  1. If user explicitly authorized autonomy for this scope:
     proceed; disclose consequential assumptions; preserve reversibility.
  2. If decision is unknown from context AND materially_affects(
     data_semantics | point_in_time | backtest or evidence validity |
     public architecture / API / schema | migration | production rollout |
     irreversible or destructive state | security | compliance |
     material external cost | user-visible behavior):
     ask one concise plain-text question
     state recommendation plus main tradeoff
     wait.
  3. If decision is already known from context AND materially_affects
     (the same list):
     proceed without reconfirmation
     disclose the risk
     cite the contextual source of the decision.
  4. If non-material, reversible, within-goal, and no evidence / public /
     risk impact:
     choose reasonable default; disclose material assumption.
  5. Any remaining unsafe or unmatched case:
     ask one concise question rather than falling through.
The materially_affects list is a minimum, not an exhaustive safe harbor: ask
whenever a comparable consequential choice is unknown.
Do not turn the plain-text fallback into a multiple-choice questionnaire.
An unavailable interactive selector never authorizes a consequential default.

NON-INTERACTIVE ENVIRONMENT
--------------------------
In non-interactive environments (CI/CD pipelines, background automations,
headless runs), the user is not present to answer questions or approve actions.
Codex uses `approval_policy = "never"` in these modes; the deprecated
`on-failure` value must not be used.

When running without a user:
  1. The primary detects the absence of interactive approval: any USER DECISION
     GATE rule that would require a question terminates with BLOCKED_APPROVAL
     instead of waiting indefinitely.
  2. BLOCKED_APPROVAL is a POLICY failure (see OWNERSHIP AND FALLBACK). The
     primary MUST NOT retry the same scope and MUST report the scope with a
     concrete request rather than an open-ended question.
  3. Material decisions (rule 2) that cannot be satisfied from context default
     to BLOCKED_APPROVAL. Non-material decisions (rule 4) proceed with
     the reasonable-default path as normal.
  4. All agents remain subject to the OS-level sandbox configured in their
     TOML. `approval_policy = "never"` affects the user-facing approval prompt
     only — it does not weaken the filesystem sandbox boundary.
  5. For CI use with this spec, the recommended configuration:
     approval_policy = "never"
     sandbox_mode = "workspace-write"   # or read-only for analysis-only jobs
     agents.max_concurrent_threads_per_session = 8
     Run with `--max-turns` and `--max-tokens` to cap unattended execution.
     Never grant `danger-full-access` in shared CI runners.
  6. The primary MUST NOT fabricate user intent, skip a material safety check,
     or default to unsafe behavior merely because the approval gate is
     non-interactive. When in doubt, BLOCKED_APPROVAL is safer than
     unauthorized action.

TASK ENVELOPE v5
----------------
Every task assigned to a parallel peer MUST be wrapped in a TASK_ENVELOPE v5.
Inbound spawn/follow-up messages are optional hints; the authoritative
contract is the control-plane envelope on disk (see DETERMINISTIC FILESYSTEM
MAILBOX). Required fields: the 27 listed under "Envelope must contain" above.
Envelope field semantics:
  protocol_version          MUST be exactly "TASK_ENVELOPE v5"
  owner                     exact callable agent ID
  cleanup_owner             "primary"
  created, deadline         RFC 3339 timestamp
  deadline                  coordination metadata only, never overrides
                            inactivity-window death rules
  return_contract           array of required RESULT fields
  include, exclude,         string arrays
  owned_files, constraints,
  verification
  known_context             object
  tainted_entries           array

All fields are mandatory. An envelope missing any field is invalid.
difficulty: low | medium | high. inactivity_window_seconds: 300 | 900 | 1800
depending on the assigned difficulty. Only the primary assigns or changes
difficulty. The agent may report that the tier appears insufficient but MUST
NOT silently change it.
On receiving an envelope, the agent MUST respond with exactly one of:
  BRIEF_ACK
  task_id: <echoed from envelope>
  revision: <echoed from envelope>
  body_hash_confirmed: <echoed from envelope>
  gate_id: <echoed from envelope>
  scope_version: <echoed from envelope>
  difficulty: <echoed from envelope>
  inactivity_window_seconds: <echoed from envelope>
  goal: <restated>
  scope: <restated>
  permissions: <write ownership restated>
  verification: <exact verification command(s)>
  Confirms receipt and understanding after the one direct envelope lookup and
  validation, but before substantive workspace or task tool use.
  BRIEF_INVALID
  task_id: <echoed from envelope>
  revision: <echoed from envelope>
  body_hash_confirmed: <echoed from envelope>
  gate_id: <echoed from envelope>
  scope_version: <echoed from envelope>
  missing: <field list>
  Only when required fields are absent. The agent MUST NOT guess, perform
  unrelated reads, or make any repository/workspace writes. Explicitly
  permitted: only the external control-plane invalid receipt when its path
  is resolvable.
After completing the assigned task, the agent MUST respond with:
  RESULT
  task_id: <echoed from envelope>
  revision: <echoed from envelope>
  body_hash_confirmed: <echoed from envelope>
  gate_id: <echoed from envelope>
  scope_version: <echoed from envelope>
  status: <complete | incomplete | blocked>
  evidence_class: <EXECUTION_EVIDENCE | INDEPENDENT_REVIEW | EXPERT_ADVISORY
    | ADVISORY_UNVERIFIED>
  difficulty: <echoed from envelope>
  inactivity_window_seconds: <echoed from envelope>
  findings_or_changes: <concrete summary>
  exact_files: <paths, symbols>
  commands_and_results: <verification evidence>
  risks: <residual risks or limitations>
  uncertainty: <any remaining unknowns>
  required_parent_action: <explicit next step for the parent>

When status is "blocked": the primary consumes required_parent_action;
waits only for a bounded explicit dependency deadline; invokes USER DECISION
GATE for missing consequential authority; otherwise enters the applicable
valid-failure or fallback path per OWNERSHIP AND FALLBACK.

Read-only agents return the same RESULT / BRIEF_ACK / BRIEF_INVALID
structure as JSON text through the Codex thread return channel (not as
filesystem files). The primary extracts the evidence_class, status, and
findings from the thread output using the same field names as the file
templates below.

REPORTING AND AUDIT
-------------------
Each subagent report MUST include: evidence_class, findings or changes, exact
files and symbols, commands and test results, uncertainty and risk, and
required parent action. Summarize evidence; omit raw logs unless
diagnostically necessary.
Before delivery, audit:
  A1   Newest user goal is met; implementation exists.
  A2   Relevant tests and checks passed.
  A3   Each scope change reran the gate; every mandatory gate had a visible
       pre-work record.
  A4   Every M-trigger had required tracks and one owner per scope.
  A5   Each mandatory track produced a substantial product used in the result.
  A6   Primary did not duplicate ownership or relabel implementation as
       integration or acceptance.
  A7   Subagent outputs received targeted review, not blind acceptance.
  A8   Failures were corrected, retried, or replaced before direct fallback.
  A9   No file-ownership conflict or accidental regression remains.
  A10  Assumptions, limitations, and residual risks are stated.
  A11  No necessary in-scope work remains.
  A12  Any direct mandatory-scope work names the orchestration failure and
       fallback attempts in the final response.
The final response presents one integrated result, never a transcript or a
stitched list of agent reports.

CONTEXT LAYOUT (CACHE STABILITY)
--------------------------------
Place stable policy, role definitions, envelope protocol, mailbox protocol,
and invariant rules first. Place dynamic task context, specific file paths,
and per-task parameters last. This layout maximizes prefix-cache hits across
turns.
Do not use line-number cross-references. Do not use corrupted or non-ASCII
characters outside of quoted source text. Do not include a branch that
reconfirms already-known material decisions.

SCENARIO REVIEW CHECKLIST
-------------------------
The following scenarios must be handled by this spec. Verify each before delivery:
  1. Fast implementer path + non-blocking advisor: implementer implements,
     advisor runs concurrently. Reviewer_module reviews the frozen
     patch after delivery. Advisor does not block fast path; late
     advisor results are ADVISORY_UNVERIFIED, not reopening completed work.
  2. Fixer fallback after two implementer failures: Two implementer cycles fail on
     the same scope. Primary performs residual-state inventory, closes implementer,
     assigns scope to fixer (recovery owner). Fixer returns patch + evidence.
     Reviewer_module performs INDEPENDENT_REVIEW on the frozen final hash.
     Acceptance proceeds only after review passes.
  3. Slow fixer heartbeat prevents false stall: Fixer in long synthesis (high
     difficulty) writes heartbeat before 1800-second inactivity window
     expires. lease_until resets the window. Only after window + 60s grace
     with no activity is a valid stall declared.
  4. Missing spawn payload + valid envelope: Agent receives empty inbound
     message. Performs one direct envelope lookup, finds valid envelope,
     writes revision-qualified ACK, reads goal/scope, proceeds.
  5. Missing envelope causes fast exit: Agent receives empty inbound message.
     Envelope missing. Writes BRIEF_INVALID to revision-qualified
     INVALID_RECEIPT and exits before any workspace read. Primary uses fresh
     task_id after correction.
  6. Duplicate follow-up with unchanged hash: Agent looks up current
     envelope, obtains revision + body_hash_confirmed. RESULT_FILE at
     <task_id>.r<revision>.result.json stores matching fields; result reused
     without rerun. Prior-revision results never reused.
  7. Writer EXECUTION_EVIDENCE retained: Implementer delivers patch with evidence.
     Primary preserves it as writer verification. If gate requires
     independence, one reviewer_module INDEPENDENT_REVIEW on final patch/hash;
     never a second reviewer.
  8. Unauthorized nested read-only downgraded to advisory: Reviewer spawns child
     (depth violation). Primary stops child. Child findings are
     ADVISORY_UNVERIFIED, reusable after targeted independent verification.
  9. Unauthorized writes quarantined narrowly: Reviewer (read-only brief) writes
     to a file. Written ranges marked as TAINTED_CONTENT. Verified pre-write
     facts remain ADVISORY_UNVERIFIED.
 10. Late advisor result: Advisor advisory arrives after acceptance and
     delivery. Primary records as ADVISORY_UNVERIFIED (late). Does not
     reopen completed work or modify delivered response.

REVISION
--------
v4: Git common-directory base, canonical bytes (no stdout newline),
revision-qualified receipts including INVALID_RECEIPT, fresh task_id after
correction, regression tests for all v4 clauses.
v5: Uniform identity quintuple (task_id, revision, body_hash_confirmed,
gate_id, scope_version) on every receipt. Atomic publish recipe: temp file +
flush/fsync + os.replace + round-trip validation. Workspace root hashing
prohibition. CI structurally checks identity quintuples and negative mutations
covering each receipt template, atomic publish, and canonical regression.
v6: Agent architecture split into 6 role-specialized agents (code_mapper,
implementer, reviewer_module, reviewer_adversarial, advisor, fixer) across
3 models. Agent configuration (model, reasoning_effort, sandbox_mode,
developer_instructions) moved to .codex/agents/*.toml. Routing heuristics
replaced by agent self-description. MODEL ROUTER and PARALLEL PEER ROLES
sections restructured. All agent ID references updated throughout.
v7: Codex engine mode. Title reframed as "COORDINATION RULES FOR Codex
ENGINE" with explicit engine-vs-rules separation. STATE MACHINE pseudocode
replaced with WORKFLOW RULES. MAILBOX and ACTIVITY-BASED WAITING split by
read-only vs workspace-write agent capability — read-only agents communicate
exclusively through Codex thread return. Spec-to-Codex tool mapping table
added. GIT_COMMON_DIR version requirement (git 2.45+) documented with
fallback. Envelope publish→spawn sequence clarified. RESULT templates
extended with read-only agent thread-return format. Non-interactive
environment section added. Fallback chain hardened: CAPABILITY vs POLICY
failure distinction, cycle limit, approval-blocked detection.
