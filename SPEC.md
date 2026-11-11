# Code Checker — Product Spec v1

This file is the single source of truth. If code and this file disagree, this file wins.

## 1. Overview

A web app where students solve Python tasks in a fullscreen browser session. Code is graded
automatically in a sandbox. Teachers build assessments from a shared task bank and review
results and violation logs.

- Used for: class works, quizzes, midterms.
- Users: admin, teachers, students. Peak load: 35 students at once.
- Setting: classroom, students' own laptops, teacher present, all rules announced in advance.
- Deployment: stage 1 runs on the teacher's Windows laptop (Docker Desktop + WSL2) over the
  classroom Wi-Fi. Stage 2 moves to a VPS. Everything runs through `docker compose`.
- Browsers: Chrome or Edge only. Other browsers get a "use Chrome or Edge" page.
- Interface languages: Uzbek, English, Russian with a switcher. Task text is not translated.

Explicitly out of scope: second-monitor detection, PINs or passwords for students, webcam
photos, device binding, plagiarism detection (v2).

## 2. Roles and structure

| Role | Can do |
| --- | --- |
| Admin | Manage teachers, groups, courses; import student CSVs; see everything |
| Teacher | Create tasks (form or JSON), build assessments, run sessions, restore and release sessions, see results of own courses, use the shared task bank |
| Student | Log in, take assessments, see own results after an assessment closes |

- Group (e.g. DT-23-01) holds Students. A student belongs to one group.
- Course (subject, owner teacher) enrols one or more groups. A group can be in several courses.
- Assessment belongs to a course: kind (classwork, quiz, midterm), optional timer in minutes,
  opens_at, closes_at, shuffle on/off, strike settings (section 6).
- Slot is one position in an assessment and holds 1+ task variants. Each student gets one
  random variant per slot. Slot order is shuffled per student when shuffle is on.
- Task bank is shared by all teachers. Adding a task to an assessment copies it into a frozen
  snapshot. Editing the bank never changes past or running assessments.
- Retake: a teacher reopens an assessment for selected students. Retake score = min(score, 70).

## 3. Login and identity

- Staff (admin, teacher): email + password.
- Students: pick group from a dropdown, then name from a dropdown. No password.
- Students are imported from CSV with columns `full_name,group`. (full_name, group) is unique.
- One active session per student. A second login while a session is active is blocked (it
  does not kick out the first). A teacher frees the account with Release.
- Resume: the session is tied to an httpOnly cookie. Reopening the site in the same browser
  continues the session without the teacher.
- Every login attempt is logged: time, IP, user agent, result.
- Before start the browser checks: Chrome or Edge, and fullscreen entered.

## 4. Task types

| | Algorithm | Theoretical |
| --- | --- | --- |
| Student writes | Body of a given function | A program from scratch |
| Check | Call the function with each test's args, compare with expected | Exit code 0 and expected stdout (with optional stdin); AST rules are optional extras |
| Comparison | `exact` (default), `unordered`, or `float` with tolerance | Stdout compared after stripping trailing whitespace on each line and at the end |
| Visible tests | 1-3, shown with args and expected | None |
| Run button | Visible tests only; free; max one run per 5 s | None: every run is an attempt |
| Submit | All tests; counts as an attempt | Counts as an attempt |
| Partial credit | passed / total | Pass or fail |

A theoretical task must have `expected_stdout` or at least one AST rule. Reject tasks with neither.

AST rules: `defines_function` (name), `defines_class` (name), `uses_loop`, `uses_construct`
(one of: try, with, list_comprehension, lambda, recursion), `min_lines` (value).

Per task: attempt mode (`limited` with max N, or `unlimited` with time bands best/average/worst
in minutes), time limit per test (default 2 s), comparison mode.

A task locks when fully passed (all tests / all checks). No more submissions after that.

### Feedback after Submit

- Theoretical: show pass/fail, and on failure the error message and traceback, or "output
  does not match" / the failed AST rule name.
- Algorithm: show "X of Y tests passed". Never show which hidden tests failed, their args or
  expected values.
- Scores are never shown before the assessment closes.

### JSON import format

