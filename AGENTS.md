# Instructions for the coding agent

You are building the Code Checker app in this repository, start to finish, without a human
in the loop.

## Files that direct you

- `SPEC.md`: what to build. It is the source of truth.
- `PLAN.md`: the order to build it in, with a "Done when" list per step.
- `DECISIONS.md`: where you record assumptions.

## How to work

1. Read SPEC.md and PLAN.md completely before writing code.
2. Follow PLAN.md step by step. Do not skip ahead. Tick each checkbox when it is true.
3. Do not stop to ask questions. When something is unclear or missing:
   - pick the simplest option that fits SPEC.md,
   - add an entry to DECISIONS.md (what was unclear, what you chose, why),
   - keep going.
4. Stop and ask only if continuing would delete data outside this repository, needs a secret
   or account you do not have, or a required tool (Docker, Node, Python) is not installed.
5. Write tests alongside the code. For step 2 (grading and scoring) write the tests first.
6. After every change, run the relevant tests. Never tick a "Done when" box with failing
   tests. If a test fails, fix the code, not the test, unless the test contradicts SPEC.md.
7. Commit at the end of each step: `step N: <title>`. Do not push.
8. Do not build anything listed as out of scope or v2 in SPEC.md. Do not add features that
   are not in SPEC.md.

## Commands

Everything runs through Docker so it behaves the same on Windows.

```
docker compose up -d --build            # start everything
docker compose exec api pytest          # backend tests
docker compose exec api alembic upgrade head
docker compose logs -f api worker       # debugging
cd frontend && npm install && npm run lint && npm run build && npm test
cd frontend && npx playwright test      # end-to-end, needs the stack running
```

## Hard rules (never break these)

- Student code runs only inside the sandbox container, with the exact flags in SPEC.md
  section 7. Never `exec`, `eval` or import student code in the api or worker process.
- Expected values and hidden tests never enter the sandbox and never reach a student's
  browser. Comparison happens in the worker.
- The server is the only authority on strikes, timers, attempts and scores. Never trust a
  count or a time sent by the browser.
- Every session endpoint checks that the session belongs to the logged-in student.
- Scoring lives in one pure module with no database access, fully unit-tested.
- Use server time (UTC) for all decisions. Store timestamps with time zone.
- No secrets in the repository. Configuration comes from `.env`; keep `.env.example` current.
- All user-visible frontend strings go through i18next. Write en first, then uz and ru.

## Conventions

- Backend: Python 3.13, FastAPI, async SQLAlchemy 2, Alembic, Pydantic v2, pytest, ruff.
- Frontend: React, Vite, TypeScript (strict), CodeMirror 6, i18next, vitest, Playwright.
- Keep it simple: no microservices, no extra infrastructure beyond what SPEC.md section 8
  lists. Plain polling, no WebSockets.
- Plain, readable UI. No design system needed; consistent spacing and clear labels are enough.

## When you finish

Write HANDOVER.md as described in PLAN.md step 7, and make sure DECISIONS.md is complete.
