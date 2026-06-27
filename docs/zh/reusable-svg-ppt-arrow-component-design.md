# 可复用 SVG/PPT 箭头组件设计方案

---

## 一、方案定位

### 1.1 建设背景

当前 SVG 与 PowerPoint 内容生产工具通常采用标准线端标记、预设箭头形状或逐页手工绘制方式表达方向关系。上述方式能够满足基本方向提示，但在正式汇报材料中容易出现箭头比例失衡、曲线衔接生硬、端点贴边、视觉权重过高以及跨页面样式漂移等问题。

本方案建设一套可供 SVG、PowerPoint 及其他矢量内容工具复用的独立箭头组件。组件统一处理路径几何、节点边界、终端切线、视觉分档和箭头资产调用，输出可编辑矢量结果。PPT Master 作为首个接入方，后续其他工具可通过相同数据契约复用。

### 1.2 建设目标

| 目标 | 设计要求 |
|---|---|
| 通用复用 | 核心组件不依赖 PPT Master 项目结构，可独立复制、调用和版本管理 |
| 视觉统一 | 箭身、箭头端部、节点留白和视觉权重由统一设计令牌控制 |
| 场景适配 | 支持直线、贝塞尔曲线、圆角正交线及汇聚、分流、流程、反馈等关系 |
| 资产扩展 | 几何引擎与设计资产库同时存在，支持标准预设和特殊箭头素材 |
| 原生编辑 | SVG 输出保持矢量可编辑，PPTX 输出优先转换为 DrawingML 原生形状 |
| 平滑接入 | 不改变 PPT Master 逐页构建、质量检查和导出主路径 |
| 可迁移 | 组件目录边界清晰，后续仓库主线升级时可整体迁移 |

### 1.3 非目标

| 非目标 | 处理原则 |
|---|---|
| 不生成整页 SVG | 组件仅生成箭头几何、SVG 片段或独立箭头资产 |
| 不承担页面语义判断 | 页面关系类型仍由上层 Strategist、Executor 或调用方确定 |
| 不替代专业流程图引擎 | 首期聚焦箭头绘制质量，不处理复杂自动布图 |
| 不强制所有关系使用箭头 | 无方向关系继续使用无箭头连接线 |
| 不依赖外部图形服务 | 首期采用本地矢量计算和本地资产 |

---

## 二、总体架构

### 2.1 架构原则

采用“几何核心＋设计资产库＋渲染适配器”组合架构。几何核心解决通用计算问题，设计资产库提供经过设计校准的箭头端部和完整关系配方，渲染适配器面向不同输出格式转换。

```text
上层调用方
    │
    │ ArrowSpec
    ▼
几何核心 Geometry Core
    ├─ 路径构造
    ├─ 节点边界求交
    ├─ 终点切线计算
    ├─ 路径裁切与留白
    ├─ 尺寸分档
    └─ 碰撞检查
    │
    │ ArrowGeometry
    ▼
设计资产库 Design Library
    ├─ Terminal Presets
    ├─ Connector Recipes
    └─ Special Arrow Assets
    │
    │ ResolvedArrow
    ▼
渲染适配器 Renderers
    ├─ SVG Group Renderer
    ├─ PPTX Geometry Adapter
    └─ JSON Geometry Exporter
```

### 2.2 组件目录

建议形成可整体迁移的独立目录：

```text
components/vector-arrow/
├── README.md
├── VERSION
├── schema/
│   ├── arrow-spec.schema.json
│   └── arrow-geometry.schema.json
├── core/
│   ├── geometry.py
│   ├── routes.py
│   ├── boundaries.py
│   ├── terminals.py
│   └── validation.py
├── presets/
│   ├── terminals/
│   ├── recipes/
│   └── themes/
├── assets/
│   └── special/
├── renderers/
│   ├── svg.py
│   ├── pptx_geometry.py
│   └── json_geometry.py
├── cli/
│   ├── arrow_render.py
│   └── arrow_validate.py
└── examples/
    ├── arrow_matrix.json
    └── arrow_matrix.svg
```

组件内部不得引用 `projects/`、`skills/ppt-master/` 或具体模板目录。PPT Master 通过适配层调用组件。

---

## 三、核心数据契约