```json
{
  "type": "algorithm",
  "title": "Two sum",
  "statement": "Return indices of two numbers that add up to target.",
  "function_name": "two_sum",
  "compare": "unordered",
  "attempts": {"mode": "limited", "max": 5},
  "test_time_limit_s": 2,
  "tests": [
    {"args": [[2, 7, 11, 15], 9], "expected": [0, 1], "visible": true},
    {"args": [[3, 3], 6], "expected": [0, 1], "visible": false}
  ]
}
```

```json
{
  "type": "theoretical",
  "title": "Student class",
  "statement": "Define class Student with name and grade. Read a name and a grade, print them as 'Ali: 90'.",
  "attempts": {"mode": "unlimited", "time_bands_min": {"best": 5, "average": 10, "worst": 20}},
  "checks": {
    "stdin": "Ali\n90\n",
    "expected_stdout": "Ali: 90",
    "ast": [{"rule": "defines_class", "name": "Student"}]
  }
}
```

An import file may hold one task object or a list of them.

## 5. Scoring

Every task scores 0-100. Assessment score = plain average of its tasks (unattempted = 0).
Students see scores only after the assessment closes: one score per task plus the final.

### Limited attempts (N)

Each submission k (1-based) gets its own value. The task score is the best of them.

- Algorithm: `value_k = max(0, passed_k / total * 100 - (k - 1) * 100 / N)`; score = max over k.
- Theoretical: `100 - (k - 1) * 100 / N` for the attempt k that passed, floor 0; 0 if never passed.

Example, N = 5: 3/5 on attempt 1 = 60. Attempt 2 with 2/5 = 20, score stays 60. Attempt 2
with 5/5 = 80, score becomes 80.

### Unlimited attempts (time-based)

T = wall-clock minutes from the moment the student first opens the task until the submission
being scored. The clock keeps running while other tasks are open. It excludes time while the
assessment is paused by the teacher.

| T | Band score |
| --- | --- |
| T <= best | 100 |
| best < T <= average | linear from 100 down to 80 |
| average < T <= worst | linear from 80 down to 60 |
| T > worst | 40 |
| never solved (theoretical) / no submission | 0 |

- Algorithm: `value_k = band(T_k) * passed_k / total`; score = max over submissions.
- Theoretical: band(T) at the passing submission; 0 if never passed.

### Other rules

- Retake: final = min(score, 70).
- Timer expiry: a server job (every 30 s) closes expired sessions even if the browser is
  gone. Each unlocked task whose saved draft differs from its last submission (or has no
  submission and a non-empty draft) is auto-submitted. This counts as one attempt.
- Terminated session (strike limit): same auto-submit, graded normally, outcome recorded as
  Fail. The teacher sees the real score next to the Fail flag.
- Round scores to 1 decimal for display; store unrounded.

## 6. Anti-cheat

The strike limit is 3 (per assessment setting, default 3). The server counts strikes. The
student never sees the counter before termination.

| Event type | Detected by | Default effect |
| --- | --- | --- |
| `fullscreen_exit` | `fullscreenchange` | Strike |
| `tab_hidden` | `visibilitychange` hidden | Strike |
| `blur` | window lost focus and still away after 1.5 s | Strike |
| `blur_short` | focus returned within 1.5 s and fullscreen intact | Flag only |
| `away` | server: every further 30 s with heartbeat reporting not fullscreen or not focused | Strike |
| `resize_small` | window under 95% of screen width or height | Strike |
| `clipboard` | paste, copy, cut in the page; action blocked | Strike (configurable) |
| `context_menu` | right-click; action blocked | Strike (configurable) |
| `big_insert` | over 40 characters inserted in one editor change without paste | Flag only |
| `heartbeat_gap` | server: no heartbeat for over 20 s | Flag only |

Rules:

1. Debounce (server side): strike-type events within 2 s of the first event of an incident
   belong to that incident. One incident = one strike.
2. Overlay: on any fullscreen exit or focus loss the task is covered by a neutral overlay:
   "This assessment requires fullscreen. Click to continue." Clicking re-enters fullscreen.
   The timer keeps running. Nothing hints at strikes.
