---
description: Compare MR pipeline e2e failures vs develop baseline
argument-hint: <MR-url-or-pipeline-id>
allowed-tools: Bash(glab:*), Read, Grep, Glob
---

Given $ARGUMENTS:

1. Fetch last pipeline for MR via `glab ci view` / `glab api`.
2. Extract failed e2e jobs + test names from that pipeline.
3. Fetch last develop pipeline, extract same.
4. Diff: new failures (red) vs pre-existing (noise).
5. Verdict:
   - **Red** (new failures exist): investigate each new failure. Read test file, read MR diff, trace whether MR changes touch code paths exercised by failing test. Report root cause hypothesis per failure.
   - **Green** (all failures pre-existing on develop): sanity-check. Scan MR diff for files/selectors/routes referenced by currently-failing tests. Confirm no overlap. If overlap found, flag as suspicious despite green verdict.

Output format:
- One-line verdict: `GREEN` or `RED`.
- Table: failing test | status on MR | status on develop | new?
- If RED: per-new-failure investigation block.
- If GREEN: confirmation line that MR diff does not intersect failing-test code paths.
