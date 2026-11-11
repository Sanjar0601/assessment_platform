# Build plan

Work through the steps in order. Do not start a step until the previous step's "Done when"
list is fully true. Tick the boxes as you go and commit at the end of every step with the
message `step N: <title>`.

Read SPEC.md fully before step 0. Section numbers below refer to SPEC.md.

## Repo layout

```
backend/
  app/            FastAPI app: api/, models/, services/, schemas/, core/
  worker/         Celery app, grading tasks, beat schedule
  sandbox/        Dockerfile + runner.py for the sandbox image
  alembic/
  tests/
frontend/         React + Vite + TypeScript
deploy/           nginx config
samples/          example tasks, assessment and students.csv
docker-compose.yml
.env.example
README.md
```

## Step 0: Scaffold

- [ ] Create the layout above, `docker-compose.yml` (api, worker, beat, postgres, rabbitmq,
      nginx), `.env.example`, backend `pyproject.toml`, frontend Vite project.
- [ ] `GET /api/health` returns ok and checks the database connection.
- [ ] pytest runs inside the api container against a separate test database.
- [ ] README.md: how to start, stop, run tests, and create the first admin.

Done when:
- [ ] `docker compose up -d --build` starts every service with no errors.
- [ ] `docker compose exec api pytest` passes (one health test).

## Step 1: Database

- [ ] SQLAlchemy models for every table in section 9, with the unique constraints.
- [ ] One Alembic migration creating all of them.
- [ ] CLI command `python -m app.cli create-admin <email> <password>`.
- [ ] CLI command `python -m app.cli seed` loading everything in `samples/`.
- [ ] `samples/`: 4 algorithm tasks, 3 theoretical tasks (at least one using stdin, one using
      AST rules, one with `unordered` compare), `students.csv` with 2 groups of 6 students.

Done when:
- [ ] `alembic upgrade head` works on an empty database, and `alembic downgrade base` works.
- [ ] Model tests pass (create one of each, constraints reject duplicates).

## Step 2: Sandbox and grading (most important step)

- [ ] `backend/sandbox`: image from `python:3.13-slim` with `runner.py` (section 7).
- [ ] Worker function `run_in_sandbox(payload) -> results` using the exact container flags.
- [ ] Comparison modes: exact, unordered, float.
- [ ] AST checks (section 4) without executing code.
- [ ] Pure scoring module, no database access: limited attempts, time bands, retake cap,
      assessment average (section 5).
- [ ] Celery tasks `grade_run` and `grade_submit` that store a submission and update
      attempts_used, locked and score.

Done when, with tests for each line:
- [ ] Correct solution passes all tests; wrong solution gets the right passed count.
- [ ] Infinite loop in one test fails only that test.
- [ ] Student `print()` calls do not affect results.
- [ ] Syntax error, runtime error, missing function each give a clean failure with a message.
- [ ] Code cannot reach the network, and cannot write outside /tmp.
- [ ] A fork bomb and a 1 GB allocation are contained and reported as failures.
- [ ] `input()` in a theoretical task reads the task's stdin.
- [ ] Scoring: every formula in section 5, including the worked example, boundary values of
      each time band, floor at 0, retake cap, best-of-submissions.
- [ ] Each AST rule passes and fails on purpose-written snippets.

## Step 3: Student API and session logic

- [ ] Student login, single session, cookie resume, login log.
- [ ] Start session: shuffle, variant choice, snapshot use.
- [ ] Session endpoints from section 10: open, draft, run, submit, jobs, finish, result.
- [ ] Feedback rules (section 4) and response filtering: no hidden tests, no expected values.
- [ ] Events and heartbeat: debounce, strike settings, away strikes, flags, termination.
- [ ] Beat job every 30 s: timer expiry, auto-submit, heartbeat gap flags, away strikes.
- [ ] Pause handling: no strikes, clocks frozen.
- [ ] Rate limit on Run; code size limit.

Done when, with tests for each line:
- [ ] Second login is refused with 409; same cookie resumes.
- [ ] Student A gets 403/404 on student B's session.
- [ ] No student response contains hidden test args or any expected value.
- [ ] fullscreen_exit + blur + tab_hidden within 2 s = exactly one strike.
- [ ] Third strike terminates, auto-submits and grades; result is Fail with a real score.
- [ ] `blur_short` creates a flag and no strike.
- [ ] 30 s of heartbeats with focused=false adds a strike; none while paused.
- [ ] Expired timer closes the session without any browser request.
- [ ] Limited attempts are enforced; a locked task refuses submissions.
- [ ] Scores are hidden until the assessment is closed.

## Step 4: Staff API

- [ ] Staff login, roles, admin staff management.
- [ ] Task bank CRUD, JSON import with validation errors that name the field.
- [ ] Courses, groups, CSV import, assessments, slots, variants, snapshots.
- [ ] Live board, pause, resume, extend, close, restore, terminate, release, retake.
- [ ] Results table, session detail, violation log, Excel export (openpyxl).

Done when, with tests for each line:
- [ ] A teacher cannot see another teacher's course results; admin can.
- [ ] Editing a bank task after adding it to an assessment does not change the snapshot.
- [ ] Restore requires a reason, resets strikes to 0, keeps old events, applies a new timer.
- [ ] Extend adds minutes to every active session.
- [ ] Retake result is capped at 70.
- [ ] A theoretical task with no expected_stdout and no AST rule is rejected.
- [ ] The Excel file opens with openpyxl and has one row per student.

## Step 5: Student frontend

- [ ] Browser check, login, assessment list, pre-start page with the rules and a Start button
      that enters fullscreen.
- [ ] Working screen (section 11) with CodeMirror, Python highlighting, autosave.
- [ ] Guard module: all event types in section 6, blur 1.5 s timing, paste/copy/cut and
      right-click blocked, heartbeat every 5 s, overlay, paused screen.
- [ ] Job polling, feedback panel, termination screen, results window.
- [ ] i18next with en, uz, ru files and a switcher. All UI strings go through it.

Done when:
- [ ] `npm run build` and `npm run lint` pass with no errors.
- [ ] Unit tests (vitest) for the guard module pass: event mapping and blur timing.
- [ ] One Playwright test passes against the running stack: log in, start, submit a correct
      sample solution, see "passed" feedback.

## Step 6: Staff frontend

- [ ] Login, task bank with form editor and JSON import, assessment builder.
- [ ] Live board (polls every 5 s) with all buttons; restore dialog requires a reason.
- [ ] Results table with export, session detail, violation log.
- [ ] Admin pages: staff, groups with CSV import, courses.

Done when:
- [ ] `npm run build` and `npm run lint` pass.
- [ ] One Playwright test passes: teacher creates an assessment from sample tasks, opens it,
      a student completes it, teacher closes it and sees the score in the results table.

## Step 7: Finish

- [ ] `backend/tests/load_test.py`: 35 simulated students each submit at the same moment;
      print the slowest grading time.
- [ ] README.md final: setup on Windows with Docker Desktop, firewall port, first admin,
      seeding samples, running a lesson, backing up the database (`pg_dump` command).
- [ ] Write HANDOVER.md: what was built, anything in SPEC.md not done and why, and a manual
      test list for a human on real laptops (fullscreen exit, Alt+Tab, window snap, paste,
      right-click, notification popup, browser crash and resume).

Done when:
- [ ] Full test suite passes from a clean `docker compose down -v && docker compose up -d --build`.
- [ ] Load test finishes with every submission graded.
- [ ] DECISIONS.md lists every assumption made along the way.
