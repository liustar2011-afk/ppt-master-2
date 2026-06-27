# 可复用 SVG/PPT 箭头组件实施计划

> 面向执行代理：实施时使用 `superpowers:subagent-driven-development`，或使用 `superpowers:executing-plans` 按任务逐项执行。全部步骤使用复选框跟踪。

目标：建设可独立迁移的矢量箭头组件，形成通用几何核心、设计资产库、SVG 与 PPTX 适配能力，并接入 PPT Master 主流程。

架构：组件采用“几何核心＋设计资产库＋渲染适配器”架构。核心组件位于 `components/vector-arrow/`，不依赖 PPT Master 项目目录；PPT Master 通过 SVG 绝对路径、执行契约和质量检查接入。

技术栈：Python 3 标准库、JSON、SVG 1.1、DrawingML、现有 `svg_to_pptx` 转换器、现有 PPT Master 质量检查工具。

## 全局约束

1. 不新增第三方依赖，不修改环境变量和权限。
2. 不新增 `tests/` 目录、测试文件或测试框架依赖。
3. 使用命令行冒烟检查、SVG 样例矩阵和端到端导出验证。
4. 组件不得生成完整演示文稿页面，只生成箭头、箭头组或独立样例。
5. PPT Master 继续由主代理逐页构建 SVG，不引入批量页面生成器。
6. 面向 PPT Master 的 SVG 输出默认将终端变换烘焙为绝对路径坐标。
7. `floating-stream` 为正式汇报场景默认箭头端部。
8. 标准 SVG marker 继续作为兼容降级路径。
9. 组件代码、接口和资产命名使用英文；用户文档使用中文；运行时参考文件遵循所在目录语言。
10. 每个任务只提交本任务文件，不纳入工作区中既有的无关修改。

---

## 文件结构

实施后形成以下文件：

```text
components/vector-arrow/
├── README.md
├── VERSION
├── schema/
│   ├── arrow-spec.schema.json
│   └── arrow-geometry.schema.json
├── vector_arrow/
│   ├── __init__.py
│   ├── models.py
│   ├── boundaries.py
│   ├── routes.py
│   ├── terminals.py
│   ├── geometry.py
│   ├── validation.py
│   ├── svg_renderer.py
│   ├── pptx_geometry.py
│   └── catalog.py
├── presets/
│   ├── terminals.json
│   ├── recipes.json
│   └── themes.json
├── assets/
│   └── special/
│       ├── ribbon-converge.svg
│       ├── fork-choice.svg
│       ├── loop-cycle.svg
│       ├── strategic-rise.svg
│       └── milestone-path.svg
├── cli/
│   ├── arrow_render.py
│   ├── arrow_validate.py
│   └── arrow_catalog.py
└── examples/
    ├── arrow_matrix.json
    ├── arrow_matrix.svg
    ├── fuel_fan_in.json
    └── fuel_fan_in.svg

skills/ppt-master/
├── references/
│   ├── arrow-components.md
│   ├── executor-base.md
│   ├── shared-standards.md
│   └── strategist.md
├── templates/
│   ├── design_spec_reference.md
│   ├── spec_lock_reference.md
│   └── charts/hub_inward_arrows.svg
├── scripts/
│   └── svg_quality_checker.py
└── SKILL.md
```

---

### 任务一：建立独立组件契约与预设目录

文件：

- 新建：`components/vector-arrow/VERSION`
- 新建：`components/vector-arrow/README.md`
- 新建：`components/vector-arrow/schema/arrow-spec.schema.json`
- 新建：`components/vector-arrow/schema/arrow-geometry.schema.json`
- 新建：`components/vector-arrow/vector_arrow/__init__.py`
- 新建：`components/vector-arrow/vector_arrow/models.py`
- 新建：`components/vector-arrow/vector_arrow/catalog.py`
- 新建：`components/vector-arrow/presets/terminals.json`
- 新建：`components/vector-arrow/presets/recipes.json`
- 新建：`components/vector-arrow/presets/themes.json`

接口：

- 输入：`dict` 形式的 `ArrowSpec`
- 输出：`ArrowSpec`、`ArrowGeometry`、`ValidationIssue` 数据对象
- 公开函数：`arrow_spec_from_dict`、`geometry_to_dict`、`list_presets`、`load_preset`

- [ ] 步骤 1：运行导入冒烟命令，确认组件尚未建立

运行：

```bash
PYTHONPATH=components/vector-arrow python3 -c "import vector_arrow"
```

预期：失败并提示 `ModuleNotFoundError: No module named 'vector_arrow'`。

- [ ] 步骤 2：建立数据模型

在 `models.py` 中定义：

```python
from __future__ import annotations

from dataclasses import asdict, dataclass, field
from typing import Any


@dataclass(frozen=True)
class Point:
    x: float
    y: float


@dataclass(frozen=True)
class RouteSpec:
    type: str
    start: Point
    end: Point
    controls: tuple[Point, ...] = ()
    corner_radius: float = 12.0
    waypoints: tuple[Point, ...] = ()


@dataclass(frozen=True)
class BoundarySpec:
    type: str
    x: float
    y: float
    width: float
    height: float
    radius: float = 0.0


@dataclass(frozen=True)
class TerminalSpec:
    preset: str = "floating-stream"
    scale: str = "standard"
    position: str = "end"


@dataclass(frozen=True)
class ArrowStyle:
    color: str = "#236AA5"
    guide_color: str = "#9FC2DF"
    guide_width: float = 1.4
    opacity: float = 0.72


@dataclass(frozen=True)
class ArrowConstraints:
    target_clearance: float = 7.0
    terminal_gap: float = 2.0
    avoid_regions: tuple[BoundarySpec, ...] = ()


@dataclass(frozen=True)
class ArrowSpec:
    version: str
    id: str
    semantic_role: str
    directionality: str
    route: RouteSpec
    terminal: TerminalSpec
    style: ArrowStyle
    source_boundary: BoundarySpec | None = None
    target_boundary: BoundarySpec | None = None
    constraints: ArrowConstraints = field(default_factory=ArrowConstraints)
    metadata: dict[str, Any] = field(default_factory=dict)


@dataclass(frozen=True)
class ValidationIssue:
    severity: str
    code: str
    message: str


@dataclass(frozen=True)
class ArrowGeometry:
    id: str
    semantic_role: str
    directionality: str
    guide_path: str
    terminal_path: str
    terminal_closed: bool
    source_anchor: Point
    target_anchor: Point
    target_tangent: Point
    target_clearance: float
    bounds: tuple[float, float, float, float]
    terminal_preset: str
    terminal_scale: str
    style: ArrowStyle
    warnings: tuple[ValidationIssue, ...] = ()


def geometry_to_dict(geometry: ArrowGeometry) -> dict[str, Any]:
    return asdict(geometry)
```

