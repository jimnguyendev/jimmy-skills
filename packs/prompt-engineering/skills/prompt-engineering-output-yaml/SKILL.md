---
name: prompt-engineering-output-yaml
description: "Emit YAML for configuration, infrastructure-as-code, and human-edited artifacts. Indentation rules, quoting, comments, schema enforcement, and the JSON-vs-YAML decision. Use for k8s manifests, Compose files, CI configs, pipeline definitions, and anywhere humans both read and edit the output."
user-invocable: true
license: MIT
compatibility: "Stack-agnostic. YAML 1.2 by default — be explicit about boolean and null literals to avoid the Norway problem (NO → false)."
metadata:
  author: jimnguyendev
  version: "0.1.0"
allowed-tools: Read Edit Write Glob Grep
---

# YAML Output Contracts

YAML earns its place when humans both read and edit the output. Configuration, infrastructure-as-code, CI pipelines, runbooks. Compared to JSON it's easier to read, supports comments, and tolerates trailing commas — at the cost of indentation sensitivity and a famously surprising type system.

For code-only consumption with strict typing, prefer JSON (`prompt-engineering-output-json`). For prose-with-structure, prefer markdown (`prompt-engineering-output-structured`).

## When to use YAML

| Situation | YAML | JSON |
|---|---|---|
| Kubernetes / Helm manifest | ✅ | — |
| Docker Compose | ✅ | — |
| GitHub Actions / GitLab CI / CircleCI | ✅ | — |
| Ansible / Puppet | ✅ | — |
| Application config (read by ops) | ✅ | — |
| Application config (read only by code) | — | ✅ |
| API response | — | ✅ |
| Tool-call arguments | — | ✅ |
| Logs / event payloads | — | ✅ |

Rule of thumb: if a human is going to open it in an editor and change a value, YAML wins. If only code reads it, JSON wins.

## Schema-first prompting

Same approach as JSON — define the structure before the prose:

```text
<output>
Return ONLY a YAML document matching this structure. No markdown fences. No commentary.

apiVersion: <string>      # required, must be "apps/v1" for Deployments
kind: <string>            # required, one of: Deployment | StatefulSet | DaemonSet
metadata:
  name: <string>          # required, RFC 1123 lowercase + dashes, max 63 chars
  labels: <map<string,string> | null>
spec:
  replicas: <integer>     # required, 1..50
  selector:
    matchLabels: <map<string,string>>
  template:
    metadata:
      labels: <map<string,string>>   # MUST equal spec.selector.matchLabels
    spec:
      containers:
        - name: <string>             # required, RFC 1123
          image: <string>            # required, image:tag — never :latest
          ports:
            - containerPort: <integer>  # 1..65535
          resources:
            requests: { cpu: <string>, memory: <string> }
            limits:   { cpu: <string>, memory: <string> }

Rules:
- Use null (not "" or omission) when a field is unknown.
- Use 2-space indentation. No tabs.
- Quote string values that look like booleans, numbers, dates, or YAML keywords
  (yes, no, on, off, null, ~, true, false, 1.0).
- Comments are allowed only above a key, never inline.
</output>
```

## YAML's three sharp edges

### 1. The Norway problem (and friends)

YAML 1.1 interprets a long list of bare strings as booleans:

```yaml
country_code: NO    # parsed as false in YAML 1.1
on_call:    yes     # parsed as true
version:    1.10    # parsed as 1.1 (number, trailing zero dropped)
zip_code:   01234   # parsed as octal
```

YAML 1.2 narrows this, but many parsers still default to 1.1 behavior. **Always quote** string values that could collide with booleans, numbers, dates, or null. Make this an explicit rule in the prompt.

### 2. Indentation is structure

JSON is forgiving; YAML is not. Two spaces vs. four spaces silently changes meaning. Tell the model:

> Use 2-space indentation. Never tabs. Lists items at the same level as their parent key, indented two spaces.

When the model emits both styles in the same document, it's almost always because the few-shot examples mixed styles. Audit your examples.

### 3. Anchors and references are usually a footgun

YAML supports `&anchor` / `*ref` / `<<` merge keys. They make hand-written configs DRY but they're confusing for LLM-generated output:

- The model may emit anchors that aren't actually used (cosmetic noise).
- It may emit dangling refs.
- Many JSON-to-YAML round trips don't preserve them.

