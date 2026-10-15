# 学习通作业页 DOM 取数/填充脚本

以下脚本用于 `browser_evaluate`。逐字复制即可，不要改造其中的转义写法。

> 通用禁忌：脚本字符串里**不要出现 `\n` 等反斜杠转义**（传到页面会变成真实换行，触发 `SyntaxError: Invalid regular expression: missing /`）。需要拆分/截断时用 `split()`、`slice()`。

## 1. 取 iframe 真实地址

顶层 ref 触达不到 iframe 内容，必须先拿到地址再整体导航。

```js
JSON.stringify(Array.from(document.querySelectorAll('iframe')).map(f => f.src))
```

## 2. 抽取全部题目（题干 + 选项 + 公式）

关键点：选中项的判分字母是 `.num_option` 的 `data` 属性（不是显示字母）；公式藏在 `img[class*=ans-latex-moudle]` 的 `data` 属性里，是 URL 编码的 LaTeX，必须 `decodeURIComponent`。

```js
Array.from(document.querySelectorAll('.questionLi')).map(function (q) {
  var stem = q.querySelector('h3');
  var stemImgs = stem ? Array.from(stem.querySelectorAll('img')).map(function (m) {
    return decodeURIComponent(m.getAttribute('data') || '');
  }) : [];
  var opts = Array.from(q.querySelectorAll('.answerBg')).map(function (d) {
    var letter = d.querySelector('.num_option');
    var img = d.querySelector('img');
    var txt = d.querySelector('.answer_p');
    return {
      shown: letter ? letter.innerText.trim() : '',
      key: letter ? letter.getAttribute('data') : '',
      formula: img ? decodeURIComponent(img.getAttribute('data') || '') : '',
      text: txt ? txt.innerText.trim() : ''
    };
  });
  var hid = document.getElementById('answer' + q.id.replace('question', ''));
  return {
    id: q.id,
    type: q.getAttribute('typename'),
    stem: stem ? stem.innerText.slice(0, 200) : '',
    stemFormulas: stemImgs,
    options: opts,
    current: hid ? hid.value : null
  };
})
```

输出前用 `JSON.stringify(...)` 包一层再 `text()`，避免嵌套对象被 `[object Object]` 吞掉。

## 3. 从 snapshot 建立「题号 → 选项 ref」映射

取一次 `browser_snapshot({ maxNodes: 500 })`，按出现顺序解析快照行：

- `option "题目 N." [eN]` —— 题目分界，记录当前题号；
- `radio "<label>" [eN]` —— 紧跟其后的选项，归到当前题号下。

因为 `.num_option` 的显示字母与判分字母不一致，`aria-label` 往往只有「A/B/C/D 选择」，映射时必须回看第 2 步的 `text`/`formula` 字段来判断该 ref 对应哪个答案。

## 4. 批量填填空题（UEditor）

先用 `Object.keys(UE.instants).map(k => (UE.instants[k].textarea || {}).id || UE.instants[k].key)` 枚举 key（普通填空为 `answer<qid>`，带编号空格的为 `answerEditor<qid>1`），再把答案按该 key 填进 `answers`：

```js
var answers = {};
// answers['answer406366384'] = '严重';
var log = [];
Object.keys(UE.instants).forEach(function (k) {
  var ed = UE.instants[k];
  var id = (ed.textarea && ed.textarea.id) || ed.key;
  if (answers[id] === undefined || answers[id] === null) { return; }
  ed.setContent(answers[id]);
  ed.sync();
  log.push(id + ' => ' + ed.getContent());
});
log.join(' | ')
```

`sync()` 不可省略：它把编辑器正文写回 textarea，而表单提交读的是 textarea 的值。编辑器正文在 iframe 里，原生 ref 触达不到，只能走这个 API。

## 5. 提交前逐题校验

一次性回读所有单选题的选中结果与所有填空题 textarea：

```js
JSON.stringify(Array.from(document.querySelectorAll('.questionLi')).map(function (q) {
  var sel = null;
  Array.from(q.querySelectorAll('.answerBg')).forEach(function (d) {
    var s = d.querySelector('.num_option');
    if (s && s.classList.contains('check_answer')) { sel = d; }
  });
  var img = sel ? sel.querySelector('img') : null;
  var hid = document.getElementById('answer' + q.id.replace('question', ''));
  return {
    q: q.id,
    picked: sel ? sel.getAttribute('aria-label') : null,
    pickedFormula: img ? decodeURIComponent(img.getAttribute('data') || '') : null,
    value: hid ? hid.value : null
  };
}))
```

填空题的三个 textarea 分别是 `answer<qid>`（4 题）与 `answerEditor<qid>1`（编号空题型）。

## 6. 提交弹窗定位

点「提交」后弹出自定义弹窗（不是浏览器原生 confirm），先定位再取 ref 点击：

```js
var pop = null;
Array.from(document.querySelectorAll('div.popDiv')).forEach(function (d) {
  if (d.innerText.indexOf('确认提交') >= 0) { pop = d; }
});
pop ? (pop.querySelector('#popcontent').innerText + ' | 确认按钮存在=' + !!pop.querySelector('#popok')) : '未出现弹窗'
```

`a#popok` 为「提交」，`a#popno` 为「取消」。点击 `#popok` 后页面跳转 `mooc-ans/mooc2/work/prompt?...`，出现「提交成功 / 等待教师批阅 / 提交时间」即为成功。

## 7. 视口健康检查（点击静默失效时先跑这个）

```js
'innerH=' + innerHeight + ' innerW=' + innerWidth + ' vis=' + document.visibilityState
```

`innerH` / `innerW` 为 0 表示窗口被最小化或移出屏幕：此时原生点击不报错但完全无效，`browser_take_screenshot` 会报「tab may be offscreen」。不要重试点击，直接 `browser_waiting_for_user_interaction` 请用户还原窗口。