`arrow_spec_from_dict` 必须显式读取 `route`、`terminal`、`style` 和 `constraints`，`directionality` 缺失时依据 `semantic_role` 使用稳定默认值；缺失 `version`、`id`、`semantic_role` 或 `route` 时抛出 `ValueError`。

- [ ] 步骤 3：建立首期预设目录

`terminals.json` 至少包含：

```json
{
  "floating-stream": {
    "preset_version": "1.0",
    "path": "M0,-3 C8,-2.8 14,-1.8 22,0 C14,1.8 8,2.8 0,3 L5,0 Z",
    "length": 22,
    "width": 6,
    "closed": true
  },
  "micro-triangle": {
    "preset_version": "1.0",
    "path": "M0,-3 L9,0 L0,3 Z",
    "length": 9,
    "width": 6,
    "closed": true
  },
  "none": {
    "preset_version": "1.0",
    "path": "",
    "length": 0,
    "width": 0,
    "closed": false
  }
}
```

`recipes.json` 首期包含 `process-standard`、`data-light`、`fan-in-elegant`、`support-neutral`；`themes.json` 首期包含 `formal-light`、`formal-dark`、`technical-light`。

- [ ] 步骤 4：实现预设读取接口

`catalog.py` 提供：

```python
def list_presets(kind: str = "all") -> list[dict[str, object]]:
    """List terminal, recipe, and theme presets."""


def load_preset(kind: str, name: str) -> dict[str, object]:
    """Load one named preset or raise ValueError with valid names."""
```

路径以 `Path(__file__).resolve().parent.parent / "presets"` 解析，不依赖当前工作目录。

- [ ] 步骤 5：验证数据契约与预设可读取

运行：

```bash
PYTHONPATH=components/vector-arrow python3 -c \
"from vector_arrow.catalog import list_presets, load_preset; \
items=list_presets('terminal'); \
assert [x['name'] for x in items] == ['floating-stream','micro-triangle','none']; \
assert load_preset('terminal','floating-stream')['length'] == 22; \
print('contract-ok')"
```

预期输出：

```text
contract-ok
```

继续运行：

```bash
jq -e . components/vector-arrow/schema/arrow-spec.schema.json >/dev/null
jq -e . components/vector-arrow/schema/arrow-geometry.schema.json >/dev/null
jq -e . components/vector-arrow/presets/terminals.json >/dev/null
```

预期：三个命令均退出 0。

- [ ] 步骤 6：提交组件契约

```bash
git add components/vector-arrow
git commit -m "feat: add vector arrow component contracts"
```

---

### 任务二：实现路径、边界与裁切几何

文件：

- 新建：`components/vector-arrow/vector_arrow/routes.py`
- 新建：`components/vector-arrow/vector_arrow/boundaries.py`
- 修改：`components/vector-arrow/vector_arrow/__init__.py`

接口：

- 输入：`RouteSpec`、`BoundarySpec`、裁切距离
- 输出：绝对路径字符串、锚点、单位切线
- 公开函数：`resolve_route`、`route_point`、`route_tangent`、`trim_route_end`、`resolve_boundary_anchor`

- [ ] 步骤 1：运行路径接口冒烟命令，确认接口尚不存在

```bash
PYTHONPATH=components/vector-arrow python3 -c \
"from vector_arrow.routes import resolve_route"
```

预期：失败并提示无法导入 `vector_arrow.routes`。

- [ ] 步骤 2：实现向量基础函数与边界求交

`boundaries.py` 提供：

```python
from __future__ import annotations

import math

from .models import BoundarySpec, Point


def normalize(vector: Point) -> Point:
    length = math.hypot(vector.x, vector.y)
    if length <= 1e-9:
        raise ValueError("Cannot normalize a zero-length vector")
    return Point(vector.x / length, vector.y / length)


def resolve_boundary_anchor(
    source: Point,
    target: Point,
    boundary: BoundarySpec | None,
) -> Point:
    if boundary is None:
        return target
    if boundary.type in {"rect", "rounded_rect"}:
        cx = boundary.x + boundary.width / 2
        cy = boundary.y + boundary.height / 2
        dx = target.x - source.x
        dy = target.y - source.y
        if abs(dx) <= 1e-9 and abs(dy) <= 1e-9:
            raise ValueError("Source and target centers must differ")
        candidates: list[tuple[float, Point]] = []
        for x in (boundary.x, boundary.x + boundary.width):
            if abs(dx) > 1e-9:
                t = (x - source.x) / dx
                y = source.y + math.prod((t, dy))
                if t >= 0 and boundary.y <= y <= boundary.y + boundary.height:
                    candidates.append((t, Point(x, y)))
        for y in (boundary.y, boundary.y + boundary.height):
            if abs(dy) > 1e-9:
                t = (y - source.y) / dy
                x = source.x + math.prod((t, dx))
                if t >= 0 and boundary.x <= x <= boundary.x + boundary.width:
                    candidates.append((t, Point(x, y)))
        if not candidates:
            return Point(cx, cy)
        return min(candidates, key=lambda item: item[0])[1]
    if boundary.type in {"circle", "ellipse"}:
        cx = boundary.x + boundary.width / 2
        cy = boundary.y + boundary.height / 2
        rx = boundary.width / 2
        ry = boundary.height / 2
        direction = normalize(Point(source.x - cx, source.y - cy))
        scale = 1.0 / math.sqrt(
            math.pow(direction.x / rx, 2) + math.pow(direction.y / ry, 2)
        )
        return Point(
            cx + math.prod((direction.x, scale)),
            cy + math.prod((direction.y, scale)),
        )
    raise ValueError(f"Unsupported boundary type: {boundary.type}")
```

