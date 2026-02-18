# Helm Template DSL Reference (AI generated)

> A beginner-friendly reference for the Go template language as used in Helm charts.  
> All examples are drawn from the actual `myapp` subchart and `umbrella-chart` in this workspace.

---

## Table of Contents

1. [How Helm Templating Works](#1-how-helm-templating-works)
2. [Delimiters and Whitespace](#2-delimiters-and-whitespace)
3. [Comments](#3-comments)
4. [Built-in Objects](#4-built-in-objects)
5. [Accessing Values](#5-accessing-values)
6. [The Pipe Operator](#6-the-pipe-operator)
7. [Variables](#7-variables)
8. [Flow Control](#8-flow-control)
   - [if / else / end](#if--else--end)
   - [with / end](#with--end)
   - [range / end](#range--end)
9. [Named Templates (define / include / template)](#9-named-templates-define--include--template)
10. [String Functions](#10-string-functions)
11. [Type Conversion Functions](#11-type-conversion-functions)
12. [Logic and Comparison Functions](#12-logic-and-comparison-functions)
13. [YAML & Indentation Functions](#13-yaml--indentation-functions)
14. [Dict (Dictionary) Functions](#14-dict-dictionary-functions)
15. [List Functions](#15-list-functions)
16. [Math Functions](#16-math-functions)
17. [Flow Control Functions](#17-flow-control-functions)
18. [Other Useful Functions](#18-other-useful-functions)
19. [Quick Reference: Patterns from Our Charts](#19-quick-reference-patterns-from-our-charts)

---

## 1. How Helm Templating Works

Helm templates are **plain YAML files** with **Go template actions** embedded in `{{ }}` delimiters.
Helm processes every file in `templates/` through the Go template engine, replacing actions with computed values.

**Critical rule:** `values.yaml` is **never** templated. Only files inside `templates/` are processed.

```
templates/deployment.yaml   ← Processed by template engine ({{ }} works here)
values.yaml                 ← Plain YAML only ({{ }} will cause errors)
```

---

## 2. Delimiters and Whitespace

### Basic delimiters

| Syntax | Meaning |
|--------|---------|
| `{{ }}` | Template action — evaluated and replaced with output |
| `{{- }}` | Trim whitespace **before** the action |
| `{{ -}}` | Trim whitespace **after** the action |
| `{{- -}}` | Trim whitespace **both sides** |

### Why this matters

YAML is whitespace-sensitive. Without trimming, template actions leave blank lines.

```yaml
# WITHOUT whitespace trimming — leaves a blank line when condition is false:
{{ if .Values.autoscaling.enabled }}
replicas: {{ .Values.replicaCount }}
{{ end }}

# WITH whitespace trimming — clean output:
{{- if .Values.autoscaling.enabled }}
replicas: {{ .Values.replicaCount }}
{{- end }}
```

### From our deployment.yaml:
```yaml
{{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
{{- end }}
```
The `{{-` trims the newline before it, so if autoscaling IS enabled, no empty line appears.

---

## 3. Comments

```yaml
{{/* This is a comment — it produces no output */}}

{{/*
Multi-line comments
work like this.
*/}}
```

### From our _helpers.tpl:
```yaml
{{/*
Expand the name of the chart.
*/}}
{{- define "myapp.name" -}}
```

---

## 4. Built-in Objects

Helm injects these objects into every template:

| Object | What it contains | Example |
|--------|-----------------|---------|
| `.Values` | Merged values from `values.yaml` + overrides | `.Values.replicaCount` → `1` |
| `.Chart` | Contents of `Chart.yaml` | `.Chart.Name` → `myapp` |
| `.Release` | Info about the current release | `.Release.Name` → `my-release` |
| `.Template` | Current template info | `.Template.Name` → `myapp/templates/deployment.yaml` |
| `.Files` | Access to non-template files in the chart | `.Files.Get "config.ini"` |
| `.Capabilities` | Kubernetes cluster capabilities | `.Capabilities.APIVersions.Has "apps/v1"` |

### .Release sub-fields

| Field | Description |
|-------|-------------|
| `.Release.Name` | The release name (e.g. `helm install **my-release** ./chart`) |
| `.Release.Namespace` | The namespace being deployed to |
| `.Release.Service` | Always `"Helm"` |
| `.Release.IsUpgrade` | `true` if this is an upgrade |
| `.Release.IsInstall` | `true` if this is an install |
| `.Release.Revision` | The revision number (starts at 1) |

### .Chart sub-fields

| Field | Description |
|-------|-------------|
| `.Chart.Name` | Chart name from Chart.yaml |
| `.Chart.Version` | Chart version |
| `.Chart.AppVersion` | Application version |
| `.Chart.Description` | Chart description |

### From our deployment.yaml:
```yaml
containers:
  - name: {{ .Chart.Name }}                          # → "myapp"
    image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
```

---

## 5. Accessing Values

### Dot notation for nested values
```yaml
# Given values.yaml:
#   image:
#     repository: nginx
#     tag: "1.21"

{{ .Values.image.repository }}    # → nginx
{{ .Values.image.tag }}           # → 1.21
```

### Global values (accessible in subcharts)
```yaml
# In umbrella values.yaml:
#   global:
#     environment: production

# In ANY subchart template:
{{ .Values.global.environment }}   # → production
```

### From our deployment.yaml:
```yaml
environment: {{ .Values.global.environment }}
```

---

## 6. The Pipe Operator

The pipe `|` sends the output of one expression as the **last argument** to the next function.
This is **the most important concept** in Helm templating.

```
EXPRESSION | FUNCTION
```
is equivalent to:
```
FUNCTION EXPRESSION
```

### Chaining pipes

```yaml
# These are equivalent:
{{ trimSuffix "-" (trunc 63 (default .Chart.Name .Values.nameOverride)) }}
{{ default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
```
The pipe version reads left-to-right and is much clearer.

### From our _helpers.tpl:
```yaml
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
```
**Read as:** "Take `.Chart.Name` as default (if `.Values.nameOverride` is empty), then truncate to 63 chars, then remove any trailing dash."

---

## 7. Variables

Variables start with `$` and are assigned with `:=`.

```yaml
{{- $name := default .Chart.Name .Values.nameOverride }}
```

Once assigned, use them anywhere in the same scope:
```yaml
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
```

### Special variable: `$`

`$` always refers to the **root scope** (the top-level `.`). This is essential inside `range` and `with` blocks where `.` is rebound.

```yaml
{{- range .Values.items }}
  # Inside range, "." is the current item
  # Use $ to access the root:
  release: {{ $.Release.Name }}
{{- end }}
```

### From our _helpers.tpl:
```yaml
{{- define "myapp.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}
```

---

## 8. Flow Control

### if / else / end

```yaml
{{- if CONDITION }}
  # rendered if CONDITION is truthy
{{- else if OTHER_CONDITION }}
  # rendered if OTHER_CONDITION is truthy
{{- else }}
  # rendered if nothing above matched
{{- end }}
```

**What counts as "falsy":** `false`, `0`, `""` (empty string), `nil`, empty collections (`[]`, `{}`).

### From our deployment.yaml:
```yaml
{{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
{{- end }}
```
**Read:** "Only set replicas if autoscaling is NOT enabled."

### From our _helpers.tpl:
```yaml
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
```
**Read:** "Only add the version label if AppVersion is set in Chart.yaml."

---

### with / end

`with` does two things:
1. **Conditional:** Only renders the block if the value is truthy
2. **Rebinds `.`:** Inside the block, `.` becomes the value you passed

```yaml
{{- with .Values.podAnnotations }}
annotations:
  {{- toYaml . | nindent 8 }}
{{- end }}
```
**Read:** "If `podAnnotations` is non-empty, render them as YAML. Inside the block, `.` IS `podAnnotations`."

### From our deployment.yaml — multiple `with` blocks:
```yaml
{{- with .Values.imagePullSecrets }}
imagePullSecrets:
  {{- toYaml . | nindent 8 }}
{{- end }}

{{- with .Values.nodeSelector }}
nodeSelector:
  {{- toYaml . | nindent 8 }}
{{- end }}

{{- with .Values.tolerations }}
tolerations:
  {{- toYaml . | nindent 8 }}
{{- end }}
```

> ⚠️ **Gotcha:** Inside `with`, you **cannot** access `.Values` or `.Release` directly because `.` has been rebound. Use `$` to reach the root scope.

---

### range / end

Iterate over lists or maps.

```yaml
# Iterate over a list:
{{- range .Values.ingress.hosts }}
- host: {{ .host }}
{{- end }}

# Iterate with index:
{{- range $index, $item := .Values.items }}
  item-{{ $index }}: {{ $item }}
{{- end }}

# Iterate over a map (dict):
{{- range $key, $value := .Values.labels }}
  {{ $key }}: {{ $value }}
{{- end }}
```

> ⚠️ **Gotcha:** Inside `range`, `.` is rebound to the current item. Use `$` to access root scope.

---

## 9. Named Templates (define / include / template)

Named templates (also called "partials") live in `_helpers.tpl` files. The `_` prefix tells Helm **not** to render this file as a Kubernetes manifest.

### define — Declare a reusable template

```yaml
{{- define "myapp.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}
```

**Explanation**: This creates a named template called `"myapp.name"` that can be reused elsewhere. It generates a chart name by:
- Using `nameOverride` from `values.yaml` if set, otherwise falling back to the chart's name (`.Chart.Name`).
- Truncating the result to 63 characters (Kubernetes DNS limit).
- Removing any trailing dashes for clean naming.
This ensures consistent, valid names across your chart without repeating code.

### include — Call a named template (preferred)

```yaml
name: {{ include "myapp.fullname" . }}
```
- First argument: the template name (string)
- Second argument: the scope to pass (`.` passes the current context)
- Returns a **string** that can be piped

### include with pipe (this is why `include` is preferred):
```yaml
labels:
  {{- include "myapp.labels" . | nindent 4 }}
```
**Read:** "Call the `myapp.labels` template, then indent its output by 4 spaces with a leading newline."

### template — Call a named template (older, avoid)

```yaml
{{ template "myapp.name" . }}
```
- Does **not** return a string — output goes directly into the document
- **Cannot** be piped (this is why `include` is preferred)

### Scope matters

Every chart has its own `_helpers.tpl` scope:
- `umbrella-chart/templates/_helpers.tpl` → only available in umbrella templates
- `umbrella-chart/charts/myapp/templates/_helpers.tpl` → only available in myapp templates

They are **not shared** between parent and subcharts.

### From our charts:

**myapp/_helpers.tpl** defines: `myapp.name`, `myapp.fullname`, `myapp.chart`, `myapp.labels`, `myapp.selectorLabels`, `myapp.serviceAccountName`

**Used in myapp/deployment.yaml:**
```yaml
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
```

**Used in umbrella-chart/configmap.yaml:**
```yaml
metadata:
  name: {{ include "umbrella-chart.fullname" . }}-config
  labels:
    {{- include "umbrella-chart.labels" . | nindent 4 }}
```

---

## 10. String Functions

### Most commonly used in our charts

| Function | Syntax | Result | Used in our chart? |
|----------|--------|--------|--------------------|
| `default` | `default "fallback" .Val` | Returns `.Val` if non-empty, else `"fallback"` | ✅ _helpers.tpl |
| `printf` | `printf "%s-%s" .Release.Name $name` | Format string like C's printf | ✅ _helpers.tpl |
| `trunc` | `trunc 63 "long string"` | Truncate to 63 characters | ✅ _helpers.tpl |
| `trimSuffix` | `trimSuffix "-" "hello-"` | Remove trailing suffix → `"hello"` | ✅ _helpers.tpl |
| `contains` | `contains "sub" "string"` | `true` if "string" contains "sub" | ✅ _helpers.tpl |
| `replace` | `replace "+" "_" "v1.0+build"` | → `"v1.0_build"` | ✅ _helpers.tpl |
| `quote` | `quote "hello"` | → `"\"hello\""` | ✅ deployment.yaml |
| `lower` | `lower "HELLO"` | → `"hello"` | |
| `upper` | `upper "hello"` | → `"HELLO"` | |
| `trim` | `trim "  hi  "` | → `"hi"` | |
| `indent` | `indent 4 $text` | Indent every line by 4 spaces | |
| `nindent` | `nindent 4 $text` | Newline + indent every line by 4 spaces | ✅ everywhere |
| `hasPrefix` | `hasPrefix "cat" "catch"` | → `true` | |
| `hasSuffix` | `hasSuffix "sh" "bash"` | → `true` | |
| `repeat` | `repeat 3 "ha"` | → `"hahaha"` | |
| `substr` | `substr 0 5 "hello world"` | → `"hello"` | |

### From our _helpers.tpl — `printf`, `replace`, `trunc`, `trimSuffix` in one line:
```yaml
{{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
```
**Read:** `"myapp-0.1.0"` → replace `+` with `_` → truncate to 63 → remove trailing `-`

### From our deployment.yaml — `default`:
```yaml
image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
```
**Read:** "Use `.Values.image.tag`, but if it's empty, fall back to `.Chart.AppVersion`."

### From our _helpers.tpl — `quote`:
```yaml
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
```
**Read:** Wraps the value in double quotes — important when the value might be interpreted as a number.

---

## 11. Type Conversion Functions

| Function | Converts to | Example |
|----------|------------|---------|
| `toString` | string | `toString 123` → `"123"` |
| `toJson` | JSON string | `toJson .Values.config` |
| `toYaml` | YAML string | `toYaml .Values.labels` |
| `fromYaml` | Go object from YAML | `$obj := fromYaml $yamlString` |
| `fromJson` | Go object from JSON | `$obj := fromJson $jsonString` |
| `int` | integer | `int "123"` → `123` |
| `int64` | int64 | `int64 "123"` |
| `float64` | float64 | `float64 "1.5"` |
| `atoi` | int (from string) | `atoi "123"` → `123` |
| `quote` | double-quoted string | `quote .Values.name` |
| `squote` | single-quoted string | `squote .Values.name` |

### The most important one — `toYaml`:

```yaml
# Given values.yaml:
#   resources:
#     limits:
#       cpu: 100m
#       memory: 128Mi

resources:
  {{- toYaml .Values.resources | nindent 2 }}

# Renders as:
resources:
  limits:
    cpu: 100m
    memory: 128Mi
```

### From our deployment.yaml — the `toYaml + nindent` pattern:
```yaml
{{- with .Values.resources }}
resources:
  {{- toYaml . | nindent 12 }}
{{- end }}
```
This is the **most common Helm pattern** — dump a YAML block with correct indentation.

---

## 12. Logic and Comparison Functions

| Function | Meaning | Example |
|----------|---------|---------|
| `and` | Boolean AND | `and .Val1 .Val2` |
| `or` | Boolean OR | `or .Val1 .Val2` |
| `not` | Boolean NOT | `not .Values.autoscaling.enabled` |
| `eq` | Equal `==` | `eq .Values.env "production"` |
| `ne` | Not equal `!=` | `ne .Values.env "test"` |
| `lt` | Less than `<` | `lt .Values.replicas 3` |
| `le` | Less or equal `<=` | `le .Values.replicas 3` |
| `gt` | Greater than `>` | `gt .Values.replicas 1` |
| `ge` | Greater or equal `>=` | `ge .Values.replicas 1` |
| `empty` | Check if empty | `empty .Values.name` |
| `required` | Fail if empty | `required "name is required!" .Values.name` |
| `fail` | Unconditional error | `fail "something went wrong"` |
| `coalesce` | First non-empty value | `coalesce .Val1 .Val2 "default"` |
| `ternary` | Ternary operator | `ternary "yes" "no" .Values.enabled` |

### From our deployment.yaml — `not`:
```yaml
{{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
{{- end }}
```

### Combining logic operators:
```yaml
{{- if and .Values.ingress.enabled (eq .Values.ingress.className "nginx") }}
  # Only when ingress is enabled AND className is nginx
{{- end }}
```

### `required` — Force users to provide a value:
```yaml
image: {{ required "image.repository must be set!" .Values.image.repository }}
```

### `coalesce` — First non-empty value wins:
```yaml
namespace: {{ coalesce .Values.namespace .Release.Namespace "default" }}
```

### `ternary` — Inline conditional:
```yaml
readOnly: {{ ternary "true" "false" .Values.readOnlyMode }}
```

---

## 13. YAML & Indentation Functions

These are **critical** for producing valid YAML output.

| Function | What it does | Example |
|----------|-------------|---------|
| `toYaml` | Converts Go object to YAML string | `toYaml .Values.labels` |
| `nindent N` | Prepends newline + indents every line by N spaces | `nindent 4 $text` |
| `indent N` | Indents every line by N spaces (no leading newline) | `indent 4 $text` |

### The `toYaml | nindent` pattern (you'll use this constantly):

```yaml
# Pattern:
labels:
  {{- include "myapp.labels" . | nindent 4 }}

# Why nindent 4?
# "labels:" is at indent 0
#  Each label under it needs 4 spaces (2 for YAML nesting × 2 levels, or whatever context needs)
```

### Choosing the right indent number:

```yaml
# deployment.yaml context shows why numbers vary:
metadata:          # indent 0
  labels:          # indent 2 — so labels content needs nindent 4
    {{- include "myapp.labels" . | nindent 4 }}

spec:
  template:
    metadata:
      labels:      # indent 6 — so labels content needs nindent 8
        {{- include "myapp.labels" . | nindent 8 }}
```

---

## 14. Dict (Dictionary) Functions

Dicts are key/value maps. Useful for building dynamic data.

| Function | What it does | Example |
|----------|-------------|---------|
| `dict` | Create a dictionary | `dict "key1" "val1" "key2" "val2"` |
| `get` | Get value by key | `get $myDict "key1"` |
| `set` | Add/update key | `$_ := set $myDict "key" "val"` |
| `unset` | Remove a key | `$_ := unset $myDict "key"` |
| `hasKey` | Check if key exists | `hasKey $myDict "key"` |
| `keys` | Get all keys (unsorted) | `keys $myDict` |
| `values` | Get all values | `values $myDict` |
| `merge` | Merge dicts (first wins) | `merge $dest $source` |
| `mergeOverwrite` | Merge dicts (last wins) | `mergeOverwrite $dest $source` |
| `pick` | Keep only specified keys | `pick $myDict "key1" "key2"` |
| `omit` | Remove specified keys | `omit $myDict "key1"` |
| `dig` | Safely traverse nested dicts | `dig "a" "b" "c" "default" $myDict` |
| `deepCopy` | Deep copy a dict | `deepCopy $myDict` |

### Practical example — passing extra data to a template:
```yaml
{{- include "myapp.labels" (dict "root" . "extra" "value") }}
```
Inside the template, access with `{{ .root.Release.Name }}` and `{{ .extra }}`.

---

## 15. List Functions

| Function | What it does | Example |
|----------|-------------|---------|
| `list` | Create a list | `list "a" "b" "c"` |
| `first` | First element | `first $myList` |
| `last` | Last element | `last $myList` |
| `rest` | Everything except first | `rest $myList` |
| `append` | Add to end | `append $myList "d"` |
| `prepend` | Add to front | `prepend $myList "z"` |
| `concat` | Join lists | `concat $list1 $list2` |
| `has` | Check if element exists | `has "a" $myList` |
| `without` | Remove elements | `without $myList "a" "b"` |
| `uniq` | Remove duplicates | `uniq $myList` |
| `reverse` | Reverse order | `reverse $myList` |
| `compact` | Remove empty entries | `compact $myList` |
| `index` | Get Nth element | `index $myList 0` |
| `len` | Length of list | `len $myList` |
| `until` | Generate 0..N-1 | `until 5` → `[0 1 2 3 4]` |
| `sortAlpha` | Sort strings alphabetically | `sortAlpha $myList` |
| `chunk` | Split into sub-lists | `chunk 3 $myList` |

### Practical: iterating a generated range
```yaml
{{- range $i := until 3 }}
worker-{{ $i }}: enabled
{{- end }}
# Produces:
# worker-0: enabled
# worker-1: enabled
# worker-2: enabled
```

---

## 16. Math Functions

| Function | What it does | Example |
|----------|-------------|---------|
| `add` | Addition | `add 1 2` → `3` |
| `sub` | Subtraction | `sub 5 2` → `3` |
| `mul` | Multiplication | `mul 2 3` → `6` |
| `div` | Integer division | `div 10 3` → `3` |
| `mod` | Modulo | `mod 10 3` → `1` |
| `max` | Maximum | `max 1 5 3` → `5` |
| `min` | Minimum | `min 1 5 3` → `1` |
| `add1` | Increment by 1 | `add1 5` → `6` |
| `ceil` | Round up (float) | `ceil 1.1` → `2` |
| `floor` | Round down (float) | `floor 1.9` → `1` |
| `round` | Round to N decimals | `round 3.456 2` → `3.46` |

---

## 17. Flow Control Functions

These supplement the `if/with/range` blocks:

| Function | What it does | Example |
|----------|-------------|---------|
| `default` | Default if empty | `default "nginx" .Values.image` |
| `empty` | Check if empty | `if empty .Values.name` |
| `required` | Fail with error if empty | `required "msg" .Values.name` |
| `fail` | Always fail with message | `fail "not supported"` |
| `coalesce` | First non-empty from list | `coalesce .a .b "fallback"` |
| `ternary` | If-then-else inline | `ternary "a" "b" $condition` |

---

## 18. Other Useful Functions

### Encoding
| Function | What it does |
|----------|-------------|
| `b64enc` | Base64 encode |
| `b64dec` | Base64 decode |
| `b32enc` | Base32 encode |

### Regex
| Function | What it does |
|----------|-------------|
| `regexMatch` | Test if string matches regex |
| `regexFind` | Find first regex match |
| `regexReplaceAll` | Replace all regex matches |
| `regexSplit` | Split string by regex |

### Crypto
| Function | What it does |
|----------|-------------|
| `sha256sum` | SHA-256 hash of string |
| `randAlphaNum` | Random alphanumeric string |
| `htpasswd` | Generate bcrypt password hash |

### UUID
| Function | What it does |
|----------|-------------|
| `uuidv4` | Generate random UUID v4 |

### Kubernetes
| Function | What it does |
|----------|-------------|
| `lookup` | Query live cluster resources (empty in `helm template`) |

### File Paths
| Function | What it does |
|----------|-------------|
| `base` | Last element of path: `base "a/b/c"` → `"c"` |
| `dir` | Directory part: `dir "a/b/c"` → `"a/b"` |
| `ext` | File extension: `ext "file.yaml"` → `".yaml"` |
| `isAbs` | Check if path is absolute |

### Reflection
| Function | What it does |
|----------|-------------|
| `kindOf` | Go kind of value: `kindOf "hi"` → `"string"` |
| `typeOf` | Go type of value |
| `deepEqual` | Deep equality comparison |

---

## 19. Quick Reference: Patterns from Our Charts

### Pattern 1: Safe name generation
```yaml
# From myapp/_helpers.tpl — "myapp.name"
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
```
**Why:** Kubernetes DNS names must be ≤63 chars and not end with `-`.

### Pattern 2: Fully qualified name with collision avoidance
```yaml
# From myapp/_helpers.tpl — "myapp.fullname"
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
```
**Why:** Avoids `my-myapp-myapp` duplication when release name already contains the chart name.

### Pattern 3: Chart label with version safety
```yaml
# From myapp/_helpers.tpl — "myapp.chart"
{{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
```
**Why:** SemVer build metadata uses `+` which is invalid in label values, so we replace it with `_`.

### Pattern 4: Standard Kubernetes labels
```yaml
# From myapp/_helpers.tpl — "myapp.labels"
helm.sh/chart: {{ include "myapp.chart" . }}
{{ include "myapp.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
```

### Pattern 5: Conditional whole-section rendering with `with`
```yaml
# From deployment.yaml — render block only if value is non-empty
{{- with .Values.podAnnotations }}
annotations:
  {{- toYaml . | nindent 8 }}
{{- end }}
```

### Pattern 6: toYaml + nindent for arbitrary YAML blocks
```yaml
# From deployment.yaml — dump any YAML at correct indent level
{{- with .Values.resources }}
resources:
  {{- toYaml . | nindent 12 }}
{{- end }}
```

### Pattern 7: Include helper + nindent for labels
```yaml
# From deployment.yaml — apply labels at the right indent
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
```

### Pattern 8: Conditional service account
```yaml
# From myapp/_helpers.tpl — "myapp.serviceAccountName"
{{- if .Values.serviceAccount.create }}
{{- default (include "myapp.fullname" .) .Values.serviceAccount.name }}
{{- else }}
{{- default "default" .Values.serviceAccount.name }}
{{- end }}
```
**Read:** "If we're creating a SA, use its name (or fallback to fullname). Otherwise use the name (or fallback to 'default')."

### Pattern 9: Image tag with appVersion fallback
```yaml
# From deployment.yaml
image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
```
**Why:** Lets users override the tag, but defaults to the chart's `appVersion`.

### Pattern 10: Global values in subcharts
```yaml
# From deployment.yaml — accessing umbrella globals
environment: {{ .Values.global.environment }}

# From umbrella-chart/templates/_helpers.tpl
environment: {{ .Values.global.environment }}
```
**Why:** `global:` values from the umbrella chart automatically propagate to all subcharts via `.Values.global`.

---

## Cheat Sheet: Reading Template Expressions

When you see a complex expression, read it **left to right** following the pipes:

```yaml
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
```

| Step | Operation | Result |
|------|-----------|--------|
| 1 | `printf "%s-%s" .Release.Name $name` | `"my-release-myapp"` |
| 2 | `\| trunc 63` | `"my-release-myapp"` (already <63) |
| 3 | `\| trimSuffix "-"` | `"my-release-myapp"` (no trailing -) |

```yaml
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
```

| Step | Operation | Result |
|------|-----------|--------|
| 1 | `default .Chart.Name .Values.nameOverride` | `.Values.nameOverride` if set, else `.Chart.Name` |
| 2 | `\| trunc 63` | Truncate to 63 chars |
| 3 | `\| trimSuffix "-"` | Remove trailing dash |

---

## Debugging Commands

```bash
# Render templates locally without installing:
helm template my-release ./umbrella-chart

# Render with verbose debug output:
helm template my-release ./umbrella-chart --debug

# Render only a specific template:
helm template my-release ./umbrella-chart -s templates/configmap.yaml

# Check chart for common issues:
helm lint ./umbrella-chart

# Dry-run against a real cluster (validates against API server):
helm install my-release ./umbrella-chart --dry-run

# Show the computed values (merged from all sources):
helm show values ./umbrella-chart
```

---

> **Source:** Official Helm docs at [helm.sh/docs/chart_template_guide/function_list](https://helm.sh/docs/chart_template_guide/function_list/) and Go `text/template` package. All "from our chart" examples come from actual files in this workspace.
