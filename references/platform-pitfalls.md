# 平台踩坑记录（本次实战）

> 只记非显而易见、下次能省时间的内容。控件怎么点的细节在 `chaoxing-ops/references/atomic-operations.md`；界面语义在 `chaoxing-ops/references/paper-exam/`。

## 1. 两套系统（最高代价的坑）

- `mooc2-ans.chaoxing.com`：课程内在线考试、监考、试卷库；也有智能导入页 `.../qbank/smartimport`。
- `v10596.kaoshi.chaoxing.com`：线下纸考／网评，含考试管理／试卷管理／题库管理／考试扫描管理。
- **智能导入会跳到 mooc2-ans 再回跳线下 `zjSuccess`**——这是正常链路，不要在 mooc2-ans 里建线下试卷。本次最初把线下考试误判为线上，浪费了一轮；旧 mooc2-ans 试卷库里的 A 卷是进错系统建的，未删。
- 判线上／线下见 `chaoxing-ops` P6与 SKILL.md 阶段 0。

## 2. 线下字段：`_lx1` 后缀

- 发布页线下字段 **id 带 `_lx1` 后缀，name 不带**：
  - `input#startTime_lx1`（name=`startTime`）、`#endTime_lx1`（name=`endTime`）、`#limitTime_lx1`（name=`limitTime`）、`#passingStandard_lx1`（name=`passingStandard`）。
  - 勾选框：`#multiHalfScoreCheckbox_lx1`、`#anonymousReview_lx1`、`#reviewFastAllocationCheckbox_lx1` 等。
- 页面上另有隐藏的 `input[name=startTime]`／`limitTime=120`（线上遗留值）。**本次报错"考试开始时间不可为空"的根因就是填了隐藏的线上字段，而线下读的是 `_lx1`。**
- 用 id 定位（`document.getElementById('startTime_lx1')`）最稳；按 name 选会命多个（线上隐藏 + 线下可见）。

## 3. 时限不能大于起止时间差

- 报错："时限不能大于起始时间差"。窗口 16:00–17:30 = 90 分钟，但答题时长填了 120。
- 修法：时长改 90（或与用户确认缩短／延长窗口）。终检 B3 必查。

## 4. UEditor 须知：退回第一步会重置

- PC 端与移动端须知各一个 UEditor 实例（`UE.instants[0]`=`examPCNotes`、`[1]`=`examNotes`）。
- **从第二步返回第一步再进第二步，实例重新初始化，之前追加的"地点／主副监考"全部丢失**，只剩默认文字。
- 修法：在第二步用 `window.UE.instants[k].setContent(完整内容)` 重新写入，两端都写，再核对。
- 不要用 `UE.getEditor()`（会重渲染写不进去）。

## 5. "已添加考生 0 人"是显示假象

- 发布页显示"已添加考生0人"，但隐藏字段 `input[name=classids]` 已是正确班级ID（本次 `155688692`）；发布后考试管理列表考生人数 63、考生来源"智能制造装备—智能23"，证明添加成功。
- 根因：平台读不到 `data-studentcount` 之类属性导致的显示 bug。
- **不要因此重新添加**——会造成重复考生。以隐藏 `classids` + 发布后考生管理页实测人数为准。

## 6. 发布按钮的位置

- `a#submitButton`（`.publishSubmitButton`，`onclick="pubishExam();"`）。
- 第一步它在 `div.l_setbtn.twoStep` 里，`display:none`、零尺寸——在第一步用 `page.click('#submitButton')` 会报 "`none can receive input; element has a zero-sized bounding box`"。
- 必须先点 `a.nextStep` 进入第二步，按钮才可见可点。
- 提交后 `pubishExam()` → `examlx==1` → `pubishExamlx1()`。

## 7. 通知文案与输入框

- 发布成功弹窗"考试发布成功，是否发放考试通知？"→"是"→"发送通知"对话框。
- 标题框 `input#notice_title`（name=`text`）；**页面多处 `input[name=text]`**（含考试等级名称"优秀"等），`querySelector('input[name="text"]')` 会选错元素。用 id。
- 正文框 `textarea#notice_context`。
- 按 Q6 写文案：不出现 A/B 卷。默认标题是"新建试卷<时间戳>考试通知"，必须改。

## 8. 题号与透题

- 客观题按题型分块编号（单选1-8、多选1-4、判断1-6），系统不支持全局连续编号。强行改 Word 卷面连续编号会与答题卡／成绩表错位，通常保持分块。
- 批阅按题单独给分，题号分块不影响给分。
- 导入前做透题检查：本次第1题与第14题考点重复 → 第14题换成振动 RMS／峭度判断题；第13／15题轻微同考点但保留。
- 分值方案：Word 底稿不写分；导入页 24 个 `input[name=score].score_input` 填分；方案 8×2+4×2+6×1+4×10+2×15=100。

## 9. 课程名称／批次／场次

- `verifyCourse`／`verifySemester`／`verifyBatch` 三个隐藏字段均为 0 → 选填，不校验。
- 教务下拉无数据时无法填写；用户核查后决定留空。学期默认带"2026-2027学年第一学期（当前学期）"。
- 不要因为用户曾说"把该填信息填成全的"就硬填不存在的下拉项——先查 verify 标志和下拉数据，再向用户说明。
