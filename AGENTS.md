<!-- BEGIN ARBITE INSTRUCTIONS -->
# Arbite Ticketing System

## Docs

Arbite documents itself from the command line: run `arbite docs` for the overview,
`arbite docs list` for every topic, and `arbite docs commands <name>` for a command's
flags. This block is the short version; `arbite docs` is the source of truth.

## Ticketing
Use arbite for all tasks. Create a ticket if required and claim it before starting work. Tickets live under `.arbite/` in state folders (`open/`, `in_progress/`, `review/`, `blocked/`, `shelved/`, `closed/YYYY-MM/`); `wishlist/` and `plans/` are buckets, not statuses, and `raw/processed/` holds frozen snapshots of promoted captures. The store and the review behaviour come from `.arbite/project.yaml` (`sink:`, `review:`).

# Agent Identity

When claiming a ticket, use an identity like "claude.opus-5.001" -- company.model.instance, your best educated guess unless told otherwise. If orchestrating, tell subagents their identity and instance number.

## Sole command: "Work Next|All <epic>"

```bash
arbite list next                      # next workable ticket
arbite list --topo --status open [--epic <epic>]
arbite list next --recursive          # the same queue swept across every project beneath this one
arbite list --topo --status open --include DIR   # a named set of projects, one report each
```

If there are no tickets, use "Classify". If "Work All", orchestrate if that is in your skill set, otherwise work in sequence until finished. A sweep reads; `list next --claim` is the one write it makes, landing each claim in the project that owns the ticket. Each row names its `project` (ids are minted per store, so `(project, id)` identifies a ticket), and you work a ticket in its own project with `--root <project>`.

## Sole command "Classify"

```bash
arbite list raw
arbite list raw --recursive           # every project's raw backlog, grouped by project
arbite fetch --recursive              # the oldest raw capture across the project set
```

Classify every raw ticket with `arbite promote <id> --title ... --tier ... --domain ...` (add `--description`/`--epic`/`--priority`/`--tags` as you learn more). Add `--agent <your-id>` to classify and claim one you will work yourself; a wish is reclassified and filed in the wishlist bucket instead of being opened. `promote` writes one project, so it takes no aggregate flags: run it as `arbite --root <project> promote <id> ...`, with `<project>` from the row you fetched.

## Workflow: claim -> in_progress -> submit -> review -> accept

```bash
arbite claim <id> --agent <your-id>        # take it: status -> in_progress
arbite stream write <id> -                 # narrate as you work (piped from your output)
arbite note <id> <your-id> "what changed"  # log progress as you go
arbite submit <id>                         # finish: status -> review, assignee kept
arbite accept <id> --agent <reviewer-id>   # the reviewer closes it, credited to them
```

Send work back with `arbite reopen <id> --reason "<why>"` (the reason is required -- it is the only record of what the author must fix). With `review: false` in `.arbite/project.yaml`, `arbite submit` closes the ticket directly instead of parking it in review.

# Ticketing etiquette addendum

When closing a ticket, add a note paragraph explaining what the user, QA, or other agents will be able to observe via integration testing (and any new effects that will be observable).

## Git
By default and unless otherwise specified, check into main/master after closing a ticket.

Do not modify this file without explicit permission.

## Comment Protocol

Do not fill the codebase with comments containing history, musings, or overly wordy explanations. Comments should be maximally useful and concise. Put long explanations and related context in Arbite tickets, and refer to the relevant ticket from a code comment when needed.
<!-- END ARBITE INSTRUCTIONS -->

<!-- QFN DELEGATION -->
# Delegating tickets to local QFN workers

A local Qwen model ("QFN") is served through LiteLLM at http://127.0.0.1:4000/v1
(provider `litellm`). The model name picks the machine: `radiance` runs 3-4 requests at
once; `strata` runs one at a time and is 2-4x slower (same model, 131072-token window).
Radiance's own server on :8000 (provider `local_radiance`) still answers.
`carnot do --worker` runs QFN as a headless coding agent; `omp` is the fallback, see
below. The orchestrator is a metered model: it dispatches and reviews, and QFN does the
work. Save orchestrator tokens, not wall time.

- Model: with `pools` set, each pool's `model` is the `--model` its workers start with
  (`litellm:radiance`). Without `pools` the worker uses `active.provider` and its model
  from `config/config.yaml`. `carnot do --model PROVIDER:MODEL` overrides both; a bare
  name is matched like `/set-model` and the chosen pair is printed.