### 3.1 输入对象 `ArrowSpec`

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `version` | string | 是 | 数据契约版本 |
| `id` | string | 是 | 箭头实例标识 |
| `semantic_role` | string | 是 | `process_flow`、`data_flow`、`fan_in_result` 等 |
| `route` | object | 是 | 路径类型、起点、终点及控制参数 |
| `source_boundary` | object | 否 | 来源节点边界 |
| `target_boundary` | object | 否 | 目标节点边界 |
| `terminal` | object | 是 | 箭头端部预设、尺寸档位和方向 |
| `style` | object | 是 | 颜色、线宽、透明度、层级 |
| `constraints` | object | 否 | 留白、禁入区域和碰撞策略 |
| `metadata` | object | 否 | 调用方、页面、语义说明等追踪信息 |

建议结构：

```json
{
  "version": "1.0",
  "id": "p05-input-a",
  "semantic_role": "fan_in_result",
  "route": {
    "type": "cubic",
    "start": [258, 140],
    "end": [347, 195],
    "controls": [[302, 146], [318, 176]]
  },
  "target_boundary": {
    "type": "rect",
    "x": 347,
    "y": 168,
    "width": 106,
    "height": 100
  },
  "terminal": {
    "preset": "floating-stream",
    "scale": "standard",
    "position": "end"
  },
  "style": {
    "color": "#236AA5",
    "guide_color": "#9FC2DF",
    "guide_width": 1.4,
    "opacity": 0.72
  },
  "constraints": {
    "target_clearance": 6,
    "terminal_gap": 2,
    "avoid_regions": []
  }
}
```

### 3.2 中间对象 `ArrowGeometry`

| 字段 | 说明 |
|---|---|
| `guide_path` | 裁切后的导引线路径 |
| `terminal_path` | 局部坐标系中的箭头端部闭合路径 |
| `terminal_transform` | 终端平移、旋转和缩放矩阵 |
| `source_anchor` | 来源节点边界锚点 |
| `target_anchor` | 目标节点边界锚点 |
| `target_tangent` | 终点方向单位向量 |
| `bounds` | 箭头整体包围盒 |
| `warnings` | 碰撞、尺寸、曲率等检查结果 |

### 3.3 输出对象 `ResolvedArrow`

输出对象包含格式无关的几何数据和渲染提示。SVG、PPTX、Canvas 等适配器不得重新计算路径语义，只进行格式转换。

---

## 四、设计资产库

### 4.1 资产分层

| 层级 | 资产类型 | 使用场景 |
|---|---|---|
| `terminal` | 标准箭头端部 | 大多数流程、数据流、汇聚关系 |
| `recipe` | 路径与端部组合配方 | 汇聚、分流、双向、反馈闭环 |
| `special` | 完整 SVG 特殊箭头 | 关键结论、战略路径、特殊语义图形 |
| `theme` | 色彩、尺寸和视觉权重令牌 | 正式汇报、技术架构、轻量信息图等风格 |

### 4.2 首期终端预设

| 预设 | 形态 | 默认用途 |
|---|---|---|
| `floating-stream` | 浮动流线端部 | 通用默认，适合正式汇报材料 |
| `micro-triangle` | 紧凑实心三角 | 高密度流程图 |
| `open-chevron` | 开放式细折角 | 弱方向提示 |
| `needle` | 窄长尖端 | 数据流、技术路径 |
| `diamond` | 菱形端部 | 条件、里程碑和状态关系 |
| `none` | 无端部 | 支撑、归因和弱关联 |

### 4.3 首期连接配方

| 配方 | 路径策略 | 默认终端 |
|---|---|---|
| `process-standard` | 直线或圆角正交线 | `micro-triangle` |
| `data-light` | 低权重曲线或正交线 | `needle` |
| `fan-in-elegant` | 多路径向目标收束 | `floating-stream` |
| `fan-out-standard` | 单节点向多节点分流 | `floating-stream` |
| `bidirectional-light` | 两端弱方向 | `open-chevron` |
| `feedback-loop` | 回环曲线 | `micro-triangle` |
| `support-neutral` | 无方向低权重连接 | `none` |

### 4.4 特殊箭头资产

特殊资产以独立 SVG 文件保存，用于几何核心不适合表达的完整视觉对象。

| 资产 | 适用场景 |
|---|---|
| `ribbon-converge` | 多项能力合成为统一结果 |
| `strategic-rise` | 战略升级、能力跃迁 |
| `fork-choice` | 路径分叉、方案选择 |
| `loop-cycle` | 业务闭环、持续改进 |
| `milestone-path` | 阶段推进、关键节点 |

