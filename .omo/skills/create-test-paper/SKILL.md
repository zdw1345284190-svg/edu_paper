---
name: create-test-paper
description: "创建交互式A4试卷HTML，支持在线答题、自动评分、打印输出。包含单选题/多选题/填空题/解答题四种题型，自动评分JS引擎，MathJax公式渲染。适合各学科教师出卷。Triggers: '出卷', '试卷', 'test paper', 'create test', '生成试卷', '交互试卷', '在线答题'."
---

# Create Test Paper — 交互式 A4 试卷生成技能

生成一份可直接在浏览器打开答题、提交后自动评分、同时支持打印到 A4 纸的完整 HTML 试卷。

## 架构概览

生成的 HTML 试卷包含三个层面：

```
试卷文件（如：math_grade10_ch1_sections1-5.html）
 ├─ 试卷正文（2-4 页 A4，含所有题目）
 ├─ 提交按钮 + 成绩弹窗
 └─ 参考答案区（默认隐藏，提交后可查）
```

## 项目约定

### 文件命名
```
{subject}_grade{level}_ch{chapter}_sections{range}.html
```
- 全小写英文，下划线分隔
- subject: math（数学）/ bio（生物）/ physics（物理）/ chem（化学）/ geo（地理）/ history（历史）/ politics（政治）/ chinese（语文）/ english（英语）
- 示例：`math_grade10_ch1_sections1-5.html`、`bio_grade10_ch2_sections1-3.html`

### 存放位置
文件直接放在项目根目录，不建子目录。

## 题型系统

| 题型 | HTML 标签 | 评分方式 |
|------|-----------|----------|
| 单选题 | `<input type="radio">` | 选对满分，选错 0 分 |
| 多选题 | `<input type="checkbox">` | 全对满分，部分选对 2 分，有错选 0 分 |
| 填空题 | `<input class="fill-input">` | 精确匹配，支持备选答案（`data-alt`） |
| 解答题 | `<textarea class="answer-area">` | 不自动评分，标记"需人工批改" |

## 经验要点（实战总结）

### 1. QID 分配规范

| 题型 | qid 范围 | 映射方式 |
|------|---------|---------|
| 单选题 | `q1` ~ `qN` | 题号 = qid 序号 |
| 多选题 | `q101` ~ `q199` | 题号从 9 起类推 |
| 填空题 | `q11` ~ `q19` | 题号从 12 起类推 |
| 解答题 | `q17` ~ `q20` | 题号从 18 起类推 |

> **经验**：qid 与显示题号是两套系统，通过 `QUESTION_LABELS` 映射。方便后续增删题型时不影响 qid 索引。

### 2. 评分引擎配置（三段式）

```javascript
// ① 答案键（集中管理）
const ANSWER_KEY = {
  q1: 'C', q2: 'B', q3: 'A',       // 单选题
  q101: ['A','B','C'],               // 多选题（数组）
  // 填空题答案写在 HTML data-answer 上
};

// ② 分值映射
const SCORE_MAP = {
  single: 4,   // 单选题每题分值
  multi: 4,    // 多选题每题分值
  fill: 3,     // 填空题每空分值
  calc: { q17: 8, q18: 10, q19: 10, q20: 10 }  // 解答题每题不同分值
};

// ③ 题号映射（qid → 显示题号）
const QUESTION_LABELS = {
  q1: '1', q2: '2', /* ... */
  q101: '9',         // 第一道多选题
  q11: '12',         // 第一道填空题
  q17: '18',         // 第一道解答题
};

// ④ 题型映射
const QUESTION_TYPES = {
  q1: 'single', q2: 'single', /* ... */
  q101: 'multi', /* ... */
  q11: 'fill',  /* ... */
  q17: 'calc',  /* ... */
};
```

### 3. 填空题 data-alt 备选答案规范

```html
<!-- 单个备选 -->
<input type="text" class="fill-input" data-answer="斐林" data-alt="斐林试剂">

<!-- pipe 分隔多个备选，评分引擎自动拆分 -->
<input type="text" class="fill-input" data-answer="糖原" data-alt="肝糖原|肌糖原">
```

**经验**：不要忽略近似答案的可能性（如"斐林"和"斐林试剂"），尤其生物/化学学科。

### 4. 解答题命名与计分