圆角矩形首期按矩形求交，任务四再增加圆角弧精确求交。

- [ ] 步骤 3：实现路径解析和终点切线

`routes.py` 为每种路径建立统一采样函数：

```python
def route_point(route: RouteSpec, t: float) -> Point:
    """Return a point at normalized parameter t."""
    if not 0.0 <= t <= 1.0:
        raise ValueError("Route parameter t must be between 0 and 1")
    if route.type == "straight":
        return _linear_point(route.start, route.end, t)
    if route.type == "quadratic" and len(route.controls) == 1:
        return _quadratic_point(route.start, route.controls[0], route.end, t)
    if route.type == "cubic" and len(route.controls) == 2:
        return _cubic_point(
            route.start,
            route.controls[0],
            route.controls[1],
            route.end,
            t,
        )
    raise ValueError(f"Unsupported route definition: {route.type}")


def route_tangent(route: RouteSpec, t: float = 1.0) -> Point:
    """Return a normalized tangent at t."""
    if route.type == "straight":
        vector = Point(route.end.x - route.start.x, route.end.y - route.start.y)
    elif route.type == "quadratic" and len(route.controls) == 1:
        vector = _quadratic_derivative(route, t)
    elif route.type == "cubic" and len(route.controls) == 2:
        vector = _cubic_derivative(route, t)
    else:
        raise ValueError(f"Unsupported route definition: {route.type}")
    return normalize(vector)


def resolve_route(route: RouteSpec) -> str:
    """Return an absolute SVG path string."""
    if route.type == "straight":
        return _format_path("M", route.start, "L", route.end)
    if route.type == "quadratic" and len(route.controls) == 1:
        return _format_path("M", route.start, "Q", route.controls[0], route.end)
    if route.type == "cubic" and len(route.controls) == 2:
        return _format_path(
            "M",
            route.start,
            "C",
            route.controls[0],
            route.controls[1],
            route.end,
        )
    raise ValueError(f"Unsupported route definition: {route.type}")
```

首期支持：

```text
straight  → M x0 y0 L x1 y1
quadratic → M x0 y0 Q cx cy x1 y1
cubic     → M x0 y0 C c1x c1y c2x c2y x1 y1
```

二次和三次贝塞尔点位与导数必须使用标准公式，不允许通过字符串插值估算切线。

- [ ] 步骤 4：实现路径末端裁切

`trim_route_end(route, trim_distance)` 使用 96 段采样估算路径长度，从终点向前定位裁切参数，再通过 De Casteljau 分割保留前段。返回：

```python
@dataclass(frozen=True)
class TrimmedRoute:
    route: RouteSpec
    end: Point
    tangent: Point
    original_length: float
    retained_length: float
```

路径总长小于裁切距离时抛出 `ValueError("Route is shorter than terminal allowance")`。

- [ ] 步骤 5：验证直线、曲线和矩形边界

运行：

```bash
PYTHONPATH=components/vector-arrow python3 - <<'PY'
from vector_arrow.boundaries import resolve_boundary_anchor
from vector_arrow.models import BoundarySpec, Point, RouteSpec
from vector_arrow.routes import resolve_route, route_tangent, trim_route_end

boundary = BoundarySpec("rect", 100, 40, 20, 20)
anchor = resolve_boundary_anchor(Point(0, 50), Point(110, 50), boundary)
assert anchor == Point(100, 50)

route = RouteSpec("straight", Point(0, 50), anchor)
trimmed = trim_route_end(route, 22)
assert trimmed.end == Point(78, 50)
assert trimmed.tangent == Point(1, 0)
assert resolve_route(trimmed.route) == "M 0 50 L 78 50"

curve = RouteSpec(
    "cubic",
    Point(0, 0),
    Point(100, 50),
    (Point(40, 0), Point(70, 50)),
)
tangent = route_tangent(curve)
assert tangent.x > 0 and abs(tangent.y) < 1e-9
print("geometry-ok")
PY
```

预期输出：

```text
geometry-ok
```

- [ ] 步骤 6：提交基础几何

```bash
git add components/vector-arrow/vector_arrow
git commit -m "feat: add vector arrow route geometry"
```

---

### 任务三：实现方向端部与箭头解析

文件：

- 新建：`components/vector-arrow/vector_arrow/terminals.py`
- 新建：`components/vector-arrow/vector_arrow/geometry.py`
- 新建：`components/vector-arrow/vector_arrow/validation.py`
- 修改：`components/vector-arrow/vector_arrow/__init__.py`

接口：

- 输入：`ArrowSpec`
- 输出：`ArrowGeometry`
- 公开函数：`resolve_arrow`、`build_terminal_path`、`validate_arrow`

- [ ] 步骤 1：运行解析接口冒烟命令，确认接口尚不存在

```bash
PYTHONPATH=components/vector-arrow python3 -c \
"from vector_arrow import resolve_arrow"
```

预期：失败并提示无法导入 `resolve_arrow`。

- [ ] 步骤 2：实现终端尺寸分档

`terminals.py` 定义：

```python
SCALE_FACTORS = {
    "light": 0.78,
    "standard": 1.0,
    "emphasis": 1.42,
}


def terminal_dimensions(preset: dict[str, object], scale: str) -> tuple[float, float]:
    factor = SCALE_FACTORS.get(scale)
    if factor is None:
        raise ValueError(f"Unsupported terminal scale: {scale}")
    return (
        math.prod((float(preset["length"]), factor)),
        math.prod((float(preset["width"]), factor)),
    )
```

- [ ] 步骤 3：实现局部路径转换为绝对路径

`build_terminal_path` 读取预设路径点，在内部将预设规范化为命令序列。所有点执行以下变换：

```python
absolute_x = origin.x + math.prod((tangent.x, local_x)) + math.prod((normal.x, local_y))
absolute_y = origin.y + math.prod((tangent.y, local_x)) + math.prod((normal.y, local_y))
```

终端路径解析器只接受首期预设使用的 `M`、`L`、`C` 和 `Z` 命令。发现其他命令时抛出 `ValueError`，不得静默忽略。

其中：

```python
normal = Point(-tangent.y, tangent.x)
```

终端基点按以下方式计算：

