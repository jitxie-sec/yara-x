# Security Policy

## Supported Versions

the main branch of yara-x project

## Reporting a Vulnerability

YARA-X crashes (debug builds) or produces incorrect behavior (release builds) when compiling regex patterns with top-level word boundary assertions `\<` or `\>`.

## Root Cause

**File**: `lib/src/re/parser.rs`, Line 603

The `Transformer::traverse()` function has an empty branch for `Ast::Assertion`:

```rust
match ast {
    Ast::Assertion(_) => {}  // ❌ Does nothing - should call replace_word_boundary_assertions()
    Ast::Group(group) => {
        self.replace_word_boundary_assertions(group.ast.as_mut());  // ✅ Correct
        // ...
    }
    // Other branches correctly handle transformation
}
```

**Problem**: Top-level assertions like `/\>/` bypass transformation, leaving unsupported `WordBoundaryEndAngle` in the AST. This triggers a debug assertion at line 400:

```rust
debug_assert!(!matches!(assertion.kind, 
    AssertionKind::WordBoundaryStartAngle | 
    AssertionKind::WordBoundaryEndAngle
));
```
**Crash variants**:
```yara
rule triggers {
  strings:
    $a = /\>/      // Word boundary end
    $b = /\</      // Word boundary start
    $c = /\w>/     // With character class
  condition: any of them
}
```

## Fix

Add transformation call to the empty branch:

```diff
-Ast::Assertion(_) => {}
+Ast::Assertion(assertion) => {
+    self.replace_word_boundary_assertions(assertion);
+}
```

## Contact

**Reporter**: [jitxie at Tencent Security YUNDING LAB]  
**Email**: [jitxie@tencent.com]  
**Discovery**: Fuzzing with LibFuzzer + AddressSanitizer (~40 hours)
