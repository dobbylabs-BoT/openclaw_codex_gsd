# SYSTEM_base_codex

## 1. Purpose

This file defines a common base for:

- OpenClaw as the runtime, channel, and overall coordination layer
- Codex as the real execution engine
- `codex app-server` as the automation bridge
- GSD as the structured workflow layer per project when its use is confirmed

This file is not a project plan. 
It is an operational base that is meant to be consumed in order to:

- deploy the technical solution correctly
- prepare the shared environment
- deploy the support wrapper and reusable bootstrap
- establish a stable way to initialize projects
- define how work must be executed when a project uses GSD

---

## 2. When the agent must read this SYSTEM

The agent must read this file in these cases:

### Mandatory reading
- when deploying the base for the first time
- when starting in a new environment
- when initializing a new project
- when changing a project's mode
- when repairing the OpenClaw ↔ Codex bridge
- when re-validating the pure GSD gate

### Not required before every minor step
Once a project is already initialized and stable, the agent must not re-read this SYSTEM before every minor action.

During normal operation it should work mainly from:

1. `AGENTS.md`
2. `STATUS.md`
3. `HANDOFF.md`
4. `BLOCKERS.md`

The SYSTEM becomes the primary source again only if:
- the base must be reconfigured
- there is a technical bridge issue
- the pure GSD gate must be re-validated
- there is uncertainty about execution rules

---

## 3. Separation of scopes

This system has two clearly separate scopes:

### Scope A: Base solution deployment
This runs once per environment in order to leave the shared base correctly installed.

### Scope B: Project execution
This runs whenever the user wants to work on a specific project.

The agent must complete Scope A first. 
Only then may it execute Scope B.

---

# SCOPE A. BASE SOLUTION DEPLOYMENT

## A0. General rule

This scope does not build product features. 
It only makes the base platform operational so that projects can later be executed correctly.

If a phase in this scope fails, the agent must stop and must not continue to the next phase.

The report for each phase must use this format:

```md
Phase:
- name:
- target:
- action:
- evidence:
- result:
- next step:
```

### Mandatory phase execution rule
The agent must execute exactly one phase at a time.

It must not:
- announce a future phase instead of executing the current one
- move to the next phase before closing the current one
- report a phase as if it were completed without executed evidence
- replace execution with planning language or intention language

A phase is considered closed only if the response contains all of:
- executed action
- concrete evidence
- `result: PASS | BLOCKED | NEEDS_CONFIRMATION`
- `next step`

If the current phase is not closed, the agent must not proceed to the next phase.

### Auto-advance rule
If a phase closes with `PASS` and the next phase is executable without human input, the agent must continue immediately to that next phase in the same run.

It must not stop after a successful phase closeout unless:
- the next phase is blocked
- the next phase requires human input
- the user explicitly asked to pause

### Protocol violation rule
The following are protocol violations:
- reporting an upcoming step instead of an executed phase
- skipping the explicit phase closeout
- advancing to the next phase without `PASS`
- giving only planning narration where executed evidence is required

If a protocol violation occurs, the agent must:
1. stop
2. acknowledge the violation
3. return to the last unclosed phase
4. execute it correctly

### No futurible rule during phased execution
When the guide requires phased execution, the agent must not answer with unexecuted future-tense progress such as:
- "I will..."
- "next I'll..."
- "the next step is..."

unless the current phase has already been executed and closed.

---

## A1. Environment health verification

The agent must verify:

- OpenClaw is reachable
- the gateway is healthy
- at least one secure channel is functioning
- Node is available
- npm is available
- git is available
- Codex CLI is available
- `codex app-server` is available
- Codex login is valid

### Minimum recommended sequence

```bash
openclaw doctor
openclaw status --deep
openclaw gateway status

node --version
npm --version
git --version
codex --help
codex login status
codex app-server --help
```

### PASS result
The technical base may continue deploying.

### Dependency completion rule
If a required base dependency is missing during Scope A, and:
- the dependency has a clear local installation path
- the corrective action is deterministic
- no additional human decision is required

then the agent must:
1. install or restore the missing dependency
2. re-run the affected verification checks
3. only then close the phase as `PASS` or `BLOCKED`

The agent must not stop early with `BLOCKED` if the only missing step is a straightforward local installation already implied by consuming the base.

