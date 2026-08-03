# Phase 6 Test Plan: Deterministic Rule Synthesis

## 1. Purpose and test discipline

This is the exhaustive regression plan for
[Phase 6](phase-6-plan.md). It covers the field-region producer correction,
the closed rule grammar, context alignment, coupled anti-unification, carrier
checks, duplicate compression, canonicalization, deterministic serialization,
and the standalone offline command.

Every synthesis case below gives exact normalized input. The compact Python
constructors in Section 3 are notation for literal JSON dictionaries; their
definitions are complete, contain no synthesis logic, and are the exact test
fixture builders. A case's `doc(...)` expression is therefore the complete
schema-version-1 input file. Keep each case's `doc(...)` call local to its test
even if the implementation shares the constructor helpers.

Rust producer tests contain their exact source locally and use the existing
source-string observation compiler harness. They must not invoke `crat-tool`,
write project files, or use a project-root `tests/` directory. Python tests are
offline and must not invoke Rust, Cargo, a provider, or the network.

The planned suite contains 69 named cases:

| Area | Cases |
| --- | ---: |
| Field-region production | 5 |
| Closed model and serialization | 9 |
| Canonical coupled-synthesis examples | 26 |
| Additional grammar and algorithm coverage | 18 |
| Enumeration, determinism, and command boundary | 11 |

## 2. Execution and comparison policy

Rust tests live in `crates/tools/src/observation.rs`. Run from
`proctor/stages/crat`:

```bash
cargo test -p tools observation::tests
cargo test -p tools
cargo test --workspace
cargo fmt
cargo clippy --workspace --all-targets
```

Python tests remain in `proctor/tests/test_local_transformation.py`. Run from
`proctor`:

```bash
uv run pytest tests/test_local_transformation.py
uv run pytest
uv run ruff check stages/local-transformation tests/test_local_transformation.py
uv run ruff format --check stages/local-transformation tests/test_local_transformation.py
uv run mypy proctor
uv run mypy stages/local-transformation/model.py \
  stages/local-transformation/rule_synthesis.py \
  stages/local-transformation/extract_rules.py
```

For every accepted pair, assert both canonical patterns, the complete context,
variable sorts and indices, and exact reconstruction of seed one and seed two.
For every rejected pair, assert an empty rule array and the named pair-local
reason through an internal result enum or test-only trace; the public command
need not print pair-local reasons. Representative documents compare exact
pretty JSON bytes including one terminal newline. Determinism cases compare
complete bytes.

## 3. Exact normalized-JSON notation

The following definitions are exact fixture code. `copy.deepcopy` is used when
a value occurs more than once so a test cannot accidentally share mutable
subtrees.

```python
import copy

def prim(name):
    return {"kind": "primitive", "name": name}

I32 = prim("i32")
BOOL = prim("bool")
USIZE = prim("usize")
RAW_I32 = {"kind": "raw_pointer", "mutability": "const", "pointee": I32}
REF_I32 = {"kind": "reference", "mutability": "shared", "pointee": I32}
SLICE_I32 = {"kind": "slice", "element": I32}
MUT_SLICE_I32 = {
    "kind": "reference", "mutability": "mutable", "pointee": SLICE_I32
}

def local_adt(kind, index=0):
    return {"kind": "local", "id": f"<{kind}{index}>"}

def local_adt_type(kind="struct", index=0):
    return {
        "kind": "adt", "adt_kind": kind,
        "identity": local_adt(kind, index), "arguments": [],
    }

def local_member(kind, owner_kind="struct", owner_index=0, index=0):
    return {
        "kind": "local",
        "owner": local_adt(owner_kind, owner_index),
        "id": f"<{kind}{index}>",
    }

def external(crate, *path):
    return {"kind": "external", "crate": crate, "path": list(path)}

def binding(index):
    return {"kind": "path", "value": {"kind": "binding", "id": f"<id{index}>"}}

def local_value(kind, index=0):
    prefix = {
        "function": "fn", "constant": "const", "static": "static",
        "method": "method",
    }[kind]
    return {"kind": "path", "value": {"kind": kind, "id": f"<{prefix}{index}>"}}

def ext_path(name):
    return {"kind": "path", "value": external("fixture", name)}

def foreign_path(kind, symbol):
    return {"kind":"path", "value":{"kind":kind, "symbol":symbol}}

def constructor_path(owner_kind="enum", owner_index=0, variant_index=0):
    return {
        "kind":"path",
        "value":{
            "kind":"constructor",
            "adt":local_adt(owner_kind, owner_index),
            "variant":local_member(
                "variant", owner_kind, owner_index, variant_index
            ),
        },
    }

def call(name, *arguments):
    return {"kind": "call", "callee": ext_path(name), "arguments": list(arguments)}

def call_expr(callee, *arguments):
    return {"kind": "call", "callee": callee, "arguments": list(arguments)}

def method(receiver, crate, path, *arguments):
    return {
        "kind": "method_call", "receiver": receiver,
        "method": external(crate, *path), "arguments": list(arguments),
    }

def local_method(receiver, index=0, *arguments):
    return {
        "kind": "method_call", "receiver": receiver,
        "method": {"kind": "method", "id": f"<method{index}>"},
        "arguments": list(arguments),
    }

def unary(operator, operand):
    return {"kind": "unary", "operator": operator, "operand": operand}

def binary(operator, left, right):
    return {"kind": "binary", "operator": operator, "left": left, "right": right}

def integer(value, ty):
    return {
        "kind": "literal",
        "value": {"kind": "integer", "value": str(value), "type": ty},
    }

def integer_text(value, ty):
    return {
        "kind": "literal",
        "value": {"kind": "integer", "value": value, "type": ty},
    }

def boolean(value):
    return {"kind": "literal", "value": {"kind": "bool", "value": value}}

def literal(value):
    return {"kind":"literal", "value":value}

def offset(base, amount):
    return method(base, "core", ("ptr", "const_ptr", "offset"), amount)

def range_from(start):
    return {"kind": "range", "start": start, "end": None, "limits": "half_open"}

def index(base, value):
    return {"kind": "index", "base": base, "index": value}

def mutable_slice_from(base, start):
    return {
        "kind": "address_of", "borrow": "reference", "mutability": "mut",
        "expression": index(base, range_from(start)),
    }

def field(base, owner_kind="struct", owner_index=0, field_index=0):
    return {
        "kind": "field", "base": base,
        "field": local_member("field", owner_kind, owner_index, field_index),
    }

def named_struct(value):
    return {
        "kind": "struct", "adt": local_adt("struct", 0), "variant": None,
        "fields": [{"field": local_member("field"), "value": value}],
        "rest": None,
    }

def anchor(index, target_type=MUT_SLICE_I32):
    return {
        "id": f"<id{index}>", "source_type": copy.deepcopy(RAW_I32),
        "target_type": copy.deepcopy(target_type),
    }

def obs(source, target, *, anchors=None, root_types=None):
    if anchors is None:
        anchors = [anchor(0)]
    if root_types is None:
        root_types = (copy.deepcopy(I32), copy.deepcopy(I32),
                      copy.deepcopy(I32), copy.deepcopy(I32))
    return {
        "source_expression": source,
        "target_expression": target,
        "pointer_anchors": anchors,
        "source_type": root_types[0],
        "source_adjusted_type": root_types[1],
        "target_type": root_types[2],
        "target_adjusted_type": root_types[3],
    }

def doc(*observations):
    return {"schema_version": 1, "observations": list(observations)}

def var(sort, index):
    return {"kind": "variable", "sort": sort, "index": index}
```

The slightly unusual `offset` call expands exactly to the external identity
`{"kind":"external","crate":"core","path":["ptr","const_ptr","offset"]}`.
All `doc(...)` values below pass the existing strict observation loader unless
the case explicitly tests loader failure.

## 4. Field-region production

### P6-FIELD-01 `promotes_immediate_field_and_serializes_owner`

Exact Rust input:

```rust
struct Pair { value: i32 }
unsafe fn source_copy(mut pointer: *const Pair) -> i32 {
    #[proctor(0)] (*pointer).value
}
unsafe fn target(mut pointer: &Pair) -> i32 {
    #[proctor(0)] pointer.value
}
```

Expected result: one observation. Its roots are the complete source
`(*pointer).value` and target `pointer.value`, not their bases. Both serialized
field identities are exactly local `<struct0>::<field0>`, and the anchor is
`<id0>`.

### P6-FIELD-02 `nested_field_promotes_only_inner_parent`

```rust
struct Inner { value: i32 }
struct Outer { inner: Inner }
unsafe fn source_copy(mut pointer: *const Outer) -> i32 {
    #[proctor(0)] (*pointer).inner.value
}
unsafe fn target(mut pointer: &Outer) -> i32 {
    #[proctor(0)] pointer.inner.value
}
```

Expected result: one observation rooted at `(*pointer).inner` and
`pointer.inner`. The outer `.value` is alignment context and does not appear in
either serialized root.

### P6-FIELD-03 `different_resolved_field_skips_unit`

```rust
struct Pair { left: i32, right: i32 }
unsafe fn source_copy(mut pointer: *const Pair) -> i32 {
    #[proctor(0)] (*pointer).left
}
unsafe fn target(mut pointer: &Pair) -> i32 {
    #[proctor(0)] pointer.right
}
```

