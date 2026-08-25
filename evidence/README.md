# Evidence Index

This directory is organized to match the reviewer feedback.

- System 1: 29-test pytest output, the cited run summary, and `claim_02_stolen_bike.jsonl`.
- System 2: the cited `budget.json`, `eval.jsonl`, `eval_control.jsonl`, and 30-test pytest output.
- System 3: the complete `.claude/` configuration, validator output, and 35-test pytest output.
- System 4: a recorded-response shift run, resulting hot state and scratchpad, fork scratchpads, isolation verification, and 33-test pytest output.

System 4 uses the project's provided recorded response fixture and offline CLI mode, so no live Claude API call is required.
