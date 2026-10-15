---
name: chaoxing-homework-automation
description: Complete Chaoxing (学习通) course homework in the browser — read the on-page questions, answer from domain knowledge, and submit. Use when asked to finish a 学习通/超星 assignment. Do not use for web answer search.
---

# 学习通作业自动化

目标：完成用户指定的学习通课程作业，全程基于页面内容作答并提交。严禁用浏览器搜索答案——题干、公式、表格都在页面上，直接读取后按学科知识作答。

可直接逐字复制的取数、填充、校验、定位弹窗脚本见 [references/dom-recipes.md](references/dom-recipes.md)。

## 一、进入作业页面

1. 打开 https://i.chaoxing.com。若出现登录页，用 `browser_waiting_for_user_interaction` 请用户登录。
2. 页面主体在 iframe 里，先取 iframe 地址再直接导航（顶层 ref 无法触达 iframe 内容）：

   ```js
   JSON.stringify(Array.from(document.querySelectorAll('iframe')).map(f=>f.src))
   ```

3. 课程列表中取目标课程链接后直接导航：

   ```js
   document.querySelector('a[href*="courseid=<id>"]')?.href
   ```

4. 会落到 `mooc2-ans.chaoxing.com/mooc2-ans/mycourse/stu?...`。再取一次 iframe，「作业」页签的地址形如 `mooc1.chaoxing.com/mooc2/work/list?courseId=..&classId=..&cpi=..&enc=..`，直接导航过去。
5. 作业列表项文本形如 `<作业名> ; 未交 / 待批阅 / 已完成`。按用户给的日期或名称选最匹配的一项点击——它会在**新标签页**打开 `mooc-ans/mooc2/work/dowork?...&workId=..&answerId=..`，先用 `browser_tabs` 切到该标签页再继续操作。
6. 核对页头「题量 / 满分 / 作答时间」，记录截止时间。
7. 「考试」页签结构相同，同样按 iframe 取地址的方式进入。

## 二、读取题目

结构：`div.questionLi`（id 为 `question<qid>`）内是 `h3` 题干；选项行为 `div.answerBg`，其中 `span.num_option` 是显示字母、`div.answer_p` 是选项内容；隐藏域 `#answer<qid>` 保存当前选择。

一次性抽取全部题目（答案键 = 题号、题干、每个选项的字母与其文本或公式 data）：

```js
Array.from(document.querySelectorAll('.questionLi')).map(q => q.id + ' |' + q.querySelector('h3').innerText.slice(0,120) + '| ' + Array.from(q.querySelectorAll('.answerBg')).map(d => (d.querySelector('.num_option')||{}).innerText + '=' + (d.querySelector('img') ? d.querySelector('img').getAttribute('data') : (d.querySelector('.answer_p')||{}).innerText)).join(' ; ')).join(' @@ ')
```

关键规则：

- 选项顺序按学生随机打乱。**显示字母不是判分字母**；判分用 `.num_option` 的 `data` 属性，`#answer<qid>` 累加的就是这些 data 值。按选项**文字（或公式含义）**判断对错，不要按显示字母。
- 公式是图片，class 为 `ans-latex-moudle`，其 `data` 属性是 URL 编码的 LaTeX，用 `decodeURIComponent` 解码即得到真实公式（例如 `"\mathrm{VIF}_j\geqslant10"`、`"R_a^2\leqslant R^2"`、`"k>1000"`）。图片选项的 innerText 只有「选择」，必须先解码图片再作答。
- 填空题干里的数值同样藏在图片中，按同样方式解码。
- 题干里成对出现的`（        ）`是答题括号，可忽略。

## 三、单选题作答

1. 用 `browser_snapshot({ maxNodes: 500 })` 取一次快照。
2. 顺序解析快照行：`option "题目 N."` 为题目、`radio "<label>"` 为选项，建立题号→ref 映射；图片选项的 `aria-label` 只有「A/B/C/D 选择」，需结合第二步的 data 判断。
3. 每题确定唯一 ref，在**同一次 Exec 内批量点击**（type/fill/click 之间无需逐个重新快照），随后一次性回读全部 `#answer<qid>` 校验。

## 四、填空题作答

每个填空题都挂了 UEditor 实例，实例在 `UE.instants` 中，key 是 textarea 的 id（普通填空为 `answer<qid>`，带编号空格的为 `answerEditor<qid>1`）。

```js
var log=[]; for(var kk in UE.instants){ var ed=UE.instants[kk]; var id=(ed.textarea&&ed.textarea.id)||ed.key; if(m[id]){ ed.setContent(m[id]); ed.sync(); log.push(id+' => '+ed.getContent()); } } log.join(' | ')
```

`sync()` 必须调用——它把编辑器正文写回 textarea，表单提交读的是 textarea 的值。编辑器正文位于 iframe 内，原生 ref 触达不到，只能用该 API 写入。

## 五、校验与提交

1. 回读全部 `#answer*` 隐藏域与 textarea，逐题确认选中项的文字/公式与预期一致，再提交。
2. 点顶部「提交」。会弹出定制弹窗 `div.popDiv.Marking`，内有 `p#popcontent`「确认提交？」以及 `a#popok`（提交）/ `a#popno`（取消）——点 `#popok`。
3. 成功后跳到 `mooc-ans/mooc2/work/prompt?...`，显示「提交成功 / 等待教师批阅 / 提交时间」。再打开作业列表确认该项状态变为「待批阅」。

## 硬性规则与坑

- 作答后**不要刷新、不要离开** `dowork` 页面：答案只存在于 DOM 中，刷新即全部丢失。
- 浏览器窗口最小化或离屏时视口会变成 0×0：原生点击静默失效（不报错但无任何效果）、`browser_take_screenshot` 报「tab may be offscreen」。先用 `innerHeight` 探测确认；确认后**不要反复重试**，直接调 `browser_waiting_for_user_interaction` 请用户还原窗口，恢复后再继续。
- 原生点击确实无法落地但页面仍完好时，可用页面自带的处理器兜底：`addChoice(el)`，其中 `el` 为 `div.answerBg` 元素；它负责切换选中态并写入 `#answer<qid>`。
- 在 Exec 里不要写 `\n` 这类反斜杠转义：传过去会变成真实换行，页面抛 `SyntaxError: Invalid regular expression: missing /`。需要截断/拆分时用 `split()`、`slice()` 代替正则。
- 提交不可逆（页面提示「本作业提交后不允许查看或修改答案」），提交前必须逐题校验。
- 用户只想留档时点「暂时保存」，不要点「提交」。
- 务必报告最终状态与作答内容（题号→答案），便于用户复核。