```python
terminal_origin = Point(
    target_anchor.x - math.prod((target_tangent.x, terminal_length)),
    target_anchor.y - math.prod((target_tangent.y, terminal_length)),
)
```

局部路径的 `x=0` 位于终端尾部，`x=terminal_length` 位于箭头尖端。输出路径不得包含 `transform`。数值统一格式化为最多 3 位小数，整数不保留小数点。

- [ ] 步骤 4：实现 `resolve_arrow`

执行顺序：

1. 计算目标边界锚点。
2. 使用目标留白将尖端沿反方向退让，并以该尖端重建路径终点。
3. 读取终端预设和尺寸档位。
4. 按“终端长度＋终端间距”裁切导引路径。
5. 以“目标锚点减去终端长度”为终端基点，使用终点切线构造绝对坐标终端路径。
6. 计算导引路径与终端路径联合包围盒。
7. 返回 `ArrowGeometry`。

函数签名：

```python
def resolve_arrow(spec: ArrowSpec) -> ArrowGeometry:
    """Resolve one arrow into renderer-independent geometry."""
```

`terminal.preset == "none"` 时不裁切终端长度，只应用目标留白，`terminal_path` 返回空字符串。

- [ ] 步骤 5：实现验证规则

`validate_arrow` 返回 `ValidationIssue` 列表，首期规则：

| 代码 | 条件 | 等级 |
|---|---|---|
| `route-too-short` | 路径长度小于终端长度 2.5 倍 | warning |
| `invalid-opacity` | 透明度不在 0 至 1 | error |
| `invalid-guide-width` | 线宽小于等于 0 | error |
| `missing-terminal` | 方向关系使用 `none` | warning |
| `unexpected-terminal` | 无方向关系使用箭头 | warning |
| `clearance-out-of-range` | 留白小于 4 或大于 14 | warning |

- [ ] 步骤 6：验证默认浮动箭头

运行：

```bash
PYTHONPATH=components/vector-arrow python3 - <<'PY'
from vector_arrow import arrow_spec_from_dict, resolve_arrow
from vector_arrow.validation import validate_arrow

spec = arrow_spec_from_dict({
    "version": "1.0",
    "id": "demo",
    "semantic_role": "fan_in_result",
    "route": {"type": "straight", "start": [0, 50], "end": [110, 50]},
    "target_boundary": {"type": "rect", "x": 100, "y": 40, "width": 20, "height": 20},
    "terminal": {"preset": "floating-stream", "scale": "standard", "position": "end"},
    "style": {
        "color": "#236AA5",
        "guide_color": "#9FC2DF",
        "guide_width": 1.4,
        "opacity": 0.72
    },
    "constraints": {"target_clearance": 6, "terminal_gap": 2}
})
issues = validate_arrow(spec)
assert not [issue for issue in issues if issue.severity == "error"]
geometry = resolve_arrow(spec)
assert geometry.guide_path == "M 0 50 L 70 50"
assert geometry.terminal_path.endswith("Z")
assert "transform" not in geometry.terminal_path
assert geometry.target_anchor.x == 94
print("floating-stream-ok")
PY
```

预期输出：

```text
floating-stream-ok
```

- [ ] 步骤 7：提交箭头解析核心

```bash
git add components/vector-arrow/vector_arrow
git commit -m "feat: resolve custom vector arrow terminals"
```

---

### 任务四：实现 SVG、JSON 和命令行输出

文件：

- 新建：`components/vector-arrow/vector_arrow/svg_renderer.py`
- 新建：`components/vector-arrow/vector_arrow/pptx_geometry.py`
- 新建：`components/vector-arrow/cli/arrow_render.py`
- 新建：`components/vector-arrow/cli/arrow_validate.py`
- 新建：`components/vector-arrow/cli/arrow_catalog.py`
- 新建：`components/vector-arrow/examples/arrow_matrix.json`
- 生成：`components/vector-arrow/examples/arrow_matrix.svg`
- 修改：`components/vector-arrow/README.md`

接口：

- 输入：单个 `ArrowSpec`，或包含 `canvas.width`、`canvas.height` 和完整 `arrows` 数组的箭头组
- 输出：SVG `<g>`、独立 SVG、JSON 几何对象
- 公开函数：`render_svg_group`、`render_svg_document`、`render_pptx_geometry`

- [ ] 步骤 1：运行 CLI 冒烟命令，确认入口尚不存在

```bash
python3 components/vector-arrow/cli/arrow_render.py --help
```

预期：失败并提示文件不存在。

- [ ] 步骤 2：实现 SVG 分组渲染

`render_svg_group` 输出结构：

```python
def render_svg_group(geometry: ArrowGeometry) -> str:
    terminal = ""
    if geometry.terminal_path:
        if geometry.terminal_closed:
            terminal_style = (
                f'fill="{geometry.style.color}" '
                f'fill-opacity="{geometry.style.opacity:.3f}"'
            )
        else:
            terminal_style = (
                f'fill="none" stroke="{geometry.style.color}" '
                f'stroke-width="{geometry.style.guide_width:g}" '
                f'stroke-linecap="round" stroke-linejoin="round" '
                f'stroke-opacity="{geometry.style.opacity:.3f}"'
            )
        terminal = (
            f'<path data-part="terminal" d="{geometry.terminal_path}" '
            f'{terminal_style}/>'
        )
    return (
        f'<g id="{geometry.id}" data-role="arrow-component" '
        f'data-arrow-preset="{geometry.terminal_preset}" '
        f'data-semantic-role="{geometry.semantic_role}" '
        f'data-directionality="{geometry.directionality}" '
        f'data-target-clearance="{geometry.target_clearance:g}">'
        f'<path data-part="guide" d="{geometry.guide_path}" fill="none" '
        f'stroke="{geometry.style.guide_color}" '
        f'stroke-width="{geometry.style.guide_width:g}" '
        f'stroke-linecap="round"/>'
        f'{terminal}</g>'
    )
```

实际实现从 `ArrowGeometry` 携带真实 `target_clearance` 和 `directionality`，不得写死。

- [ ] 步骤 3：实现独立 SVG 文档和 JSON 输出

`render_svg_document` 接收画布宽高和多个箭头，输出：