Expected result: successful empty observation document. The promoted roots are
both fields, but their resolved members differ.

### P6-FIELD-04 `promoted_regions_participate_in_overlap_check`

```rust
struct Pair { value: i32 }
unsafe fn source_copy(mut pointer: *const Pair, mut index: *const isize) -> i32 {
    #[proctor(0)] (*pointer.offset(*index)).value
}
unsafe fn target(mut pointer: &[Pair], mut index: &isize) -> i32 {
    #[proctor(0)] pointer[*index as usize].value
}
```

Expected result: the `pointer` anchor grows through `offset`, dereference, and
the immediate field; the `index` anchor grows through its dereference inside
that selected field. The promoted outer root contains the index region, so the
post-promotion overlap check skips the labeled unit before alignment and emits
no observations.

### P6-FIELD-05 `nonfield_region_behavior_is_unchanged`

```rust
unsafe fn source_copy(mut pointer: *const i32) -> i32 {
    #[proctor(0)] *pointer
}
unsafe fn target(mut pointer: &i32) -> i32 {
    #[proctor(0)] *pointer
}
```

Expected result: the existing unary-dereference source root and target root,
with no promoted-field guard. This is an exact regression for the ordinary
selected-root fast path.

## 5. Closed rule model and serialization

### P6-MODEL-01 `minimal_exact_rule_document_round_trips`

Exact input to `load_rules`:

```json
{
  "schema_version": 1,
  "rules": [
    {
      "source_pattern": {"kind": "unary", "operator": "deref", "operand": {"kind": "path", "value": {"kind": "variable", "sort": "anchor", "index": 0}}},
      "target_pattern": {"kind": "path", "value": {"kind": "variable", "sort": "anchor", "index": 0}},
      "pointer_anchors": [{"id": {"kind": "variable", "sort": "anchor", "index": 0}, "source_type": {"kind": "raw_pointer", "mutability": "const", "pointee": {"kind": "primitive", "name": "i32"}}, "target_type": {"kind": "reference", "mutability": "shared", "pointee": {"kind": "primitive", "name": "i32"}}}],
      "source_type": {"kind": "primitive", "name": "i32"},
      "source_adjusted_type": {"kind": "primitive", "name": "i32"},
      "target_type": {"kind": "primitive", "name": "i32"},
      "target_adjusted_type": {"kind": "primitive", "name": "i32"}
    }
  ]
}
```

Expected result: successful load and byte-stable `rules_to_json` output with
two-space indentation and exactly one terminal newline.

### P6-MODEL-02 `all_variable_positions_are_closed`

Start with the exact document in P6-MODEL-01 and make each mutation separately:

```python
unknown_sort = copy.deepcopy(value)
unknown_sort["rules"][0]["source_pattern"]["operand"]["value"]["sort"] = "value"
boolean_index = copy.deepcopy(value)
boolean_index["rules"][0]["source_pattern"]["operand"]["value"]["index"] = True
negative_index = copy.deepcopy(value)
negative_index["rules"][0]["source_pattern"]["operand"]["value"]["index"] = -1
wrong_position = copy.deepcopy(value)
wrong_position["rules"][0]["source_pattern"]["operand"]["value"]["sort"] = "expression"
extra_variable_key = copy.deepcopy(value)
extra_variable_key["rules"][0]["source_pattern"]["operand"]["value"]["extra"] = 0
too_large_index = copy.deepcopy(value)
too_large_index["rules"][0]["source_pattern"]["operand"]["value"]["index"] = 2**64
```

Expected result: each load fails.

### P6-MODEL-03 `integer_magnitude_variable_is_only_literal_value`

Exact accepted fragment:

```json
{"kind":"literal","value":{"kind":"integer","value":{"kind":"variable","sort":"integer_magnitude","index":0},"type":"isize"}}
```

Let `value` be the parsed exact document from P6-MODEL-01 and set its
`source_pattern` to the fragment above. Expected: accepted. Exact rejected
mutations of that accepted value:

```python
in_type = copy.deepcopy(value)
in_type["rules"][0]["source_pattern"]["value"]["type"] = var(
    "integer_magnitude", 0
)
wrong_value_sort = copy.deepcopy(value)
wrong_value_sort["rules"][0]["source_pattern"]["value"]["value"] = var(
    "expression", 0
)
complete_magnitude = copy.deepcopy(value)
complete_magnitude["rules"][0]["source_pattern"] = var(
    "integer_magnitude", 0
)
```

### P6-MODEL-04 `local_member_owner_remains_structural`

Exact accepted document:

```python
owned_field = {
    "kind":"local",
    "owner":var("struct", 0),
    "id":var("field", 0),
}
source = {
    "kind":"field",
    "base":{"kind":"path", "value":var("anchor", 0)},
    "field":owned_field,
}
target = copy.deepcopy(source)
accepted = {
    "schema_version":1,
    "rules":[{
        "source_pattern":source,
        "target_pattern":target,
        "pointer_anchors":[{
            "id":var("anchor", 0),
            "source_type":copy.deepcopy(RAW_I32),
            "target_type":copy.deepcopy(REF_I32),
        }],
        "source_type":{
            "kind":"adt", "adt_kind":"struct",
            "identity":var("struct", 0), "arguments":[],
        },
        "source_adjusted_type":copy.deepcopy(I32),
        "target_type":copy.deepcopy(I32),
        "target_adjusted_type":copy.deepcopy(I32),
    }],
}
rule = accepted["rules"][0]
```

Expected: accepted; context establishes `struct` and source establishes its
owned field. Exact rejected mutations:

```python
whole_member_variable = copy.deepcopy(accepted)
whole_member_variable["rules"][0]["source_pattern"]["field"] = var("field", 0)
wrong_owner_sort = copy.deepcopy(accepted)
wrong_owner_sort["rules"][0]["source_pattern"]["field"]["owner"] = var("field", 0)
wrong_member_sort = copy.deepcopy(accepted)
wrong_member_sort["rules"][0]["source_pattern"]["field"]["id"] = var("struct", 0)
```

Exact accepted owned-variant subcase:

```python
accepted_variant = copy.deepcopy(accepted)
variant_rule = accepted_variant["rules"][0]
owned_variant = {
    "kind":"local", "owner":var("enum", 0),
    "id":var("variant", 0),
}
constructor = {
    "kind":"path",
    "value":{"kind":"constructor", "adt":var("enum", 0),
             "variant":owned_variant},
}
anchor_path = {"kind":"path", "value":var("anchor", 0)}
pattern = {"kind":"call", "callee":constructor, "arguments":[anchor_path]}
variant_rule["source_pattern"] = pattern
variant_rule["target_pattern"] = copy.deepcopy(pattern)
variant_rule["source_type"] = {
    "kind":"adt", "adt_kind":"enum", "identity":var("enum", 0),
    "arguments":[],
}

whole_variant_variable = copy.deepcopy(accepted_variant)
whole_variant_variable["rules"][0]["source_pattern"]["callee"]["value"]["variant"] = var("variant", 0)
wrong_variant_owner = copy.deepcopy(accepted_variant)
wrong_variant_owner["rules"][0]["source_pattern"]["callee"]["value"]["variant"]["owner"] = var("variant", 0)
wrong_variant_id = copy.deepcopy(accepted_variant)
wrong_variant_id["rules"][0]["source_pattern"]["callee"]["value"]["variant"]["id"] = var("enum", 0)
```

Expected: `accepted` and `accepted_variant` load successfully with owners
explicit. All six named rejected mutations fail, each derived unambiguously
from its matching accepted document.

### P6-MODEL-05 `document_rule_and_nested_unknown_fields_reject`

Starting from `value = json.loads(<the exact P6-MODEL-01 JSON>)`, create:

```python
unknown_locations = []
for path in [
    (), ("rules", 0), ("rules", 0, "pointer_anchors", 0),
    ("rules", 0, "source_pattern"),
    ("rules", 0, "source_pattern", "operand", "value"),
    ("rules", 0, "source_type"),
]:
    mutation = copy.deepcopy(value)
    nested = mutation
    for component in path:
        nested = nested[component]
    nested["extra"] = True
    unknown_locations.append(mutation)
versions = []
for version in [2, 0, True, "1"]:
    mutation = copy.deepcopy(value)
    mutation["schema_version"] = version
    versions.append(mutation)
```

Add the local-member `extra` mutation to the exact accepted value from
P6-MODEL-04. Every load fails without returning a partial document.

### P6-MODEL-06 `canonical_indices_and_target_availability_are_checked`

Starting from `value = json.loads(<the exact P6-MODEL-01 JSON>)`, construct:

```python
noncanonical_anchor = copy.deepcopy(value)
noncanonical_anchor["rules"][0]["pointer_anchors"][0]["id"]["index"] = 1
duplicate_anchor = copy.deepcopy(value)
duplicate_anchor["rules"][0]["pointer_anchors"].append(
    copy.deepcopy(duplicate_anchor["rules"][0]["pointer_anchors"][0])
)
missing_e0 = copy.deepcopy(value)
missing_e0["rules"][0]["source_pattern"] = var("expression", 1)
target_only_e = copy.deepcopy(value)
target_only_e["rules"][0]["target_pattern"] = var("expression", 0)
target_only_field = copy.deepcopy(value)
target_only_field["rules"][0]["target_pattern"] = {
    "kind":"field",
    "base":{"kind":"path", "value":var("anchor", 0)},
    "field":{"kind":"local", "owner":var("struct", 0),
             "id":var("field", 0)},
}

anchor_a0 = copy.deepcopy(value["rules"][0]["pointer_anchors"][0])
anchor_a1 = copy.deepcopy(anchor_a0)
anchor_a1["id"] = var("anchor", 1)
anchor_order_a1_a0 = copy.deepcopy(value)
anchor_order_a1_a0["rules"][0]["pointer_anchors"] = [anchor_a1, anchor_a0]
anchor_order_a1_a0["rules"][0]["source_pattern"] = {
    "kind":"call", "callee":ext_path("pair"),
    "arguments":[
        {"kind":"path", "value":var("anchor", 1)},
        {"kind":"path", "value":var("anchor", 0)},
    ],
}
anchor_order_a1_a0["rules"][0]["target_pattern"] = copy.deepcopy(
    anchor_order_a1_a0["rules"][0]["source_pattern"]
)

source_order_b1_b0 = copy.deepcopy(value)
source_order_b1_b0["rules"][0]["source_pattern"] = {
    "kind":"call", "callee":ext_path("ordered"),
    "arguments":[
        {"kind":"path", "value":var("binding", 1)},
        {"kind":"path", "value":var("binding", 0)},
        {"kind":"path", "value":var("anchor", 0)},
    ],
}
source_order_b1_b0["rules"][0]["target_pattern"] = copy.deepcopy(
    source_order_b1_b0["rules"][0]["source_pattern"]
)
```

Every load fails. `anchor_order_a1_a0` and `source_order_b1_b0` each contain a
contiguous index set `{0,1}`, but first occurrence is `1,0`; canonicality is
about traversal order, not mere contiguity. Exact positive subcase:

```python
available_anchor = {"kind":"path", "value":var("anchor", 0)}
repeated = {
    "kind":"call", "callee":ext_path("pair"),
    "arguments":[available_anchor, copy.deepcopy(available_anchor)],
}
positive = copy.deepcopy(value)
positive["rules"][0]["source_pattern"] = repeated
positive["rules"][0]["target_pattern"] = copy.deepcopy(repeated)
```

It loads successfully because both occurrences reuse available `A0`.

### P6-MODEL-07 `empty_document_has_exact_bytes`

Exact value: `RuleDocument(rules=())`. Exact bytes:

```json
{
  "schema_version": 1,
  "rules": []
}
```

There is exactly one newline after `}` and no other trailing bytes.

### P6-MODEL-08 `rule_loader_rejects_concrete_local_ids`

Start from the exact rule document in P6-MODEL-01. Apply these independent
exact mutations:

```python
concrete_anchor = copy.deepcopy(value)
concrete_anchor["rules"][0]["pointer_anchors"][0]["id"] = "<id0>"

concrete_path = copy.deepcopy(value)
concrete_path["rules"][0]["source_pattern"]["operand"]["value"] = {
    "kind":"binding", "id":"<id0>"
}

concrete_adt = copy.deepcopy(value)
concrete_adt["rules"][0]["source_type"] = local_adt_type("struct", 0)

concrete_member = copy.deepcopy(value)
concrete_member["rules"][0]["source_pattern"] = field(
    unary("deref", {"kind":"path", "value":var("anchor", 0)})
)
```

Expected: every `load_rules` call fails. A rule document contains variables for
all local semantic identities; observation alpha names are never legal rule
constants. The external identities and foreign symbols used in later cases
remain legal concrete values.

### P6-MODEL-09 `observation_loader_rejects_every_variable_position`

Use this exact valid base document:

```python
base = doc(obs(unary("deref", binding(0)), binding(0),
               anchors=[anchor(0, REF_I32)]))
```

Apply these independent mutations:

```python
expression_variable = copy.deepcopy(base)
expression_variable["observations"][0]["source_expression"] = var("expression", 0)

identity_variable = copy.deepcopy(base)
identity_variable["observations"][0]["source_expression"]["operand"]["value"] = var("anchor", 0)

anchor_variable = copy.deepcopy(base)
anchor_variable["observations"][0]["pointer_anchors"][0]["id"] = var("anchor", 0)

magnitude_variable = copy.deepcopy(base)
magnitude_variable["observations"][0]["source_expression"] = {
    "kind":"literal",
    "value":{"kind":"integer", "value":var("integer_magnitude", 0),
             "type":"isize"},
}

adt_base = doc(obs(
    unary("deref", binding(0)), binding(0),
    anchors=[anchor(0, REF_I32)],
    root_types=(local_adt_type("struct", 0), I32, I32, I32),
))
adt_identity_variable = copy.deepcopy(adt_base)
adt_identity_variable["observations"][0]["source_type"]["identity"] = var(
    "struct", 0
)

member_base = doc(obs(
    field(unary("deref", binding(0))), field(binding(0)),
    anchors=[anchor(0, REF_I32)],
))
member_owner_variable = copy.deepcopy(member_base)
member_owner_variable["observations"][0]["source_expression"]["field"]["owner"] = var(
    "struct", 0
)
member_id_variable = copy.deepcopy(member_base)
member_id_variable["observations"][0]["source_expression"]["field"]["id"] = var(
    "field", 0
)

pattern_source = {
    "kind":"block",
    "block":{"statements":[
        {"kind":"let",
         "pattern":{"kind":"binding", "id":"<id0>",
                    "mutability":"mutable", "by_ref":"no"},
         "type":copy.deepcopy(I32),
         "initializer":unary("deref", binding(1))},
        {"kind":"expression", "expression":binding(0), "semicolon":False},
    ]},
}
pattern_target = copy.deepcopy(pattern_source)
pattern_target["block"]["statements"][0]["initializer"] = binding(1)
pattern_base = doc(obs(
    pattern_source, pattern_target, anchors=[anchor(1, REF_I32)]
))
binding_pattern_variable = copy.deepcopy(pattern_base)
binding_pattern_variable["observations"][0]["source_expression"]["block"]["statements"][0]["pattern"]["id"] = var(
    "binding", 0
)
```

Expected: `load_observations(json.dumps(mutation))` fails for every mutation.
`adt_base`, `member_base`, and `pattern_base` themselves load successfully
before mutation.

Exact positive `load_rules` controls for the newly covered positions are the
`accepted` ADT/member document from P6-MODEL-04 and this binding-pattern rule,
derived from the exact P6-MODEL-01 document:

```python
binding_pattern_rule = copy.deepcopy(value)
binding_rule = binding_pattern_rule["rules"][0]
rule_pattern = {
    "kind":"block",
    "block":{"statements":[
        {"kind":"let",
         "pattern":{"kind":"binding", "id":var("binding", 0),
                    "mutability":"mutable", "by_ref":"no"},
         "type":copy.deepcopy(I32),
         "initializer":{"kind":"path", "value":var("anchor", 0)}},
        {"kind":"expression",
         "expression":{"kind":"path", "value":var("binding", 0)},
         "semicolon":False},
    ]},
}
binding_rule["source_pattern"] = rule_pattern
binding_rule["target_pattern"] = copy.deepcopy(rule_pattern)

wrong_adt_rule_sort = copy.deepcopy(accepted)
wrong_adt_rule_sort["rules"][0]["source_type"]["identity"] = var("field", 0)
wrong_member_owner_rule_sort = copy.deepcopy(accepted)
wrong_member_owner_rule_sort["rules"][0]["source_pattern"]["field"]["owner"] = var("field", 0)
wrong_member_id_rule_sort = copy.deepcopy(accepted)
wrong_member_id_rule_sort["rules"][0]["source_pattern"]["field"]["id"] = var("struct", 0)
anchor_pattern_rule = copy.deepcopy(binding_pattern_rule)
anchor_pattern_rule["rules"][0]["source_pattern"]["block"]["statements"][0]["pattern"]["id"] = var("anchor", 0)
anchor_pattern_rule["rules"][0]["target_pattern"] = copy.deepcopy(
    anchor_pattern_rule["rules"][0]["source_pattern"]
)

wrong_pattern_rule_sort = copy.deepcopy(binding_pattern_rule)
wrong_pattern_rule_sort["rules"][0]["source_pattern"]["block"]["statements"][0]["pattern"]["id"] = var("expression", 0)
```

Expected: `accepted`, `binding_pattern_rule`, and `anchor_pattern_rule` load
successfully. An anchor variable is valid in a binding-pattern ID when it
denotes the identity already established by the pointer-anchor context. Each
named wrong-sort rule mutation fails. Rule support is position- and
sort-specific and never makes observation loading permissive.

## 6. Canonical coupled-synthesis cases

Unless a case overrides it, the exact input is one `doc` containing the shown
two observations, each with `anchor(0)` and the exact four `I32` root trees
supplied by `obs`. Accepted-rule notation uses `A0`, `B0`, `E0`, `N0`, `S0`,
and `F0` only as prose abbreviations for the exact `var` nodes.

### P6-SYN-01 `integer_magnitude_correspondence`

```python
doc(
    obs(offset(binding(0), integer(1, "isize")),
        mutable_slice_from(binding(0), integer(1, "usize"))),
    obs(offset(binding(0), integer(2, "isize")),
        mutable_slice_from(binding(0), integer(2, "usize"))),
)
```

