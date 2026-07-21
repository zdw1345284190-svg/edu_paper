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

### 9. Ctrl+Enter 快捷键

```javascript
document.addEventListener('keydown', function(e) {
  if (e.ctrlKey && e.key === 'Enter') submitPaper();
});
```

### 10. 试卷头部核心信息

```
标题：{学科} · {教材版本} 第{X}章（X.X-X.X）单元测试卷
一行副信息：总分 100 分 | 考试时间 45 分钟
```

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

> 新建试卷时，可直接参考上述两个文件的源码结构。

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
