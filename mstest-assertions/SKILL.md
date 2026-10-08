---
name: mstest-assertions
description: Replacements for the MSTEST0037 analyzer warnings in MSTest 3.x/4.x tests, with the argument order that trips people up (the bound or expected value comes first). Use when fixing MSTEST0037 warnings or writing MSTest assertions on collections, strings and comparisons.
---

# MSTEST0037: use the specific assertion

| Old | New |
|---|---|
| `Assert.IsTrue(s.Contains(x))` | `Assert.Contains(x, s)` |
| `Assert.IsFalse(s.Contains(x))` | `Assert.DoesNotContain(x, s)` |
| `Assert.AreEqual(0, c.Count)` | `Assert.IsEmpty(c)` |
| `Assert.AreEqual(n, c.Count)` | `Assert.HasCount(n, c)` |
| `Assert.IsTrue(c.Count > 0)` | `Assert.IsNotEmpty(c)` |
| `Assert.IsTrue(a < b)` | `Assert.IsLessThan(b, a)` |
| `Assert.IsTrue(a <= b)` | `Assert.IsLessThanOrEqualTo(b, a)` |
| `Assert.IsTrue(a > b)` | `Assert.IsGreaterThan(b, a)` |
| `Assert.IsTrue(a >= b)` | `Assert.IsGreaterThanOrEqualTo(b, a)` |
| `Assert.IsTrue(s.StartsWith(x))` | `Assert.StartsWith(x, s)` |
| `Assert.IsTrue(s.EndsWith(x))` | `Assert.EndsWith(x, s)` |
| `Assert.AreEqual(true, x)` | `Assert.IsTrue(x)` |

- The bound, substring or expected value is the first argument and the value under test the second; the optional message stays last.
- Rebuild with `--no-incremental` to see every warning, then run the suite: a swapped pair compiles and fails only when the assertion would have failed.
