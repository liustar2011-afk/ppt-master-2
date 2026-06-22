# 迁移方案：`brands/中电联公共元素_轻量版` → ppt-master 新版本

状态：草稿，待执行。资产文件本身已经在 `/Volumes/DOC/ppt-master/skills/ppt-master/templates/brands/中电联公共元素_轻量版/` 落地且与旧仓库逐字节一致，**不需要再复制文件**。这次迁移没成功的原因不是资产没拷过来，而是新仓库的主流程已经把 chrome-only brand 的"消费方式"换了一套（脚本注入 → spec_lock 驱动的逐页手绘），但配套的索引工具和文档没跟着升级，导致这个 brand 处于"部分迁移"状态。本文档列出旧/新两套调用模式的对照，以及把这个 brand 真正接回新主流程所需的具体改动。

## 0. 结论先行

| 项目 | 状态 |
|---|---|
| brand 资产文件（`design_spec.md` / `brand_rules.json` / `master_elements.svg` / `01_cover.svg` / `04_ending.svg` / `cover_bg.jpg` / `logo.png` / `master_elements.reference.txt`） | ✅ 已迁移，文件级一致 |
| `brands_index.json` 里的条目 | ✅ 当前内容正确（`brand_mode` / `color_policy` / 空 `primary_color` 都在），但是**易碎**——见 §3.1 |
| `SKILL.md` Step 3 对 chrome-only brand 的读取/落 `spec_lock.md` 规则 | ✅ 已经完整实现，且比旧仓库更细（见 §1） |
| `SKILL.md` 默认品牌声明（原 `default_template.json` 机制） | ✅ 已迁移，新仓库改为 `SKILL.md` 行内声明，等价生效 |
| `references/executor-base.md §2.1` 逐页手绘 chrome 的强制规则 | ✅ 已完整实现 |
| `templates/spec_lock_reference.md §master_chrome / §cover_regions` | ✅ 已完整实现 |
| `scripts/register_template.py --kind brand` 对 chrome-only 的支持 | ❌ **已退化** — 不再解析 `brand_mode` / `color_policy`，会在下次重建索引时把这两个字段冲掉 |
| `templates/brands/README.md` 的 chrome-only 章节 | ❌ **已被删除** — 与实际跑通的主流程矛盾 |
| `docs/zh/templates-architecture.md` 对 chrome-only 概念的描述 | ❌ **完全缺失** — 全文 0 次提到 chrome |
| 旧版三个脚本 `apply_brand_page_templates.py` / `verify_chrome_only_brand.py` / `inject_brand_chrome.py` | ⚪ **不需要迁移** — 新架构不再走"导出后注入 PPTX 版式"路线，这三个脚本的职责已被 `executor-base.md §2.1`（逐页手绘）取代 |
| `svg_quality_checker.py` 对 `protected_region` / chrome 的质量门禁 | ⚠️ **新仓库无对应检查** — 旧仓库靠脚本硬门禁，新仓库目前完全靠 Strategist/Executor 的文字纪律，没有自动化兜底 |
| `design_spec.md §IV` 里 `templates/brands/中电联公司/` 路径 | ❌ **悬空引用**（旧仓库就没有这个目录，应为 `中电联公司_现代能源科技`）——与本次迁移无关的前置 bug，顺手记录 |

## 1. 旧 / 新两套调用模式对照

### 1.1 旧仓库（ppt-master210）：脚本注入模式

```
Step 3  读 design_spec.md(kind:brand, brand_mode:chrome-only)
        → 整个 brand 目录原样拷进 <project>/templates/
Step 7.3 svg→pptx 导出
Step 7.4 apply_brand_page_templates.py   (替换封面/封底占位符)
         verify_chrome_only_brand.py     (硬门禁：正文页不准重绘 chrome)
         inject_brand_chrome.py          (把 chrome 注入 PPTX slideLayout，正文页统一引用该 layout)
```
默认品牌通过独立文件 `templates/default_template.json` 声明，`brand_rules.json` 是这三个脚本唯一的运行期契约（`master_elements` / `page_protected_regions` / `brand_page_templates` / `brand_assets`）。

### 1.2 新仓库（ppt-master）：spec_lock 驱动 + 逐页手绘模式