Expected: one rule equivalent to
`A0.offset(N0_isize) -> &mut A0[N0_usize..]`. The same exact
`integer_magnitude` variable occurs in both literal values despite different
fixed source/target integer types.

### P6-SYN-02 `complete_expression_correspondence`

```python
doc(
    obs(offset(binding(0), binary("add", binding(1), integer(1, "isize"))),
        mutable_slice_from(binding(0), binary("add", binding(1), integer(1, "isize")))),
    obs(offset(binding(0), binary("multiply", binding(1), integer(2, "isize"))),
        mutable_slice_from(binding(0), binary("multiply", binding(1), integer(2, "isize")))),
)
```

Expected: `A0.offset(E0) -> &mut A0[E0..]`. `E0` substitutions contain
ordinary binding `<id1>` in both seeds and no other carrier for that binding.

### P6-SYN-03 `repeated_complete_expression_correspondence`

```python
x1 = binary("add", binding(1), integer(1, "isize"))
x2 = binary("multiply", binding(1), integer(2, "isize"))
doc(
    obs(call("pair", offset(binding(0), copy.deepcopy(x1)),
                    offset(binding(0), copy.deepcopy(x1))),
        call("pair", mutable_slice_from(binding(0), copy.deepcopy(x1)),
                    mutable_slice_from(binding(0), copy.deepcopy(x1)))),
    obs(call("pair", offset(binding(0), copy.deepcopy(x2)),
                    offset(binding(0), copy.deepcopy(x2))),
        call("pair", mutable_slice_from(binding(0), copy.deepcopy(x2)),
                    mutable_slice_from(binding(0), copy.deepcopy(x2)))),
)
```

Expected: both source occurrences and both target occurrences reuse `E0`; no
`E1` is allocated.

### P6-SYN-04 `source_only_magnitude_variable`

```python
doc(
    obs(method(offset(binding(0), integer(1, "isize")),
               "core", ("ptr", "const_ptr", "is_null")), boolean(False),
        root_types=(BOOL, BOOL, BOOL, BOOL)),
    obs(method(offset(binding(0), integer(2, "isize")),
               "core", ("ptr", "const_ptr", "is_null")), boolean(False),
        root_types=(BOOL, BOOL, BOOL, BOOL)),
)
```

Expected: `A0.offset(N0_isize).is_null() -> false`; source-only `N0` is valid.

### P6-SYN-05 `reordered_binding_identities`

```python
value = obs(
    call("mix", unary("deref", binding(0)), binding(1), binding(2)),
    call("mix", binding(0), binding(2), binding(1)),
    anchors=[anchor(0, REF_I32)],
)
doc(copy.deepcopy(value), copy.deepcopy(value))
```

Expected exact identity relationship:
`mix(*A0,B0,B1) -> mix(A0,B1,B0)`. Equal alpha strings across the two
occurrences still become variables; no concrete `<idN>` remains.

### P6-SYN-06 `ordered_disagreement_pairs_do_not_collapse`

```python
doc(
    obs(call("triple", unary("deref", binding(0)), integer(1, "usize"), integer(2, "usize")),
        call("triple", binding(0), integer(1, "usize"), integer(2, "usize")),
        anchors=[anchor(0, REF_I32)]),
    obs(call("triple", unary("deref", binding(0)), integer(2, "usize"), integer(1, "usize")),
        call("triple", binding(0), integer(2, "usize"), integer(1, "usize")),
        anchors=[anchor(0, REF_I32)]),
)
```

Expected: first magnitude position is `N0` for `(1,2)` and second is `N1`
for `(2,1)`. Target reuses them in the same positions.

### P6-SYN-07 `exact_rule_from_repeated_equal_observations`

```python
value = obs(unary("deref", binding(0)), binding(0),
            anchors=[anchor(0, REF_I32)])
doc(copy.deepcopy(value), copy.deepcopy(value))
```

Expected: exact `*A0 -> A0`. A singleton copy of `value` emits no rule.

### P6-SYN-08 `expression_equal_source_and_target_is_retained`

```python
value = obs(unary("deref", binding(0)), unary("deref", binding(0)),
            anchors=[anchor(0, REF_I32)])
doc(copy.deepcopy(value), copy.deepcopy(value))
```

Expected: exact `*A0 -> *A0`; equality of expression patterns is not a reject
condition.

### P6-SYN-09 `ordinary_binding_identities_are_explicit`

```python
value = obs(
    offset(binding(0), binary("add", binding(1), binding(2))),
    mutable_slice_from(binding(0), binary("add", binding(1), binding(2))),
)
doc(copy.deepcopy(value), copy.deepcopy(value))
```

Expected: `A0.offset(B0+B1) -> &mut A0[(B0+B1)..]`.

### P6-SYN-10 `identity_encapsulated_by_one_expression_variable`

```python
doc(
    obs(binary("add", integer(1, "i32"),
               unary("deref", offset(binding(0), binary("add", binding(1), binding(2))))),
        binary("add", integer(1, "i32"),
               index(binding(0), binary("add", binding(1), binding(2))))),
    obs(binary("add", binding(0),
               unary("deref", offset(binding(1), binary("add", binding(2), binding(3))))),
        binary("add", binding(0),
               index(binding(1), binary("add", binding(2), binding(3)))),
        anchors=[anchor(1)]),
)
```

Expected: `E0 + *(A0.offset(B0+B1)) -> E0 + A0[B0+B1]`. Seed two's
`<id0>` is carried only by `E0`; it is not an anchor and occurs nowhere else.

### P6-SYN-11 `updated_named_struct_owner_and_field_identity`

```python
value = obs(
    named_struct(unary("deref", binding(0))),
    named_struct(binding(0)),
    anchors=[anchor(0, REF_I32)],
    root_types=(local_adt_type(), local_adt_type(),
                local_adt_type(), local_adt_type()),
)
doc(copy.deepcopy(value), copy.deepcopy(value))
```

Expected: one exact rule whose source and target are the same named-struct
tree except for `*A0 -> A0`. The `adt` is `S0`; the named field retains
`{"kind":"local","owner":S0,"id":F0}` on both sides. This updated,
producer-representable case deliberately tests both the local ADT and its
owned field.

### P6-SYN-12 `promoted_local_field_identity_and_owner`

```python
value = obs(
    field(unary("deref", binding(0))),
    field(binding(0)),
    anchors=[anchor(0, REF_I32)],
)
doc(copy.deepcopy(value), copy.deepcopy(value))
```

Expected: `(*A0).<S0::F0> -> A0.<S0::F0>`. Owner and field variables are
structurally nested, not detached conditions.

### P6-SYN-13 `rigid_external_function_is_preserved`

```python
doc(
    obs(call("load", offset(binding(0), integer(1, "isize"))),
        call("load", mutable_slice_from(binding(0), integer(1, "usize")))),
    obs(call("load", offset(binding(0), integer(2, "isize"))),
        call("load", mutable_slice_from(binding(0), integer(2, "usize")))),
)
```

Expected: the exact external identity `fixture::load` remains concrete and
the magnitude becomes shared `N0`.

### P6-SYN-14 `child_list_arity_is_hidden_by_enclosing_expression`

```python
doc(
    obs(offset(binding(0), call("f", binding(1))),
        mutable_slice_from(binding(0), call("f", binding(1)))),
    obs(offset(binding(0), call("f", binding(1), binding(2))),
        mutable_slice_from(binding(0), call("f", binding(1), binding(2)))),
)
```

Expected: the complete nested call is `E0`, producing
`A0.offset(E0) -> &mut A0[E0..]`. There is no sequence variable.

### P6-SYN-15 `operator_difference_is_hidden_by_expression`

```python
doc(
    obs(offset(binding(0), binary("add", binding(1), integer(1, "isize"))),
        mutable_slice_from(binding(0), binary("add", binding(1), integer(1, "isize")))),
    obs(offset(binding(0), binary("subtract", binding(1), integer(1, "isize"))),
        mutable_slice_from(binding(0), binary("subtract", binding(1), integer(1, "isize")))),
)
```

Expected: the differing binary expression is `E0`; operators never become
variables themselves.

### P6-SYN-16 `target_only_magnitude_disagreement_rejects`

```python
doc(
    obs(method(binding(0), "core", ("ptr", "const_ptr", "read")), integer(0, "i32")),
    obs(method(binding(0), "core", ("ptr", "const_ptr", "read")), integer(1, "i32")),
)
```

Expected: no rule because target `(0,1)` has no source `N` disagreement.

### P6-SYN-17 `different_source_and_target_disagreement_pairs_reject`

```python
doc(
    obs(offset(binding(0), integer(1, "isize")),
        mutable_slice_from(binding(0), integer(0, "usize"))),
    obs(offset(binding(0), integer(2, "isize")),
        mutable_slice_from(binding(0), integer(1, "usize"))),
)
```

Expected: source records `N` key `(1,2)`; target asks for `(0,1)` and rejects.

### P6-SYN-18 `identical_sources_with_conflicting_targets_reject`

```python
doc(
    obs(unary("deref", binding(0)), binding(0), anchors=[anchor(0, REF_I32)]),
    obs(unary("deref", binding(0)),
        {"kind":"address_of","borrow":"reference","mutability":"mut",
         "expression":unary("deref", binding(0))},
        anchors=[anchor(0, REF_I32)]),
)
```

