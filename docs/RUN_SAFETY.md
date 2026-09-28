# Run safety: what a worst-case test found, and what changed

The architecture work kept behaviour the same. This next piece did the opposite:
we started the platform and drove it on purpose with bad input, to see how it
behaved when pushed. It found five issues. Unlike the refactor, these changes do
change behaviour, and each one is written up here.

## What we did

We started the server and gave it deliberately hostile or broken input: requests
with no credentials, file paths that tried to escape their project, projects
with bad names, an oversized upload, the AI steps with no model key, a run that
matched no tests, a run pointed at a Journey that does not exist, several runs
started at once, and deleting a project while a run was still in flight.

## What held

The guards that already existed were real, and the failure paths were mostly
calm:

- Path traversal was blocked.
- Project creation rejected missing fields, bad slugs, duplicate slugs, and
  malformed JSON.
- An oversized upload was refused.
- Extraction and generation returned a clean "no model key" message rather than
  breaking.
- A run that matched no tests ended as `empty`, not failed.
- Unknown ids returned `404`.
- Deleting a project removed its folder and rows with no leftover browser
  processes.

## What was wrong, and what changed

| # | The problem | The change |
| --- | --- | --- |
| 18 | Starting a run against a Journey that does not exist was accepted. The whole suite ran with no environment, recorded the wrong stage, and failed every test. | The run is refused with `404`, the same as upload and extraction already did. |
| 19 | Nothing limited how many runs could start. Six ran at once with no cap, and a run that hung stayed "running" forever. | A Project may have three runs in flight; a further start is refused with a clear message. A run is stopped after a timeout (default thirty minutes). Both are configurable. |
| 20 | Deleting a Project while a run was in flight removed the folder and rows underneath it, so the run failed against a missing config. | Delete cancels the Project's runs, waits for them to stop, then removes anything. |
| 21 | A missing browser was only discovered when every test failed with the runner's own "executable does not exist" error. | A run checks the runner and its browser before it starts, and refuses with a plain message. |
| 22 | The Test Library reads each test's last status and last run time, but the server never sent them, so those indicators could never light up. | One query finds each test's most recent result, and the list includes it. |

An earlier fix already in the code heals runs interrupted by a crash: at
start-up, any run still marked "running" is marked cancelled.

## Why it matters

Every gap was in what happens when the platform is pushed, not in the ordinary
path. Two of the fixes close sharp edges for a single operator — a bad stage
name, a missing browser. The rest are groundwork for hosting many customers: a
limit on work in flight, a way to stop work that is stuck, and a clean handover
when a tenant is removed.
