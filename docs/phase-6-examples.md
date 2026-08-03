# Rule-Synthesis Examples

## 1. Integer-magnitude correspondence

```text
source 1: <anchor0>.offset(1_isize)
target 1: &mut <anchor0>[1_usize..]

source 2: <anchor0>.offset(2_isize)
target 2: &mut <anchor0>[2_usize..]

rule: [A0].offset([N0]_isize) => &mut [A0][[N0]_usize..]
```

## 2. Complete-expression correspondence

```text
source 1: <anchor0>.offset(<var0> + 1)
target 1: &mut <anchor0>[(<var0> + 1)..]

source 2: <anchor0>.offset(<var0> * 2)
target 2: &mut <anchor0>[(<var0> * 2)..]

rule: [A0].offset([E0]) => &mut [A0][[E0]..]
```

## 3. Repeated complete-expression correspondence

```text
source 1: pair(<anchor0>.offset(<var0> + 1), <anchor0>.offset(<var0> + 1))
target 1: pair(&mut <anchor0>[(<var0> + 1)..], &mut <anchor0>[(<var0> + 1)..])

source 2: pair(<anchor0>.offset(<var0> * 2), <anchor0>.offset(<var0> * 2))
target 2: pair(&mut <anchor0>[(<var0> * 2)..], &mut <anchor0>[(<var0> * 2)..])

rule: pair([A0].offset([E0]), [A0].offset([E0]))
      => pair(&mut [A0][[E0]..], &mut [A0][[E0]..])
```

## 4. Source-only magnitude metavariable

```text
source 1: <anchor0>.offset(1_isize).is_null()
target 1: false

source 2: <anchor0>.offset(2_isize).is_null()
target 2: false

rule: [A0].offset([N0]_isize).is_null() => false
```

## 5. Reordered binding identities

```text
source 1: mix(*<anchor0>, <var0>, <var1>)
target 1: mix(<anchor0>, <var1>, <var0>)

source 2: mix(*<anchor0>, <var0>, <var1>)
target 2: mix(<anchor0>, <var1>, <var0>)

rule: mix(*[A0], [B0], [B1]) => mix([A0], [B1], [B0])
```

## 6. Ordered disagreement pairs

```text
source 1: triple(*<anchor0>, 1_usize, 2_usize)
target 1: triple(<anchor0>, 1_usize, 2_usize)

source 2: triple(*<anchor0>, 2_usize, 1_usize)
target 2: triple(<anchor0>, 2_usize, 1_usize)

rule: triple(*[A0], [N0]_usize, [N1]_usize)
      => triple([A0], [N0]_usize, [N1]_usize)
```

## 7. Exact rule from repeated equal observations

```text
source 1: *<anchor0>
target 1: <anchor0>

source 2: *<anchor0>
target 2: <anchor0>

rule: *[A0] => [A0]
```

## 8. Expression-equal source and target

```text
source 1: *<anchor0>
target 1: *<anchor0>

source 2: *<anchor0>
target 2: *<anchor0>

rule: *[A0] => *[A0]
```

## 9. Ordinary binding identities

```text
source 1: <anchor0>.offset(<var0> + <var1>)
target 1: &mut <anchor0>[(<var0> + <var1>)..]

source 2: <anchor0>.offset(<var0> + <var1>)
target 2: &mut <anchor0>[(<var0> + <var1>)..]

rule: [A0].offset([B0] + [B1]) => &mut [A0][([B0] + [B1])..]
```

## 10. Identity encapsulated by an expression metavariable

```text
source 1: 1 + *(<anchor0>.offset(<var0> + <var1>))
target 1: 1 + <anchor0>[<var0> + <var1>]

source 2: <var0> + *(<anchor0>.offset(<var1> + <var2>))
target 2: <var0> + <anchor0>[<var1> + <var2>]

rule: [E0] + *([A0].offset([B0] + [B1]))
      => [E0] + [A0][[B0] + [B1]]
```

## 11. Local struct identity

```text
source 1: ffi::read_as::<struct0>(*<anchor0>)
target 1: ffi::read_as::<struct0>(<anchor0>)

source 2: ffi::read_as::<struct0>(*<anchor0>)
target 2: ffi::read_as::<struct0>(<anchor0>)

rule: ffi::read_as::<[S0]>(*[A0]) => ffi::read_as::<[S0]>([A0])
```

## 12. Local field identity and owner

```text
source 1: (*<anchor0>).<struct0::field0>
target 1: <anchor0>.<struct0::field0>

source 2: (*<anchor0>).<struct0::field0>
target 2: <anchor0>.<struct0::field0>

rule: (*[A0]).<[S0]::[F0]> => [A0].<[S0]::[F0]>
```

## 13. Rigid external function

```text
source 1: ffi::load(<anchor0>.offset(1_isize))
target 1: ffi::load(&mut <anchor0>[1_usize..])

source 2: ffi::load(<anchor0>.offset(2_isize))
target 2: ffi::load(&mut <anchor0>[2_usize..])

rule: ffi::load([A0].offset([N0]_isize))
      => ffi::load(&mut [A0][[N0]_usize..])
```