3. Per-assessment settings: for each event type the teacher chooses strike or flag only.
   Defaults are in the table.
4. At the limit: session terminated, work auto-submitted, student sees a termination screen
   listing every incident with time and type.
5. Server authority: the browser sends events immediately and a heartbeat every 5 s
   (seq, fullscreen, focused, current_task_id). Only the server counts strikes and terminates.
6. Autosave: draft code saved every 10 s and on every Run or Submit.
7. Restore: teacher only, reason required, one restore per click, optional new timer in
   minutes. Strike counter resets to 0; old events stay in the log.
8. Pause and extend: a teacher can pause/resume an assessment for everyone, or add minutes to
   every active session's timer. While paused, students see a "Paused" screen, no strikes are
   counted, and timers and time-band clocks do not advance.

## 7. Code execution sandbox

Student code only ever runs inside a throwaway container. Expected values never enter it.

- Sandbox image: `python:3.13-slim` plus `runner.py` baked in. No pip packages.
- The worker starts one container per job with:
  `--rm -i --network none --memory 128m --cpus 0.5 --pids-limit 64 --read-only
  --tmpfs /tmp:size=16m --user 1000:1000 --cap-drop ALL --security-opt no-new-privileges`
- The job payload (student code, function name, test args or stdin) is sent on the
  container's stdin as JSON. No bind mounts.
- `runner.py` runs each test in its own subprocess with its own time limit. A hang or crash
  fails only that test. It captures the student's stdout/stderr per test and returns one JSON
  document: per test `{status: ok|error|timeout, result, stdout, stderr}`.
- The worker compares results with expected values outside the container, and runs AST
  checks itself with Python's `ast` module (never executing the code).
- Limits: output per test capped at 64 KB; whole container killed after the sum of test
  limits plus 5 s.
- Results must be JSON-serialisable. Tuples compare equal to lists. A non-serialisable
  return value fails that test.
- The worker reaches Docker through the mounted `docker.sock`. Acceptable on the laptop.

## 8. Stack and architecture

Browser -> FastAPI -> PostgreSQL. FastAPI queues grading jobs in RabbitMQ; a Celery worker
starts sandbox containers and writes results to PostgreSQL. The browser polls for job results.

| Layer | Choice |
| --- | --- |
| Backend | Python 3.13, FastAPI, SQLAlchemy 2 (async), Alembic, PostgreSQL 16 |
| Queue | RabbitMQ + Celery (4 worker processes), Celery beat for the 30 s job |
| Sandbox | Docker, one fresh container per job |
| Frontend | React + Vite + TypeScript, CodeMirror 6, i18next (uz, en, ru) |
| Deploy | docker compose: api, worker, beat, postgres, rabbitmq, nginx (serves frontend, proxies /api) |

## 9. Data model

| Table | Key fields |
| --- | --- |
| `staff_user` | id, role (admin/teacher), email, password_hash, full_name |
| `group` | id, code |
| `student` | id, full_name, group_id; unique (full_name, group_id) |
| `course`, `course_group` | id, name, owner_id / course_id, group_id |
| `task` | id, author_id, type, title, statement, function_name, compare_mode, checks jsonb, attempt_mode, max_attempts, time_best, time_avg, time_worst, test_time_limit |
| `test_case` | id, task_id, args jsonb, expected jsonb, visible, position |
| `task_snapshot` | id, source_task_id, frozen jsonb (task + tests) |
| `assessment` | id, course_id, title, kind, timer_min, opens_at, closes_at, shuffle, strike_limit, strike_settings jsonb, status (draft/open/paused/closed) |
| `slot`, `slot_variant` | id, assessment_id, position / slot_id, snapshot_id |
| `session` | id, assessment_id, student_id, token_hash, status (active/finished/terminated/expired), is_retake, started_at, ends_at, strikes, last_heartbeat_at, last_seq |
| `session_task` | id, session_id, slot_id, snapshot_id, order_index, draft_code, first_opened_at, attempts_used, locked, score |
| `submission` | id, session_task_id, kind (run/submit/auto), code, passed, total, feedback jsonb, status, created_at |
| `event` | id, session_id, type, client_ts, server_ts, incident_no, is_strike, details jsonb |
| `restore` | id, session_id, teacher_id, reason, new_timer_min, created_at |
| `login_log` | id, student_id, ip, user_agent, result, created_at |

