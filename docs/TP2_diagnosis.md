# Lab sheet 2: diagnosis from the pipeline log

| Defect | Gate that failed | Name in the log | One-line cause |
|---|---|---|---|
| Test 1 | tests | `tests/test_challenges.py::test_defaults_are_applied` | The default `decay_solves` is 100 instead of the expected 10. |
| Test 2 | tests | `tests/test_challenges.py::test_a_missing_file_is_reported_clearly` | A missing challenge file lets a raw `FileNotFoundError` escape instead of reporting it clearly. |
| Credential | secret scan | `generic-api-key` (`src/scoreboard/db.py:31`) | A fake `STAGING_DB_CREDENTIAL` value is hard-coded in `db.py`. |
| Dependency | dependency audit | `urllib3` 2.0.5, `PYSEC-2026-1995` | `requirements.txt` pins `urllib3` 2.0.5, which has known vulnerabilities. |

How each test failed:

* Test 1: assertion (`AssertionError: assert 100 == 10`).
* Test 2: exception (`FileNotFoundError: [Errno 2] No such file or directory: '/nonexistent/challenges.yaml'`).
