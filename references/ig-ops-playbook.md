# iGenie Studio Acceptance Operations Playbook

This reference records the reusable two-stage acceptance workflow: reviewer check, pending-acceptance link collection, explicit approval, and batch approval across every pending Job ID.

## Route Map

| Purpose | URL |
|---|---|
| Task search | `https://igeniestudio.agibot.com/data/collection/tasks?rf=1` |
| Job list | `https://igeniestudio.agibot.com/data/collection/tasks/{taskId}/jobs?rf=1` |
| Episode list | `https://igeniestudio.agibot.com/data/collection/tasks/{taskId}/jobs/{jobId}/episodes` |
| Annotation/审核 page | `https://igeniestudio.agibot.com/data/collection/tasks/{taskId}/jobs/{jobId}/episodes/{episodeId}/check` |

## Stage 1: Reviewer Check

After searching a Task ID, find the numeric **审核员** value in the task row and click it. It opens the `{任务名}_审核员` drawer.

Open the **验收** tab. Search the visible reviewer rows for the exact reviewer.

- If found and enabled: record display name, account, join time, and status; do not click **新增审核员**.
- If not found: click **新增审核员**, keep **类型 = 验收**, search the exact reviewer in **审核员**, select the exact account, and click **添加**.
- If found but disabled or if multiple similar accounts exist: stop and ask instead of adding a duplicate.

Observed controls:

- `新增审核员`
- `类型`: `验收`, `标注`, `审核`
- `用户组`: optional
- `审核员`: searchable selector
- `添加`
- `取消`

Success evidence:

- a new row appears with reviewer name, account, join time, review count, and enabled switch;
- the `验收` tab count increments;
- a toast such as `成功添加1个审核员` may appear.

For the common reviewer used in this workflow:

- display name: `精标-欧阳金钢`
- account: `j-oyangjingang`

Use these values only after confirming they match the user's requested reviewer.

## Stage 1: Enumerate Every Pending Job

A Task may contain multiple Job IDs with pending acceptance. Enumerate all of them before returning links.

Recommended sequence:

1. Open the Task's Job list.
2. Use the Job-list **阶段 = 待验收** filter when available and click **搜索**.
3. Also inspect each visible Job row's `待验收：N` text or progress columns for corroborating evidence.
4. For every candidate Job, open `/tasks/{taskId}/jobs/{jobId}/episodes`.
5. Set **阶段 = 待验收**, click **搜索**, and keep only Jobs whose filtered count is greater than zero.
6. Record: Task ID, Job ID, pending count, and the visible pending episode IDs.

Do not stop after the first pending Job. Do not infer that all Jobs belong to the same stage.

## Stage 1: Random `/check` Links

From every pending Job:

- randomly select the requested number of distinct pending-acceptance episodes;
- default to one random episode per Job if no sample count is provided;
- click the row's **标注** button;
- verify the URL ends in `/check`, the breadcrumb includes **审核**, and the page contains **标注工具**.

Return links grouped by Task and Job:

```markdown
Task <taskId>
- Job <jobId>，待验收 <count> 条
  - [Episode <episodeId> 标注页面](https://igeniestudio.agibot.com/data/collection/tasks/<taskId>/jobs/<jobId>/episodes/<episodeId>/check)
```

Stop after returning the links. The user will inspect the data and explicitly approve the batch step.

## Stage 2: Batch Approval Gate

Do not enter batch approval mode before the user says to proceed.

When approved, process each approved Job sequentially and report its result separately.

### Enter selection mode

The episode page initially has no selection checkboxes. Click **批量标注** once to enter selection mode. This reveals:

- a checkbox column;
- a disabled/state-changed `批量标注` button;
- a `取消` button;
- header and row checkboxes.

### Prepare the complete selection

1. Set **阶段 = 待验收** and click **搜索**.
2. Record the filtered count.
3. Change the page size to a value that contains the entire filtered result: `20`, `50`, `100`, `200`, `500`, or `1000`.
4. Select the header checkbox.
5. Verify the selected count equals the filtered count.
6. Do not change page size after selecting, because doing so can clear the selection.

### Fill the dialog

Click **批量标注** again. The dialog contains:

- required `审核结果`: `通过`, `不通过`, `异常数据`;
- optional `审核标签`;
- `通过质量` when `审核结果 = 通过`: `优秀`, `可接受`, `差`;
- `取消` and `保存`.

For this workflow:

- `审核结果 = 通过`;
- leave `通过质量` at the default `优秀`;
- leave `审核标签` empty and do not change any other field;
- click **保存**.

### Verify persistence

The request can remain in flight briefly after the click. Verify success with independent signals:

1. Re-apply **阶段 = 待验收** and click **搜索**; the processed rows should no longer be returned, generally leaving `0` for a fully processed Job.
2. Query one processed episode ID without a stage filter; its **更新时间** should be recent, such as `几秒前`.
3. Check for a success toast or updated status count if available.
4. Do not treat dialog closure alone as proof.

Repeat for every approved Job ID, then report:

```text
Task <taskId>
- Job <jobId>: 待验收 <beforeCount> 条；已批量通过 <savedCount> 条；处理后待验收 <afterCount> 条
```

## Recovery Patterns

### Task ID search does not submit

Entering a Task ID and pressing Enter may leave the previous table in place. Click **搜索** explicitly and wait for the table to refresh. If needed, refill the ID and click **搜索** again.

### Reviewer count click is intercepted

The clickable number is inside the **审核员** column. A fixed right-hand column can cover its center. Refetch the visible rectangle, coordinate-click the exact numeric element, and confirm the `{任务名}_审核员` drawer heading before proceeding.

### Shadow-DOM controls fail

Some dialogs are behind a shadow boundary where accessibility activation fails. Use the visible rectangle or keyboard activation, then verify the changed state immediately.

### Reviewer selector is large

Type the exact name into the selector input to reduce the list, then choose the unique matching account. Do not pick a similar-looking reviewer.

### Page-size change clears selection

Always set page size before selecting all filtered rows. If selection is cleared, repeat the selection and count verification.

### Search result and filter count disagree

Treat the episode-level **阶段 = 待验收** count as authoritative. Do not select rows from a stale table or an earlier Job.

## Final Verification Checklist

Before the first mutation:

- Task ID matches the request.
- The reviewer is checked only in the **验收** tab.
- If the reviewer is absent, the exact reviewer account is selected before **添加**.
- All pending Job IDs have been enumerated, not just the first one.
- Each returned `/check` link opens the annotation page.

Before batch save:

- The Job ID is explicitly approved by the user.
- `阶段 = 待验收`.
- Filtered count and selected count match.
- `审核结果 = 通过`.
- `通过质量` remains `优秀`.
- `审核标签` and other fields are untouched.

After batch save:

- The pending-acceptance filter no longer returns the processed rows.
- At least one processed episode has a recent update time.
- The result is summarized per Job ID.

## Example Runs From Discovery

These examples illustrate the workflow; they are not fixed task data.

- Task `82819`: the `验收` reviewer list existed but `精标-欧阳金钢` was absent; the reviewer was added once and the count changed `4 -> 5`.
- Task `43727`: `精标-欧阳金钢` was added to `验收`; count changed `6 -> 7`. Pending acceptance had four episodes; links for `25445235` and `25445247` were returned before approval.
- Task `111973` / Job `6413101`: filtered `阶段 = 待验收`, selected `33` episodes, set `审核结果 = 通过` with default `通过质量 = 优秀`, and verified that the pending filter returned `0` afterward.

When a Task has multiple pending Jobs, repeat the link collection and batch approval separately for each Job and preserve the Job grouping in the response.