解答题仍然使用 `calc-question` 类名（继承自计算题体系），但在主观题场景中标注"需人工批改"。

小问分值标注方式：
```html
(1) 第一小问；<span style="float:right;">（4 分）</span><br>
(2) 第二小问。<span style="float:right;">（6 分）</span>
```

### 5. 试卷头部信息栏

```html
<div class="info-row">
  <label>姓名：<input type="text" style="width:80px;border:none;border-bottom:1px solid #000;"></label>
  <label>班级：<input type="text" style="width:60px;border:none;border-bottom:1px solid #000;"></label>
  <label>学号：<input type="text" style="width:80px;border:none;border-bottom:1px solid #000;"></label>
  <label style="margin-left:auto;">得分：<span id="displayScore">______</span></label>
</div>
```

### 6. A4 尺寸与页面布局

```
.page {
  width: 210mm;
  min-height: 297mm;
  padding: 18mm 20mm 15mm 20mm;
}
```

- 多页时自动撑长（`min-height` 而非 `height`）
- 每页之间用 margin-bottom 留白

### 7. 多选题评分（arraysEqual）

```javascript
function arraysEqual(a, b) {
  if (a.length !== b.length) return false;
  const sa = [...a].sort();
  const sb = [...b].sort();
  for (let i = 0; i < sa.length; i++) {
    if (sa[i] !== sb[i]) return false;
  }
  return true;
}
```

全对满分 → `SCORE_MAP.multi`，部分选对（无错选）→ 2 分，有错选 → 0 分。

### 8. 填空题输入归一化

```javascript
function normalizeStr(s) {
  return s.trim().replace(/\s+/g, '');
}
```

答案比对前必须 trim + 去空格，避免用户不小心多打了空格导致误判。

### 9. 答案自动缓存 + 清空按钮（防手机切后台丢失）

所有试卷需内置 localStorage 答案自动缓存机制。学生用手机打开试卷时，切到后台不会丢失已填答案。

#### 9.1 PAPER_ID 命名规则

每份试卷定义唯一 ID，作为 localStorage key 的一部分：

```javascript
// 命名格式：{subject}_grade{level}_ch{chapter}_sections{range}
const PAPER_ID = 'physics_grade10_ch1';
const STORAGE_KEY = 'test_answers_' + PAPER_ID;
```

不同试卷 key 不同，即使用户同时打开多份试卷也不会混淆。

#### 9.2 完整实现代码

**注意 textarea 的识别方式**：textarea 不一定有 id 属性，需要根据其父元素来定位。有两种常见 DOM 结构：

- **模式 A（data-qid 型）**：`<div class="calc-question" data-qid="q17">` → 用 `[data-qid="q17"] textarea` 定位
- **模式 B（id 型）**：`<div class="question" id="q17">` → 用 `#q17 textarea` 定位

两种模式都要兼容：

