# Fix: `resource.MustParse` silently overflows int64 near `math.MaxInt64`

**Issue:** [#135487](https://github.com/kubernetes/kubernetes/issues/135487)
**Component:** `k8s.io/apimachinery/pkg/api/resource`
**SIG:** `sig/api-machinery`

---

## What was the bug?

Kubernetes uses `resource.MustParse()` everywhere to read resource values like CPU, memory, and storage:

```yaml
resources:
  requests:
    memory: "9223372036854775808"  # bigger than math.MaxInt64
    cpu: "500m"
```

If the value was **too large to fit in an `int64`**, instead of rejecting it with an error, `MustParse` would **silently return a wrong value** — no panic, no warning, just garbage data flowing into the cluster.

---

## Why is this dangerous?

Every resource request in Kubernetes — pod CPU limits, memory requests, PVC storage sizes — goes through `resource.MustParse` internally.

A silent overflow means:
- A node could think it has **0 bytes** of storage available
- A pod could be scheduled with an **impossible resource allocation**
- No error surfaces — the bug is invisible until something breaks at runtime

---

## What we fixed

### 1. Added overflow helper in `quantity.go`

```go
// mulInt64WithOverflowCheck detects int64 overflow (issue #135487).
func mulInt64WithOverflowCheck(a, b int64) (int64, bool) {
    if a == 0 || b == 0 { return 0, false }
    result := a * b
    if result/b != a { return 0, true }
    return result, false
}
```

Classic Go overflow detection — multiply, then divide back. If you don't get the original number, it overflowed.

### 2. Added round-trip guard in `MustParse`

After parsing succeeds, we verify:

> *"If I convert this value back to a string, do I get the same number I started with?"*

If not → **panic immediately** with a clear message instead of returning silent garbage.

### 3. Added regression test in `quantity_test.go`

```go
func TestMustParseNearMaxInt64(t *testing.T) {
    cases := []struct {
        input     string
        shouldErr bool
    }{
        {"0",                    false},
        {"9223372036854775807",  false}, // exact math.MaxInt64 — valid
        {"9223372036854775808",  true},  // MaxInt64+1 — must panic
        {"9999999999999999999G", true},  // massive overflow — must panic
    }
    ...
}
```

This test permanently locks the behavior — any future PR that accidentally reintroduces the bug will be caught automatically by CI.

---

## Before / After

| | Before | After |
|---|---|---|
| Input `"9223372036854775808"` | Returns wrong value silently ❌ | Panics with clear error ✅ |
| Input `"5Gi"` | Works correctly ✅ | Works correctly ✅ |
| Input `"500m"` | Works correctly ✅ | Works correctly ✅ |

---

## How to test

```bash
cd staging/src/k8s.io/apimachinery
go test ./pkg/api/resource/... -v -run TestMustParse
```

All existing tests must pass. `TestMustParseNearMaxInt64` must pass green.