Default rule in prompts: **"Do NOT use YAML anchors, references, or merge keys. Repeat values literally."** Reserve anchors for human-authored YAML.

## Comments

YAML comments are a feature, not a bug — use them for human readability:

```text
- Add a one-line `# comment` above any non-obvious value.
- Comment any value derived from another (e.g. "# matches spec.selector.matchLabels").
- Do NOT add comments inside lists in CI configs (some parsers strip them).
```

Cap comment volume — a config that is 60% comments is a sign the schema needs a different shape.

## Validation

Treat YAML output the same way you treat JSON:

```python
import yaml
from pydantic import BaseModel, Field
from typing import Literal

class Container(BaseModel):
    name: str
    image: str
    containerPort: int | None = None
    # ... etc

class DeploymentSpec(BaseModel):
    apiVersion: Literal["apps/v1"]
    kind: Literal["Deployment", "StatefulSet", "DaemonSet"]
    # ...

def parse(raw: str) -> DeploymentSpec:
    raw = raw.strip()
    if raw.startswith("```"):              # strip code fences if model added them
        raw = raw.split("\n", 1)[1].rsplit("```", 1)[0]
    data = yaml.safe_load(raw)             # NEVER use yaml.load (RCE risk)
    return DeploymentSpec.model_validate(data)
```

Two non-negotiables:

1. **Always `yaml.safe_load`** (not `yaml.load`). The unsafe loader can execute arbitrary Python via `!!python/object` tags. LLM outputs are untrusted input.
2. **Validate against a schema after parse.** YAML's permissiveness is exactly why you need typed validation (Pydantic, JSON Schema via `jsonschema`, or `kubeval` / `kustomize` for k8s).

## Multi-document YAML

For Kubernetes and similar, you often want multiple documents in one stream:

```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  key: value
---
apiVersion: apps/v1
kind: Deployment
# ...
```

Tell the model:

> Separate multiple resources with `---` on its own line. Each resource is a complete YAML document.

Parse with `yaml.safe_load_all`.

## Common failure modes

1. **Unquoted booleans / numbers** that should be strings (Norway problem).
2. **Mixed indentation** (2-space and 4-space in the same file).
3. **Code-fence wrapping** (```` ```yaml ````).
4. **Comments inside flow-style maps** (`{key: value, # comment}` is invalid).
5. **Hallucinated anchors** that don't bind.
6. **Tabs in indentation** (illegal in YAML).

Prompt rules + a parse-and-validate step catch all six.

## Combining JSON output with YAML target

When the LLM is more reliable at JSON but the artifact must be YAML, generate JSON and convert:

```python
import json, yaml
data = json.loads(model_json_output)
yaml_text = yaml.safe_dump(data, sort_keys=False, default_flow_style=False)
```

Pros: full schema enforcement on the JSON side, deterministic conversion. Cons: lose the model's ability to add helpful comments — re-add them in code if needed. Useful when YAML must validate against a strict tool (k8s API, GitHub Actions schema).

## Anti-patterns

- **`yaml.load` on LLM output.** Remote code execution; use `safe_load`.
- **Letting the model invent the schema.** Always pin the structure.
- **Trusting the YAML parser to enforce types.** Validate with Pydantic / JSON Schema after.
- **Anchors and merge keys in generated configs.** Confusing and brittle; ban them in the prompt.
- **Skipping the quote rule on user-provided strings.** Causes the Norway problem in production.
- **Tabs.** Some editors silently insert them. The prompt and the parser should both reject.
- **Mixing flow style (`{a: 1, b: 2}`) and block style.** Pick one — usually block — for the whole document.

## Quick template

```text
<output>
Return ONLY a YAML document. No markdown fences. No commentary.

Rules:
- 2-space indentation. No tabs.
- Quote any string value that could be parsed as a boolean, number, date, or YAML keyword.
- Use null for unknown values; do not invent.
- Do NOT use anchors, references, or merge keys.
- Add a `# comment` above any non-obvious value.
- For multiple resources, separate with `---` on its own line.

Schema:
<paste schema here>
</output>
```

## Cross-references

- `jimmy-skills@prompt-engineering-core` — Output Contract.
- `jimmy-skills@prompt-engineering-output-json` — when only code consumes the output.
- `jimmy-skills@prompt-engineering-output-structured` — for human-rendered prose.
- `jimmy-skills@prompt-engineering-edge-cases` — empty / long / adversarial input handling.