```
Step 3  SKILL.md:188 行内声明默认品牌（不再用 default_template.json 文件）
        → design_spec.md(kind:brand, brand_mode:chrome-only) 命中后：
          - design_spec.md + 非图片资产 → <project>/templates/
          - 读 brand_rules.json，把 master_elements + page_protected_regions
            原样写入 spec_lock.md §master_chrome
          - 把 content_regions.cover_title_region / cover_meta_region
            原样写入 spec_lock.md §cover_regions
Executor 每页生成前重读 spec_lock.md（executor-base.md §2.1，强制规则）：
        - 非封面/封底页：把 §master_chrome 的每个元素手绘进当前 SVG（不是注入，是真的重新画一遍，因为 SVG→PPTX 没有 slide-master 合成）
        - 命中该 brand 自己的 01_cover.svg / 04_ending.svg 的页面：跳过 §master_chrome（避免重复 logo/footer），改用 §cover_regions 约束标题/元信息落位
质量校验  没有脚本级硬门禁，完全靠 §2.1 的文字纪律 + Strategist 的 spec_lock 复核
```

**核心差异**：旧版是"画一次、导出后批量注入"，新版是"spec_lock 当唯一事实源，Executor 每页都要重读并重画"。这对该 brand 而言是兼容的——`brand_rules.json` 里的字段名（`master_elements`、`page_protected_regions`、`content_regions.cover_title_region/cover_meta_region`）恰好和新仓库 `SKILL.md:232` 行描述的读取路径完全对得上，**不需要改 `brand_rules.json` 的 schema**。真正需要动的是下面 §2、§3 列出的配套工具/文档。

## 2. 不需要迁移的部分（明确排除，避免误操作）

- `apply_brand_page_templates.py`、`verify_chrome_only_brand.py`、`inject_brand_chrome.py`：**不要**把这三个脚本搬到新仓库的 `skills/ppt-master/scripts/`。新架构下封面/封底模板的应用方式是 Executor 直接读 `templates/01_cover.svg` / `04_ending.svg` 做版式继承（参见 `executor-base.md:64-67` 的通用 Cover/Ending 继承表，与 `template-designer.md` 的页面类型约定一致），chrome 校验也已转为 §2.1 的文字规则，三个脚本的职责已被取代，迁移它们反而会造成"两套机制并存、互相矛盾"。
- `templates/default_template.json`：新仓库没有这个文件，也不需要补。默认品牌已经在 `SKILL.md:188` 用行内声明的方式等价实现，且已经指向正确路径 `templates/brands/中电联公共元素_轻量版/`。**不要**在新仓库里新建 `default_template.json`，否则会和 `SKILL.md` 的行内声明产生两份"默认品牌"的事实源。

## 3. 需要落地的改动

### 3.1 `skills/ppt-master/scripts/register_template.py` — 补回 chrome-only 支持

现状（新仓库）：
- 文件头注释只写 `brand: { summary, primary_color }`，没有 chrome-only 的说明。
- `_extract_entry()` 里直接 `primary_color = fm.get("primary_color") or _extract_primary_color(body) or ""`，不区分 `brand_mode`。对这个 brand 来说，因为它的章节标题是 `## II. Public Chrome Elements` 而不是 `## II. Color Scheme`，`_extract_primary_color()` 的正则碰巧匹配不到，目前侥幸跑出空字符串——**这是侥幸，不是保障**，换一个 chrome-only brand、章节顺序或命名稍有不同就会被误抓一个颜色塞进 `primary_color`。
- 完全没有把 `brand_mode` / `color_policy` 写回 entry，意味着只要有人在新仓库跑一次：
  ```bash
  python3 skills/ppt-master/scripts/register_template.py 中电联公共元素_轻量版 --kind brand
  ```
  当前 `brands_index.json` 里这两个字段就会被**静默冲掉**。

需要的改动（对照旧仓库 `ppt-master210` 同名函数移植即可，逻辑已经验证过）：

1. 在 `_extract_entry()` 里读取 `brand_mode` / `color_policy`：
   ```python
   brand_mode = str(fm.get("brand_mode", "") or "")
   color_policy = str(fm.get("color_policy", "") or "")
   if kind == "brand" and brand_mode == "chrome-only":
       primary_color = fm.get("primary_color", "")
   else:
       primary_color = fm.get("primary_color") or _extract_primary_color(body) or ""
   ```