This applies especially to:
- Codex CLI missing
- `codex app-server` unavailable only because Codex CLI is missing
- wrapper missing
- bootstrap missing
- required base directories missing

### BLOCKED result
The agent must stop only if:
- installation fails
- authentication fails and requires human action
- the runtime remains unavailable after installation
- a permission or environment error prevents continuation
- the next corrective step is not deterministic

---

## A2. Reusable global structure

The agent must ensure a reusable structure for the base.

At minimum, an equivalent structure must exist for:

```bash
mkdir -p ~/bin
mkdir -p ~/.openclaw/workspace
mkdir -p ~/.openclaw/workspace/docs
mkdir -p ~/.openclaw/workspace/projects
mkdir -p ~/.openclaw/workspace/templates
mkdir -p ~/.openclaw/workspace/templates/gsd-codex
mkdir -p ~/.openclaw/workspace/templates/gsd-codex/bootstrap
```

It must not delete existing content.

---

## A3. Support wrapper

The agent must deploy a stable support wrapper named `gsd-codex`.

### Stable path
- `~/bin/gsd-codex`

### Allowed responsibilities

The wrapper's role is limited to:

- `doctor`
- `status`
- `bootstrap`
- `install-local`
- `gsd-check`
- `codex`
- `exec`
- `logs`
- `where`

### Critical rule
The wrapper is not the GSD workflow. 
The wrapper cannot declare pure GSD by itself.

It may only declare:

- local GSD installed
- repo prepared
- skills present
- backend present

It may not declare:

- skill executed
- real GSD flow active
- GSD phase completed

### Wrapper code

The wrapper must be deployed as this baseline implementation or an explicitly versioned and validated equivalent implementation that preserves the same operational contract.
Write the following file to `~/bin/gsd-codex` and make it executable:

```bash
#!/usr/bin/env bash
set -euo pipefail

SELF_PATH="${BASH_SOURCE[0]}"
SELF_DIR="$(cd "$(dirname "$SELF_PATH")" && pwd)"

CODex_CONFIG_CONTENT='model = "gpt-5.4"
model_reasoning_effort = "high"
approval_policy = "never"
sandbox_mode = "workspace-write"

[sandbox_workspace_write]
network_access = true
'

usage() {
 cat <<'EOF'
Usage:
 gsd-codex doctor
 gsd-codex where
 gsd-codex status [project_path]
 gsd-codex bootstrap <project_path> [--force] [--backup]
 gsd-codex install-local <project_path>
 gsd-codex gsd-check <project_path>
 gsd-codex codex [args...]
 gsd-codex exec <project_path> <command...>
 gsd-codex logs [project_path]
EOF
}

need_cmd() {
 local cmd="$1"
 if ! command -v "$cmd" >/dev/null 2>&1; then
 echo "MISSING: $cmd"
 return 1
 fi
}

abs_path() {
 local target="$1"
 if [[ -d "$target" ]]; then
 (cd "$target" && pwd)
 else
 local dir
 dir="$(cd "$(dirname "$target")" && pwd)"
 echo "$dir/$(basename "$target")"
 fi
}

ensure_file() {
 local path="$1"
 local content="$2"
 local force="$3"
 local backup="$4"

 mkdir -p "$(dirname "$path")"

 if [[ -f "$path" && "$force" != "1" ]]; then
 echo "SKIP existing: $path"
 return 0
 fi

 if [[ -f "$path" && "$force" == "1" && "$backup" == "1" ]]; then
 cp "$path" "$path.bak"
 echo "BACKUP created: $path.bak"
 fi

 printf "%s" "$content" > "$path"
 echo "WROTE: $path"
}

doctor_cmd() {
 local rc=0

 echo "== gsd-codex doctor =="

 for cmd in bash node npm git codex; do
 if command -v "$cmd" >/dev/null 2>&1; then
 echo "OK: $cmd -> $(command -v "$cmd")"
 else
 echo "MISSING: $cmd"
 rc=1
 fi
 done

 if codex app-server --help >/dev/null 2>&1; then
 echo "OK: codex app-server"
 else
 echo "MISSING: codex app-server"
 rc=1
 fi

 if command -v openclaw >/dev/null 2>&1; then
 echo "OK: openclaw -> $(command -v openclaw)"
 else
 echo "WARN: openclaw not found in PATH"
 fi

 return "$rc"
}

where_cmd() {
 echo "wrapper_path=$SELF_PATH"
 echo "wrapper_dir=$SELF_DIR"
 echo "pwd=$(pwd)"
 echo "path=$PATH"
}

status_cmd() {
 local project="${1:-.}"
 project="$(abs_path "$project")"

 echo "== gsd-codex status =="
 echo "project=$project"

 if [[ -d "$project/.git" ]]; then
 echo "git_repo=yes"
 else
 echo "git_repo=no"
 fi

 if [[ -d "$project/.codex" ]]; then
 echo ".codex=yes"
 else
 echo ".codex=no"
 fi

 if [[ -f "$project/.codex/config.toml" ]]; then
 echo "codex_config=yes"
 else
 echo "codex_config=no"
 fi

 if compgen -G "$project/.codex/skills/*/SKILL.md" >/dev/null; then
 echo "skills=yes"
 find "$project/.codex/skills" -maxdepth 2 -name SKILL.md | sort
 else
 echo "skills=no"
 fi

 for f in AGENTS.md STATUS.md HANDOFF.md BLOCKERS.md; do
 if [[ -f "$project/$f" ]]; then
 echo "$f=yes"
 else
 echo "$f=no"
 fi
 done
}

bootstrap_cmd() {
 if [[ $# -lt 1 ]]; then
 usage
 exit 1
 fi

 local project="$1"
 shift || true

 local force=0
 local backup=0

 while [[ $# -gt 0 ]]; do
 case "$1" in
 --force) force=1 ;;
 --backup) backup=1 ;;
 *) echo "Unknown flag: $1"; exit 1 ;;
 esac
 shift
 done

 project="$(abs_path "$project")"
 mkdir -p "$project/.codex"

 ensure_file "$project/AGENTS.md" "# AGENTS

Objective:
Work on this project through small, verifiable, safe changes.

Rules:
- Use the repository as the primary source of truth.
- Read README, docs, STATUS.md, HANDOFF.md and BLOCKERS.md before changing code.
- Update STATUS.md and HANDOFF.md when a subphase is completed.
- Record real blockers in BLOCKERS.md.
- If this project uses GSD, use GSD as the real workflow.
" "$force" "$backup"

 ensure_file "$project/STATUS.md" "# STATUS

Project:
System active: base_codex
Workflow mode:
Phase:
Subphase:
Current goal:
Last real change:
Last validation:
Current state:
Next step:
Open risks:
" "$force" "$backup"

 ensure_file "$project/HANDOFF.md" "# HANDOFF

Last completed block:
Immediate pending:
Read first to resume:
Useful commands:
Notes:
" "$force" "$backup"

 ensure_file "$project/BLOCKERS.md" "# BLOCKERS

## Active

- [ ] No real blocker documented yet

## Resolved
" "$force" "$backup"

 ensure_file "$project/.codex/config.toml" "$CODex_CONFIG_CONTENT" "$force" "$backup"
}

install_local_cmd() {
 if [[ $# -lt 1 ]]; then
 usage
 exit 1
 fi

 local project
 project="$(abs_path "$1")"

 need_cmd node
 need_cmd npm

 cd "$project"
 npx get-shit-done-cc@latest --codex --local
}

gsd_check_cmd() {
 if [[ $# -lt 1 ]]; then
 usage
 exit 1
 fi

 local project
 project="$(abs_path "$1")"

 echo "== gsd-codex gsd-check =="
 echo "project=$project"

 [[ -d "$project/.codex" ]] || { echo "FAIL: missing .codex"; exit 1; }

 if compgen -G "$project/.codex/skills/*/SKILL.md" >/dev/null; then
 echo "OK: skills found"
 find "$project/.codex/skills" -maxdepth 2 -name SKILL.md | sort
 else
 echo "FAIL: no skills found under .codex/skills"
 exit 1
 fi

 if [[ -f "$project/.codex/config.toml" ]]; then
 echo "OK: codex config present"
 else
 echo "FAIL: missing .codex/config.toml"
 exit 1
 fi
}

codex_cmd() {
 exec codex "$@"
}

exec_cmd() {
 if [[ $# -lt 2 ]]; then
 usage
 exit 1
 fi

 local project
 project="$(abs_path "$1")"
 shift

 cd "$project"
 "$@"
}

logs_cmd() {
 local project="${1:-.}"
 project="$(abs_path "$project")"

 echo "== gsd-codex logs =="
 echo "project=$project"

 find "$project" -maxdepth 2 \( -name "*.log" -o -name "STATUS.md" -o -name "HANDOFF.md" -o -name "BLOCKERS.md" \) | sort
}

main() {
 if [[ $# -lt 1 ]]; then
 usage
 exit 1
 fi

 local cmd="$1"
 shift || true

 case "$cmd" in
 doctor) doctor_cmd "$@" ;;
 where) where_cmd "$@" ;;
 status) status_cmd "$@" ;;
 bootstrap) bootstrap_cmd "$@" ;;
 install-local) install_local_cmd "$@" ;;
 gsd-check) gsd_check_cmd "$@" ;;
 codex) codex_cmd "$@" ;;
 exec) exec_cmd "$@" ;;
 logs) logs_cmd "$@" ;;
 -h|--help|help) usage ;;
 *) echo "Unknown command: $cmd"; usage; exit 1 ;;
 esac
}

main "$@"
```