- Concurrency: the `pools` in `.carnot/orchestrate.yaml` set it.
  `dispatch <id> "<tests>" --pool NAME` picks a pool; without it the first pool with a
  free slot is used. Keep one machine at 3 workers or fewer. Strata has proven
  unreliable as a worker: give it no slots.
- This repo has no `.carnot/orchestrate.yaml` and no test suite yet, so the loop runs
  one pool -- the active provider, 3 slots -- and gates on `python -m pytest -q`. Add
  the file once the project has tests:

      worker: qwen.flash-next
      test: "python -m pytest -q"   # the suite's own command; test files are added after it
      pools:
        - {name: radiance, model: 'litellm:radiance', slots: 3}

- Identities: orchestrator `claude.opus-5.001` (or your own). Workers
  `qwen.flash-next.001` to `.003`: one number for each parallel slot.
- Test environment: one shared venv at `$MAIN/.venv` (the main checkout), holding the
  dependencies and pytest. If it is missing, make it in `$MAIN`. A worktree has no
  `.env`, and the tests must not need one. Where config files are not in git, the
  worktree gets the `.example` files copied to their plain names --
  `orchestrate dispatch` does this, and never copies a `.env`.
- Make the suite parallel once it is big enough (`-n 4`, not `-n auto`: the workers'
  gates overlap, and `-n auto` lets each run take every core). The gate of a worker is
  the full suite; give the ticket's own test files for its fast inner loop.
- Main is the reference: keep it green, so a red gate is a regression and not something
  a known-failure list excuses. A red that appears only in a full run is
  order-dependence, not flakiness: run that file alone.
- A worktree is not the main checkout: it has no git-ignored directories. A test that
  reads a directory listing can pass in a worktree and fail in `$MAIN`, which is why
  `carnot orchestrate full` runs in `$MAIN`.
- Branch: the main branch of this repo is `main`.

## Orchestrator: "Work All" loop

The loop is built into carnot: one `orchestrate` tool with an `action` (`status`,
`dispatch`, `review`, `accept`, `reject`, `full`, `health`, and the CLI-only `wait`),
and the same verbs on the CLI, `carnot orchestrate <action>`. carnot runs it itself when
told "self orchestrate" (the `orchestrate` skill); another orchestrator uses the CLI.
Each verb checks its state and refuses a wrong step with the reason: trust the refusal,
do not work around it. This repo's settings are in `.carnot/orchestrate.yaml` (worker
name, test command); the skill's `reference.md` has every setting, file and exit code.

0. **Preconditions (once per run).** `carnot orchestrate health` prints ok for each
   pool. Main has no uncommitted changes outside `.arbite/`.
   `carnot orchestrate full` passes. Read `.arbite/AGENTS.md` once per session.
1. **Resume from arbite, not memory.** `carnot orchestrate status` lists what is in
   progress or in review, the running workers and the free slots. Finish those
   before new work. Do this again after a context compaction.
2. **Pick work.** Up to one ticket per free slot from "next workable". Not two at once
   that edit the same function or file heavily; new modules in parallel are fine.
3. **Route by tier.** low/medium go to QFN. high goes to QFN only if tests define
   "done" (or you make them so in step 4); if not, work it yourself. frontier: work it
   yourself, or `arbite block <id> --reason "needs human"`.
4. **Make the ticket dispatchable.** Acceptance criteria, where the code is, the test
   files: add what is missing with `arbite note`. For a design-heavy ticket, write
   `.arbite/plans/<id>-spec.md` (API, rules, codes, required tests), `arbite ref add` it
   and commit it before the dispatch. QFN implements specs well but must not invent
   semantics.