2. 写回 entry 时按需追加字段：
   ```python
   if kind == "brand":
       entry = OrderedDict(summary=summary, primary_color=str(primary_color))
       if brand_mode:
           entry["brand_mode"] = brand_mode
       if color_policy:
           entry["color_policy"] = color_policy
   ```
3. 顶部 docstring 同步补一句：``brand: { summary, primary_color }``；chrome-only brands may also include ``brand_mode`` and ``color_policy`` with an empty ``primary_color``。

> 旧仓库版本还多带了一个 `_parse_simple_frontmatter`（PyYAML 缺失时的兜底解析），新仓库现在是直接 `raise SpecParseError` 要求装 PyYAML。这部分和 chrome-only 无关，是否一起搬看新仓库环境是否保证有 PyYAML；如果新仓库 CI/运行环境已固定装了 PyYAML，可以不搬，只搬 chrome-only 相关的三处改动即可。

验证方式：改完后跑 `register_template.py --rebuild-all --kind brand`，确认输出的 `brands_index.json` 里 `中电联公共元素_轻量版` 仍然是：
```json
{
  "summary": "...",
  "primary_color": "",
  "brand_mode": "chrome-only",
  "color_policy": "visual-style-owned"
}
```

### 3.2 `skills/ppt-master/templates/brands/README.md` — 补回 chrome-only 章节

新仓库这份 README 把旧版的下列内容整段删掉了，需要按旧仓库 `ppt-master210/skills/ppt-master/templates/brands/README.md` 原文补回（只是文档对齐，不涉及行为变更，因为行为本来就在 `SKILL.md` / `executor-base.md` 跑通了）：

- "How are brands consumed" 表格里缺的一行：
  > 显式 chrome-only brand 路径（frontmatter `brand_mode: chrome-only`）→ 拷贝 public chrome 资产和规则到 `<project_path>/templates/`；Strategist 只锁 master chrome / protected regions，内容配色、字体、图标、视觉风格仍按正常 Eight Confirmations 推荐。
- "Package structure" 一节下需要补充 chrome-only 的 frontmatter 示例：
  ```yaml
  kind: brand
  brand_mode: chrome-only
  color_policy: visual-style-owned
  primary_color: ""
  ```
  以及它对应的六个必需小节：I Brand Overview / II Public Chrome Elements / III Safety Regions / IV Assets / V Non-Locked Design Areas / VI Runtime Rule（注意这和"全身份 brand"的六节标题不同，必须分开写清楚，否则有人按"全身份 brand"模板去检查这个 brand 会得出"缺章节"的错误结论）。
- "Discovery index" 一段补回：chrome-only brand 在 `brands_index.json` 里可以带 `brand_mode` / `color_policy`，`primary_color` 留空。

### 3.3 `docs/zh/templates-architecture.md` — 补充 chrome-only 概念

当前新仓库这份架构文档对 chrome-only 完全没有提及（`grep chrome` 零命中），但 `SKILL.md`、`executor-base.md`、`spec_lock_reference.md` 三处都已经围绕这个概念写了相当篇幅的强制规则。需要在该文档里至少补一段，覆盖：

- chrome-only 是 `kind: brand` 的一个子模式（`brand_mode: chrome-only`），定位：只锁定"公共版面元素"（页眉分割线、角标 logo、页脚条、机构名、页码）和安全区，不锁内容配色/字体/图标/语调/页面结构。
- 它和"全身份 brand"在 schema、必需章节、index 字段上的区别（直接引用 §3.2 改完后的 `brands/README.md` 即可，不用重复写一遍表格）。
- 运行期落地路径：`brand_rules.json` → `spec_lock.md §master_chrome` / `§cover_regions` → Executor 逐页重绘（指向 `executor-base.md §2.1`，不要在架构文档里重复展开规则细节，避免后续两处文档分裂维护）。

### 3.4（建议，非阻断）`svg_quality_checker.py` — 补一道 chrome 门禁

旧仓库用独立脚本 `verify_chrome_only_brand.py` 做硬门禁：扫正文 SVG，发现疑似 logo / 红色分割线 / 蓝色页脚条 / 页脚文字 / 静态页码就报错。新仓库目前完全没有等价检查——`executor-base.md §2.1` 只是对 Executor 的文字要求，没有任何脚本会在 Strategist/Executor 漏画或多画 chrome 时拦截。

