---
name: agibot-acceptance-workflow
description: Run the two-stage iGenie Studio acceptance workflow for one or more Task IDs. Ensure the requested reviewer exists in 验收, enumerate every Job ID with 待验收 episodes, return random pending-acceptance /check links for user review, wait for explicit approval, then batch-mark each approved Job's filtered 待验收 episodes as 通过 with default quality 优秀. Use only for this end-to-end acceptance workflow; do not use for simple metadata or actual-review-count lookup.
metadata:
  short-description: iGenie 先验收审核员检查再待验收批量通过
---

# iGenie Acceptance Workflow

## Outcome

For each supplied Task ID, perform this two-stage workflow:

1. Ensure the requested reviewer exists in the **验收** reviewer team.
2. Find every Job ID under the Task that has **阶段 = 待验收** episodes.
3. Return random pending-acceptance `/check` annotation links from those Jobs for the user to inspect.
4. Stop and wait for explicit user approval.
5. After approval, batch-mark all pending-acceptance episodes in each approved Job:
   - filter **阶段 = 待验收**;
   - select all filtered rows;
   - set **审核结果 = 通过**;
   - leave **通过质量** at its default **优秀**;
   - do not modify **审核标签** or any other field;
   - save and verify.

A Task may have multiple pending-acceptance Job IDs. Never stop after the first one.

## Required Inputs

- One or more Task IDs or Task URLs.
- Exact reviewer name/account. The common workflow reviewer is `欧阳金钢`; in iGenie the exact selectable account is typically `精标-欧阳金钢 (j-oyangjingang)`. Use the reviewer supplied or confirmed by the user.
- Optional sample count per Job. If the user does not specify, return one random pending-acceptance episode link per pending Job.
- Optional target Job IDs. If omitted, consider all Job IDs with pending acceptance.

## Hard Gates

- **Reviewer gate:** Search only the **验收** tab. If the requested reviewer already exists and is enabled, do not add them again. If they are absent, add exactly one matching reviewer and verify the result. If the reviewer exists but is disabled or ambiguous, report it and do not create a duplicate.
- **Approval gate:** After returning the `/check` links, stop. Do not enter batch mode or click **保存** until the user explicitly approves the batch operation.
- **Filter gate:** Before any batch selection, set **阶段 = 待验收**, click **搜索**, and record the filtered count. Never select from an unfiltered or stale list.
- **Selection gate:** Set page size large enough to show the complete filtered result before selecting rows. Verify the selected count equals the filtered count.
- **Scope gate:** Batch-approve only the pending-acceptance episodes in the Jobs approved by the user. Never widen the batch to other stages or Jobs.

## Phase 1: Reviewer Check and `/check` Link Collection

Process each Task ID sequentially.

1. Open `https://igeniestudio.agibot.com/data/collection/tasks?rf=1`, search the Task ID, and explicitly click **搜索** if needed.
2. Click the numeric value in the task row's **审核员** column to open `{任务名}_审核员`.
3. Open the **验收** tab and check the reviewer list:
   - if the exact reviewer exists and is enabled, record the existing row and continue;
   - if absent, click **新增审核员**, keep **类型 = 验收**, search and select the exact reviewer account, click **添加**, then verify the count increased and the row appeared.
4. Open the Task's Job list and identify every Job ID that has pending acceptance:
   - use the Job-level **阶段 = 待验收** filter when available;
   - independently inspect `待验收：N` cells or the progress columns for evidence;
   - open each candidate Job's episode list, set **阶段 = 待验收**, click **搜索**, and keep the Job only when the filtered count is greater than zero.
5. From each pending Job, randomly select the requested number of distinct pending-acceptance episodes. Default to one per Job when no count is given.
6. Open each selected row's **标注** page and verify the URL ends with `/check` and the page shows breadcrumb **审核** and **标注工具**.
7. Return links grouped by Task ID and Job ID, with the pending count for each Job.

Return format:

```markdown
Task <taskId>
- Job <jobId>，待验收 <count> 条
  - [Episode <episodeId> 标注页面](https://igeniestudio.agibot.com/data/collection/tasks/<taskId>/jobs/<jobId>/episodes/<episodeId>/check)
```

Stop after returning the links and ask for approval.

## Phase 2: Batch Approval After User Approval

Only after the user explicitly approves, process each approved Job sequentially.

1. Open the Job's episode list and set **阶段 = 待验收**.
2. Click **搜索** and record the filtered count.
3. Change the page size so all filtered rows fit on one page.
4. Click **批量标注** once to enter selection mode.
5. Select the header checkbox and verify every filtered row is selected.
6. Click **批量标注** again to open the dialog.
7. Set **审核结果 = 通过**. Leave **通过质量 = 优秀** at the default. Leave **审核标签** and all other fields unchanged.
8. Click **保存**.
9. Verify per Job:
   - re-filter **阶段 = 待验收**; the processed rows should disappear and the count should drop as expected, usually to `0`;
   - check at least one processed episode and confirm its **更新时间** is recent;
   - do not use dialog closure alone as proof of success.
10. Report per Job: Job ID, filtered count before approval, saved count, pending count afterward, and verification result.

If any batch is unconfirmed, do not retry automatically. Report the exact Job and state.

## UI and Browser Notes

- The **审核员** value is a clickable `action_count` element. A fixed right-hand table column can intercept a normal click; use a coordinate click on the exact visible value if needed and verify the reviewer drawer opened.
- Generated `el-id-*` values change. Identify controls by visible labels such as `Task ID/UUID`, `审核员`, `验收`, `新增审核员`, `阶段`, `待验收`, `批量标注`, `审核结果`, `通过`, `通过质量`, and `保存`.
- Element Plus dialogs may sit behind a shadow boundary. If accessibility activation fails, use the visible rectangle or keyboard activation, then verify the resulting state.
- Changing page size can clear row selection. Set page size before selecting all.
- `审核结果 = 通过` reveals **通过质量** with default **优秀**. Do not change it unless the user explicitly asks.

## Supporting Reference

Read `references/ig-ops-playbook.md` for detailed UI mappings, multi-Job enumeration guidance, recovery patterns, and verification evidence.