### Deployment of the wrapper

The agent must deploy it with a concrete sequence such as:

```bash
cat > ~/bin/gsd-codex <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
# ...full wrapper content exactly as defined above...
EOF

chmod +x ~/bin/gsd-codex
```

### Minimum wrapper validation

The agent must leave the wrapper operational and validate at least:

```bash
gsd-codex where
gsd-codex doctor
```

### PASS result
- the wrapper exists
- it is executable
- it is in `PATH`
- `where` works
- `doctor` works

---

## A4. Reusable bootstrap

The agent must deploy a reusable bootstrap that creates or preserves:

- `AGENTS.md`
- `STATUS.md`
- `HANDOFF.md`
- `BLOCKERS.md`
- `.codex/config.toml`

It must support:

- create if missing
- no overwrite by default
- `--force`
- `--backup`
- `--backup --force`

### Bootstrap rules
- if the file does not exist, create it
- if it exists and there is no `--force`, preserve it
- if `--backup --force` is used, create a backup before overwriting
- do not delete useful content without a backup

### Minimum bootstrap validation
The agent must test the bootstrap in:
- a case with no files
- a case with existing files
- a case with backup and overwrite

---

## A5. Minimum valid Codex configuration

When creating `.codex/config.toml`, the agent must write exactly this minimum valid configuration:

```toml
model = "gpt-5.4"
model_reasoning_effort = "high"
approval_policy = "never"
sandbox_mode = "workspace-write"

[sandbox_workspace_write]
network_access = true
```

It must not invent other sections as the default baseline unless the project requires them.

---

## A6. Codex automation bridge

The agent must validate the correct bridge:

- `codex app-server`
- `stdio` transport
- JSON-RPC flow
- capability for:
 - `initialize`
 - `initialized`
 - `thread/start`
 - `thread/resume`
 - `turn/start`

### Critical rule
The Codex TUI must not be used as an automation path.

The TUI must not be treated as an API. 
Bash, `gsd-tools.cjs`, and the wrapper must not be used as a substitute for the real Codex runtime.

### Connection rule
For each `stdio` connection to `codex app-server`, the agent must:

1. send `initialize`
2. send `initialized`
3. not repeat `initialize` on that same connection

---

## A7. Base deployment closeout

The agent must close this scope by reporting:

- system health
- channel health
- Codex
- `codex app-server`
- wrapper
- bootstrap
- shared paths
- blockers
- next step

Only after this is the base considered correctly consumed.

---

# SCOPE B. PROJECT EXECUTION

## B0. General rule

This scope performs actual project work.

There are two possible modes here:

- `codex_without_gsd`
- `codex_gsd_pure`

The agent must not mix modes without declaring the change.

---

## B1. Project activation

When the user says something like:

- I want to build app X
- I want to work on this repo
- prepare this project
- I want to start a new development project

the agent must enter this scope.

---

## B2. Target identification

The agent must determine whether the target is:

- an existing repo
- an existing directory to initialize
- a new project that needs a path

If minimum information is missing, it must ask only for what is necessary.

It must not invent paths.

### Minimum target verification

```bash
pwd
ls -la
git rev-parse --is-inside-work-tree
```

---

## B3. Confirm whether GSD is required

The agent must explicitly ask whether this project should use the GSD workflow.

### If the answer is no
It must leave the project in:

- `codex_without_gsd`

### If the answer is yes
It must prepare the project and pass through the pure GSD gate.

---

## B4. Project initialization

