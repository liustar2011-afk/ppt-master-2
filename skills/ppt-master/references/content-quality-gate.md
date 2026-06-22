# Content Quality Gate Reference Manual

Validate that each generated page visibly renders the content committed in its outline.

---

## 1. Page Contract

**Hard rule**: After `design_spec.md` and `spec_lock.md` are locked, write `<project>/analysis/page_contract.json`.

| Field | Behavior |
|---|---|
| `page_id` | `P01` format, one record per output page |
| `claim` | The page's central conclusion |
| `required_refs` | Semantic units that must be visible in the SVG |
| `required_relations` | Required sequence, hierarchy, comparison, cause, or flow links |
| `density.minimum_coverage` | Optional minimum estimated content coverage, from zero to one |
| `density.required` | Turn a coverage miss into an error when `true` |

```json
{
  "version": "1.0",
  "pages": [{
    "page_id": "P06",
    "claim": "核验嵌入企业入场审批，形成可审计闭环",
    "required_refs": ["step-project", "step-authorize", "evidence-output"],
    "required_relations": ["flow-project-authorize"],
    "density": {"minimum_coverage": 0.35, "required": false}
  }]
}
```

---

## 2. SVG Mapping

**Hard rule**: Every contract ref must map to one visible SVG group.

```xml
<g id="step-project" data-role="entity" data-ref="step-project">...</g>
<g id="flow-project-authorize" data-role="relation" data-ref="flow-project-authorize">...</g>
<g id="output-report" data-role="evidence" data-ref="evidence-output">...</g>
```

| `data-role` | Use |
|---|---|
| `claim` | Page conclusion |
| `entity` | Object, module, stage, actor, or metric |
| `relation` | Sequence, hierarchy, comparison, cause, dependency, or flow |
| `evidence` | Fact, data, mechanism, example, explanation, or output |
| `action` | Decision, owner, next step, or delivery |
| `context` | Scope, time, source, definition, or constraint |

---

## 3. Gate

**Mandatory**: Run after `svg_quality_checker.py` and before notes or post-processing.

```bash
python3 scripts/page_content_checker.py <project_path> --require-contract
```

**Validation**: Missing contract refs or relations are errors. Density is advisory unless the contract marks it required. Existing projects without a contract remain compatible when the command runs without `--require-contract`.