输出根节点必须使用 `http://www.w3.org/2000/svg` 命名空间，`viewBox`、`width` 和 `height` 采用调用方传入的实际画布值；根节点内按输入顺序写入每个 `render_svg_group` 结果。

不得输出 `<style>`、`class`、`marker`、脚本或事件属性。

`render_pptx_geometry` 返回：

```python
{
    "group_id": geometry.id,
    "guide": {"type": "path", "d": geometry.guide_path},
    "terminal": {"type": "path", "d": geometry.terminal_path},
    "grouped": True,
    "editable": True
}
```

- [ ] 步骤 4：实现 CLI 参数

`arrow_render.py`：

```text
usage: arrow_render.py INPUT --format {svg,svg-group,json,pptx-geometry} [-o OUTPUT]
```

`arrow_validate.py`：

```text
usage: arrow_validate.py INPUT [--strict]
```

`arrow_catalog.py`：

```text
usage: arrow_catalog.py [--kind {all,terminal,recipe,theme}] [--json]
```

三个入口均使用 `argparse`，`--help` 不产生文件和目录。

- [ ] 步骤 5：建立箭头矩阵样例

`arrow_matrix.json` 至少包含：

1. 八个方向的 `floating-stream`。
2. 三种尺寸档位。
3. `floating-stream`、`micro-triangle`、`none` 三种终端。
4. 白色和深色两个背景分区。

运行：

```bash
python3 components/vector-arrow/cli/arrow_render.py \
  components/vector-arrow/examples/arrow_matrix.json \
  --format svg \
  -o components/vector-arrow/examples/arrow_matrix.svg
```

- [ ] 步骤 6：验证 SVG 合规和绝对路径

```bash
python3 - <<'PY'
from pathlib import Path
from xml.etree import ElementTree as ET

path = Path("components/vector-arrow/examples/arrow_matrix.svg")
root = ET.fromstring(path.read_text(encoding="utf-8"))
assert root.tag.endswith("svg")
text = path.read_text(encoding="utf-8")
assert 'data-role="arrow-component"' in text
assert 'data-part="terminal"' in text
assert "marker-end" not in text
assert 'transform=' not in text
assert "<style" not in text
print("svg-render-ok")
PY
```

预期输出：

```text
svg-render-ok
```

- [ ] 步骤 7：提交渲染器和命令行工具

```bash
git add components/vector-arrow
git commit -m "feat: render reusable vector arrow svg"
```

---

### 任务五：扩展复杂路径、配方和特殊资产

文件：

- 修改：`components/vector-arrow/vector_arrow/routes.py`
- 修改：`components/vector-arrow/vector_arrow/boundaries.py`
- 修改：`components/vector-arrow/vector_arrow/geometry.py`
- 修改：`components/vector-arrow/presets/terminals.json`
- 修改：`components/vector-arrow/presets/recipes.json`
- 新建：`components/vector-arrow/assets/special/ribbon-converge.svg`
- 新建：`components/vector-arrow/assets/special/fork-choice.svg`
- 新建：`components/vector-arrow/assets/special/loop-cycle.svg`
- 新建：`components/vector-arrow/assets/special/strategic-rise.svg`
- 新建：`components/vector-arrow/assets/special/milestone-path.svg`
- 新建：`components/vector-arrow/examples/fuel_fan_in.json`
- 生成：`components/vector-arrow/examples/fuel_fan_in.svg`

接口：

- 新增路径：`orthogonal`、`rounded_orthogonal`、`loop`
- 新增终端：`open-chevron`、`needle`、`diamond`
- 新增配方：`fan-out-standard`、`bidirectional-light`、`feedback-loop`

- [ ] 步骤 1：运行复杂路径冒烟命令，确认尚不支持

```bash
PYTHONPATH=components/vector-arrow python3 - <<'PY'
from vector_arrow.models import Point, RouteSpec
from vector_arrow.routes import resolve_route

resolve_route(RouteSpec(
    "rounded_orthogonal",
    Point(0, 0),
    Point(100, 50),
    waypoints=(Point(60, 0), Point(60, 50)),
    corner_radius=10,
))
PY
```

预期：失败并提示 `Unsupported route type: rounded_orthogonal`。

- [ ] 步骤 2：实现正交与圆角正交路径

`orthogonal` 将 `start`、`waypoints`、`end` 输出为 `M/L` 路径。

`rounded_orthogonal` 对每个转角：

1. 沿入线和出线分别退让 `min(corner_radius, segment_length / 2)`。
2. 使用 `Q corner_x corner_y exit_x exit_y` 建立圆角。
3. 删除连续重复点和零长度线段。

- [ ] 步骤 3：实现回环路径

`loop` 使用 `route.controls` 中的两个控制点生成三次贝塞尔。没有控制点时，根据起终点相对位置和 `corner_radius` 自动构造外扩控制点，外扩距离不得小于 48。

- [ ] 步骤 4：增加圆角矩形精确求交

矩形直边候选点落入圆角区域时，改用对应四分之一椭圆求交。无法稳定求交时回退到矩形边界，并返回 `rounded-boundary-fallback` 信息项。

- [ ] 步骤 5：增加终端预设与特殊资产

`terminals.json` 增加：

```json
{
  "open-chevron": {
    "preset_version": "1.0",
    "path": "M0,-4 L10,0 L0,4",
    "length": 10,
    "width": 8,
    "closed": false
  },
  "needle": {
    "preset_version": "1.0",
    "path": "M0,-2 C8,-1.6 15,-0.8 24,0 C15,0.8 8,1.6 0,2 L7,0 Z",
    "length": 24,
    "width": 4,
    "closed": true
  },
  "diamond": {
    "preset_version": "1.0",
    "path": "M0,0 L6,-4 L12,0 L6,4 Z",
    "length": 12,
    "width": 8,
    "closed": true
  }
}
```

特殊资产只使用绝对路径、内联属性和标准 `viewBox`，不得包含文本。

- [ ] 步骤 6：生成燃料经营汇聚样例

`fuel_fan_in.json` 使用四条三次贝塞尔路径、`fan-in-elegant` 配方和 `floating-stream` 终端，输出一个 1280×720 独立箭头组样例。