```javascript
// ===== 答案自动缓存 =====
const PAPER_ID = 'physics_grade10_ch1';  // 每份试卷唯一
const STORAGE_KEY = 'test_answers_' + PAPER_ID;

function saveAnswers() {
  var data = {};
  // 单选
  document.querySelectorAll('input[type="radio"]').forEach(function(r) {
    if (r.checked && r.name) {
      if (!data.r) data.r = {};
      data.r[r.name] = r.value;
    }
  });
  // 多选（存选中值数组）
  document.querySelectorAll('input[type="checkbox"]').forEach(function(c) {
    if (c.checked && c.name) {
      if (!data.c) data.c = {};
      if (!data.c[c.name]) data.c[c.name] = [];
      data.c[c.name].push(c.value);
    }
  });
  // 填空输入（优先 id，其次 data-qid）
  document.querySelectorAll('input[type="text"], input.fill-input').forEach(function(t) {
    var key = t.id || (t.closest('[data-qid]') && t.closest('[data-qid]').getAttribute('data-qid'));
    if (key) {
      if (!data.t) data.t = {};
      data.t[key] = t.value;
    }
  });
  // 解答题 textarea（通过父元素定位）
  document.querySelectorAll('textarea').forEach(function(ta) {
    var parent = ta.closest('[data-qid], .question, .calc-question');
    var key = parent ? (parent.id || parent.getAttribute('data-qid')) : null;
    if (key) {
      if (!data.a) data.a = {};
      data.a[key] = ta.value;
    }
  });
  try { localStorage.setItem(STORAGE_KEY, JSON.stringify(data)); } catch(e) {}
}

function restoreAnswers() {
  try {
    var raw = localStorage.getItem(STORAGE_KEY);
    if (!raw) return;
    var data = JSON.parse(raw);
    // 恢复单选
    if (data.r) {
      Object.keys(data.r).forEach(function(name) {
        var el = document.querySelector('input[type="radio"][name="' + name + '"][value="' + data.r[name] + '"]');
        if (el) el.checked = true;
      });
    }
    // 恢复多选
    if (data.c) {
      Object.keys(data.c).forEach(function(name) {
        data.c[name].forEach(function(val) {
          var el = document.querySelector('input[type="checkbox"][name="' + name + '"][value="' + val + '"]');
          if (el) el.checked = true;
        });
      });
    }
    // 恢复填空
    if (data.t) {
      Object.keys(data.t).forEach(function(key) {
        var el = document.getElementById(key) || document.querySelector('input[name="' + key + '"]');
        if (el) el.value = data.t[key];
      });
    }
    // 恢复解答题
    if (data.a) {
      Object.keys(data.a).forEach(function(key) {
        var ta = document.querySelector('#' + key + ' textarea')
                || document.querySelector('[data-qid="' + key + '"] textarea');
        if (ta) ta.value = data.a[key];
      });
    }
  } catch(e) {}  // localStorage 不可用时静默失败
}

function clearAnswers() {
  if (!confirm('确定要清空所有已填写的答案吗？此操作不可撤销！')) return;
  localStorage.removeItem(STORAGE_KEY);
  document.querySelectorAll('input[type="radio"]').forEach(function(r) { r.checked = false; });
  document.querySelectorAll('input[type="checkbox"]').forEach(function(c) { c.checked = false; });
  document.querySelectorAll('input[type="text"], input.fill-input, textarea').forEach(function(el) { el.value = ''; });
}

function setupAutoSave() {
  document.addEventListener('change', function(e) {
    if (e.target.matches('input[type="radio"], input[type="checkbox"]')) saveAnswers();
  });
  document.addEventListener('input', function(e) {
    if (e.target.matches('input[type="text"], input.fill-input, textarea')) saveAnswers();
  });
}

// 页面加载即恢复 + 开启自动保存
restoreAnswers();
setupAutoSave();
```

#### 9.3 关键设计要点

| 要点 | 说明 |
|------|------|
| key 唯一性 | `test_answers_` + `PAPER_ID`，不同试卷互不干扰 |
| 提交前保存 | `submitPaper()`/`submitExam()` 开头调用 `saveAnswers()`，保证最后输入不丢 |
| 清空确认 | `clearAnswers()` 先弹出 `confirm()`，防误触 |
| 异常静默 | `try/catch` 包裹 localStorage 操作，兼容无痕模式等受限环境 |
| textarea 定位 | 通过 `父元素.id` 或 `父元素.data-qid` 回退匹配，不自带 id 也能工作 |

#### 9.4 清空按钮

在提交按钮旁添加清空按钮，样式与提交按钮保持一致但颜色区分：

```html
<!-- 提交栏示例 -->
<div class="submit-bar">
  <button class="btn-submit" onclick="submitPaper()">📝 提交试卷</button>
  <button class="btn-ref" onclick="toggleAnswer()">📖 查看参考答案</button>
  <button onclick="clearAnswers()" style="background:#888; color:#fff; border:none; padding:10px 18px; border-radius:6px; cursor:pointer;">🗑️ 清空答案</button>
</div>
```

### 10. Ctrl+Enter 快捷键

```javascript
document.addEventListener('keydown', function(e) {
  if (e.ctrlKey && e.key === 'Enter') {
    e.preventDefault();
    submitPaper();
  }
});
```

建议在提交按钮上标注"按下 Ctrl+Enter 也可提交"。

### 11. 试卷头部核心信息

```
标题：{学科} · {教材版本} 第{X}章（X.X-X.X）单元测试卷
一行副信息：总分 100 分 | 考试时间 45 分钟
信息行：姓名 ______  班级 ______  学号 ______  得分 ______
```