Expected: target-only structural disagreement has no source `E` key; no rule.

### P6-SYN-19 `lone_expression_source_is_degenerate`

```python
doc(
    obs(call("left", unary("deref", binding(0))),
        call("left", unary("deref", binding(0)))),
    obs(call("right", offset(binding(0), integer(1, "isize"))),
        call("right", offset(binding(0), integer(1, "isize")))),
)
```

Expected: the conflicting rigid external callee identities cannot be hidden at
the bare path, so the mismatch bubbles to the strictly enclosing root call.
The tentative rule is `E0 -> E0` and is rejected because the complete source
is one `E`. Instrument the target traversal helper and assert it is never
called; this rejection occurs immediately after source synthesis.

### P6-SYN-20 `anchor_hidden_by_expression_rejects`

```python
doc(
    obs(call("consume", unary("deref", binding(0))),
        call("consume", unary("deref", binding(0)))),
    obs(call("consume", offset(binding(0), integer(1, "isize"))),
        call("consume", offset(binding(0), integer(1, "isize")))),
)
```

Expected tentative pattern `consume(E0) -> consume(E0)`, then rejection because
both seed substitutions hide their anchor identity inside `E0`.

### P6-SYN-21 `anchor_hidden_despite_explicit_occurrence_rejects`

```python
doc(
    obs(call("combine", unary("deref", binding(0)), call("read", binding(0))),
        call("combine", binding(0), call("read", binding(0))),
        anchors=[anchor(0, REF_I32)]),
    obs(call("combine", unary("deref", binding(0)), offset(binding(0), integer(1, "isize"))),
        call("combine", binding(0), offset(binding(0), integer(1, "isize"))),
        anchors=[anchor(0, REF_I32)]),
)
```

Expected: `A0` remains explicit in the first argument, but the same anchor is
also inside `E0` in the second argument; reject.

### P6-SYN-22 `one_of_multiple_anchors_hidden_rejects`

```python
doc(
    obs(call("combine", unary("deref", binding(0)), call("read", binding(1))),
        call("combine", binding(0), call("read", binding(1))),
        anchors=[anchor(0, REF_I32), anchor(1)]),
    obs(call("combine", unary("deref", binding(0)), offset(binding(1), integer(1, "isize"))),
        call("combine", binding(0), offset(binding(1), integer(1, "isize"))),
        anchors=[anchor(0, REF_I32), anchor(1)]),
)
```

Expected: `A1`, though not `A0`, is hidden by `E0`; reject.

### P6-SYN-23 `local_identity_split_between_explicit_and_expression_rejects`

```python
doc(
    obs(binary("add", binding(0),
               unary("deref", offset(binding(1), binding(0)))),
        binary("add", binding(0), index(binding(1), binding(0))),
        anchors=[anchor(1)]),
    obs(binary("add", binding(0),
               unary("deref", offset(binding(1), integer(1, "usize")))),
        binary("add", binding(0), index(binding(1), integer(1, "usize"))),
        anchors=[anchor(1)]),
)
```

Expected tentative explicit `B0` plus `E0` in the offset/index position.
Seed one's `<id0>` is carried by both, so reject.

### P6-SYN-24 `inconsistent_binding_equality_partition_rejects`

```python
doc(
    obs(call("combine", unary("deref", binding(0)), binding(1), binding(1)),
        call("combine", binding(0), binding(1), binding(1)),
        anchors=[anchor(0, REF_I32)]),
    obs(call("combine", unary("deref", binding(0)), binding(1), binding(2)),
        call("combine", binding(0), binding(1), binding(2)),
        anchors=[anchor(0, REF_I32)]),
)
```

Expected tentative `B0` and `B1`; seed one's `<id1>` has two identity-variable
carriers and rejects.

### P6-SYN-25 `target_only_local_field_identity_rejects`

```python
value = obs(
    unary("deref", binding(0)),
    field(binding(0)),
    anchors=[anchor(0, REF_I32)],
)
doc(copy.deepcopy(value), copy.deepcopy(value))
```

Expected: the document is valid under the authoritative observation loader,
but the target's equal local owner/field identities have no context/source
variables. Lookup-only target synthesis rejects; it does not allocate `S0` or
`F0`.

### P6-SYN-26 `different_rigid_external_functions_reject`

```python
doc(
    obs(call("load", unary("deref", binding(0))), call("load", binding(0)),
        anchors=[anchor(0, REF_I32)]),
    obs(call("peek", unary("deref", binding(0))), call("peek", binding(0)),
        anchors=[anchor(0, REF_I32)]),
)
```

Expected: the conflicting rigid callee identities bubble to the enclosing root
call, making the entire source `E0`; degenerate-source rejection yields no
rule.

## 7. Additional grammar and algorithm coverage

### P6-ALG-01 `remaining_local_identity_sorts_are_emitted`

Use this exact repeated observation:

```python
enum_expr = {
    "kind": "struct",
    "adt": local_adt("enum", 0),
    "variant": local_member("variant", "enum", 0, 0),
    "fields": [{
        "field": local_member("field", "enum", 0, 0),
        "value": call_expr(
            local_value("function", 0),
            unary("deref", binding(0)),
            local_value("constant", 0),
            local_value("static", 0),
            local_method(binding(1), 0),
        ),
    }],
    "rest": None,
}
target_expr = copy.deepcopy(enum_expr)
target_expr["fields"][0]["value"]["arguments"][1] = binding(0)
value = obs(enum_expr, target_expr, anchors=[anchor(0, REF_I32)])
doc(copy.deepcopy(value), copy.deepcopy(value))
```

Expected: exact variables of sorts `anchor`, `binding`, `function`, `enum`,
`field`, `variant`, `constant`, `static`, and `method`. The field and variant
each retain the same explicit `enum` owner. Add an exact union subcase:

```python
union_expr = {
    "kind":"struct", "adt":local_adt("union", 0), "variant":None,
    "fields":[{"field":local_member("field", "union", 0, 0),
               "value":unary("deref", binding(0))}], "rest":None,
}
union_target = copy.deepcopy(union_expr)
union_target["fields"][0]["value"] = binding(0)
u = obs(union_expr, union_target, anchors=[anchor(0, REF_I32)])
doc(copy.deepcopy(u), copy.deepcopy(u))
```

Expected: `union` and owned `field` variables. Together with the canonical
cases, every declared variable sort is exercised.

### P6-ALG-02 `local_nominal_context_alignment_is_one_environment`

```python
left_ty = local_adt_type("struct", 0)
right_ty = local_adt_type("struct", 0)
doc(
    obs(unary("deref", binding(0)), binding(0), anchors=[anchor(0, REF_I32)],
        root_types=(left_ty, left_ty, left_ty, left_ty)),
    obs(unary("deref", binding(0)), binding(0), anchors=[anchor(0, REF_I32)],
        root_types=(right_ty, right_ty, right_ty, right_ty)),
)
```

Expected: one `struct` variable reused in all four root trees. Equal alpha
strings in separate observations are still observation-local and become a
variable rather than a rigid constant. P6-SYN-11 is the exact case proving
that a source field owner reuses the context-established variable.

### P6-ALG-03 `anchor_count_order_and_type_are_context`

Exact three independent pairs:

```python
one_vs_two = doc(
    obs(call("f", binding(0)), call("f", binding(0)), anchors=[anchor(0)]),
    obs(call("f", binding(0), binding(1)), call("f", binding(0), binding(1)),
        anchors=[anchor(0), anchor(1)]),
)
reordered_types = doc(
    obs(call("f", binding(0), binding(1)), call("f", binding(0), binding(1)),
        anchors=[anchor(0, REF_I32), anchor(1, MUT_SLICE_I32)]),
    obs(call("f", binding(0), binding(1)), call("f", binding(0), binding(1)),
        anchors=[anchor(0, MUT_SLICE_I32), anchor(1, REF_I32)]),
)
mutability_mismatch = doc(
    obs(unary("deref", binding(0)), binding(0), anchors=[anchor(0, REF_I32)]),
    obs(unary("deref", binding(0)), binding(0), anchors=[anchor(0, MUT_SLICE_I32)]),
)
```

Expected: all three produce no rule at context comparison. Anchor lists are
positional and types are structural; the synthesizer never reorders anchors.

### P6-ALG-04 `root_type_constructor_external_and_arity_mismatch_skip`

Exact inputs:

```python
left = obs(
    unary("deref", binding(0)), binding(0), anchors=[anchor(0, REF_I32)],
    root_types=(local_adt_type("struct", 0),) * 4,
)
primitive = copy.deepcopy(left)
primitive["source_type"] = I32
external_type = copy.deepcopy(left)
external_type["source_type"] = {
    "kind":"adt", "adt_kind":"struct",
    "identity":external("fixture", "External"), "arguments":[],
}
arity = copy.deepcopy(left)
arity["source_type"] = local_adt_type("struct", 0)
arity["source_type"]["arguments"] = [I32]
primitive_mismatch = doc(copy.deepcopy(left), primitive)
external_mismatch = doc(copy.deepcopy(left), external_type)
arity_mismatch = doc(copy.deepcopy(left), arity)
```

Expected: each context mismatch skips without `E`, because types are not
expression-generalization positions.