```bash
python3 components/vector-arrow/cli/arrow_render.py \
  components/vector-arrow/examples/fuel_fan_in.json \
  --format svg \
  -o components/vector-arrow/examples/fuel_fan_in.svg
```

- [ ] 步骤 7：验证全部预设和配方

```bash
python3 components/vector-arrow/cli/arrow_catalog.py --kind all --json | jq -e \
'.terminals | length == 6'

python3 components/vector-arrow/cli/arrow_validate.py \
  components/vector-arrow/examples/fuel_fan_in.json \
  --strict
```

预期：目录校验返回 `true`，严格校验退出 0。

- [ ] 步骤 8：提交复杂路径和资产库

```bash
git add components/vector-arrow
git commit -m "feat: add vector arrow recipes and assets"
```

---

### 任务六：验证 PowerPoint 原生可编辑输出

文件：

- 修改：`components/vector-arrow/vector_arrow/pptx_geometry.py`
- 修改：`components/vector-arrow/README.md`
- 不修改：`skills/ppt-master/scripts/svg_to_pptx/drawingml_elements.py`
- 不修改：`skills/ppt-master/scripts/svg_to_pptx/drawingml_converter.py`

接口：

- 输入：`ArrowGeometry`
- 输出：使用绝对路径的 PPTX 中立几何记录
- 验证目标：现有转换器将导引线与终端转换为 `<a:custGeom>`，并将多子元素 `<g>` 转换为 `<p:grpSp>`

- [ ] 步骤 1：实现 PPTX 几何记录验证

`render_pptx_geometry` 必须拒绝包含 `transform`、相对路径命令或 marker 的输入：

```python
def render_pptx_geometry(geometry: ArrowGeometry) -> dict[str, object]:
    if "transform" in geometry.terminal_path:
        raise ValueError("PPTX geometry requires baked absolute coordinates")
    return {
        "group_id": geometry.id,
        "guide": {"type": "path", "d": geometry.guide_path},
        "terminal": {"type": "path", "d": geometry.terminal_path},
        "grouped": True,
        "editable": True,
    }
```

- [ ] 步骤 2：建立临时冒烟项目

```bash
rm -rf projects/_smoke_vector_arrow
python3 skills/ppt-master/scripts/project_manager.py init \
  _smoke_vector_arrow \
  --format ppt169
```

将 `fuel_fan_in.svg` 复制到临时项目 `svg_output/01_arrow.svg`，并补充最小 `notes/total.md`。临时项目不得纳入提交。

- [ ] 步骤 3：运行 SVG 质量检查与 PPTX 导出

```bash
python3 skills/ppt-master/scripts/svg_quality_checker.py \
  projects/_smoke_vector_arrow

python3 skills/ppt-master/scripts/total_md_split.py \
  projects/_smoke_vector_arrow

python3 skills/ppt-master/scripts/finalize_svg.py \
  projects/_smoke_vector_arrow

python3 skills/ppt-master/scripts/svg_to_pptx.py \
  projects/_smoke_vector_arrow
```

预期：质量检查 0 error，导出 1 页原生 PPTX。

- [ ] 步骤 4：检查 PPTX 内部原生几何和分组

```bash
PPTX_FILE="$(find projects/_smoke_vector_arrow/exports -maxdepth 1 -type f \
  | rg '\\.pptx$' | sort | tail -1)"
unzip -p "$PPTX_FILE" ppt/slides/slide1.xml > /tmp/vector-arrow-slide1.xml
rg -c "<a:custGeom>" /tmp/vector-arrow-slide1.xml
rg -c "<p:grpSp>" /tmp/vector-arrow-slide1.xml
```

预期：

```text
8
4
```

实际计数可以因外层关系组增加而更高，但不得低于 8 个自定义几何和 4 个箭头组件组。

- [ ] 步骤 5：渲染 PPTX 并检查视觉一致性

```bash
mkdir -p projects/_smoke_vector_arrow/render_check
soffice --headless --convert-to pdf \
  --outdir projects/_smoke_vector_arrow/render_check \
  "$PPTX_FILE"
PDF_FILE="$(find projects/_smoke_vector_arrow/render_check -maxdepth 1 \
  -type f | rg '\\.pdf$' | sort | tail -1)"
pdftoppm -png -r 144 \
  "$PDF_FILE" \
  projects/_smoke_vector_arrow/render_check/slide
```

人工检查：

1. 四个箭头方向与 SVG 一致。
2. 终端未发生旋转漂移。
3. 导引线与终端间距一致。
4. 箭头端部未进入中央节点。
5. PowerPoint 解除组合后可分别选择导引线和端部。

- [ ] 步骤 6：记录既有转换器兼容结论

在组件 README 中记录：

```text
PPT Master native export compatibility:
- absolute SVG path coordinates: supported
- multi-child arrow group: preserved as PowerPoint group
- terminal path: exported as DrawingML custom geometry
- composite path transform: not used by the component
```

- [ ] 步骤 7：提交 PPTX 适配说明

```bash
git add components/vector-arrow/vector_arrow/pptx_geometry.py \
        components/vector-arrow/README.md
git commit -m "docs: verify vector arrow pptx compatibility"
```

---

### 任务七：接入 SVG 质量检查

文件：

- 修改：`skills/ppt-master/scripts/svg_quality_checker.py`
- 新建：`skills/ppt-master/scripts/docs/vector-arrow-components.md`

接口：

- 输入：含 `data-role="arrow-component"` 的 SVG
- 输出：错误、警告和组件统计信息
- 新增方法：`_check_arrow_components`

- [ ] 步骤 1：建立不合规箭头冒烟样例

在 `/tmp/vector-arrow-invalid.svg` 写入一个缺少终端、带 `transform`、缺少 `data-arrow-preset` 的箭头组件。

运行：

```bash
python3 skills/ppt-master/scripts/svg_quality_checker.py \
  /tmp/vector-arrow-invalid.svg
```

预期：当前版本未报告箭头组件专项错误，证明检查缺口存在。

- [ ] 步骤 2：接入检查调用

在 `check_file` 的 `_check_forbidden_elements` 之后增加：

```python
self._check_arrow_components(content, result)
```

- [ ] 步骤 3：实现组件检查

方法签名：

```python
def _check_arrow_components(self, content: str, result: Dict) -> None:
    """Validate custom SVG arrow component structure and metadata."""
```