### 12. 一题多空（填空题多个输入框）

化学/生物学科经常有"将____逐滴加入____中，继续煮沸至____"这种一题多空。每个空各占一个 `fill-input`，分别配 `data-answer` 和 `data-alt`。

```html
<div class="fill-question" data-qid="q11">
  15. Fe(OH)₃胶体的制备方法：向
  <input type="text" class="fill-input input-sm" data-answer="沸水" style="width:50px;">
  中逐滴加入
  <input type="text" class="fill-input" data-answer="FeCl₃饱和" data-alt="FeCl₃饱和溶液|饱和FeCl₃|饱和氯化铁" style="width:100px;">
  溶液，继续煮沸至溶液呈红褐色，停止加热。
</div>
```

**经验**：评分引擎遍历 `fill-question` 下的所有 `fill-input` 逐个评分，每个空独立计分。分值为 `SCORE_MAP.fill` × 该题空数。例如一题2空，每空3分则该题6分。

**注意填空输入框宽度**：
| class | 宽度 | 适用场景 |
|-------|------|---------|
| `fill-input`（默认）| 80px | 一般术语 |
| `input-sm` | 50px | 元素符号、数字、字母 |
| `input-long` | 120px | 长句子 |

### 13. 解答题内嵌自动评分（calc-question 混合模式）

部分解答题中某些小问可以用输入框自动评分（如化学中的"氧化剂：____ 还原剂：____"，或计算题中的"加速度 = ____ m/s²"），其余小问仍需人工评阅。

实现方式：在 `calc-question` 里混用 `<input>` + `<textarea>`。评分引擎会：
- 扫描 `calc-question` 内带 `data-answer` 的 `<input>` → 自动评分
- 标记 `<textarea>` → 需人工批改

```html
<div class="calc-question" data-qid="q19">
  <span class="q-head">22.（10分）</span>
  <span class="q-text">用双线桥法分析氧化还原反应：</span>
  <div class="q-text" style="font-size:16px; text-align:center; margin:8px 0;">
    \(2KMnO₄ + 16HCl = 2KCl + 2MnCl₂ + 5Cl₂↑ + 8H₂O\)
  </div>
  <div style="margin:2px 0 4px 22px;">
    (1) 用双线桥标出电子转移方向和数目；<span style="float:right;">（5分）</span><br>
    (2) 氧化剂：<input type="text" class="fill-input input-sm" data-answer="KMnO₄" style="width:70px;">
       还原剂：<input type="text" class="fill-input input-sm" data-answer="HCl" style="width:70px;">
       <span style="float:right;">（2分）</span><br>
    (3) 转移电子的物质的量：<input type="text" class="fill-input input-sm" data-answer="5" style="width:50px;"> mol
       <span style="float:right;">（3分）</span>
  </div>
  <textarea class="answer-area" placeholder="请在此书写(1)的双线桥分析过程..."></textarea>
</div>
```

**关键**：calc-question 内的 `<input>` 必须带 `data-answer` 属性，评分引擎按 name/qid 不匹配的方式单独处理，与外面的填空题互通评分逻辑。

### 14. MathJax 化学式/离子符号规范

化学试卷中的元素符号、离子、化学式、反应式均用 MathJax 渲染。常用写法：

| 显示效果 | 写法 | 说明 |
|---------|------|------|
| \(Fe^{3+}\) | `\(Fe^{3+}\)` | 离子：元素 + 上标电荷 |
| \(CO_3^{2-}\) | `\(CO_3^{2-}\)` | 多原子离子：下标+上标 |
| \(H_2O\) | `\(H_2O\)` | 分子式：下标 |
| \(2KMnO₄\) | `\(2KMnO₄\)` | 带系数：直接写 |
| \(H_2SO_4\) | `\(H_2SO_4\)` | 酸：下标数字 |
| \(CO_2↑\) | `\(CO_2↑\)` | 气体符号 |
| \(BaSO_4↓\) | `\(BaSO_4↓\)` | 沉淀符号 |
| \(⇌\) | `\(\rightleftharpoons\)` | 可逆反应箭头 |
| \(→(△)→\) | `\(\xrightarrow{\triangle}\)` | 加热条件 |
| \(2H⁺ + CO_3^{2-} = CO_2↑ + H_2O\) | `\(2H⁺ + CO_3^{2-} = CO_2↑ + H_2O\)` | 完整离子方程式 |