### P6-ALG-05 `namespace_bijection_conflict_in_context_skips`

```python
pair_ty_left = {
    "kind":"tuple", "elements":[local_adt_type("struct", 0),
                                  local_adt_type("struct", 0)]
}
pair_ty_right = {
    "kind":"tuple", "elements":[local_adt_type("struct", 0),
                                  local_adt_type("struct", 1)]
}
doc(
    obs(unary("deref", binding(0)), binding(0), anchors=[anchor(0, REF_I32)],
        root_types=(pair_ty_left, pair_ty_left, pair_ty_left, pair_ty_left)),
    obs(unary("deref", binding(0)), binding(0), anchors=[anchor(0, REF_I32)],
        root_types=(pair_ty_right, pair_ty_right, pair_ty_right, pair_ty_right)),
)
```

Expected: context bijection conflict, no rule. Exact reverse input:

```python
reverse_left = {
    "kind":"tuple", "elements":[local_adt_type("struct", 0),
                                  local_adt_type("struct", 1)]
}
reverse_right = {
    "kind":"tuple", "elements":[local_adt_type("struct", 0),
                                   local_adt_type("struct", 0)]
}
reverse = doc(
    obs(unary("deref", binding(0)), binding(0), anchors=[anchor(0, REF_I32)],
        root_types=(reverse_left,) * 4),
    obs(unary("deref", binding(0)), binding(0), anchors=[anchor(0, REF_I32)],
        root_types=(reverse_right,) * 4),
)
```

It also skips.

### P6-ALG-06 `integer_literal_type_controls_narrow_magnitude_rule`

```python
same_value = doc(
    obs(offset(binding(0), integer(1, "isize")), binding(0)),
    obs(offset(binding(0), integer(1, "isize")), binding(0)),
)
different_type = doc(
    obs(offset(binding(0), integer(1, "isize")), binding(0)),
    obs(offset(binding(0), integer(2, "usize")), binding(0)),
)
```

Expected: `same_value` is concrete and allocates no `N`. `different_type`
uses one `E` for the complete differing literal expression; it does not create
`N` across seed literal types. The exact targets are `binding(0)` in both, so
the source-only `E` is accepted.

### P6-ALG-07 `magnitude_variables_require_canonical_ascii_decimal`

All exact documents below are accepted by the unchanged observation loader:

```python
equal_leading_zero = doc(
    obs(offset(binding(0), integer_text("01", "isize")), binding(0)),
    obs(offset(binding(0), integer_text("01", "isize")), binding(0)),
)
unequal_leading_zero = doc(
    obs(offset(binding(0), integer_text("01", "isize")), binding(0)),
    obs(offset(binding(0), integer_text("02", "isize")), binding(0)),
)
mixed_canonical = doc(
    obs(offset(binding(0), integer_text("1", "isize")), binding(0)),
    obs(offset(binding(0), integer_text("02", "isize")), binding(0)),
)
equal_unicode = doc(
    obs(offset(binding(0), integer_text("١", "isize")), binding(0)),
    obs(offset(binding(0), integer_text("١", "isize")), binding(0)),
)
unequal_unicode = doc(
    obs(offset(binding(0), integer_text("١", "isize")), binding(0)),
    obs(offset(binding(0), integer_text("٢", "isize")), binding(0)),
)
canonical_zero_one = doc(
    obs(offset(binding(0), integer_text("0", "isize")), binding(0)),
    obs(offset(binding(0), integer_text("1", "isize")), binding(0)),
)
canonical_nine_ten = doc(
    obs(offset(binding(0), integer_text("9", "isize")), binding(0)),
    obs(offset(binding(0), integer_text("10", "isize")), binding(0)),
)
```

Expected: equal `"01"` and equal Arabic-Indic `"١"` remain concrete literal
values. Unequal `("01","02")`, mixed `("1","02")`, and unequal
Arabic-Indic `("١","٢")` become ordinary source-only `E` disagreements at
the complete literal expression; none allocates `N`. Exact canonical pairs
`("0","1")` and `("9","10")` do allocate `N`. The tests assert the
observation loader itself accepts every value above, preserving its current
`str.isdigit()` behavior.

### P6-ALG-08 `rigid_identity_conflict_can_hide_in_larger_nonroot_expression`

```python
doc(
    obs(call("outer", call("load", binding(0)), unary("deref", binding(0))),
        call("outer", call("load", binding(0)), binding(0)),
        anchors=[anchor(0, REF_I32)]),
    obs(call("outer", call("peek", binding(0)), unary("deref", binding(0))),
        call("outer", call("peek", binding(0)), binding(0)),
        anchors=[anchor(0, REF_I32)]),
)
```

Expected: the differing callee path cannot itself become `E`; the nested inner
call becomes `E0`. The outer call remains concrete, so the source is not a lone
`E`. Source and target contain the same ordered inner-call disagreement, so
target lookup reuses `E0`; carrier analysis then rejects because the inner-call
substitutions hide the anchor. The exact no-hidden-identity subcase is:

```python
doc(
    obs(call("outer", call("load", integer(0, "i32")),
                      unary("deref", binding(0))),
        call("outer", call("load", integer(0, "i32")), binding(0)),
        anchors=[anchor(0, REF_I32)]),
    obs(call("outer", call("peek", integer(1, "i32")),
                      unary("deref", binding(0))),
        call("outer", call("peek", integer(1, "i32")), binding(0)),
        anchors=[anchor(0, REF_I32)]),
)
```

It is accepted as `outer(E0,*A0) -> outer(E0,A0)`, proving that a larger
expression may hide rigid identities.

### P6-ALG-09 `source_variable_may_be_unused_by_target`

```python
doc(
    obs(call("select", unary("deref", binding(0)), integer(1, "i32")), binding(0),
        anchors=[anchor(0, REF_I32)]),
    obs(call("select", unary("deref", binding(0)), integer(2, "i32")), binding(0),
        anchors=[anchor(0, REF_I32)]),
)
```

Expected: one rule with source-only `N0`; target contains only `A0`.

### P6-ALG-10 `target_lookup_does_not_widen_or_fallback`

```python
doc(
    obs(offset(binding(0), binary("add", binding(1), integer(1, "isize"))),
        mutable_slice_from(binding(0), integer(1, "isize"))),
    obs(offset(binding(0), binary("multiply", binding(1), integer(2, "isize"))),
        mutable_slice_from(binding(0), integer(2, "isize"))),
)
```

Expected: source records `E` for the binary subtree. Target sees a narrow
integer disagreement and looks for `N`, not the source `E`; it rejects instead
of widening the target or reusing a different-sort key.

### P6-ALG-11 `member_owner_and_member_have_independent_carriers`

Exact accepted input:

```python
value = obs(
    call("pair", field(unary("deref", binding(0)), field_index=0),
                 field(unary("deref", binding(0)), field_index=1)),
    call("pair", field(binding(0), field_index=0),
                 field(binding(0), field_index=1)),
    anchors=[anchor(0, REF_I32)],
)
doc(copy.deepcopy(value), copy.deepcopy(value))
```

Expected: one `S0` owner and distinct `F0`, `F1` variables; repeated owner
occurrences are one carrier. Exact rejection input:

```python
left = value
right = obs(
    call("pair", field(unary("deref", binding(0)), field_index=0),
                 field(unary("deref", binding(0)), field_index=0)),
    call("pair", field(binding(0), field_index=0),
                 field(binding(0), field_index=0)),
    anchors=[anchor(0, REF_I32)],
)
doc(left, right)
```

Seed two's field identity is paired with two distinct seed-one fields, so its
identity partition has two carriers and the pair rejects.

### P6-ALG-12 `distinct_expression_variables_may_have_equal_substitutions`

```python
doc(
    obs(call("triple", call("left", integer(0, "i32")),
                       call("left", integer(0, "i32")),
                       unary("deref", binding(0))),
        binding(0), anchors=[anchor(0, REF_I32)]),
    obs(call("triple", call("right", integer(1, "i32")),
                       call("other", integer(2, "i32")),
                       unary("deref", binding(0))),
        binding(0), anchors=[anchor(0, REF_I32)]),
)
```

Expected: two different ordered subtree pairs allocate `E0` and `E1`. A
test-only reconstruction substitution may assign equal concrete values to
different `E` variables without violating injectivity. Exact `N` subcase:

```python
doc(
    obs(call("triple", unary("deref", binding(0)),
                       integer(1, "usize"), integer(1, "usize")),
        binding(0), anchors=[anchor(0, REF_I32)]),
    obs(call("triple", unary("deref", binding(0)),
                       integer(2, "usize"), integer(3, "usize")),
        binding(0), anchors=[anchor(0, REF_I32)]),
)
```

It allocates distinct `N0=(1,2)` and `N1=(1,3)`; seed one's substitutions are
equal, which is allowed. Identity variables remain injective.

### P6-ALG-13 `conflicting_and_specific_rules_are_all_retained`

Exact input combines five observations:

```python
a = obs(offset(binding(0), integer(1, "isize")),
        mutable_slice_from(binding(0), integer(1, "usize")))
b = obs(offset(binding(0), integer(2, "isize")),
        mutable_slice_from(binding(0), integer(2, "usize")))
c = obs(offset(binding(0), integer(1, "isize")),
        call("alternate", binding(0), integer(1, "usize")))
d = obs(offset(binding(0), integer(2, "isize")),
        call("alternate", binding(0), integer(2, "usize")))
doc(copy.deepcopy(a), copy.deepcopy(b), copy.deepcopy(a),
    copy.deepcopy(c), copy.deepcopy(d))
```