检查规则：

| 条件 | 等级 |
|---|---|
| 缺少 `data-arrow-preset` | error |
| 缺少 `data-semantic-role` | error |
| 缺少 `data-directionality` | error |
| 缺少一个 `data-part="guide"` | error |
| 有方向且缺少 `data-part="terminal"` | error |
| `floating-stream` 终端不是闭合 `<path>` | error |
| 终端含 `transform` | error |
| 组件内部使用 marker | warning |
| `data-target-clearance` 小于 4 或大于 14 | warning |
| 单页使用超过 2 种终端预设 | warning |

解析使用 `xml.etree.ElementTree`，通过本地标签名兼容命名空间，不使用正则解析 XML 层级。

- [ ] 步骤 4：增加检查统计

在 `result["info"]` 中写入：

```python
result["info"]["arrow_components"] = {
    "count": component_count,
    "presets": sorted(presets),
}
```

- [ ] 步骤 5：验证不合规与合规样例

```bash
python3 skills/ppt-master/scripts/svg_quality_checker.py \
  /tmp/vector-arrow-invalid.svg
```

预期：至少 3 个 error。

```bash
python3 skills/ppt-master/scripts/svg_quality_checker.py \
  components/vector-arrow/examples/fuel_fan_in.svg
```

预期：0 error，`arrow_components.count` 为 4。

- [ ] 步骤 6：编写脚本说明

`vector-arrow-components.md` 说明元数据、检查规则、错误示例和修复方式，并引用独立组件 README。

- [ ] 步骤 7：提交质量检查

```bash
git add skills/ppt-master/scripts/svg_quality_checker.py \
        skills/ppt-master/scripts/docs/vector-arrow-components.md
git commit -m "feat: validate custom svg arrow components"
```

---

### 任务八：接入主 Skill 执行规范

文件：

- 新建：`skills/ppt-master/references/arrow-components.md`
- 修改：`skills/ppt-master/templates/spec_lock_reference.md`
- 修改：`skills/ppt-master/templates/design_spec_reference.md`
- 修改：`skills/ppt-master/references/strategist.md`
- 修改：`skills/ppt-master/references/executor-base.md`
- 修改：`skills/ppt-master/references/shared-standards.md`
- 修改：`skills/ppt-master/SKILL.md`

接口：

- Strategist 输出：箭头渲染模式、终端预设、尺寸档位和目标留白
- Executor 输入：`connector_policy` 与每页 `Connector intent`
- SVG 输出：`connector-group` 外层和 `arrow-component` 内层

- [ ] 步骤 1：编写箭头组件运行时参考

`arrow-components.md` 使用英文，包含：

1. 触发条件。
2. 语义角色与默认配方。
3. `floating-stream` SVG 绝对路径示例。
4. 尺寸分档。
5. 节点边界与留白。
6. 终点切线和曲线规则。
7. 元数据要求。
8. marker 降级规则。
9. 质量自检清单。

运行时参考明确：

```text
Custom SVG arrow components use baked absolute path coordinates.
Do not place transform on data-part="terminal".
Do not use block arrows for ordinary connectors.
```

- [ ] 步骤 2：扩展 `connector_policy`

在 `spec_lock_reference.md` 增加：

```md
- arrow_render_mode: custom_svg
- default_terminal_preset: floating-stream
- fallback_terminal_preset: micro-triangle
- arrow_scale: standard
- target_clearance: 7
- process_flow: route=rounded_orthogonal terminal=micro-triangle
- data_flow: route=curved terminal=needle
- fan_in_result: route=curved terminal=floating-stream
```

保留旧 `arrow=end` 字段兼容说明。

- [ ] 步骤 3：扩展页面连接器说明

在 `design_spec_reference.md` 的 `Connector intent` 中增加：

```text
When direction is material, state render mode, terminal preset, scale,
and target clearance. Example:
fan_in_result; inbound; custom_svg; terminal=floating-stream;
scale=standard; target_clearance=7.
```

- [ ] 步骤 4：更新 Strategist 规则

新增决策顺序：

1. 判断关系是否有方向。
2. 选择连接器角色。
3. 有方向时选择 `custom_svg` 或 `native_marker`。
4. 正式汇报场景默认 `floating-stream`。
5. 高密度流程图可使用 `micro-triangle`。
6. 无方向关系保持 `arrow=none`。

- [ ] 步骤 5：更新 Executor 规则

新增每页执行要求：

1. 读取 `arrow-components.md`。
2. 按目标节点边界计算锚点和留白。
3. 导引路径使用低权重样式。
4. 终端路径使用绝对坐标。
5. 每个箭头输出 `arrow-component` 元数据。
6. 同组箭头保持预设、尺寸和留白一致。
7. 终端不得遮挡节点、文本、图标或数据标签。

- [ ] 步骤 6：更新技术规范与主流程门禁

`shared-standards.md` 增加“Custom SVG Arrow Components”章节，说明允许的 SVG 元素和 PPTX 导出行为。

`SKILL.md` Step 4 增加箭头预设契约；Step 6 增加箭头组件参考读取和自检；质量门增加 `arrow-component` 检查结果。

- [ ] 步骤 7：检查文档术语一致性

```bash
rg -n "floating-stream|arrow_render_mode|data-role=\"arrow-component\"|target_clearance" \
  skills/ppt-master/SKILL.md \
  skills/ppt-master/references/arrow-components.md \
  skills/ppt-master/references/strategist.md \
  skills/ppt-master/references/executor-base.md \
  skills/ppt-master/references/shared-standards.md \
  skills/ppt-master/templates/spec_lock_reference.md \
  skills/ppt-master/templates/design_spec_reference.md
```

预期：七个文件均至少出现一次核心字段或明确引用。

- [ ] 步骤 8：提交主 Skill 接入规范

```bash
git add skills/ppt-master/SKILL.md \
        skills/ppt-master/references/arrow-components.md \
        skills/ppt-master/references/strategist.md \
        skills/ppt-master/references/executor-base.md \
        skills/ppt-master/references/shared-standards.md \
        skills/ppt-master/templates/spec_lock_reference.md \
        skills/ppt-master/templates/design_spec_reference.md
git commit -m "feat: integrate reusable arrow component guidance"
```

