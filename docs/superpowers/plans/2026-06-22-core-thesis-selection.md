# Core Thesis Selection Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 使确认页以候选选择方式确认核心论点，并阻止空论点提交。

**Architecture:** `recommendations.json` 增加核心论点候选数组与推荐索引；前端将候选映射为最终字符串并保持既有回写格式；文档与策略约束同步到新契约。旧的单一 `thesis` 字段作为兼容输入。

**Tech Stack:** 原生 JavaScript、Flask静态确认页、Markdown协议文档。

## Global Constraints

- 不修改服务端 `result.json` 的最终 `core_thesis` 字符串格式。
- 无候选数组时必须兼容既有 `core_thesis.thesis`。
- 不新增依赖；仓库不含自动化测试，使用静态语法检查和浏览器交互验证。

---

### Task 1: 核心论点候选选择与提交校验

**Files:**
- Modify: `skills/ppt-master/scripts/confirm_ui/static/app.js`

**Interfaces:**
- Consumes: `core_thesis.{applicable,thesis,candidates,selected,scqa,supporting_arguments}`。
- Produces: `STATE.core_thesis`，并以非空字符串写入 `result.json.core_thesis`。

- [ ] **Step 1: 实现候选归一化和选择控件**

将旧 `thesis` 转换为唯一候选；渲染候选卡片、推荐标记和自定义输入。

- [ ] **Step 2: 增加提交前非空校验**

在 `confirm()` 中拦截适用核心论点的空白值，聚焦自定义输入或核心论点区。

- [ ] **Step 3: 执行语法检查**

Run: `node --check skills/ppt-master/scripts/confirm_ui/static/app.js`

Expected: exit code 0。

### Task 2: 策略协议与说明同步

**Files:**
- Modify: `skills/ppt-master/references/strategist.md`
- Modify: `skills/ppt-master/scripts/docs/confirm_ui.md`

**Interfaces:**
- Produces: 策略角色可生成候选，确认页说明可解释候选、默认项、自定义项和最终回写格式。

- [ ] **Step 1: 更新策略角色协议**

要求适用时提供2至3条候选句和推荐索引，并保留最终用户编辑的权威性。

- [ ] **Step 2: 更新确认页数据契约和说明**

记录候选字段、旧字段兼容规则、空值拦截和结果格式不变。

- [ ] **Step 3: 校验文档与实现字段一致**

Run: `rg -n "core_thesis|candidates|selected" skills/ppt-master/references/strategist.md skills/ppt-master/scripts/docs/confirm_ui.md skills/ppt-master/scripts/confirm_ui/static/app.js`

Expected: 三处均使用同一字段名。