**经验**：MathJax 中 `=` 自动渲染为等号，`−`（减号）和 `–`（短横线）不同，推荐用 `-`（连字符）保证一致性。上标电荷用 `^{n+}` / `^{n-}`，下标用 `_{n}`。

## 快速上手

### Step 1 — 准备模板

直接从现有试卷复制修改，或按以下规范从头创建。

```typescript
task(
  category="unspecified-high",
  load_skills=["create-test-paper"],
  prompt="在项目根目录创建一份高一物理第二章的交互试卷，文件名为 physics_grade10_ch2_sections2-1-2-3.html..."
)
```

### Step 2 — 按知识点设计题目

覆盖每个知识点的核心概念，难度递进：
- 单选题：基础概念（60%记忆+40%理解）
- 多选题：易混淆知识点辨析
- 填空题：关键术语/数据填空
- 解答题：综合分析/实验/情境应用

### Step 3 — 替换题目内容

#### 单选题
```html
<div class="choice-question" data-qid="q1" data-type="single" data-answer="C">
  <span class="q-head">1.</span>
  <span class="q-text">题目文字（　）</span>
  <div class="options" data-qid="q1">
    <label><input type="radio" name="q1" value="A"> A．选项A</label>
    <label><input type="radio" name="q1" value="B"> B．选项B</label>
    <label><input type="radio" name="q1" value="C"> C．选项C</label>
    <label><input type="radio" name="q1" value="D"> D．选项D</label>
  </div>
</div>
```

#### 多选题
```html
<div class="choice-question" data-qid="q101" data-type="multi" data-answer='["A","B"]'>
  <span class="q-head">9.</span>
  <span class="q-text">题目文字</span>
  <div class="options" data-qid="q101">
    <label><input type="checkbox" name="q101" value="A"> A．选项A</label>
    <label><input type="checkbox" name="q101" value="B"> B．选项B</label>
    <label><input type="checkbox" name="q101" value="C"> C．选项C</label>
    <label><input type="checkbox" name="q101" value="D"> D．选项D</label>
  </div>
</div>
```

#### 填空题
```html
<div class="fill-question" data-qid="q11">
  12. 题目文字：<input type="text" class="fill-input" data-answer="标准答案" data-alt="备选答案" style="width:80px;">
</div>
```

#### 解答题
```html
<div class="calc-question" data-qid="q17">
  <span class="q-head">18.</span>
  <span class="q-text">题目文字</span>
  <div style="margin:2px 0 4px 22px;">
    (1) 第一小问；<span style="float:right;">（4 分）</span><br>
    (2) 第二小问。<span style="float:right;">（6 分）</span>
  </div>
  <textarea class="answer-area" placeholder="请在此书写解答过程..."></textarea>
</div>
```

### Step 4 — 配置答案键

```javascript
const ANSWER_KEY = {
  // 单选题
  q1: 'C', q2: 'B', q3: 'A', q4: 'D', q5: 'C', q6: 'B', q7: 'A', q8: 'C',
  // 多选题（数组）
  q101: ['A','B','C'],
  q102: ['A','B','C','D'],
  q103: ['A','C']
};
// 填空题答案写在 HTML data-answer 上
// 解答题不自动评分
```

### Step 5 — 更新参考答案区

在 `<!-- ==================== 参考答案区 ==================== -->` 之后，提供答案和每题解析。

## 评分引擎详解

评分逻辑位于 HTML 底部的 `<script>` 中，核心函数：

| 函数 | 作用 |
|------|------|
| `submitPaper()` | 入口，禁用按钮防重复提交，触发评分 |
| `gradePaper()` | 遍历所有题型，执行评分逻辑，返回 `{ earned, total, details }` |
| `showScore(result)` | 弹出成绩弹窗，显示得分、正确率、每题详情 |
| `closeScore()` | 关闭弹窗 |
| `toggleAnswer()` | 切换参考答案区的显示/隐藏 |

### 评分规则

- **单选题**: `data-answer` 精确匹配 → `SCORE_MAP.single` 分
- **多选题**: 
  - 全对（选项完全一致，顺序无关）→ `SCORE_MAP.multi` 分
  - 部分对（选了部分正确选项，无错选）→ 2 分
  - 有错选 → 0 分