Expected: retain the generalized rule from `(a,b)`, the exact rule from the
repeated `a`, and the conflicting generalized rule from `(c,d)`. Exact
canonical deduplication removes only byte-identical semantic
cores; there is no subsumption or conflict resolution.

### P6-ALG-14 `every_closed_constructor_traverses_all_children`

Use `S = unary("deref", binding(0))` and `T = binding(0)`. For each tuple
below, create `value = obs(source, target, anchors=[anchor(0, REF_I32)])` and
input `doc(copy.deepcopy(value), copy.deepcopy(value))`. These are exact
smallest trees for expression constructors not already isolated above:

```python
S = unary("deref", binding(0))
T = binding(0)
stmt_s = {"kind":"expression", "expression":S, "semicolon":False}
stmt_t = {"kind":"expression", "expression":T, "semicolon":False}
block_s = {"statements":[stmt_s]}
block_t = {"statements":[stmt_t]}

constructor_pairs = [
    ({"kind":"array", "elements":[S]},
     {"kind":"array", "elements":[T]}),
    ({"kind":"tuple", "elements":[integer(0, "i32"), S]},
     {"kind":"tuple", "elements":[integer(0, "i32"), T]}),
    ({"kind":"cast", "expression":S, "type":I32},
     {"kind":"cast", "expression":T, "type":I32}),
    ({"kind":"if", "condition":boolean(True), "then":block_s, "else":S},
     {"kind":"if", "condition":boolean(True), "then":block_t, "else":T}),
    ({"kind":"while", "condition":boolean(True), "body":block_s},
     {"kind":"while", "condition":boolean(True), "body":block_t}),
    ({"kind":"loop", "body":block_s},
     {"kind":"loop", "body":block_t}),
    ({"kind":"assign", "left":integer(0, "i32"), "right":S},
     {"kind":"assign", "left":integer(0, "i32"), "right":T}),
    ({"kind":"assign_op", "operator":"add", "left":integer(0, "i32"), "right":S},
     {"kind":"assign_op", "operator":"add", "left":integer(0, "i32"), "right":T}),
    ({"kind":"range", "start":integer(0, "usize"), "end":S, "limits":"closed"},
     {"kind":"range", "start":integer(0, "usize"), "end":T, "limits":"closed"}),
    ({"kind":"address_of", "borrow":"raw", "mutability":"const", "expression":S},
     {"kind":"address_of", "borrow":"raw", "mutability":"const", "expression":T}),
    ({"kind":"break", "value":S},
     {"kind":"break", "value":T}),
    ({"kind":"return", "value":S},
     {"kind":"return", "value":T}),
    ({"kind":"repeat", "value":S, "count":integer(2, "usize")},
     {"kind":"repeat", "value":T, "count":integer(2, "usize")}),
    ({"kind":"block", "block":block_s},
     {"kind":"block", "block":block_t}),
    ({"kind":"array", "elements":[{"kind":"continue"}, S]},
     {"kind":"array", "elements":[{"kind":"continue"}, T]}),
]
```

Add the exact binding-pattern block subcase:

```python
source_block = {
    "kind":"block",
    "block":{"statements":[
        {"kind":"let",
         "pattern":{"kind":"binding","id":"<id0>",
                    "mutability":"mutable","by_ref":"no"},
         "type":I32, "initializer":unary("deref", binding(1))},
        {"kind":"expression","expression":binding(0),"semicolon":False},
    ]},
}
target_block = copy.deepcopy(source_block)
target_block["block"]["statements"][0]["initializer"] = binding(1)
value = obs(source_block, target_block, anchors=[anchor(1, REF_I32)])
doc(copy.deepcopy(value), copy.deepcopy(value))
```

Expected: every pair emits one exact structural rule; all fixed flags, optional
positions, statement order, pattern mutability/by-ref, and explicit types are
retained. The binding pattern ID and its later path share one `binding`
variable. Operator disagreement and list-arity generalization use the exact
P6-SYN-15 and P6-SYN-14 inputs; fixed context mismatches use P6-ALG-03 and
P6-ALG-04. No implicit mutation subcase is required here.

Exercise every type constructor as root context with this exact list, using
the repeated `*A0 -> A0` observation from P6-SYN-07 and placing the selected
tree in all four root-type slots:

```python
type_samples = [
    I32,
    {"kind":"slice", "element":I32},
    {"kind":"array", "element":I32, "length":4},
    RAW_I32,
    REF_I32,
    {"kind":"tuple", "elements":[I32, REF_I32]},
    local_adt_type("struct", 0),
    {"kind":"adt", "adt_kind":"enum",
     "identity":external("core", "option", "Option"),
     "arguments":[REF_I32]},
]
```

Expected: exact structural retention, with the local ADT becoming `S0`.
The exact mismatch inputs in P6-ALG-03 through P6-ALG-05 cover constructor,
arity, primitive, mutability, and identity conflicts. For array length, use
`left = type_samples[2]`, `right = copy.deepcopy(left)`, set
`right["length"] = 5`, and place each in all four root slots of the repeated
`*A0 -> A0` pair; it emits no rule.

### P6-ALG-15 `all_loader_reachable_target_only_namespaces_reject`

For every exact target below, create a repeated observation with source
`unary("deref", binding(0))`, `anchors=[anchor(0, REF_I32)]`, and input
`doc(copy.deepcopy(value), copy.deepcopy(value))`:

```python
target_only_values = [
    field(binding(0)),
    {"kind":"cast", "expression":binding(0),
     "type":local_adt_type("struct", 0)},
    {"kind":"cast", "expression":binding(0),
     "type":local_adt_type("union", 0)},
    call_expr(constructor_path("enum", 0, 0), binding(0)),
    call("pair", binding(0), local_value("constant", 0)),
    call("pair", binding(0), local_value("static", 0)),
    call("pair", binding(0), local_method(binding(0), 0)),
]
documents = []
for target in target_only_values:
    value = obs(unary("deref", binding(0)), target,
                anchors=[anchor(0, REF_I32)])
    documents.append(doc(copy.deepcopy(value), copy.deepcopy(value)))
```

Expected: the unchanged observation loader accepts every document. Target
lookup rejects each because the source/context established none of the local
`struct`, `union`, `enum`, `variant`, `field`, `constant`, `static`, or
`method` identities. No target traversal allocates a variable. Target-only
binding and local-function IDs remain earlier loader errors and are not
synthesis inputs.

### P6-ALG-16 `constructor_value_identity_preserves_owned_variant`

```python
source = call_expr(constructor_path("enum", 0, 0),
                   unary("deref", binding(0)))
target = call_expr(constructor_path("enum", 0, 0), binding(0))
value = obs(source, target, anchors=[anchor(0, REF_I32)])
doc(copy.deepcopy(value), copy.deepcopy(value))
```

Expected: the `ValueIdentity::Constructor` shape remains concrete while its
local ADT becomes the first `enum` variable and its owned variant becomes the
first `variant` variable; on the wire these are exactly
`var("enum",0)` and `var("variant",0)`. The variant retains
`owner=var("enum",0)` in source and target.

Exact null-variant struct-constructor subcase:

```python
struct_constructor = {
    "kind":"path",
    "value":{"kind":"constructor", "adt":local_adt("struct", 0),
             "variant":None},
}
value = obs(call_expr(struct_constructor, unary("deref", binding(0))),
            call_expr(struct_constructor, binding(0)),
            anchors=[anchor(0, REF_I32)])
doc(copy.deepcopy(value), copy.deepcopy(value))
```

Expected: one `struct` variable and no variant variable.

### P6-ALG-17 `foreign_identities_and_noninteger_literals_remain_rigid`

Exact source and target:

```python
fixed_literals = [
    literal({"kind":"bool", "value":True}),
    literal({"kind":"char", "value":"x"}),
    literal({"kind":"byte", "value":255}),
    literal({"kind":"string", "value":"text"}),
    literal({"kind":"byte_string", "value":[0, 255]}),
    literal({"kind":"c_string", "value":[65, 0]}),
    literal({"kind":"float", "bits":"3f800000", "type":"f32"}),
]
external_field = {
    "kind":"field", "base":integer(0, "i32"),
    "field":external("fixture", "External", "field"),
}
external_constructor = {
    "kind":"path",
    "value":{
        "kind":"constructor",
        "adt":external("fixture", "ExternalEnum"),
        "variant":external("fixture", "ExternalEnum", "Variant"),
    },
}
source = call_expr(
    foreign_path("foreign_function", "ffi_read"),
    foreign_path("foreign_static", "ERRNO"),
    external_field,
    external_constructor,
    *fixed_literals,
    unary("deref", binding(0)),
)
target = call_expr(
    foreign_path("foreign_function", "ffi_read"),
    foreign_path("foreign_static", "ERRNO"),
    copy.deepcopy(external_field),
    copy.deepcopy(external_constructor),
    *copy.deepcopy(fixed_literals),
    binding(0),
)
value = obs(source, target, anchors=[anchor(0, REF_I32)])
doc(copy.deepcopy(value), copy.deepcopy(value))
```