5. **Dispatch:** `carnot orchestrate dispatch <id> "<test files>"` (tool:
   `action: dispatch`; a retry adds the rejection reason after the files). It claims the
   ticket for the next free slot (`<worker>.001` to `.003`), makes the worktree
   `~/.cache/worktrees/<project>/<id>` on branch `<id>` with the example configs, writes
   the task, and starts `carnot do --worker` from the pinned harness with three gates
   (full suite, clean tree, ticket in review). It returns at once. **End your turn**:
   the harness wakes the session with the worker's receipt. Do not poll, sleep or tail
   logs.
   - From the CLI (`carnot orchestrate dispatch`) there is no session wake: run
     `carnot orchestrate wait` in the background; it returns when a worker ends.
   - Workers run from the installed carnot. (Only the carnot repo pins a harness: a
     detached worktree of the base branch, refreshed at a dispatch when no worker runs.)
   - Exit codes of a run: 0 done, 3 blocked, 4 step budget, 5 time limit, 7 gate
     failed, 8 stalled, 9 stopped, 1 carnot or provider error.
   - Watch or steer a running worker from the TUI (Left twice on an empty prompt, or
     `/workers`).
6. **Review:** `carnot orchestrate review <id>`. It merges the base into the branch
   (resolve a conflict there, commit, review again), runs the full suite in the
   worktree, and prints the run's digest and the diff stat. Read only the diff hunks you
   need; probe what the spec and tests do not cover with a short scratch script (with
   `ORCH_ROOT` pointed at a scratch directory).
7. **Decide.**
   - Accept: `carnot orchestrate accept <id> "<what you checked; what a user can now
     observe>"`. It refuses unless the last review passed at the current commit; it
     merges, closes the ticket, logs it in `.arbite/agents/<me>.md`, commits
     `.arbite/`, and removes the worktree and the branch.
   - Small defect (a few lines): fix it in the worktree, commit, review again, accept.
   - If not: `carnot orchestrate reject <id> "<file:line, failing output, what to
     change>"`, then dispatch again with the same reason. The second rejection blocks
     the ticket: move on.
8. After every third accept and at the end, `carnot orchestrate full` in the main
   checkout (main is green, so a failure there is a merge's). Stop when status shows no
   workable ticket and no running worker, and report the run.

### Fallback: dispatch with omp

Use this when carnot itself cannot run a worker (a ticket that broke the runner).
Claim the ticket and make the worktree by hand
(`WT_ROOT=~/.cache/worktrees/<project>`, `WT=$WT_ROOT/<id>`), then run:

       printf 'bash:\n  autoBackground:\n    enabled: false\n    thresholdMs: 900000\n' > "$WT_ROOT/qfn-overlay.yml"
       CARNOT_HOME="$WT_ROOT/carnot-home" \
       omp -p --session-dir "$WT_ROOT/<id>.session" \
         --model radiance/swift1.5-qwen3.8-next-flash-fp8-iq4r-moe --cwd "$WT" \
         --tools read,grep,glob,edit,write,bash --approval-mode yolo --max-time 60m \
         --config "$WT_ROOT/qfn-overlay.yml" \
         "<the task text>" < /dev/null > "$WT_ROOT/<id>.log" 2>&1   # without </dev/null omp can hang

Without the overlay, omp puts a test run in the background after 60 seconds and the
worker exits before it commits. omp has no gates and no result record: check the ticket
status, the log tail and the tests yourself.

## QFN worker rules (if your identity is qwen.*)

- Your project root for arbite is the `--root` path in your task. Always pass it
  (`arbite --root <MAIN> ...`). Never use the `.arbite/` copy in your worktree.
- Work only in your worktree. Do not touch `~/.omp` or anything outside the worktree.
  Do not change dependency or config files (`pyproject.toml`, a lockfile,
  `config/config.yaml`, `config/providers.yaml`) unless the ticket tells you to.
- Scratch files (probe scripts, notes, temporary output) go in `$CARNOT_SCRATCH`,
  never in the worktree.
- Read `CONVENTIONS.md` first, then the ticket and any spec it references.
- Implement the ticket, run the test command from your task, and commit to your branch
  with a message that starts with `<id>:`. The test files of the ticket must pass. Do
  not try to repair a failure that the ticket does not mention.
- Run the test command in the foreground and wait for it to finish. Do not put a
  command in the background.
- Finish with `arbite --root <MAIN> note <id> <your identity> "<what changed; test
  command and result; what a user/QA can now observe>"`, then
  `arbite --root <MAIN> submit <id>`, then call `finish` with status `done`.
- If you are blocked or cannot meet the acceptance criteria, add a note that says why
  and call `finish` with status `blocked`. Do not submit unfinished work.
- The `finish` summary: at most 10 lines (status, files changed, test command and result).
<!-- END QFN DELEGATION -->