- **填空题**: `data-answer` 精确匹配（trim+去空格后）；有 `data-alt` 时备选也接受 → `SCORE_MAP.fill` 分
- **解答题**: 不自动评分，标记"需人工批改"

### 视觉反馈

- ✅ 绿色高亮 → 正确选项/输入
- ❌ 红色高亮 → 错误选项/输入
- 漏选标注 → 多选中正确但未选的选项
- 正确答案标注 → 错误选项旁会显示正确答案（灰色标注）

## 完整 HTML 模板骨架

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>...</title>
  <style>
    /* 全局重置、A4页面、试卷头部、题型样式、参考答案样式、打印样式 */
  </style>
</head>
<body>
  <div class="page">
    <!-- 试卷头部：标题、总分时间、姓名/班级/学号 -->
    <!-- 一、单选题（选择题题干 + 选项） -->
    <!-- 二、多选题（选择题题干 + 选项） -->
    <!-- 三、填空题（填空输入框） -->
    <!-- 四、解答题（题目 + textarea） -->
    <!-- 提交按钮 -->
  </div>

  <!-- 参考答案区 -->
  <!-- MathJax CDN -->
  <!-- 评分引擎 JS（ANSWER_KEY + gradePaper + submitPaper + showScore + toggleAnswer） -->
</body>
</html>
```

## 已生成试卷清单（实战验证）

| 文件 | 内容 | 题量 |
|------|------|------|
| `math_grade10_ch1_sections1-5.html` | 高一数学·集合与常用逻辑用语 | 8单选+3多选+6填空+4解答（100分） |
| `bio_grade10_ch2_sections1-3.html` | 高一生物·组成细胞的分子 | 12单选+3多选+6填空+4解答（100分） |
| `physics_grade10_ch1.html` | 高一物理·运动的描述 | 10单选+3多选+6填空+4解答（100分） |
| `chem_grade10_ch1.html` | 高一化学·物质及其变化 | 10单选+3多选+6填空+4解答（100分） |

> 新建试卷时，可直接参考上述文件的源码结构。

## A4 打印

试卷已内置打印样式：

```css
@media print {
  .submit-bar, .score-overlay { display: none !important; }
  @page { size: A4; margin: 0; }
}
```

- 浏览器打开 → Ctrl+P → 选 A4 → 直接打印
- 打印时自动隐藏提交按钮、弹窗、评分标记
- 参考答案默认可打印

## MathJax 公式

试卷使用 MathJax 3 渲染 LaTeX 公式，已包含 CDN：

```html
<script>
MathJax = {
  tex: { inlineMath: [['\\(', '\\)']], displayMath: [['\\[', '\\]']] },
  svg: { fontCache: 'global' },
  options: { enableMenu: false }
};
</script>
<script src="https://cdn.jsdelivr.net/npm/mathjax@3/es5/tex-svg.js" async></script>
```

行内公式用 `\(...\)`，行间公式用 `\[...\]`。MathJax 加载需要 1-3 秒，页面加载后自动渲染。

## 自定义模板

### 修改分值
- 修改 `SCORE_MAP` 对象中的值即可
- 总分 = 各题型 `score * 题量` 之和

### 修改题型数量
- 删除某题型：直接删除对应的 HTML 块，并更新 `ANSWER_KEY`/`QUESTION_LABELS`/`QUESTION_TYPES`
- 增加题目：复制现有题目结构，更新 `data-qid`、`name`、`data-answer`

### 添加新题型
在 `gradePaper()` 中添加新的评分分支：
```javascript
document.querySelectorAll('.judge-question').forEach(function(el) {
  // ...评分逻辑
});
```

## 典型调用

在 OpenCode 中使用本技能：

```typescript
// 创建新试卷
task(
  category="unspecified-high",
  load_skills=["create-test-paper"],
  prompt="在项目根目录创建一份高一化学第一章1.1-1.3的交互试卷..."
)

// 在现有试卷基础上修改
task(
  category="quick",
  load_skills=["create-test-paper"],
  prompt="修改 math_grade10_ch1_sections1-5.html，把第5题选项D改为正确答案，更新data-answer和参考答案区。"
)
```
