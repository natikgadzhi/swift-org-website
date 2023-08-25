---
layout: post
published: true
date: 2023-08-25 12:34:56
title: What's new in Swift debugging on the 5.9 branch?
author: [adrian-prantl, augustonoronha, kastiglione]
---

## What’s new in Swift debugging on the 5.9 branch?

On the Swift 5.9 branch we introduced a couple of new features to [LLDB](https://lldb.llvm.org/ "LLDB project home page") and the Swift compiler to improve the debugging experience. In this post we are highlighting three changes that we expect to have the most noticeable impact on Swift debugging workflows.


### Faster `p` and `po`

The `p` and `po` command aliases have been redefined to the new `dwim-print` command. The `dwim-print` command prints values using the most user friendly implementation. "DWIM" is an acronym for "Do What I Mean". Specifically, when printing variables, `dwim-print` will use the same implementation as `frame variable` or `v` instead of the more expensive expression evaluator.

By default, the output of `p` no longer includes a persistent result variable, such as `$0`, or `$R0`. In addition to the overhead incurred by persisting the result, persistent result variables retain any objects they contain, which can be an unexpected side effect for the program execution. Users who want persistent results on occasion, can use `expression` (or a unique prefix such as `expr`) directly instead of `p`. To enable persistent results every time, the `p` alias can be redefined in the `~/.lldbinit` file:

```
command unalias p
command alias p dwim-print --persistent-result on --
```

The `dwim-print` command also gives `po` new functionality. The `po` command can now print Swift objects by their *address*. When running `po <object-address>`, LLDB's embedded Swift compiler will automatically evaluate the expression `unsafeBitCast(<object-address>, to: AnyObject.self)` under hood to produce the expected result.

See [Introduce dwim-print command](https://reviews.llvm.org/D138315 "LLVM review") and [Change dwim-print to default to disabled persistent results](https://reviews.llvm.org/D145609 "LLVM review") for the patches that introduced these changes.


### Using generic type parameters in expressions

LLDB now supports referring to generic type parameters in expression evaluation. For example, given the following code:

```swift
func use<T>(_ t: T) {
    print(t) // break here
}

use(5)
use("Hello!”)
```

Running `po T.self`, when stopped in `use`, will print `Int` when coming in through the first call, and `String` in the second. This can be especially useful in combination with conditional breakpoints to stop only when a generic function is instantiated with a certain concrete type. For example, adding the following expression as the condition to a breakpoint inside use will only stop when the variable `t` is a `String`: `T.self == String.self`.

More details about the implementation of this feature can be found in the [LLDB PR](https://github.com/apple/llvm-project/pull/5715) introducing it.


### More precise scope information in the Swift compiler

The Swift compiler now emits more precise lexical scopes in the debug information. Scope information allows a debugger to distinguish between the different variables that are all called `x` in the following example:

```swift
func f(x: AnyObject?) {
  // function parameter `x: AnyObject?`
  guard let x else {}
  // local variable `x: AnyObject`, which shadows the function argument `x`
  ...
}
```

In fact, the Swift language's scoping rules allow some astonishing things to be done with variable bindings:

```swift
enum E<T> {
case A(T)
case B(T)
case C(String)
case D(T, T, T)
}

func f<T>(_ e: E<T>) -> String {
  switch e {
  case .A(let a), .B(let a): return "One variable \(a): T in scope"
  case .C(let a):            return "One variable \(a): String in scope"
  case .D(let a, _, let c):  return "One \(a): T and one \(c): T in scope"
  default:                   return "Only the function argument e is in scope"
  }
}
```

All of these can be correctly disambiguated by LLDB thanks to lexical scope debug information.

The Swift compiler now uses more accurate ASTScope information to generate the lexical scope hierarchy in the debug information, which results in some behavior changes in the debugger. In the example below, the local variable `a` is not yet in scope at the call site of `getInt()` and will only be available after it has been assigned:

```
  1 func getInt() -> Int { return 42 }
  2
  3 func f() {
  4     let a = getInt()
                ^
  5     print(a)
  6 }

(lldb) p a
error: <EXPR>:3:1: error: cannot find 'a' in scope

(lldb) n
  3 func f() {
  4     let a = getInt()
  5     print(a)
        ^
  6 }

(lldb) p a
42
```

With the debug information produced by previous versions of the Swift compiler, the debugger might have displayed uninitialized memory as the contents of `a` at the call site of `getInt()`. In Swift 5.9 the variable `a` only becomes visible after it has been initialized.

For more details, see the [pull request](https://github.com/apple/swift/pull/64941) that introduced this change.


### Getting involved

If you want to learn more about Swift debugging and LLDB, provide feedback, or want to get started with improving the tooling, debug information, or the debugger itself, come join us in the [LLDB section](https://forums.swift.org/c/development/lldb/13) of the Swift development forums!