## 14. Child-list arity hidden by a complete-expression metavariable

```text
source 1: <anchor0>.offset(f(<var0>))
target 1: &mut <anchor0>[f(<var0>)..]

source 2: <anchor0>.offset(f(<var0>, <var1>))
target 2: &mut <anchor0>[f(<var0>, <var1>)..]

rule: [A0].offset([E0]) => &mut [A0][[E0]..]
```

## 15. Operator difference hidden by a complete-expression metavariable

```text
source 1: <anchor0>.offset(<var0> + 1)
target 1: &mut <anchor0>[(<var0> + 1)..]

source 2: <anchor0>.offset(<var0> - 1)
target 2: &mut <anchor0>[(<var0> - 1)..]

rule: [A0].offset([E0]) => &mut [A0][[E0]..]
```

## 16. Target-only magnitude disagreement

```text
source 1: <anchor0>.read()
target 1: 0_i32

source 2: <anchor0>.read()
target 2: 1_i32

rule: reject
```

## 17. Different source and target disagreement pairs

```text
source 1: <anchor0>.offset(1_isize)
target 1: &mut <anchor0>[0_usize..]

source 2: <anchor0>.offset(2_isize)
target 2: &mut <anchor0>[1_usize..]

rule: reject
```

## 18. Identical sources with conflicting targets

```text
source 1: *<anchor0>
target 1: <anchor0>

source 2: *<anchor0>
target 2: &mut *<anchor0>

rule: reject
```

## 19. Degenerate source pattern

```text
source 1: left(*<anchor0>)
target 1: left(*<anchor0>)

source 2: right(<anchor0>.offset(1_isize))
target 2: right(<anchor0>.offset(1_isize))

rule: [E0] => [E0] (rejected)
reason: the complete source pattern is [E0]
```

## 20. Anchor hidden by an expression metavariable

```text
source 1: consume(*<anchor0>)
target 1: consume(*<anchor0>)

source 2: consume(<anchor0>.offset(1_isize))
target 2: consume(<anchor0>.offset(1_isize))

rule: consume([E0]) => consume([E0]) (rejected)
reason: <anchor0> is represented by both [A0] and [E0]
```

## 21. Anchor hidden despite another explicit occurrence

```text
source 1: combine(*<anchor0>, read(<anchor0>))
target 1: combine(<anchor0>, read(<anchor0>))

source 2: combine(*<anchor0>, <anchor0>.offset(1_isize))
target 2: combine(<anchor0>, <anchor0>.offset(1_isize))

rule: combine(*[A0], [E0]) => combine([A0], [E0]) (rejected)
reason: <anchor0> is represented by both [A0] and [E0]
```

## 22. One of multiple anchors hidden

```text
source 1: combine(*<anchor0>, read(<anchor1>))
target 1: combine(<anchor0>, read(<anchor1>))

source 2: combine(*<anchor0>, <anchor1>.offset(1_isize))
target 2: combine(<anchor0>, <anchor1>.offset(1_isize))

rule: combine(*[A0], [E0]) => combine([A0], [E0]) (rejected)
reason: <anchor1> is represented by both [A1] and [E0]
```

## 23. Local identity split across explicit structure and a metavariable

```text
source 1: <var0> + *(<anchor0>.offset(<var0>))
target 1: <var0> + <anchor0>[<var0>]

source 2: <var0> + *(<anchor0>.offset(1_usize))
target 2: <var0> + <anchor0>[1_usize]

rule: [B0] + *([A0].offset([E0]))
      => [B0] + [A0][[E0]] (rejected)
reason: <var0> in observation 1 is represented by both [B0] and [E0]
```

## 24. Inconsistent binding-equality partitions

```text
source 1: combine(*<anchor0>, <var0>, <var0>)
target 1: combine(<anchor0>, <var0>, <var0>)

source 2: combine(*<anchor0>, <var0>, <var1>)
target 2: combine(<anchor0>, <var0>, <var1>)

rule: combine(*[A0], [B0], [B1])
      => combine([A0], [B0], [B1]) (rejected)
reason: <var0> in observation 1 is represented by both [B0] and [B1]
```

## 25. Target-only local binding identity

```text
source 1: make(*<anchor0>)
target 1: make(<anchor0>, <var0>)

source 2: make(*<anchor0>)
target 2: make(<anchor0>, <var0>)

rule: reject
```

## 26. Target-only local field identity

```text
source 1: *<anchor0>
target 1: <anchor0>.<struct0::field0>

source 2: *<anchor0>
target 2: <anchor0>.<struct0::field0>

rule: reject
```

## 27. Different rigid external functions

```text
source 1: ffi::load(*<anchor0>)
target 1: ffi::load(<anchor0>)

source 2: ffi::peek(*<anchor0>)
target 2: ffi::peek(<anchor0>)

rule: reject
```