特殊资产必须使用标准 `viewBox`、纯矢量路径和主题令牌，不得包含固定页面坐标、文本或业务内容。

---

## 五、箭头几何模型

### 5.1 组合模型

通用箭头由两部分组成：

1. 导引路径：表达来源与目标之间的空间关系。
2. 方向端部：表达方向，并作为独立 SVG 闭合路径进行精细设计。

B“浮动终端箭”作为默认组合。导引路径保持低视觉权重，方向端部在目标节点附近形成清晰焦点。

### 5.2 路径类型

| 路径类型 | 输入参数 | 使用场景 |
|---|---|---|
| `straight` | 起点、终点 | 简单流程、短距离关系 |
| `quadratic` | 起点、控制点、终点 | 单弯曲线 |
| `cubic` | 起点、两个控制点、终点 | 汇聚、分流和复杂曲线 |
| `orthogonal` | 起点、终点、转向规则 | 系统架构和审批流程 |
| `rounded_orthogonal` | 正交路径、圆角半径 | 正式架构图和复杂流程 |
| `loop` | 起点、终点、回环方向 | 反馈闭环 |

### 5.3 目标节点锚定

终点不得直接使用目标节点中心。几何核心根据目标边界计算实际锚点。

| 边界类型 | 求交方法 |
|---|---|
| 矩形 | 路径射线与四边求交 |
| 圆角矩形 | 直边与圆角弧联合求交 |
| 圆形、椭圆 | 射线与二次曲线求交 |
| 多边形 | 射线与边集合求最近交点 |
| 自定义路径 | 使用调用方提供的锚点或轮廓采样 |

目标锚点向外退让 `target_clearance`，为箭头尖端与节点边界保留稳定留白。

### 5.4 终点切线

方向端部沿路径终点切线旋转。不同路径的切线计算如下：

| 路径 | 终点切线 |
|---|---|
| 直线 | `end - start` |
| 二次贝塞尔 | `end - control` |
| 三次贝塞尔 | `end - control2` |
| 正交线 | 最后一段非零向量 |
| 回环路径 | 终点前最后一段曲线导数 |

终点切线归一化后转换为旋转角。方向端部始终在局部坐标系内沿 X 轴绘制，再应用平移、旋转和缩放。

### 5.5 路径裁切

导引路径在方向端部起点之前终止。裁切长度由终端预设决定：

```text
guide_end = target_anchor
            - target_tangent × target_clearance
            - target_tangent × terminal_length
            - target_tangent × terminal_gap
```

该策略避免导引线穿过箭头端部，保持箭头轮廓干净。

### 5.6 尺寸分档

以 1280×720 画布为基准建立三档尺寸。其他画布按短边比例缩放，并设置上下限。

| 档位 | 导引线宽 | 端部长度 | 端部最大宽度 | 节点留白 | 适用场景 |
|---|---:|---:|---:|---:|---|
| `light` | 1.2–1.5 | 14–18 | 6–8 | 5–7 | 数据流、弱方向 |
| `standard` | 1.6–2.0 | 18–24 | 8–11 | 6–9 | 通用流程、汇聚 |
| `emphasis` | 2.2–3.2 | 24–34 | 12–18 | 8–12 | 关键路径、单一主关系 |

同一页面同一语义层级应使用同一尺寸档位。多箭头页面默认使用 `light` 或 `standard`。

---

## 六、默认视觉方案

### 6.1 `floating-stream`

`floating-stream` 由低权重导引线和独立流线端部组成。

| 参数 | 默认值 |
|---|---|
| 端部轮廓 | 前尖、后部内收、上下对称 |
| 导引线端帽 | round |
| 导引线与端部间距 | 1–3 px |
| 端部颜色 | 主色或关系强调色 |
| 导引线颜色 | 端部颜色向背景混合后的降权色 |
| 端部透明度 | 0.78–1.0 |
| 导引线透明度 | 0.42–0.72 |
| 目标边界留白 | 6–9 px |

建议局部坐标轮廓：

```svg
<path d="
  M 0,-3
  C 8,-2.8 14,-1.8 22,0
  C 14,1.8 8,2.8 0,3
  L 5,0
  Z"/>
```

实际渲染时根据尺寸档位缩放，并沿终点切线旋转。

### 6.2 颜色策略

