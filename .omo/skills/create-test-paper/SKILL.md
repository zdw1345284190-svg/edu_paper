---
name: create-test-paper
description: "创建交互式A4试卷HTML，支持在线答题、自动评分、打印输出。包含单选题/多选题/填空题/实验题/计算题五种题型，自动评分JS引擎，MathJax公式渲染。适合各学科教师出卷。Triggers: '出卷', '试卷', 'test paper', 'create test', '生成试卷', '交互试卷', '在线答题'."
---

# Create Test Paper — 交互式 A4 试卷生成技能

生成一份可直接在浏览器打开答题、提交后自动评分、同时支持打印到 A4 纸的完整 HTML 试卷。

## 架构概览

生成的 HTML 试卷包含三个层面：

```
试卷文件（如：高一物理_第一章单元测试.html）
 ├─ 试卷正文（3-4 页 A4，含所有题目）
 ├─ 提交按钮 + 成绩弹窗
 └─ 参考答案区（默认隐藏，提交后可查）
```

## 题型系统

| 题型 | HTML 标签 | 评分方式 |
|------|-----------|----------|
| 单选题 | `<input type="radio">` | 选对满分，选错 0 分 |
| 多选题 | `<input type="checkbox">` | 全对满分，部分选对 2 分，有错选 0 分 |
| 填空题 | `<input class="fill-input">` | 精确匹配，支持备选答案（`data-alt`） |
| 实验题 | `<input class="fill-input">` | 同填空题，精确匹配 |
| 计算题 | `<textarea class="answer-area">` | 不自动评分，标记"需人工批改" |

## 快速上手

### Step 1 — 准备模板

```typescript
// 复制参考模板到你需要的科目目录
// 模板路径：subjects/physics/高一上学期_第一章1-3节_单元测试卷.html
bash(command="Copy-Item -Path 'subjects/physics/高一上学期_第一章1-3节_单元测试卷.html' -Destination 'subjects/物理/新学期_第X章_单元卷.html'")
```

### Step 2 — 替换题目内容

在 HTML 中按以下规范修改每个题型区域：

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

关键属性：
- `data-qid`: 唯一标识，单选题 q1-q99，多选题 q101-q199
- `data-type`: `"single"` 或 `"multi"`
- `data-answer`: 单选题填一个字母（如 `"C"`）；多选题填 JSON 数组（如 `'["A","C"]'`）
- `name`: 同一单选题组的 radio `name` 必须相同

#### 多选题
```html
<div class="choice-question" data-qid="q101" data-type="multi" data-answer='["A","B"]'>
  ...
  <label><input type="checkbox" name="q101" value="A"> A．...</label>
  <label><input type="checkbox" name="q101" value="B"> B．...</label>
  ...
</div>
```

#### 填空题
```html
<div class="fill-question" data-qid="q11">
  题目文字：<input type="text" class="fill-input" data-answer="标准答案"> 单位。
</div>
```

关键属性：
- `data-answer`: 标准答案（用于匹配评分）
- `data-alt`: 备选答案（如 `data-alt="火箭"` 表示"飞船"或"火箭"都算对）
- `class`: 用 `fill-input`（默认 80px）、`fill-input input-sm`（50px）、`fill-input input-long`（120px）控制宽度

#### 实验题（同填空题）
```html
<div class="experiment-question" data-qid="q15">
  <input type="text" class="fill-input input-sm" data-answer="交流" style="width:60px;">
</div>
```

#### 计算题
```html
<div class="calc-question" data-qid="q16">
  <span class="q-head">16.</span>
  <span class="q-text">题目文字...</span>
  <div style="margin:2px 0 4px 22px;">
    (1) 第一小问；<span style="float:right;">（4 分）</span><br>
    (2) 第二小问。<span style="float:right;">（6 分）</span>
  </div>
  <textarea class="answer-area" placeholder="请在此书写解答过程..."></textarea>
</div>
```

计算题不自动评分，提交后在成绩弹窗中标为"需人工批改"。

### Step 3 — 配置答案键

在底部 JavaScript 中更新 `ANSWER_KEY` 对象：

```javascript
const ANSWER_KEY = {
  // 单选题 <data-qid>: <答案字母>
  q1: 'C', q2: 'C', // ...

  // 多选题 <data-qid>: <答案数组>
  q8: ['A','C'], q9: ['A','B','C'], // ...
};
// 填空题的答案写在 HTML 的 data-answer 属性上，无需在此配置
// 计算题不自动评分
```

### Step 4 — 更新参考答案区

在 `<!-- ==================== 参考答案区 ==================== -->` 之后更新答案和解析内容。

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

- **单选题**: `data-answer` 精确匹配 → 4 分
- **多选题**: 
  - 全对（选项完全一致，顺序无关）→ 4 分
  - 部分对（选了部分正确选项，无错选）→ 2 分
  - 有错选 → 0 分
- **填空题**: `data-answer` 精确匹配；有 `data-alt` 时备选也接受 → 每空 2 分
- **计算题**: 不自动评分

### 视觉反馈

- ✅ 绿色高亮 → 正确选项/输入
- ❌ 红色高亮 → 错误选项/输入
- ·漏选 → 多选中正确但未选的选项
- 正确答案标注 → 错误选项旁会显示正确答案

## A4 打印

试卷已内置打印样式：

```
@media print {
  .submit-bar, .score-overlay { display: none !important; }
  @page { size: A4; margin: 0; }
}
```

- 浏览器打开 → Ctrl+P → 选 A4 → 直接打印
- 打印时自动隐藏题按钮、弹窗、评分标记
- 参考答案可打印（默认可见，或通过 JS 控制显示）

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
- 选择题分值：改 `gradePaper()` 中的 `totalObj += 4`（每题分值）
- 填空题分值：改 `totalObj += 2`（每空分值）
- 总分上限：改成绩弹窗中显示的文字

### 修改题型数量
- 删除某题型：直接删除对应的 HTML 块
- 增加题目：复制现有题目结构，更新 `data-qid`、`name`、`data-answer`

### 添加新题型
在 `gradePaper()` 中添加新的评分分支，遵循现有模式：
```javascript
// 示例：判断题评分
document.querySelectorAll('.judge-question').forEach(function(el) {
  // ...评分逻辑
});
```

## 典型调用

在 OpenCode 中使用本技能：

```typescript
// 方式一：创建新试卷（从模板生成）
task(
  subagent_type="general",
  load_skills=["create-test-paper"],
  prompt="在 subjects/物理/ 下创建一份高一物理第二章匀变速直线运动的交互试卷，含 8 道单选、2 道多选、4 道填空、3 道计算，总分 100 分。"
)

// 方式二：在现有试卷基础上修改
task(
  subagent_type="general", 
  load_skills=["create-test-paper"],
  prompt="修改 subjects/物理/月考卷.html，把第 5 题选项 D 改为正确答案，更新 data-answer 和参考答案区。"
)
```

## 项目目录结构

```
.omo/skills/create-test-paper/SKILL.md    ← 本技能文件
subjects/physics/*.html                   ← 物理试卷
subjects/化学/*.html                      ← 化学试卷（示例）
subjects/生物/*.html                      ← 生物试卷（示例）
```

## 参考模板

模板位于项目 `subjects/physics/` 下，包含所有题型完整示例，建议直接复制后修改。
