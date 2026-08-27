# Fleet Control-Plane Skill Specifications

Narrow proposals only. Each spec is additive: it should plug into the
existing kanban/kernel primitives already present in `hermes_cli/kanban_db.py`
and `cron/scheduler*.py`, not replace them.

## 1. fleet-router
**Trigger:** dispatcher has multiple candidate workers / profiles / providers for one task.  
**Behavior:** rank candidates by task `assignee`, `skills`, `model_override`,
`provider_override`, tenant/project scope, and current load. Return an ordered
list with explicit reject reasons.  
**Hard guard:** never route to a profile whose runtime is missing or blocked by
`lifecycle_guard.py`. Never inject inbound network binds.

## 2. task-contract
**Trigger:** `kanban_create` / worker context build.  
**Behavior:** emit a deterministic JSON blob carrying the contract fields already
supported by `kanban_db.Task`:
`task_id`, `parent`, `owner`, `capability`, `inputs`,
`immutable_constraints`, `resource_locks`, `dependencies`,
`acceptance_criteria`, `expected_artifacts`, `completion_evidence`,
`escalation`, `retry_budget`, `status`.  
**Hard guard:** `immutable_constraints` is advisory-only in code; the skill
must not mutate `config.yaml`, `SOUL.md`, memory, skills, cron, session,
vault, or Herdr files.

## 3. task-decomposer
**Trigger:** task enters `triage` with `goal_mode=True` or explicit
`kanban_decompose` request.  
**Behavior:** read task body/title, parent/child links, assignee roster, and
`capability_matrix` outputs; emit child tasks with `parents`, `assignee`,
`skills`, and acceptance criteria.  
**Hard guard:** one writer per child worktree; respect existing
`lock:repo:hermes-agent` semantics by not mutating code paths outside the
assigned worktree.

## 4. dependency-planner
**Trigger:** `link_tasks` or `parents=[...]` on create.  
**Behavior:** validate no cycles, no duplicate links, max fan-in/fan-out if
configured. Surface the plan in `task_events` as `dependency_plan`.  
**Hard guard:** never auto-create hidden cross-tenant links; preserve explicit
board tenancy.

## 5. capability-matrix
**Trigger:** dispatcher / decomposer / router.  
**Behavior:** static manifest mapping profile -> toolsets, skills, runtime
constraints, forbidden paths. Derived from profile config + `toolsets` +
`VALID_WORKSPACE_KINDS`. Cache under `kanban` board metadata; invalidate on
profile/skills change.  
**Hard guard:** do not write profile config or skills; read-only.

## 6. handoff-validator
**Trigger:** `kanban_complete`, review approval, parent completion.  
**Behavior:** enforce the same checks already present in `complete_task`:
phantom created-card detection, parent-satisfied gate, artifact existence
advisory, and summary/result metadata preservation.  
**Hard guard:** never mutate task state on validation failure; emit
`completion_blocked_hallucination` / `suspected_hallucinated_references`
events only.

## 7. acceptance-evidence
**Trigger:** goal-mode judge, reviewer, or completion.  
**Behavior:** treat task body acceptance criteria + metadata `tests_run`,
`changed_files`, and `artifacts` as the evidence set. Fail closed when
required fields are missing.  
**Hard guard:** do not auto-generate fake evidence; missing evidence blocks
completion, it does not get synthesized.

## 8. retry-and-circuit-breaker
**Trigger:** task failure paths in `cron/scheduler.py` and `kanban_db.py`.  
**Behavior:** reuse the existing `consecutive_failures`, `max_retries`,
`_record_task_failure`, and `_backoff_wait_seconds` primitives. Add an
exponential backoff summary to `task_runs.metadata` so operators can see
the retry shape.  
**Hard guard:** do not remove or weaken the existing failure counter or
auto-block transition; only extend observability.

## 9. human-escalation
**Trigger:** `block_kind in {needs_input, capability}`, unblock recurrence
limit hit, circuit-breaker trip, or explicit `escalation` field in body.  
**Behavior:** route to `blocked` -> `triage` per existing `block_task` /
`BLOCK_RECURRENCE_LIMIT` semantics; notify subscribers via
`kanban_notify_subs`.  
**Hard guard:** never auto-approve blocked tasks; human decision required.

## 10. memory-promotion
**Trigger:** task completion with high-value handoff metadata.  
**Behavior:** promote stable knowledge from run `metadata` and task comments
into shared memory/skills only when explicitly opted in. Default: no-op.  
**Hard guard:** never write vault, skills, or profile memory without an
explicit opt-in flag; respect cross-profile write guard.

## 11. remote-routing
**Trigger:** task assigned to a profile on another machine / docker / modal.  
**Behavior:** emit a portable task handoff packet referencing absolute paths
and board/task ids; do not assume same-machine filesystem access.  
**Hard guard:** no inbound A2A or port binds from this skill; routing is
outbound via existing gateway/dispatcher.

## 12. fleet-retrospective
**Trigger:** board archive, project completion, or periodic cron.  
**Behavior:** aggregate `task_events`, `task_runs`, and retry/circuit-breaker
metrics into a compact report.  
**Hard guard:** read-only; never mutate tasks, runs, or config during
retrospective.