| 页面背景 | 导引线 | 方向端部 |
|---|---|---|
| 白色或浅色 | 主色降权 35%–55% | 主色 70%–100% |
| 深色 | 白色或辅助色 35%–55% | 白色或强调色 75%–100% |
| 图片背景 | 使用局部对比色并增加外轮廓 | 使用高对比实色 |

箭头不得使用与正文相同的高对比度权重。关键路径除外。

### 6.3 视觉限制

| 限制项 | 默认规则 |
|---|---|
| 单页箭头样式 | 不超过 2 种终端预设 |
| 单页尺寸档位 | 不超过 2 档 |
| 箭头与文字 | 不得穿越标题、正文、图标和数据标签 |
| 箭头与节点 | 尖端不得进入节点内部 |
| 多箭头汇聚 | 终点间距应保持一致，避免箭头堆叠 |
| 短路径 | 路径长度不足端部长度 2.5 倍时缩小端部或取消箭头 |
| 块箭头 | 仅由 `special` 资产承担，不作为通用连接器默认 |

---

## 七、渲染适配

### 7.1 SVG 输出

SVG 渲染器输出一个独立 `<g>`，包含导引路径和方向端部。

组件中间对象可以保留方向端部的局部路径和变换矩阵。面向 PPT Master 的 SVG 渲染默认将平移、旋转和缩放烘焙到绝对路径坐标，避免现有 PPTX 转换器对复合变换的解释与浏览器产生位置偏差。仅在调用方能够完整处理 SVG 仿射变换时，才保留局部路径和 `transform`。

```svg
<g id="p05-input-a"
   data-role="arrow-component"
   data-arrow-preset="floating-stream"
   data-semantic-role="fan_in_result"
   data-directionality="inbound">
  <path data-part="guide"
        d="M258 140 C302 146 318 176 328 184"
        fill="none"
        stroke="#9FC2DF"
        stroke-width="1.4"
        stroke-linecap="round"/>
  <path data-part="terminal"
        d="M328 181 C334 181.4 339 182.5 346 184
           C339 185.5 334 186.6 328 187 L333 184 Z"
        fill="#236AA5"/>
</g>
```

SVG 输出不得依赖 `<style>`、脚本、事件或外部 CSS。通用预设优先采用路径和多边形，保证跨渲染器兼容。

### 7.2 PPTX 输出

PPTX 适配器采用以下优先级：

1. 导引路径转换为 DrawingML 自由曲线或连接线。
2. 方向端部转换为 DrawingML 自定义几何。
3. 不支持自定义几何时，以独立 SVG 图片保留矢量显示。
4. 调用方明确要求原生线端时，降级为 DrawingML `headEnd` 或 `tailEnd`。

导引路径和方向端部默认组成一个逻辑组，PowerPoint 中可整体移动，也可解除组合后分别编辑。

### 7.3 JSON 几何输出

JSON 适配器输出归一化路径、变换矩阵、颜色和包围盒，供 Canvas、Web 编辑器、Figma 插件或其他工具消费。

---

## 八、质量检查与异常处理

### 8.1 几何检查

| 检查项 | 触发条件 | 处理 |
|---|---|---|
| 路径长度不足 | 小于端部长度 2.5 倍 | 自动降档或返回 warning |
| 终点切线无效 | 最后一段长度为零 | 回退到前一有效线段 |
| 目标边界缺失 | 未提供边界 | 使用显式终点并返回提示 |
| 箭头进入节点 | 留白小于最小值 | 重新裁切路径 |
| 箭头越界 | 包围盒超出画布 | 调整控制点或返回 error |
| 穿越禁入区域 | 与 `avoid_regions` 相交 | 返回候选绕行路径或 warning |
| 曲率突变 | 终端前曲率超过阈值 | 延长终端直线段或调整控制点 |

### 8.2 视觉检查

| 检查项 | 合格标准 |
|---|---|
| 端部比例 | 长宽比符合预设范围 |
| 终端一致性 | 同组箭头的尺寸、颜色和目标留白一致 |
| 路径层级 | 导引线视觉权重低于方向端部 |
| 节点关系 | 箭头尖端清晰指向目标边界 |
| 页面秩序 | 箭头不遮挡文本、图标和数据标签 |
| 输出一致性 | SVG 预览与 PPTX 渲染方向、尺寸和颜色一致 |

### 8.3 错误分级

| 等级 | 场景 | 组件行为 |
|---|---|---|
| `error` | 参数非法、路径无法构造、输出越界严重 | 停止生成当前箭头 |
| `warning` | 路径过短、轻微碰撞、目标边界缺失 | 输出结果并附带提示 |
| `info` | 使用默认值或发生尺寸降档 | 记录处理结果 |