The agent must apply the safe bootstrap to the project and ensure:

- `AGENTS.md`
- `STATUS.md`
- `HANDOFF.md`
- `BLOCKERS.md`
- `.codex/config.toml`

### Minimum recommended content for live files

#### `AGENTS.md`

```md
# AGENTS

Objective:
Work on this project through small, verifiable, safe changes.

Rules:
- Use the repository as the primary source of truth.
- Read README, docs, STATUS.md, HANDOFF.md and BLOCKERS.md before changing code.
- Update STATUS.md and HANDOFF.md when a subphase is completed.
- Record real blockers in BLOCKERS.md.
- If this project uses GSD, use GSD as the real workflow.
```

#### `STATUS.md`

```md
# STATUS

Project:
System active: base_codex
Workflow mode:
Phase:
Subphase:
Current goal:
Last real change:
Last validation:
Current state:
Next step:
Open risks:
```

#### `HANDOFF.md`

```md
# HANDOFF

Last completed block:
Immediate pending:
Read first to resume:
Useful commands:
Notes:
```

#### `BLOCKERS.md`

```md
# BLOCKERS

## Active

- [ ] No real blocker documented yet

## Resolved
```

---

## B5. Normal GSD installation for Codex

If the project uses GSD, the agent must install GSD locally using its normal Codex installation inside the project.

### Normal local installation

```bash
cd /path/to/repo
npx get-shit-done-cc@latest --codex --local
```

### Minimum installation validation

The agent must check:

- presence of `.codex/`
- presence of installed GSD skills
- presence of the backend or local resources left by the installer

Example verification:

```bash
find .codex -maxdepth 3 -type f | sort
find .codex/skills -maxdepth 2 -name SKILL.md | sort
```

### PASS result
- local GSD is installed
- skills are present
- the project is ready to attempt the pure GSD gate

### BLOCKED result
- if local installation fails
- if the skills do not appear
- if the backend or required resources are missing

In that case:
- stop
- record the blocker
- do not declare the project ready for GSD

---

## B6. Pure GSD gate

To declare a project as:

- `codex_gsd_pure`

the agent must demonstrate all of the following:

1. it uses `codex app-server`
2. it uses `stdio`
3. it correctly completes:
 - `initialize`
 - `initialized`
 - `thread/start` or `thread/resume`
 - `turn/start`
4. it invokes a GSD skill explicitly in the turn text with a pattern such as:
 - `$gsd-help`
 - `$gsd-map-codebase`
 - `$gsd-new-project`
 - or another valid GSD skill for that point in the workflow
5. the turn reaches terminal state
6. there is a useful response attributable to the skill

### Initial validation rule
The first practical bridge validation may use a simple read-only skill, for example:

- `$gsd-help`

but that skill is not fixed as the only route in the system. 
It is only used as an initial proof that the bridge and turn consumption work correctly.

### Critical rule
The following must not be considered sufficient validation:

- typing into the Codex TUI
- using shell
- using `gsd-tools.cjs`
- using the wrapper as a substitute
- passing only a `type: "skill"` item as the sole input without a functional textual `$gsd-...` invocation

---

## B7. Bridge calling and response consumption rule

When the project operates in pure GSD mode, the agent must use this call pattern:

### Effective runtime configuration rule
The agent must not assume that `.codex/config.toml` is active only because the file exists on disk.

Before running write-capable GSD workflows, the agent must verify the effective runtime configuration returned by the real `codex app-server` session.

It must verify at least:
- effective approval policy
- effective sandbox mode
- effective working directory
- effective write capability for the project
- whether the project is trusted or project config is being ignored

If the effective runtime configuration does not match the required baseline for the intended workflow, the agent must correct it before continuing.

This includes, when needed:
- marking the project as trusted so project config is actually applied
- correcting effective sandbox mode to `workspace-write`
- correcting effective approval policy to the required baseline
- re-running the runtime verification after the correction

A write-capable GSD workflow must not start until this effective runtime verification passes.

### Per-connection handshake
1. `initialize`
2. `initialized`

### Thread management
3. `thread/start` for the first turn of the project or a new line of work
4. `thread/resume` for normal continuity of the same thread
5. `thread/fork` only if a separate work branch is needed

### Turn execution
6. `turn/start`

The actual work call happens in `turn/start`.

### Invocation format
The valid way to invoke a GSD skill is through explicit text inside the turn, for example:

```json
[
 {
 "type": "text",
 "text": "$gsd-help"
 }
]
```

or, depending on the current point in the workflow:

```json
[
 {
 "type": "text",
 "text": "$gsd-new-project"
 }
]
```

```json
[
 {
 "type": "text",
 "text": "$gsd-map-codebase"
 }
]
```

Explicit textual invocation is the operational reference for the system.

---

## B8. Turn waiting rule in `codex app-server`

In the OpenClaw ↔ Codex App Server integration, a turn is not considered finished at the first agent message.

The only valid turn completion is:

- `turn/completed`

### Events to listen for
During the turn, the agent must accumulate and process events such as:

- `turn/started`
- `item/started`
- `item/agentMessage/delta`
- `item/completed`
- `turn/completed`

### Accumulation rule
While the turn remains open, the agent must accumulate:

- all `item/agentMessage/delta`
- and, if present, the final content of `item/completed` with `type = "agentMessage"`

### What it must not do
The agent must not consider the turn finished in any of these cases:

- the first agent text
- an initial preamble
- the first user `item/completed`
- the first assistant `item/completed` if the turn is still open

### What it must do
The agent must:

1. keep the client open
2. listen until `turn/completed`
3. consider the turn complete only then
4. use the accumulated text up to that point as the final useful response

---

## B9. Context and continuity rule

It is important to manage context so that degradation does not occur.

### Thread rule
Each project must have thread continuity.

The agent must:
- save the project's operational `threadId`
- reuse `thread/resume` as the normal continuity path
- avoid opening new threads unnecessarily
- use `thread/start` only for real start points
- use `thread/fork` only when intentionally separating a work branch

### Minimum sufficient context rule
The agent must not resend the entire historical project context on every turn.

It must rely on:

- thread continuity
- `AGENTS.md`
- `STATUS.md`
- `HANDOFF.md`
- `BLOCKERS.md`

### No-degradation rule
If the thread starts accumulating too much noise or long low-value responses, the agent must:

1. consolidate the real state into `STATUS.md`
2. consolidate the short re-entry point into `HANDOFF.md`
3. keep live blockers in `BLOCKERS.md`
4. continue using those artifacts as the operational anchor

The agent must not compensate for context degradation by sending massive text dumps in each new turn.

---

## B10. GSD response interpretation rule

A final useful response from a GSD skill may fall into one of these groups:

### Type A. Executable next step for the agent
Examples:
- continue with a subsequent skill
- map the codebase
- initialize a project
- execute a specific phase
- verify a specific phase

In this case, the agent must:
1. record the result
2. execute the next valid step
3. keep `STATUS.md` updated
4. keep `HANDOFF.md` updated

### Type B. Question or request resolvable by the agent
Examples:
- information already present in `AGENTS.md`
- information already present in `STATUS.md`
- project context available in the repo or live artifacts
- objective answers derivable from the current target

In this case, the agent must:
1. resolve the question on its own
2. launch the next required turn
3. not escalate to the user if the answer is already recoverable with confidence

### Type C. Question or request not resolvable by the agent
Examples:
- business decision
- human preference
- priority among options
- external data not present in the project or live artifacts
- explicit confirmation required by the workflow

In this case, the agent must:
1. stop the affected phase
2. escalate immediately to the user
3. communicate the exact question or a brief faithful reformulation
4. include minimal context and sufficient evidence
5. not invent the answer
6. not continue until a valid answer is received

### Type D. Technical or functional blocker
Examples:
- skill with no useful response
- reproducible error
- broken bridge
- missing dependency
- internal flow contradiction
- no valid next step available

In this case, the operational blocker rule applies.

---

## B11. Execution rule in `codex_gsd_pure`

When the project is in pure GSD mode, the agent must work like this:

1. invoke the appropriate GSD skill through `codex app-server`
2. use explicit `$gsd-...` text
3. wait for terminal result
4. extract the useful response
5. classify the response as:
 - executable next step
 - question resolvable by the agent
 - question not resolvable by the agent
 - blocker
6. act according to that classification
7. update:
 - `STATUS.md`
 - `HANDOFF.md`
 - `BLOCKERS.md` if needed

### Critical rule
It must not:
- reinterpret the workflow on its own outside the GSD response
- replace the skill with shell
- manually recreate map/plan/execute
- use the TUI for automation
- invent answers to questions asked by GSD that the agent cannot resolve confidently

