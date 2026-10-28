# Agent Todo List

## In progress

- [ ] Create GitHub repo `AliNikkhah2001/agent-state` and push local agent-state submodule to it
- [ ] Configure GitHub branch protection rules on all repos:
    - [ ] Work_RAG-KB (master, develop)
    - [ ] Work_Credit-RAG_Phase1 (main, develop)
    - [ ] Work_RAG-Orchestrator (main)
    - [ ] Work_RAG-Guardrails (main)

## Pending

- [ ] Run draft supplement tests against live stack (KB :8000, orchestrator :8100)
- [ ] Verify all submodule gitlinks are correct (`git submodule status --recursive`)
- [ ] Document any remaining debugging sessions in `history/archive/`

## Completed (this session)

- [x] Remove `vast-gemma4-migration` branch from all repos (local + remote)
- [ ] Delete `legacy/*` branches from all repos
- [x] Create `develop` branch on all repos and push to origin
- [x] Create agent-state submodule with README, architecture, coding-standards, git-workflow, insights, FILE_MAP
- [x] Copy agent context, todo, and archived sessions into agent-state/
- [x] Push Work_RAG-KB master + develop to origin
- [x] Push knowledgebase submodule master + develop to origin
- [x] Push Work_Credit-RAG_Phase1 main + develop (with agent-state gitlink) to origin

## Code style reminders

- Use conventional commit prefixes: `feat:`, `fix:`, `docs:`, `chore:`
- Type hints on all public functions
- `try/except` around external calls (HTTP, DB, model)
- `async with db.session() as session:` for DB operations
- `httpx.AsyncClient(timeout=120, trust_env=False)` for localhost calls
- No `pkill python` — use `bash deploy/vast/stop.sh`