---

## 九、PPT Master 接入设计

### 9.1 接入边界

PPT Master 继续由 Strategist 判断连接器语义，由 Executor 逐页构建页面。独立组件只提供箭头几何和设计资产，不生成完整页面，不批量重写 SVG。

### 9.2 执行契约扩展

在现有 `connector_policy` 基础上增加箭头渲染字段：

```md
## connector_policy
- arrow_render_mode: custom_svg
- default_terminal_preset: floating-stream
- fallback_terminal_preset: micro-triangle
- arrow_scale: standard
- target_clearance: 7
- process_flow: route=rounded_orthogonal terminal=micro-triangle
- data_flow: route=curved terminal=needle
- fan_in_result: route=curved terminal=floating-stream
```

### 9.3 SVG 标记

每个箭头组件使用统一元数据：

```xml
<g data-role="arrow-component"
   data-arrow-preset="floating-stream"
   data-semantic-role="fan_in_result"
   data-directionality="inbound">
```

多个箭头组成关系组时，外层保留现有连接器语义标记：

```xml
<g data-role="connector-group"
   data-connector-role="fan_in_result"
   data-directionality="inbound">
```

### 9.4 主流程使用约束

| 环节 | 接入动作 |
|---|---|
| Strategist | 在 `Connector intent` 中声明方向性和终端预设 |
| Executor | 依据页面路径和节点边界调用或引用箭头组件 |
| Live Preview | 直接显示组件输出的 SVG 图形 |
| Quality Checker | 检查元数据、节点留白、样式一致性和禁入区域 |
| Finalize | 保留路径和分组，不将箭头栅格化 |
| PPTX Export | 导引路径与方向端部转换为原生可编辑形状 |

### 9.5 与逐页手工构建规则的关系

组件不得输出整页 SVG 文件。PPT Master 可采用以下两种方式：

1. Executor 依据资产库规范手工写入标准 `<g>` 片段。
2. 几何核心返回 JSON 或路径数据，由 Executor 写入当前页面。

两种方式均不改变页面内容、布局和视觉设计由主代理逐页完成的要求。

---

## 十、独立组件接口

### 10.1 Python API

建议公开以下最小接口：

```python
resolve_arrow(spec: dict) -> dict
render_svg(geometry: dict) -> str
render_pptx_geometry(geometry: dict) -> dict
validate_arrow(spec_or_geometry: dict) -> list[dict]
list_presets(kind: str = "all") -> list[dict]
```

### 10.2 命令行接口

```bash
python3 components/vector-arrow/cli/arrow_render.py spec.json --format svg -o arrow.svg
python3 components/vector-arrow/cli/arrow_render.py spec.json --format json -o geometry.json
python3 components/vector-arrow/cli/arrow_validate.py spec.json
```

命令行工具只处理单个箭头、箭头组或独立示例，不接收 PPT Master 项目目录，不生成整页演示文稿。

### 10.3 版本策略

| 版本变化 | 处理方式 |
|---|---|
| 新增预设 | 次版本升级 |
| 新增可选字段 | 次版本升级 |
| 修改默认预设几何 | 次版本升级并保留旧预设版本 |
| 删除字段或改变字段语义 | 主版本升级 |
| 修复不影响输出结构的错误 | 修订版本升级 |

每个预设包含独立 `preset_version`，保证历史项目可重现。

---

## 十一、验证方案

### 11.1 视觉样例矩阵

首期建立箭头矩阵样例，覆盖：

| 维度 | 样例 |
|---|---|
| 路径 | 直线、二次曲线、三次曲线、正交线、圆角正交线、回环 |
| 方向 | 上、下、左、右及四个斜向 |
| 尺寸 | `light`、`standard`、`emphasis` |
| 背景 | 白色、浅灰、深色 |
| 终端 | 首期全部预设 |
| 关系 | 单向、双向、汇聚、分流、反馈 |
| 输出 | SVG、PowerPoint 原生形状 |

### 11.2 冒烟验证

仓库遵循现有约定，不新增测试目录。使用独立样例和命令行冒烟检查：

1. 生成箭头矩阵 SVG。
2. 运行 XML 与数据契约校验。
3. 导出 PowerPoint 示例页。
4. 将 SVG 与 PowerPoint 渲染结果并排检查。
5. 验证解除组合后导引线和方向端部可分别编辑。

