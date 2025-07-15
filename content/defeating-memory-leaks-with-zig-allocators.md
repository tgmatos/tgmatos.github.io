+++
title = "Defeating Memory Leaks With Zig Allocators"
date = 2025-07-15
[extra]
thumbnail = "/static/images/zig_memory_leak.png"
+++
{{ resize_image(path="./images/zig_memory_leak.png", width=600, height=200, op="fit_width") }}

# Table of Contents
1. [Introduction](#introduction)
2. [Zig Allocators](#zig-allocators)
3. [Debug Allocator](#debug-allocator)
4. [Detecting Memory Leaks](#detecting-memory-leaks)

## Introduction

I started reading the book [Crafting Interpreters](https://craftinginterpreters.com/) to learn about the process of developing and implementing a programming language (_I have some criticism about the way the book presents formal concepts_). The book is divided in two parts. The first one is an interpreter for the Lox programming language, which has a syntax similar to JavaScript and is implemented in Java. The second part is the virtual machine and it is written in C. Instead of using Java for the interpreter, I decided to use [zig](https://ziglang.org/). <!-- The book starts showing how to write the lexer, then how to build the AST and how to evaluate it. -->

Zig is a systems programming language that occupies the same niche as C, Rust, C++ and others, but unlike newer programming languages in this space which tries to be memory safe — such as Rust — Zig does not try to be memory safe but instead it offers enchancements for manual memory management, like optionals for nullable values, `defer` and other features. In this post I will focus on the allocators offered by Zig's standard library.

## Zig Allocators

Contrary to C in which the standard library provides only a single allocator — and developers must rely on external libraries to use alternatives like arena allocators —, Zig goes in the other direction and tries to provide a variety of allocators in the stdlib, and make it easy to switch between then with minimal code change. As an example, this is how a programmer would use an arena allocator in Zig:

```zig
// Code taken from Zig documentation
const std = @import("std");

pub fn main() !void {
    var arena = std.heap.ArenaAllocator.init(std.heap.page_allocator);
    defer arena.deinit();

    const allocator = arena.allocator();

    const ptr = try allocator.create(i32);
    std.debug.print("ptr={*}\n", .{ptr});
}
```

As the code above shows, the `std.heap.ArenaAllocator` receives a child allocator and wraps it to provide an interface that allows allocations without individual deallocations, instead freeing all memory at once. If a programmer was to change from this allocator to another, it would do something like this:

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();

    const allocator = gpa.allocator();

    const ptr = try allocator.create(i32);
    std.debug.print("ptr={*}\n", .{ptr});
}
```

A common practice in Zig code is passing the allocator as a function parameter:

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();

    const allocator = gpa.allocator();

    const arr = createArray(allocator, 5);
    defer allocator.free(arr);
    
    // Do something with "arr"
}

fn createArray(allocator: std.mem.Allocator, size: usize) ![]i32 {
    return allocator.alloc(i32, size) catch |err| {
        return err;
    };
}
```

In the code above, the `createArray` function takes an `allocator: std.mem.Allocator` parameter. This type is an interface (_yes, Zig does have interfaces[^1]_) that allocators must implement. Using this pattern, developers can easily switch out allocators, changing memory allocation behavior with minimal refactoring.

## Debug Allocator

One of the allocators provided by Zig's stdlib is `std.heap.debug_allocator`. This allocator provides many features to help detect bugs, and I leveraged these features to detect and remove memory leaks from my interpreter. This is the list of the features provided by the allocator[^2]:

- Captures stack traces on allocation, free, and optionally resize.
- Double free detection, which prints all three traces (first alloc, first free, second free).
- Leak detection, with stack traces.
- Never reuses memory addresses, making it easier for Zig to detect branch on undefined values in case of dangling pointers. This relies on the backing allocator to also not reuse addresses.
- Uses a minimum backing allocation size to avoid operating system errors from having too many active memory mappings.
- When a page of memory is no longer needed, give it back to resident memory as soon as possible, so that it causes page faults when used.
- Cross platform. Operates based on a backing allocator which makes it work everywhere, even freestanding.
- Compile-time configuration.

The features relevant to this post are the __double free and memory leak detection__, which help developers catch memory issues without using external tools like `valgrind`.

## Detecting Memory Leaks
To understand the memory leak I was trying to solve, I must explain first what I was attemptin. In my interpreter, the `parser.zig` file generates the AST (_abstract syntax tree_) based on the tokens produced by the lexer. The AST (_at the time_) produces an expression and then returns it.

I won't enter in details, but an `expression` is represented as recursivily tagged union, which contains other structs and tagged unions. This is the code which represents the expression:

```zig
pub const Expr = union(enum) {
    const Self = @This();
    literal: *Literal,
    unary: *Unary,
    binary: *Binary,
    grouping: *Grouping,
}

pub const Literal = union(enum) {
    number: f64,
    string: []u8,
    boolean: bool,
    nil: void,
}

pub const Grouping = struct {
    expr: *Expr,
}

pub const Unary = struct {
    operator: Token,
    right: *Expr,
}

pub const Binary = struct {
    exprLeft: *Expr,
    operator: Token,
    exprRight: *Expr,
}
```

The relevant part here is the `Binary` expression, which receives an expression both on the left and right side. For example, the expression `10 - 8 + 2` is a binary expression, in which the left side expression is `10 - 8` and the right side expression is `+ 2`. This would be the internal representation of this expression:

```zig
Expr{
    .binary = .{
        .exprLeft = Expr{
            .binary = .{
                .exprLeft = Expr{
                    .literal = &Literal{ .number = 10.0 },
                },
                .operator = Token.Minus,
                .exprRight = Expr{
                    .literal = &Literal{ .number = 8.0 },
                },
            },
        },
        .operator = Token.Plus,
        .exprRight = Expr{
            .literal = &Literal{ .number = 2.0 },
        },
    },
}
```

To evaluate such expression, I created this function that receives an `Expr` and a `std.mem.Allocator` and dispatchs based on the active tag:

```zig
pub fn evaluate(expr: *Self, allocator: std.mem.Allocator) RuntimeError!*Expr {
    switch (expr.*) {
        .literal => return expr,
        .grouping => return try Grouping.evaluate(expr.grouping.*, allocator),
        .unary => return try Unary.evaluate(expr.unary.*, allocator),
        .binary => return try Binary.evaluate(expr.binary.*, allocator),
    }
}
```

In this example, calling `evaluate` on the full expression causes recursive calls to `Binary.evaluate`, each of which evaluates its sub-expression and returns a result:

```zig
pub fn evaluate(binary: Binary, allocator: std.mem.Allocator) RuntimeError!*Expr {
    const left = try binary.exprLeft.evaluate(allocator);
    const right = try binary.exprRight.evaluate(allocator);

    const expr = try allocator.create(Expr);
    errdefer allocator.destroy(expr);

    const literal = try allocator.create(Literal);
    errdefer allocator.destroy(literal);

    return switch (binary.operator.kind) {
        .MINUS => {
            if (!checkOperand(left.*, right.*)) {
                return RuntimeError.InvalidOperand;
            }
                literal.* = Literal{ .number = left.literal.number - right.literal.number };
            expr.* = .{ .literal = literal };
            return expr;
        }
        // rest of the code
    }
}
```

However, although the code seemed correct, the debug allocator reported memory leaks during executions:

```shell
$ zig build run
> 10 - 8 + 3;
5
> error(gpa): memory address 0x7f216d9e0060 leaked: 
LoxInterpreter/src/expression.zig:177:42: 0x117b7e2 in evaluate (main.zig)
        const expr = try allocator.create(Expr);
                                         ^
LoxInterpreter/src/expression.zig:24:50: 0x116e6d2 in evaluate (main.zig)
            .binary => return try Binary.evaluate(expr.binary.*, allocator),
                                                 ^
LoxInterpreter/src/expression.zig:174:50: 0x117b6bc in evaluate (main.zig)
        const left = try binary.exprLeft.evaluate(allocator);
                                                 ^
LoxInterpreter/src/expression.zig:24:50: 0x116e6d2 in evaluate (main.zig)
            .binary => return try Binary.evaluate(expr.binary.*, allocator),
                                                 ^
LoxInterpreter/src/statement.zig:16:58: 0x1167e39 in evaluate (main.zig)
                const expr = try self.expression.evaluate(allocator);
                                                         ^
LoxInterpreter/src/main.zig:60:26: 0x1168cd1 in runPrompt (main.zig)
            stmt.evaluate(allocator) catch |err| {
                         ^

error(gpa): memory address 0x7f216da00060 leaked: 
LoxInterpreter/src/expression.zig:180:45: 0x117b88e in evaluate (main.zig)
        const literal = try allocator.create(Literal);
                                            ^
LoxInterpreter/src/expression.zig:24:50: 0x116e6d2 in evaluate (main.zig)
            .binary => return try Binary.evaluate(expr.binary.*, allocator),
                                                 ^
LoxInterpreter/src/expression.zig:174:50: 0x117b6bc in evaluate (main.zig)
        const left = try binary.exprLeft.evaluate(allocator);
                                                 ^
LoxInterpreter/src/expression.zig:24:50: 0x116e6d2 in evaluate (main.zig)
            .binary => return try Binary.evaluate(expr.binary.*, allocator),
                                                 ^
LoxInterpreter/src/statement.zig:16:58: 0x1167e39 in evaluate (main.zig)
                const expr = try self.expression.evaluate(allocator);
                                                         ^
LoxInterpreter/src/main.zig:60:26: 0x1168cd1 in runPrompt (main.zig)
            stmt.evaluate(allocator) catch |err| {
```

The problem in this code lies in not freeing memory after evaluating an "intermediary" expression. For instance, the result of evaluating `10 - 8` (_which returns the literal `2`_) would not be freed after returning the result of the expression `2 + 3`. This was an annoying memory leak, which I only discovered thanks to Zig's `std.heap.debug_allocator`.

After identifying the leak, I tried to simply free the memory allocated in `left` and `right`, but doing so caused double free - which Zig's debug allocator warned me about - because when evaluating literals I would return a pointer to it. Eventually I fixed the memory leak by copying the literals and returning the copy and then freeing the intermediary expression afterward. This ensured that only the final result was reteined and returned, while everything else was properly deallocated.

```zig
pub fn evaluate(expr: *Self, allocator: std.mem.Allocator) RuntimeError!*Expr {
    switch (expr.*) {
        .literal => {
            const literal: *Literal = try allocator.create(Literal);
            // Since the string is stored in heap instead of the stack,
            // I must check if the literal is a string and then copy it.
            if (activeTag(expr.literal.*) == Literal.string) {
                const str = try allocator.alloc(u8, expr.literal.string.len);
                @memcpy(str, expr.literal.string);
                literal.* = expr.literal.*;
                literal.*.string = str;
            } else {
                literal.* = expr.literal.*;
            }
            const cpyExpr: *Expr = try allocator.create(Expr);
            cpyExpr.* = .{ .literal = literal };
            return cpyExpr;
        },
        .grouping => return try Grouping.evaluate(expr.grouping.*, allocator),
        .unary => return try Unary.evaluate(expr.unary.*, allocator),
        .binary => return try Binary.evaluate(expr.binary.*, allocator),
    }
}

pub fn evaluate(binary: Binary, allocator: std.mem.Allocator) RuntimeError!*Expr {
    const left = try binary.exprLeft.evaluate(allocator);
    defer left.deinit(allocator;) // Cleanup left expression
    
    const right = try binary.exprRight.evaluate(allocator);
    defer right.deinit(allocator); // Cleanup right expression

    const expr = try allocator.create(Expr);
    errdefer allocator.destroy(expr);

    const literal = try allocator.create(Literal);
    errdefer allocator.destroy(literal);

    return switch (binary.operator.kind) {
        .MINUS => {
            if (!checkOperand(left.*, right.*)) {
                return RuntimeError.InvalidOperand;
            }
                literal.* = Literal{ .number = left.literal.number - right.literal.number };
            expr.* = .{ .literal = literal };
            return expr;
        }
        // rest of the code
    }
}
```

With these changes I was able to fix all the memory leaks and double free in my code.
If you are interested in reading the code, this is the repository: [tgmatos/loxinterpreter](https://github.com/tgmatos/loxinterpreter)


[^1]: [Zig Interfaces](https://www.openmymind.net/Zig-Interfaces/)
[^2]: [std.heap.debug_allocator reference](https://ziglang.org/documentation/0.14.0/std/#std.heap.debug_allocator)