不属于本次"把 brand 接回主流程"的阻断项（brand 本身的 schema 和文档迁移完成后，pipeline 在文字纪律遵守的前提下是能跑通的），但建议作为后续加固：把 `verify_chrome_only_brand.py` 的检测逻辑（读 `spec_lock.md` 里落地的 `master_chrome` 字段，而不是旧版直接读 `<project>/templates/brand_rules.json`）移植成新仓库的质量门禁脚本，挂在 Step 6/Step 7 之间。

### 3.5（顺手修复，独立于本次迁移）`design_spec.md §IV` 悬空路径

`design_spec.md` 第 56 行写"这两个模板借自 `templates/brands/中电联公司/`"，但无论旧仓库还是新仓库都不存在 `中电联公司` 这个目录，只有 `中电联公司_现代能源科技`。这是 brand 自身内容的历史遗留 typo，建议顺手改成 `templates/brands/中电联公司_现代能源科技/`，与迁移无关但容易被后续读这份 spec 的人误导去找一个不存在的目录。

## 4. 执行清单（按顺序做）

- [x] 3.1：补 `register_template.py` 的 chrome-only 解析/写回逻辑，跑 `--rebuild-all --kind brand --dry-run` 验证 `brands_index.json` 不被冲掉（已验证：三个 brand 条目不变，chrome-only 条目仍带 `brand_mode`/`color_policy`/空 `primary_color`）
- [x] 3.2：补回 `templates/brands/README.md` 的 chrome-only 章节（消费表新增一行、补 frontmatter 示例与六个必需小节、discovery index 说明同步）
- [x] 3.3：在 `docs/zh/templates-architecture.md` 补一段 chrome-only 概念说明（Brand schema 下新增子模式小节 + Step 3 行为表补充一句）
- [x] 3.5：修正 `design_spec.md §IV` 的悬空路径 typo（`中电联公司/` → `中电联公司_现代能源科技/`，新旧仓库都已改）
- [x] 3.4：重写 `scripts/verify_chrome_only_brand.py`，改为读 `<project>/templates/spec_lock.md §master_chrome / §page_layouts`（而非旧版直接读 brand 目录），检测方向也反过来了——旧版查"正文页是否误画了 chrome"，新版查"正文页是否漏画了 chrome"+"品牌自带封面/封底页是否重复画了 chrome"+"内容是否侵入 protected_region"。已用 `/tmp/chrome_test` 合成 fixture 验证三种异常都能正确命中、且品牌自身的页脚文字/页码不会误报为侵入 protected_region、正确的 deck 能拿到 exit 0。已接入 `SKILL.md` Step 6 质量门禁（`svg_quality_checker.py` 之后，封面归属判定后并入 checkpoint 清单）
- [x] 端到端验证：用 `project_manager.py init` 起了一个临时项目，套用本 brand（拷 design_spec/brand_rules/模板 SVG/logo），手写了符合 §master_chrome / §cover_regions / §page_layouts 的 `spec_lock.md`，手绘 4 页 SVG（P01 用品牌 `01_cover.svg` 派生、不画 chrome；P02/P03 正文页手绘全部 5 个 chrome 要素；P04 用品牌 `04_ending.svg` 派生、不画 chrome），依次跑通：
  - `verify_chrome_only_brand.py` → 第一轮跑出一个**真实 bug**：P01/P04 品牌模板自带的"中国电力企业联合会"机构名大字（在封面下方深蓝区域，不是页脚）被误判成页脚 `footer_org_text` 的重复绘制——`_has_footer_org_text` 当时只做子串匹配、没比较坐标。已修复为坐标 + 文本双重匹配，回归两组合成 fixture（缺失 chrome / 重复 chrome / 品牌模板页自带文案）全部正确，正式 deck 跑出 `exit 0`
  - `svg_quality_checker.py` → 4/4 通过
  - `finalize_svg.py` → 图片对齐嵌入成功
  - `svg_to_pptx.py` → 成功导出真实 `.pptx`（4 页，Native DrawingML）
  临时项目已清理，未污染 `projects/` 下的真实项目