---

## B12. Execution rule in `codex_without_gsd`

When the project is in non-GSD mode, the agent must:

- work with Codex without GSD skills
- maintain real validation
- maintain state and handoff
- maintain blocker discipline

This must be made explicit in `STATUS.md`:

```md
Workflow mode: codex_without_gsd
```

---

## B13. Which files govern normal operation

Once the base is deployed and the project is initialized, the agent must operate by reading, in this order:

1. `AGENTS.md`
2. `STATUS.md`
3. `HANDOFF.md`
4. `BLOCKERS.md`

The SYSTEM must not be the primary source for every daily micro-decision. 
The SYSTEM governs:

- deployment
- initialization
- mode changes
- bridge repair
- framework rules

The project files govern:

- daily execution
- current state
- next step
- re-entry
- concrete blockers

---

## B14. Operational blocker rule

Whenever any real blocker appears, technical or functional, the agent must:

1. detect it and record it in `BLOCKERS.md`
2. communicate it immediately to the user with brief evidence
3. stop only the affected phase
4. not remain silent even if time passes
5. not retry indefinitely without communicating
6. not continue inventing steps outside the validated flow

A real blocker includes, among others:
- environment failure
- bridge failure
- skill with no useful response
- missing dependency
- reproducible error
- requirements contradiction
- pending human decision
- absence of a valid next step

Mandatory format:

```md
Blocker:
- blocker detected:
- phase affected:
- evidence:
- what was tried:
- what is needed:
- next step if unblocked:
```

---

## B15. Executed communication rule

The agent must not communicate intention as a substitute for execution.

It may only communicate one of these:

1. an already executed action with evidence
2. an already detected real blocker with evidence
3. a human decision required to continue
4. an already completed phase closeout

It must not announce:
- what it is going to do
- what it plans to try
- unexecuted working hypotheses
- empty progress
- review promises without evidence

Every communication must correspond to work already performed or a blocker already verified.

---

## B16. No progress simulation rule

The agent must not use intermediate messages to simulate activity.

If there is no:
- executed action
- new evidence
- real blocker
- or required human decision

then it must not send an update.

---

### B17. Useful completion rule for long GSD turns
For long-running GSD turns through `codex app-server`, commentary is not considered useful completion by itself.

If a skill invocation is expected to create or update concrete artifacts, the turn must end in one of these states only:
- artifacts created or updated, with evidence
- explicit blocker, with evidence
- explicit human decision required by the workflow

The agent must not treat commentary-only output, preambles, or descriptive progress text as a successful outcome.

If a GSD turn keeps producing commentary without artifact creation, blocker evidence, or decision need, the agent must:
1. treat the turn as degraded
2. interrupt or abandon that attempt after reasonable evidence of non-progress
3. report the degraded turn as a blocker or failed attempt
4. retry only with a stricter invocation or a corrected workflow assumption

### B18. Progress evidence rule for long GSD turns
When waiting on a long GSD turn, the agent must evaluate progress by material evidence, not by token stream volume.

Valid evidence of progress includes:
- expected files created
- expected files modified
- subagent completions
- required workflow state transitions
- a final useful response approaching terminal completion

Invalid evidence of progress includes:
- repeated commentary
- repeated restatement of intent
- repeated reading narration
- long descriptive text without artifact change

If material progress does not appear within a reasonable interval, the agent must stop waiting silently and surface the issue.

---

## 4. Stop conditions

The agent must stop if any of these occurs:

- there is no valid target
- Codex authentication is missing
- `codex app-server` is unavailable
- the wrapper is not operational
- the bootstrap is not operational
- the project cannot be initialized safely
- local GSD installation fails
- the pure GSD gate fails
- the next step cannot be determined safely

---

## 5. Final objective

The objective of this system is to:

- deploy a reusable base correctly
- clearly separate deployment from execution
- allow projects to be initialized consistently
- execute pure GSD only when it is actually validated through `codex app-server`
- prevent the agent from improvising or pretending a GSD workflow
- enable stable autonomous operation with well-defined project artifacts

When a project requires GSD, the system must enforce pure GSD through `codex app-server`, with explicit `$gsd-...` invocation, waiting until `turn/completed`, correct consumption of the final response, thread continuity, and immediate escalation to the user of any question or decision the agent cannot resolve on its own.
