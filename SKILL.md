---
name: weakness-quiz
description: 分析用户错题本，定位薄弱知识点，联网检索同类题目并生成一份灰阶标准卷排版的针对性练习卷 PDF（含答案解析页）。
---

# weakness-quiz

让 agent 拥有「分析错题 → 定位薄弱点 → 联网搜同类题 → 生成练习卷 PDF」的完整能力。

## 适用场景

用户上传一份错题本（PDF / 文本 / 图片），并表达「分析我的弱点 / 针对薄弱点出题 / 出一份练习卷 / 针对性训练」等意图。不限于学科，K12 及高中阶段的学科均可覆盖。

## 设计原则（固定，不可变更）

1. **题目完全联网原样收录**：从题库站检索带答案的同类题目，**不改编、不改数字、不换选项**。收录前必须核对答案与解析存在且自洽；若某题找不到可靠答案，宁可弃用换题，也不得自行编造。
2. **始终附答案解析页**：练习卷倒数第一页（或最后固定一页）必须是完整答案解析，含正确选项/答案与简要推导过程。
3. **排版严格锁定标准卷样式**：必须使用本 skill 自带模板 `assets/exam_template.html`，其灰阶配色、竖排密封线、卷头、得分总表、大题头、作答区、页脚页码均已冻结。任何调用者不得自行更换配色或版式。
4. **题型配比固定为 100 分**：单选 25 分（每题 5 分）、多选 15 分（每题 5 分）、填空 20 分（每题 5 分）、解答 40 分（每题 10 分）。
5. **标签化归类**：每道大题右上角标注薄弱点标签（如「场强叠加」「基底分解」），便于用户定位。

## 流水线

### Step 1｜分析错题本，定位薄弱点

读取用户提供的错题本，逐题归类到具体的知识点短语。归纳出 3~5 个**可检索**的薄弱点（粒度要细，例如「空间向量基底分解」「库仑力作为向心力」，而非「立体几何」这种大范围标签）。

向用户输出一份弱点诊断：按薄弱点列出涉及的题号、具体卡在哪里、一句话总括。这一步是后续搜题的输入，**必须先完成再搜题**。

### Step 2｜联网检索同类题目

针对 Step 1 归纳出的每个薄弱点，用 `web_search` 检索对应知识点 + 「题目」「解析」「答案」等关键词，优先来源：

- 中学学科网、菁优网、高考资源网等题库站
- 各省高考真题 / 模拟题官方解析
- 知乎、公众号等带完整解析的科普/教辅内容

收录规则：每道薄弱点配 3~4 道题，整卷 16 小题（单选 5 + 多选 3 + 填空 4 + 解答 4）。题目必须覆盖全部薄弱点，占比与错题中暴露的严重程度成正比。

### Step 3｜生成 HTML

以 `assets/exam_template.html` 为骨架，替换其中的标题、卷头信息、题目区与答案页。关键约束：

- 中文字体使用 `Noto Sans CJK` 或 `WenQuanYi Micro Hei`
- 公式用 KaTeX / MathJax，或转成可读的 Unicode / HTML 表示
- 页面使用 A4 分页：`@page { size: A4; }`、`.report-page` 单页高度 297mm、`overflow:hidden`、`break-after:page`
- 图片（若有）声明容器宽高 + `object-fit`，`break-inside:avoid`

### Step 4｜质检

生成 PDF 前必须运行 PDF skill 的质检脚本：

```bash
node /data/skills/pdf/scripts/check_render_quality.js --source 练习卷.html --report render-quality.json
```

通过标准：退出码 0、`ok` 为 true、每页 `scrollHeight <= clientHeight`、无文本重叠、无越界。若 FAIL，按报告中的 `code` / `selector` / `rect` 修改 HTML 后重跑，通过后才渲染。

### Step 5｜渲染 PDF

```bash
html_to_pdf --source 练习卷.html --target 练习卷.pdf
```

通过 `media_info` 以 `type: file` 交付。同时把弱点诊断文字一并返回给用户。

## 产物

最终向用户交付两样东西：

1. 一段弱点诊断说明（文字，含薄弱点表格）
2. 一份练习卷 PDF 文件（灰阶标准卷，含答案解析页）

## 依赖

- 本 skill 自带的 `assets/exam_template.html`
- `pdf` skill 的创建流程：`html_to_pdf` 与 `scripts/check_render_quality.js`
- `web_search` 用于联网检索同类题目

## 目录结构

```
weakness-quiz/
├── SKILL.md
└── assets/
    └── exam_template.html
```