### 11.3 验收指标

| 指标 | 验收标准 |
|---|---|
| 格式兼容 | SVG 可在浏览器、LibreOffice 和 PowerPoint 中正常显示 |
| 编辑能力 | PowerPoint 中箭身和端部保持矢量可编辑 |
| 视觉一致 | 同一预设跨方向、跨尺寸保持稳定比例 |
| 节点留白 | 箭头尖端与目标边界间距符合规格 |
| 关系清晰 | 方向可识别，导引线不抢占主体视觉层级 |
| 迁移能力 | 独立目录复制后可在无 PPT Master 环境下运行 |

---

## 十二、实施阶段

### 12.1 第一阶段：组件骨架与默认预设

| 工作项 | 产出 |
|---|---|
| 建立独立目录 | `components/vector-arrow/` |
| 固化数据契约 | `ArrowSpec`、`ArrowGeometry` schema |
| 实现基础路径 | 直线、二次和三次贝塞尔 |
| 实现默认端部 | `floating-stream`、`micro-triangle`、`none` |
| 建立 SVG 渲染器 | 独立 `<g>` 输出 |
| 建立视觉矩阵 | SVG 样例页 |

### 12.2 第二阶段：PPTX 与复杂路径

| 工作项 | 产出 |
|---|---|
| 增加正交路径 | `orthogonal`、`rounded_orthogonal` |
| 增加节点边界 | 矩形、圆角矩形、圆形、椭圆 |
| 增加 PPTX 适配 | DrawingML 路径和分组 |
| 扩展预设 | `open-chevron`、`needle`、`diamond` |
| 增加配方 | 汇聚、分流、双向、反馈 |

### 12.3 第三阶段：主 Skill 接入

| 工作项 | 产出 |
|---|---|
| 扩展连接器契约 | `connector_policy` 箭头渲染字段 |
| 增加 Executor 规则 | 终端预设选择和几何使用规范 |
| 增加质量检查 | 元数据、留白、碰撞和样式一致性检查 |
| 更新正式示例 | 使用截图页验证 `fan-in-elegant` |
| 保留旧路径 | 标准 marker 作为兼容降级方案 |

### 12.4 第四阶段：跨工具复用

| 工作项 | 产出 |
|---|---|
| 发布组件接口说明 | 独立 README 和版本说明 |
| 提供 JSON 适配 | Web、Canvas 和插件消费格式 |
| 扩展特殊资产 | 战略路径、闭环、分叉等 |
| 建立主题包 | 正式汇报、技术架构、轻量信息图 |

---

## 十三、迁移与兼容

### 13.1 主线升级后的迁移顺序

1. 将 `components/vector-arrow/` 整体迁移至目标仓库。
2. 运行独立箭头矩阵冒烟检查。
3. 接入目标仓库 SVG 输出适配器。
4. 接入 PowerPoint 几何转换层。
5. 扩展目标仓库连接器执行契约。
6. 使用正式截图页进行端到端验证。

### 13.2 兼容策略

| 情况 | 处理方式 |
|---|---|
| 调用方不支持自定义路径 | 降级为标准三角 marker |
| PPTX 转换器不支持自定义几何 | 保留独立 SVG 矢量对象 |
| 旧项目无箭头预设字段 | 使用现有 `arrow=end` 行为 |
| 预设版本缺失 | 使用组件当前默认版本并记录 warning |
| 特殊资产不可用 | 回退到对应标准配方 |

### 13.3 回滚路径

组件接入采用新增字段和新增适配器，不删除现有 marker 支持。出现兼容问题时，可将 `arrow_render_mode` 切换为 `native_marker`，恢复现有导出行为。

---

## 十四、决策结论

1. 采用几何核心与设计资产库并行的组合架构。
2. 将 B“浮动终端箭”固化为首个通用默认预设 `floating-stream`。
3. 组件以独立目录交付，不依赖 PPT Master 项目结构。
4. 常规箭头由几何核心和终端预设组合生成，特殊箭头由完整 SVG 资产提供。
5. SVG、PPTX 和 JSON 使用统一中间几何对象，避免多套算法漂移。
6. PPT Master 保持逐页构建纪律，组件不生成整页 SVG。
7. 标准 marker 继续作为兼容降级方案。
8. 后续实施按组件骨架、PPTX 适配、主 Skill 接入和跨工具复用四个阶段推进。