---

### 任务九：升级示例模板并完成端到端验收

文件：

- 修改：`skills/ppt-master/templates/charts/hub_inward_arrows.svg`
- 修改：`skills/ppt-master/templates/charts/charts_index.json`
- 修改：`components/vector-arrow/examples/fuel_fan_in.svg`
- 修改：`components/vector-arrow/README.md`

接口：

- 输入：中心节点与四个外围节点
- 输出：四个 `fan_in_result` 箭头组件
- 验收：SVG、PPTX、PNG 三种结果一致

- [ ] 步骤 1：替换模板中的标准 marker

删除 `hub_inward_arrows.svg` 中的 `<marker id="arrow">` 和四个 `marker-end`。

每条连接器改为：

```xml
<g data-role="arrow-component"
   data-arrow-preset="floating-stream"
   data-semantic-role="fan_in_result"
  data-directionality="inbound"
   data-target-clearance="7">
  <path data-part="guide"
        d="M440 380 L472 380"
        fill="none"
        stroke="#94A3B8"
        stroke-width="1.4"
        stroke-linecap="round"/>
  <path data-part="terminal"
        d="M472 377 C480 377.2 486 378.2 494 380
           C486 381.8 480 382.8 472 383 L477 380 Z"
        fill="#64748B"/>
</g>
```

四个端部均使用绝对路径坐标，不使用 `transform`。

- [ ] 步骤 2：补充模板索引说明

`charts_index.json` 的 `hub_inward_arrows.summary` 增加“uses custom SVG floating-stream arrow components”说明，不改变模板选择语义。

- [ ] 步骤 3：运行模板与组件质量检查

```bash
python3 skills/ppt-master/scripts/svg_quality_checker.py \
  skills/ppt-master/templates/charts/hub_inward_arrows.svg

python3 skills/ppt-master/scripts/svg_quality_checker.py \
  components/vector-arrow/examples/fuel_fan_in.svg
```

预期：两个文件均为 0 error。

- [ ] 步骤 4：验证原始问题页

在 `projects/fuel_connector_semantics_demo_ppt169_20260627` 的 `spec_lock.md` 中将该页连接器设置为：

```md
- fan_in_result: route=curved terminal=floating-stream arrow_scale=standard target_clearance=7
```

更新该项目页面 SVG，使用四个内向 `floating-stream` 箭头。项目文件只作为验收样例，不纳入组件提交。

- [ ] 步骤 5：运行项目质量门

```bash
python3 skills/ppt-master/scripts/svg_quality_checker.py \
  projects/fuel_connector_semantics_demo_ppt169_20260627

python3 skills/ppt-master/scripts/page_content_checker.py \
  projects/fuel_connector_semantics_demo_ppt169_20260627 \
  --require-contract
```

预期：0 error；页面存在 `fan_in_result` 和 4 个 `arrow-component`。

- [ ] 步骤 6：按主流程重新导出

依次运行：

```bash
python3 skills/ppt-master/scripts/total_md_split.py \
  projects/fuel_connector_semantics_demo_ppt169_20260627
```

```bash
python3 skills/ppt-master/scripts/finalize_svg.py \
  projects/fuel_connector_semantics_demo_ppt169_20260627
```

```bash
python3 skills/ppt-master/scripts/svg_to_pptx.py \
  projects/fuel_connector_semantics_demo_ppt169_20260627
```

- [ ] 步骤 7：执行视觉验收

将最新 PPTX 转换为 PDF 和 PNG，检查：

| 验收项 | 标准 |
|---|---|
| 方向 | 四个箭头均指向中央决策服务 |
| 端部 | 箭头轮廓流畅、尖端清晰、无 marker 档位感 |
| 留白 | 四个箭头尖端与中央节点边界间距一致 |
| 曲线 | 曲线进入终端前切线连续，无折角 |
| 层级 | 导引线弱于终端，终端弱于中央节点 |
| 遮挡 | 不遮挡卡片正文、图标、标题和底部价值条 |
| PPT 编辑 | 箭头组可整体移动，解除组合后路径和端部可编辑 |

- [ ] 步骤 8：运行最终仓库检查

```bash
git diff --check
python3 -m compileall -q components/vector-arrow/vector_arrow \
  components/vector-arrow/cli \
  skills/ppt-master/scripts
```

预期：无空白错误、无 Python 语法错误。

- [ ] 步骤 9：提交模板升级和说明

```bash
git add skills/ppt-master/templates/charts/hub_inward_arrows.svg \
        skills/ppt-master/templates/charts/charts_index.json \
        components/vector-arrow/examples/fuel_fan_in.svg \
        components/vector-arrow/README.md
git commit -m "feat: adopt polished vector arrows in hub template"
```

---

## 最终验收清单

- [ ] `components/vector-arrow/` 可脱离 PPT Master 独立运行。
- [ ] `floating-stream` 在八个方向和三种尺寸下比例稳定。
- [ ] 所有终端路径使用绝对坐标，不依赖复合 `transform`。
- [ ] SVG 输出不包含 `<style>`、`class`、脚本和 marker。
- [ ] PowerPoint 输出包含原生 `<a:custGeom>` 和 `<p:grpSp>`。
- [ ] `svg_quality_checker.py` 能识别箭头组件结构错误。
- [ ] 主 Skill 能在 `connector_policy` 中锁定箭头预设和留白。
- [ ] 原始燃料经营页面实现四向内收、优雅且可编辑的箭头。
- [ ] 旧项目和标准 marker 路径继续可用。
- [ ] 每个任务形成独立提交，未纳入无关工作区修改。

---

## 回滚方案

| 层级 | 回滚方式 |
|---|---|
| 页面 | 将 `arrow_render_mode` 改为 `native_marker` 或 `arrow=none` |
| 主 Skill | 删除新增箭头预设字段，保留既有连接器语义字段 |
| 质量检查 | 移除 `_check_arrow_components` 调用和方法 |
| 模板 | 恢复 `hub_inward_arrows.svg` 的标准 marker |
| 组件 | 移除 `components/vector-arrow/`，不会影响旧项目导出 |

回滚不需要修改 PPTX 导出器，因为首期实现复用其既有 path 和 group 转换能力。