One non-retake session and at most one retake session per (assessment, student).

## 10. API

All paths are under `/api`. Run and Submit return a job id; the browser polls `/jobs/{id}`.

Student:

| Method | Path | Purpose |
| --- | --- | --- |
| GET | `/groups`, `/groups/{id}/students` | Login dropdowns |
| POST | `/auth/student` | Log in; 409 if a session is active elsewhere |
| GET | `/assessments/available` | Open assessments for this student |
| POST | `/assessments/{id}/start` | Create session: shuffle slots, pick variants |
| GET | `/sessions/{id}` | Tasks, attempts left, timer, paused flag |
| POST | `/sessions/{id}/tasks/{tid}/open` | Marks first_opened_at |
| PUT | `/sessions/{id}/tasks/{tid}/draft` | Autosave |
| POST | `/sessions/{id}/tasks/{tid}/run` | Algorithm only; visible tests |
| POST | `/sessions/{id}/tasks/{tid}/submit` | Graded attempt |
| GET | `/jobs/{id}` | Run/Submit result |
| POST | `/sessions/{id}/events` | Batch of events |
| POST | `/sessions/{id}/heartbeat` | seq, fullscreen, focused, current_task_id |
| POST | `/sessions/{id}/finish` | End early |
| GET | `/sessions/{id}/result` | Only after the assessment closes (or the incident list if terminated) |

Staff:

| Method | Path | Purpose |
| --- | --- | --- |
| POST | `/auth/staff` | Email + password |
| CRUD | `/tasks`; POST `/tasks/import` | Task bank, JSON import |
| CRUD | `/courses`, `/assessments`, `/assessments/{id}/slots` | Build assessments |
| POST | `/admin/groups/import` | Student CSV |
| CRUD | `/admin/staff` | Admin manages teachers |
| GET | `/assessments/{id}/live` | Live board data (polled every 5 s) |
| POST | `/assessments/{id}/pause`, `/resume`, `/extend` | Whole class |
| POST | `/assessments/{id}/close` | Close now; results become visible |
| POST | `/sessions/{id}/restore` | reason (required), new_timer_min |
| POST | `/sessions/{id}/terminate` | Manual termination |
| POST | `/students/{id}/release` | Free a blocked account |
| POST | `/assessments/{id}/retake` | student_ids |
| GET | `/assessments/{id}/results`, `/results.xlsx` | Results and Excel export |
| GET | `/sessions/{id}/detail` | Submissions, outputs, event timeline |

Rules for every endpoint:

- Hidden tests and expected values are never sent to a student's browser.
- A student can reach only their own session. Every session endpoint checks the owner.
- Teachers see only assessments of their own courses; admin sees all. The task bank is shared.
- Run is limited to one call per 5 s per student.
- Submitted code is limited to 64 KB.

## 11. Screens

Student:

- Login (group, name), list of available assessments, pre-start check page.
- Working screen: task list in the student's order, statement, CodeMirror editor, Run
  (algorithm only) and Submit, attempts left, timer if set, feedback panel.
- Overlay, Paused screen, termination screen with incident list.
- Results window after close: score per task, final average, note if capped at 70.

Teacher:

| View | Shows |
| --- | --- |
| Task bank | List, form editor with test editor, JSON import |
| Assessment builder | Settings, slots, variants, strike settings |
| Live board | Per student: status, current task, strikes, last heartbeat, flags; Release, Restore, Terminate; Pause, Resume, Extend, Close for the class |
| Results table | Students x tasks, final score, Fail/Retake/Terminated flags; Excel export |
| Session detail | Every submission's code and output, event timeline |
| Violation log | Incidents for an assessment, filter by type |

Admin: staff management, groups and CSV import, courses.

## 12. v2 (do not build now)

Plagiarism check, task analytics, student history view, WebSocket live board, VPS hardening,
session code in front of the student lists.