Expected: one exact rule changes only `*A0` to `A0`. The foreign symbols,
external field/ADT/variant identities, and every noninteger literal kind/value
remain rigid concrete nodes and allocate no variables. Exact conflict
subcases:

```python
foreign_conflict = doc(
    obs(call_expr(foreign_path("foreign_function", "ffi_read"),
                  unary("deref", binding(0))),
        call_expr(foreign_path("foreign_function", "ffi_read"), binding(0)),
        anchors=[anchor(0, REF_I32)]),
    obs(call_expr(foreign_path("foreign_function", "ffi_peek"),
                  unary("deref", binding(0))),
        call_expr(foreign_path("foreign_function", "ffi_peek"), binding(0)),
        anchors=[anchor(0, REF_I32)]),
)
literal_conflict = doc(
    obs(call("pair", literal({"kind":"char", "value":"x"}),
                     unary("deref", binding(0))),
        call("pair", literal({"kind":"char", "value":"x"}), binding(0)),
        anchors=[anchor(0, REF_I32)]),
    obs(call("pair", literal({"kind":"char", "value":"y"}),
                     unary("deref", binding(0))),
        call("pair", literal({"kind":"char", "value":"y"}), binding(0)),
        anchors=[anchor(0, REF_I32)]),
)
```

`foreign_conflict` rejects after its rigid callee conflict bubbles to the root
call and creates a lone source `E`. `literal_conflict` accepts one `E0` at the
char literal position and reuses it in the target; it never allocates `N`.

### P6-ALG-18 `canonical_first_occurrence_order_is_context_then_patterns`

Exact repeated observation:

```python
root = local_adt_type("struct", 1)
source = call(
    "ordered",
    unary("deref", binding(0)),
    unary("deref", binding(1)),
    binding(2),
    field(binding(2), owner_kind="struct", owner_index=0, field_index=0),
)
target = call(
    "ordered",
    binding(0), binding(1), binding(2),
    field(binding(2), owner_kind="struct", owner_index=0, field_index=0),
)
value = obs(
    source, target,
    anchors=[anchor(0, REF_I32), anchor(1, REF_I32)],
    root_types=(root, root, root, root),
)
doc(copy.deepcopy(value), copy.deepcopy(value))
```

Expected canonical traversal assigns `A0` and `A1` from pointer-anchor list
order, then `S0` from root context, then `B0` from `<id2>`, then `S1` and `F0`
from the source field. Target traversal reuses all six identity variables and
allocates none. Add this exact magnitude-order subcase:

```python
doc(
    obs(call("ordered", unary("deref", binding(0)),
                        integer(1, "usize"), integer(2, "usize")),
        binding(0), anchors=[anchor(0, REF_I32)]),
    obs(call("ordered", unary("deref", binding(0)),
                        integer(3, "usize"), integer(4, "usize")),
        binding(0), anchors=[anchor(0, REF_I32)]),
)
```

Expected: source traversal assigns `N0` to `(1,3)` and `N1` to `(2,4)`.
Indices are independent across sorts, and output rule member order never
changes first-occurrence semantics.

## 8. Enumeration, determinism, and command boundary

### P6-IO-01 `duplicate_compression_crosses_documents`

Exact logical inputs are three separately loaded files:

```python
value = obs(unary("deref", binding(0)), binding(0),
            anchors=[anchor(0, REF_I32)])
first = doc(copy.deepcopy(value))
second = doc(copy.deepcopy(value))
third = doc(copy.deepcopy(value))
```

Expected: the core sees one unique value marked repeated and enumerates one
self-pair, not three occurrence pairs. The output contains one exact rule and
no support count.

### P6-IO-02 `singleton_never_self_pairs_and_empty_is_success`

Exact inputs: `doc(value)` using P6-IO-01's value, and independently `doc()`.
Each succeeds with the exact empty document bytes from P6-MODEL-07.

### P6-IO-03 `input_document_and_observation_permutations_are_byte_identical`

Use `a`, `b`, `c`, and `d` from P6-ALG-13. Compare complete output bytes for:

```python
[doc(a, b), doc(c, d, a)]
[doc(a, c), doc(d, b, a)]
[doc(a), doc(a), doc(c, b, d)]
```

Expected: identical bytes. This checks semantic sorting before ordered pair
orientation as well as final canonical sorting.

### P6-IO-04 `canonical_dedup_ignores_precanonical_variable_indices`

Take the complete accepted semantic rule from P6-SYN-05 before
canonicalization. Make two exact internal candidates by replacing every `A0`
with `var("anchor", 4)` in the first and `var("anchor", 9)` in the second;
replace `(B0,B1)` with `(var("binding",8),var("binding",3))` in the first and
`(var("binding",1),var("binding",7))` in the second, preserving every
occurrence relationship and the target's reversal. Expected: context-first
first-occurrence renaming turns both into the same canonical `A0,B0,B1` rule,
and exact deduplication retains one. This is an internal canonicalizer test;
`load_rules` correctly rejects either precanonical candidate because its
indices are not contiguous.

### P6-IO-05 `command_arguments_and_exact_success_file`

Create `a.json` containing two copies of P6-IO-01's observation. Invoke:

```python
main(["--output", str(tmp_path / "rules.json"), str(tmp_path / "a.json")])
```

Expected: return `0`, empty stdout/stderr, and `rules.json` equals
`rules_to_json(synthesize_rules((load_observations(a_text),)))` byte for byte.
No other file remains in the directory.

### P6-IO-06 `command_rejects_input_shape_and_path_aliases_before_write`

Run independent exact invocations with no observation path, a missing path, a
directory, a symlink input, the same resolved input twice, `a.json` plus
`./a.json`, and `--output a.json a.json`. Each returns exactly `1` and leaves the
input unchanged. Also write exact contents `{}\n`, two concatenated valid
documents, JSONL, invalid UTF-8, schema version 2, and a valid document with an
unknown nested key; each is fatal and leaves no output. Every invocation
returns exactly `1`, prints nothing to stdout, and prints exactly one
newline-terminated `extract_rules: <message>` line to stderr, with no usage,
traceback, or second line.

### P6-IO-07 `publication_is_atomic_and_preserves_old_output_on_failure`

Pre-create `rules.json` with exact bytes `b"old\n"`. Inject failures during
temporary creation, write, flush/close, and `os.replace`. Before replace, every
failure leaves `rules.json` exactly `b"old\n"` and removes the owned sibling
temporary. A cleanup failure is included with the primary error. On success,
an old regular file or symlink at the output path is replaced, and no temporary
remains. An existing output directory or other nonregular node is rejected
without mutation.

### P6-IO-08 `pair_nonresults_do_not_mask_fatal_command_errors`

Run the exact P6-SYN-16, P6-SYN-20, and P6-SYN-26 documents as three independent
valid invocations. Each pair-local incompatibility is skipped, the command
returns `0`, both streams are empty, and the output is exactly P6-MODEL-07's
empty bytes. Then use P6-SYN-07's valid document as the first input and exact
`{}\n` as a second input: the whole invocation returns `1`, emits one stderr
line, and publishes nothing. Inject a serializer or resource failure after
synthesis: it has the same fatal stream/return contract and preserves the old
output.

### P6-IO-09 `synthesis_never_mutates_nested_inputs`

Use one accepted document (P6-SYN-03), one carrier-rejected document
(P6-SYN-23), and the mixed five-observation document P6-ALG-13. For each:

```python
loaded = load_observations(json.dumps(input_value))
before = copy.deepcopy(loaded.observations)
first = synthesize_rules((loaded,))
assert loaded.observations == before
second = synthesize_rules((loaded,))
assert loaded.observations == before
assert rules_to_json(first) == rules_to_json(second)
```

Expected: every nested dict, list, identity, type, and expression remains
value-for-value unchanged after accepted and rejected pair processing. The
test also checks that duplicate compression and semantic freezing store no
mutable aliases back into the inputs.

### P6-IO-10 `recursive_json_member_order_is_semantically_irrelevant`

Exact permutation helper and input:

```python
def reverse_members(value):
    if isinstance(value, dict):
        return {
            key: reverse_members(value[key])
            for key in reversed(tuple(value.keys()))
        }
    if isinstance(value, list):
        return [reverse_members(item) for item in value]
    return value

original = doc(copy.deepcopy(a), copy.deepcopy(b), copy.deepcopy(a))
permuted = reverse_members(original)
left = load_observations(json.dumps(original, ensure_ascii=False))
right = load_observations(json.dumps(permuted, ensure_ascii=False))
```

Here `a` and `b` are the exact P6-ALG-13 values. Expected: both loaders
succeed, `synthesize_rules((left,))` and `synthesize_rules((right,))` have
identical semantic rules, and `rules_to_json` bytes are identical. This
recursively reverses document, observation, expression, identity, anchor, and
type object member order while preserving every meaningful array order.

### P6-IO-11 `help_is_successful_and_side_effect_free`

Exact invocation:

```python
result = main(["--help"])
```

Expected: `result == 0`; stderr is empty; stdout starts with
`"usage: extract_rules.py"`, documents `--output` and one-or-more observation
paths, and ends with one newline. Run with an empty temporary directory and
assert its entries remain exactly empty. Help does not resolve paths, read
inputs, synthesize, or create a temporary/output file.
