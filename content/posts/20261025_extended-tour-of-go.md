---
title: 'A Tour of Go — extended edition'
date: 2026-10-25
tags: [go, tutorial]
description: 'The official Tour of Go, slowed down: every section explained with the why, what happens under the hood and the common mistakes, plus all the exercises.'
draft: false
---

## Introduction

This guide walks through the same ground as the official [A Tour of Go](https://go.dev/tour/)[^1] — basics, flow control, composite types, methods and interfaces, generics and concurrency — but slows down at every step to explain *why* Go works the way it does, what happens under the hood, and which mistakes people commonly make. Every exercise from the Tour is restated so you can work through it yourself.

### How to read this guide

Read it in order the first time. Later parts build on earlier ones: interfaces only make sense once you understand methods, and channels only make sense once you understand goroutines. Each section has a short example you should type in and run yourself; changing the example and predicting what happens is the fastest way to learn.

### Running Go code

The examples assume **Go 1.23 or later**: some use features from 1.22 (per-iteration loop variables, `range` over integers) and 1.23 (iterators, as in `slices.Sorted(maps.Keys(m))`). Check with `go version`.

You have two options.

1. **The Go Playground** at [go.dev/play](https://go.dev/play/). It compiles and runs one program in a sandbox. Time is fixed (it starts at 2009-11-10 23:00:00 UTC), there is no network, and output is buffered. It is perfect for experiments.
2. **A local install** from [go.dev/dl](https://go.dev/dl/). After installing, create a project:

```bash
mkdir hello && cd hello
go mod init example.com/hello   # creates go.mod, which names your module
```

Write `main.go`:

```go
package main

import "fmt"

func main() {
	fmt.Println("Hello, 世界")
}
```

Then run `go run .` to compile and run in one step, or `go build` to produce a binary. Two tools you will use constantly:

- `gofmt` (or `go fmt ./...`) rewrites your code into the one standard style. Go has no style debates: tabs for indentation, braces on the same line, and gofmt decides the rest. Most editors run it on save.
- `go vet ./...` catches suspicious code the compiler accepts, such as a `Printf` format that does not match its arguments.

### What makes Go different

Go was designed at Google in 2007 for large codebases and large teams. Its priorities show up everywhere in this guide: fast compilation, a small language you can hold in your head, explicit error handling instead of exceptions, composition instead of inheritance, and concurrency built into the language. When a feature seems missing (no classes, no `while`, no ternary operator), it is usually a deliberate choice to keep the language small.

## Part 1 — Packages, variables and functions

### 1.1 Packages

Every Go file starts with a `package` clause, and every Go program is made of packages. A package is a directory of `.go` files that are compiled together and share one namespace: a function declared in `a.go` is visible in `b.go` of the same package without any import.

Execution starts in the function `main` of the package named `main`. Any other package name produces a library, not a program.

```go
package main

import (
	"fmt"
	"math/rand"
)

func main() {
	fmt.Println("My favorite number is", rand.Intn(10))
}
```

By convention, the package name is the **last element of the import path**. The package imported as `"math/rand"` contains files that begin with `package rand`, so you refer to it as `rand.Intn`. The path tells the toolchain *where* to find the code; the name is what you type in your source.

Since Go 1.20, the global `math/rand` generator is seeded randomly, so this program prints a different number on each run. In very old Go versions (and in older versions of the Playground) it always printed the same number because the seed was fixed. For new code, `math/rand/v2` is the preferred package.

### 1.2 Imports

You can write one `import` statement per package, or group them in parentheses (a “factored” import). The grouped form is the idiomatic one, and gofmt sorts it alphabetically.

```go
import (
	"fmt"
	"math"
)
```

Two rules surprise newcomers:

- **Unused imports are compile errors**, not warnings. This keeps dependency lists honest and compilation fast. Tools like `goimports` add and remove imports for you automatically.
- You can rename an import (`import str "strings"`), import only for side effects (`import _ "image/png"`, which registers the PNG decoder), but you should avoid dot-imports (`import . "fmt"`) because they hide where names come from.

### 1.3 Exported names

Go has no `public` or `private` keywords. Visibility is decided by the **first letter** of a name:

- Starts with a capital letter (`Pi`, `Println`, `Reader`) → **exported**, visible to other packages.
- Starts with a lowercase letter (`pi`, `helper`) → **unexported**, visible only inside its own package.

```go
fmt.Println(math.Pi)  // ok
fmt.Println(math.pi)  // compile error: cannot refer to unexported name math.pi
```

The rule applies to everything: functions, types, constants, variables, struct fields and methods. Because you can tell at a glance whether something is part of a package’s API, reading Go code requires less jumping around.

### 1.4 Functions

A function takes zero or more parameters. Note that the **type comes after the name**:

```go
func add(x int, y int) int {
	return x + y
}
```

Why types after names? C’s declaration syntax gets hard to read for complex types (a pointer to a function returning a pointer to an array…). Go’s order reads left to right: “`x` is an `int`”, “`f` is a function taking an `int` and returning a `string`”. The Go blog post [Go’s Declaration Syntax](https://go.dev/blog/declaration-syntax) explains the reasoning in detail.[^2]

When consecutive parameters share a type, you can write the type once:

```go
func add(x, y int) int {
	return x + y
}
```

### 1.5 Multiple results

A function can return any number of results. This is used everywhere in Go, most importantly for returning a value *and* an error.

```go
func swap(x, y string) (string, string) {
	return y, x
}

func main() {
	a, b := swap("hello", "world")
	fmt.Println(a, b) // world hello
}
```

The common pattern you will see thousands of times:

```go
f, err := os.Open("data.txt")
if err != nil {
	return err
}
defer f.Close()
```

If you don’t need one of the results, assign it to the **blank identifier** `_`, which discards it: `_, err := strconv.Atoi("42")`.

### 1.6 Named return values

Results can be named. Named results behave like variables declared at the top of the function, initialized to their zero values. A `return` with no arguments (a “naked” return) returns their current values.

```go
func split(sum int) (x, y int) {
	x = sum * 4 / 9
	y = sum - x
	return
}
```

Named results are useful for **documentation** — `func Dims() (width, height int)` is clearer than `(int, int)` — and they let deferred functions modify the result (see 2.12). Naked returns, however, hurt readability in anything longer than a few lines: the reader has to scroll up to see what is being returned. Use them only in short functions.

### 1.7 Variables

The `var` statement declares a list of variables, again with the type last. It can appear at package level or inside a function.

```go
var c, python, java bool

func main() {
	var i int
	fmt.Println(i, c, python, java) // 0 false false false
}
```

A variable declared inside a function that is never used is a **compile error**, just like an unused import. Package-level variables are exempt.

### 1.8 Variables with initializers

A declaration can include one initializer per variable. If an initializer is present, the type can be omitted and is taken from the initializer.

```go
var i, j int = 1, 2

func main() {
	var c, python, java = true, false, "no!"
	fmt.Println(i, j, c, python, java)
}
```

Here `c` and `python` are `bool` and `java` is `string` — each variable gets its own inferred type.

### 1.9 Short variable declarations

Inside a function, `:=` declares and initializes in one step, with the type inferred:

```go
k := 3
c, python, java := true, false, "no!"
```

Things to know about `:=`:

- It is **only allowed inside functions**. At package level every statement must start with a keyword (`var`, `func`, `const`, `type`, `import`), which keeps parsing simple.
- At least one variable on the left must be **new**. The others are simply assigned. That is why `f, err := ...` followed by `g, err := ...` works: `g` is new and `err` is reused.
- Beware of **shadowing**: `:=` in an inner block creates a *new* variable that hides the outer one. A classic bug:

```go
func load() error {
	var err error
	if cond {
		x, err := compute() // new x AND a new err, shadowing the outer one
		fmt.Println(x, err)
	}
	return err // always nil, whatever compute returned!
}
```

`go vet` doesn't catch this by default, so the habit to build is: inside a nested block, use `=` when you mean to assign to an existing variable.

### 1.10 Basic types

| Category | Types | Notes |
| --- | --- | --- |
| Boolean | `bool` | `true` / `false`; not interchangeable with integers |
| String | `string` | Immutable sequence of bytes, usually UTF-8 |
| Signed integers | `int` `int8` `int16` `int32` `int64` | `int` is 32 or 64 bits depending on the platform |
| Unsigned integers | `uint` `uint8` `uint16` `uint32` `uint64` `uintptr` | `uintptr` holds a pointer value; rarely needed |
| Aliases | `byte` = `uint8`, `rune` = `int32` | A `rune` is one Unicode code point |
| Floating point | `float32` `float64` | Prefer `float64` |
| Complex | `complex64` `complex128` | Built-in `real`, `imag`, `complex` |

```go
var (
	ToBe   bool       = false
	MaxInt uint64     = 1<<64 - 1
	z      complex128 = cmplx.Sqrt(-5 + 12i)
)

func main() {
	fmt.Printf("Type: %T Value: %v\n", ToBe, ToBe)
	fmt.Printf("Type: %T Value: %v\n", MaxInt, MaxInt)
	fmt.Printf("Type: %T Value: %v\n", z, z)
}
```

`%T` prints a value’s type and `%v` its value in a default format — both are invaluable when exploring. When you need an integer, just use `int` unless you have a specific reason (binary formats, memory layout) for a sized or unsigned type.

**Strings, bytes and runes.** `len("世界")` is 6, not 2, because `len` counts bytes and each of those characters takes 3 bytes in UTF-8. Indexing a string (`s[0]`) gives a byte. Ranging over a string (see 3.16) gives runes. Use `utf8.RuneCountInString` to count characters.

### 1.11 Zero values

Variables declared without an initializer get their type’s **zero value**:

| Type | Zero value |
| --- | --- |
| Numeric | `0` |
| `bool` | `false` |
| `string` | `""` |
| Pointers, slices, maps, channels, functions, interfaces | `nil` |
| Structs | every field set to its zero value |

There is no such thing as an uninitialized variable in Go. This removes a whole class of bugs and enables a design principle: **make the zero value useful**. A `sync.Mutex` needs no constructor — its zero value is an unlocked mutex. A `bytes.Buffer` zero value is an empty buffer ready to use. Try to design your own types the same way.

### 1.12 Type conversions

`T(v)` converts value `v` to type `T`:

```go
i := 42
f := float64(i)
u := uint(f)
```

Unlike C, Go **never converts implicitly** between numeric types, even between `int` and `int64` or from `int32` to `float64`. This looks fussy at first, but it means every lossy or sign-changing conversion is visible in the code.

```go
var x, y int = 3, 4
var f float64 = math.Sqrt(float64(x*x + y*y))
var z uint = uint(f)
```

Remove a `float64(...)` from this example and the compiler refuses it. Also note: converting float to int truncates toward zero, and converting an integer to a string with `string(65)` produces `"A"` (a rune), not `"65"` — use `strconv.Itoa` for that.

### 1.13 Type inference

When the type is omitted (`var i = ...` or `i := ...`), it is inferred from the right-hand side. If the right side is a typed value, the new variable gets the same type. If it is an **untyped constant**, the type depends on the constant’s kind:

```go
i := 42           // int
f := 3.142        // float64
g := 0.867 + 0.5i // complex128
r := 'x'          // rune (int32)
```

### 1.14 Constants

Constants are declared with `const`. They can be character, string, boolean or numeric values, and they cannot use `:=`.

```go
const Pi = 3.14

const (
	StatusOK       = 200
	StatusNotFound = 404
)
```

Constants are evaluated at compile time; you cannot assign the result of a function call like `math.Sqrt(2)` to a constant. A handy companion is `iota`, a counter that increments within a `const` block, which is how Go writes enumerations:

```go
type Weekday int

const (
	Sunday Weekday = iota // 0
	Monday                // 1
	Tuesday               // 2
)
```

### 1.15 Numeric constants

Numeric constants are **high-precision values** that have no fixed type until context demands one. The compiler represents them with at least 256 bits of precision, so intermediate results don’t overflow.

```go
const (
	Big   = 1 << 100  // far larger than any integer type
	Small = Big >> 99 // 2
)

func needInt(x int) int           { return x*10 + 1 }
func needFloat(x float64) float64 { return x * 0.1 }

func main() {
	fmt.Println(needInt(Small))   // 21
	fmt.Println(needFloat(Small)) // 0.2
	fmt.Println(needFloat(Big))   // 1.2676506002282295e+29
	// fmt.Println(needInt(Big))  // error: constant overflows int
}
```

`Big` is a perfectly legal constant; the error happens only when you try to fit it into an `int`. This is why you can write `time.Sleep(2 * time.Second)` or `x := 1.5 * someFloat` without conversions: the untyped constant adopts the type the context needs, as long as the value fits.

## Part 2 — Flow control: for, if, else, switch and defer

### 2.1 For

Go has exactly one loop keyword: `for`. The full form has three parts separated by semicolons:

```go
sum := 0
for i := 0; i < 10; i++ {
	sum += i
}
fmt.Println(sum) // 45
```

- **init** (`i := 0`) runs once before the first iteration. Variables declared here exist only inside the loop.
- **condition** (`i < 10`) is checked before every iteration; the loop stops when it is false.
- **post** (`i++`) runs at the end of every iteration.

There are no parentheses around the three parts, but the braces `{ }` are **always required**, even for a one-line body. This eliminates the “dangling statement” bugs of C, where adding a second line to an unbraced loop silently leaves it outside.

Note that `i++` is a *statement*, not an expression. You cannot write `x = i++`, and there is no `++i`.

**Loop variables are per-iteration (Go 1.22+).** When the loop *declares* its variables (`for i := ...` or `for i, v := range ...`) and the module uses Go 1.22 or later language semantics (the `go` line in `go.mod`), each iteration gets a fresh copy of `i`. Before that, all iterations shared one variable, so closures or goroutines that captured `i` could observe a later value, typically the final one, instead of the one from their own iteration. If you read older code with `i := i` inside a loop, that was the workaround. A loop that assigns to a variable declared outside it (`for i = 0; ...` or `for i = range ...`) still reuses that one variable, in every Go version.

### 2.2 For, continued

The init and post statements are optional:

```go
sum := 1
for ; sum < 1000; {
	sum += sum
}
```

### 2.3 For is Go’s “while”

Drop the semicolons too, and you have a `while` loop:

```go
sum := 1
for sum < 1000 {
	sum += sum
}
fmt.Println(sum) // 1024
```

gofmt removes the semicolons from the previous example automatically, so this is the form you will always see.

### 2.4 Forever

Omit the condition and the loop runs until something inside breaks out of it (`break`, `return`, `panic`, or the program ending):

```go
for {
	line, err := reader.ReadString('\n')
	if err != nil {
		break
	}
	process(line)
}
```

This is common in servers, event loops and readers. `continue` skips to the next iteration. Both `break` and `continue` accept a **label** to target an outer loop:

```go
outer:
	for _, row := range grid {
		for _, cell := range row {
			if cell == target {
				break outer
			}
		}
	}
```

Since Go 1.22 you can also loop over an integer: `for i := range 10 { ... }` runs with `i` from 0 to 9. Range loops are covered in 3.16.

### 2.5 If

`if` looks like `for`: no parentheses, braces required.

```go
func sqrt(x float64) string {
	if x < 0 {
		return sqrt(-x) + "i"
	}
	return fmt.Sprint(math.Sqrt(x))
}
```

The condition must be a `bool`. Unlike C or JavaScript, `if n { ... }` with an integer or `if ptr { ... }` with a pointer does not compile — write `n != 0` or `ptr != nil`.

### 2.6 If with a short statement

`if` can start with a short statement executed before the condition. Variables declared there are scoped to the `if` (and any `else` branches):

```go
func pow(x, n, lim float64) float64 {
	if v := math.Pow(x, n); v < lim {
		return v
	}
	return lim
}
```

This is the idiomatic way to handle errors without leaking variables into the surrounding scope:

```go
if err := os.WriteFile(name, data, 0o644); err != nil {
	return fmt.Errorf("saving %s: %w", name, err)
}
```

### 2.7 If and else

Variables declared in the short statement are also visible in `else` blocks:

```go
func pow(x, n, lim float64) float64 {
	if v := math.Pow(x, n); v < lim {
		return v
	} else {
		fmt.Printf("%g >= %g\n", v, lim)
	}
	// v is out of scope here
	return lim
}
```

The `else` must be on the same line as the closing brace. That is because Go inserts semicolons automatically at the end of lines that could end a statement; a `}` on its own line would end the `if` before the `else` arrives.

**Idiom: keep the happy path unindented.** Go code rarely uses `else` after a `return`. Instead of nesting, handle the exceptional case and return early:

```go
// Preferred
if err != nil {
	return err
}
doTheWork()
```

### 2.8 Exercise: Loops and functions — square root with Newton’s method

**Task.** Implement `Sqrt(x float64) float64` without `math.Sqrt`. Start with a guess `z` and repeatedly improve it with

```text
z ← z − (z² − x) / (2z)
```

The term z² − x measures how far z² is from x; dividing by 2z (the derivative of z²) tells you how much to adjust. First run the loop 10 times and print `z` each time, then change it to stop when `z` stops changing.

Try it in the Tour: [Exercise: Loops and Functions](https://go.dev/tour/flowcontrol/8)

### 2.9 Switch

A `switch` is a shorter way to write a chain of `if`/`else` statements. It runs the **first** case whose value equals the condition.

```go
switch os := runtime.GOOS; os {
case "darwin":
	fmt.Println("macOS.")
case "linux":
	fmt.Println("Linux.")
default:
	// freebsd, openbsd, plan9, windows...
	fmt.Printf("%s.\n", os)
}
```

The big difference from C, Java and JavaScript: **Go only runs the selected case**, then leaves the switch. There is no implicit fall-through and no need for `break` at the end of each case. If you really want to continue into the next case, write `fallthrough` explicitly (rarely needed).

Other differences:

- Cases need not be constants, and the values need not be integers — strings, function calls, anything comparable works.
- One case can list several values: `case "saturday", "sunday":`.
- `break` inside a switch exits the switch, not an enclosing `for`. Use a labeled break to leave the loop.

### 2.10 Switch evaluation order

Cases are evaluated **top to bottom**, and evaluation stops at the first match. Later case expressions are never evaluated:

```go
switch i {
case 0:
case f(): // f is not called if i == 0
}
```

This lets you put cheap or most likely cases first, and it matters when case expressions have side effects.

```go
today := time.Now().Weekday()
switch time.Saturday {
case today + 0:
	fmt.Println("Today.")
case today + 1:
	fmt.Println("Tomorrow.")
case today + 2:
	fmt.Println("In two days.")
default:
	fmt.Println("Too far away.")
}
```

### 2.11 Switch with no condition

`switch` with no condition is the same as `switch true`. Each case is then a boolean expression, and the first true one wins. This is the cleanest way to write long if-else chains:

```go
t := time.Now()
switch {
case t.Hour() < 12:
	fmt.Println("Good morning!")
case t.Hour() < 17:
	fmt.Println("Good afternoon.")
default:
	fmt.Println("Good evening.")
}
```

### 2.12 Defer

A `defer` statement postpones a function call until the **surrounding function returns** — whether by reaching the end, by `return`, or by panicking.

```go
func main() {
	defer fmt.Println("world")
	fmt.Println("hello")
}
// hello
// world
```

The crucial detail: the deferred call’s **arguments are evaluated immediately**, at the `defer` statement. Only the call itself is delayed.

```go
x := 1
defer fmt.Println("deferred:", x) // captures 1 now
x = 2
fmt.Println("now:", x)
// now: 2
// deferred: 1
```

The main use of `defer` is cleanup placed right next to acquisition, so you cannot forget it on any return path:

```go
f, err := os.Open(path)
if err != nil {
	return err
}
defer f.Close()

mu.Lock()
defer mu.Unlock()
```

Two more facts that you will need later:

- A deferred closure can read and modify **named results**, which is how functions annotate errors on the way out: `defer func() { if err != nil { err = fmt.Errorf("load: %w", err) } }()`.
- `recover()` stops a panic, but only when a deferred function calls it **directly**, while the panic is unwinding the **same goroutine**. That is the only place it has any effect. If the deferred function calls a helper that in turn calls `recover`, the helper gets `nil` and the panic continues. A panic in another goroutine can’t be recovered from here at all; it has to be recovered by a deferred call in that goroutine, or it crashes the whole program.

Avoid `defer` inside a long loop: the calls pile up until the *function* returns, not the loop iteration. Move the loop body into its own function instead.

### 2.13 Stacking defers

Deferred calls are pushed onto a stack and run in **last-in, first-out** order:

```go
func main() {
	fmt.Println("counting")
	for i := 0; i < 4; i++ {
		defer fmt.Println(i)
	}
	fmt.Println("done")
}
// counting
// done
// 3
// 2
// 1
// 0
```

LIFO order is exactly what cleanup needs: resources are released in the reverse order they were acquired, so something opened later (which may depend on something opened earlier) is closed first. The Go blog post [Defer, Panic, and Recover](https://go.dev/blog/defer-panic-and-recover) covers the topic in depth.[^3]

## Part 3 — More types: structs, slices and maps

### 3.1 Pointers

A pointer holds the **memory address** of a value. The type `*T` is a pointer to a `T`; its zero value is `nil`.

- `&x` (“address of”) produces a pointer to `x`.
- `*p` (“dereference”) reads or writes the value `p` points to.

```go
i, j := 42, 2701

p := &i         // p points to i
fmt.Println(*p) // 42, read i through p
*p = 21         // set i through p
fmt.Println(i)  // 21

p = &j          // now point to j
*p = *p / 37
fmt.Println(j)  // 73
```

Why pointers at all? Go passes **everything by value**: a function receives a copy of its arguments. To let a function modify the caller’s variable, or to avoid copying a large struct, you pass a pointer.

```go
func reset(n *int) { *n = 0 }

x := 5
reset(&x)
fmt.Println(x) // 0
```

What Go deliberately leaves out: **pointer arithmetic**. You cannot write `p + 1` to walk through memory. That (together with garbage collection) makes Go memory-safe. It is also perfectly fine to return a pointer to a local variable — the compiler’s escape analysis moves the variable to the heap when needed:

```go
func newCounter() *int {
	c := 0
	return &c // safe in Go, a bug in C
}
```

Dereferencing a `nil` pointer causes a runtime panic (“invalid memory address or nil pointer dereference”).

### 3.2 Structs

A struct is a typed collection of named fields — Go’s way of grouping data. There are no classes; structs plus methods (Part 4) fill that role.

```go
type Vertex struct {
	X int
	Y int
}

func main() {
	fmt.Println(Vertex{1, 2}) // {1 2}
}
```

The fields are laid out contiguously in memory, in the order declared. A struct is a **value**: assigning it or passing it to a function copies all its fields.

### 3.3 Struct fields

Fields are accessed with a dot:

```go
v := Vertex{1, 2}
v.X = 4
fmt.Println(v.X) // 4
```

The capitalization rule applies to fields: `X` is visible to other packages, a field named `x` is not. This matters for packages like `encoding/json`, which can only see exported fields. Struct **tags** let you control how such packages treat a field:

```go
type User struct {
	Name  string `json:"name"`
	Email string `json:"email,omitempty"`
	pass  string // unexported: ignored by encoding/json
}
```

### 3.4 Pointers to structs

With a pointer `p` to a struct, the field can be written `(*p).X`. That is clumsy, so Go lets you write just `p.X` — the dereference is automatic.

```go
v := Vertex{1, 2}
p := &v
p.X = 1e9
fmt.Println(v) // {1000000000 2}
```

In practice structs are very often handled through pointers (`*Vertex`), especially when they are large or when methods need to modify them.

### 3.5 Struct literals

A struct literal creates a new struct value. You can list values in field order, or name fields with `Name: value` in any order (omitted fields get their zero value). Prefixing with `&` returns a pointer.

```go
var (
	v1 = Vertex{1, 2}  // X:1 Y:2
	v2 = Vertex{X: 1}  // Y:0 is implicit
	v3 = Vertex{}      // X:0 Y:0
	p  = &Vertex{1, 2} // type *Vertex
)
```

**Prefer named fields** for any struct from another package, and for structs with more than two or three fields. Positional literals break as soon as someone adds or reorders a field; `go vet` warns about unkeyed literals of imported struct types.

Anonymous structs are handy for one-off data, especially table-driven tests:

```go
tests := []struct {
	in   int
	want int
}{
	{1, 2},
	{2, 4},
}
```

### 3.6 Arrays

The type `[n]T` is an array of exactly `n` values of type `T`. **The length is part of the type**, so `[4]int` and `[5]int` are different, incompatible types, and arrays cannot be resized.

```go
var a [2]string
a[0] = "Hello"
a[1] = "World"
fmt.Println(a[0], a[1]) // Hello World

primes := [6]int{2, 3, 5, 7, 11, 13}
fmt.Println(primes)
```

Arrays are values: assigning one array to another copies every element, and passing an array to a function passes a copy. `[...]int{1, 2, 3}` lets the compiler count the length. You will rarely use arrays directly; they are the storage that **slices** are built on.

### 3.7 Slices

A slice is a **dynamically sized, flexible view** into an array. The type `[]T` (no length) is a slice of `T`. Slices are what you use for lists in Go.

```go
primes := [6]int{2, 3, 5, 7, 11, 13}

var s []int = primes[1:4]
fmt.Println(s) // [3 5 7]
```

`a[low:high]` is a **half-open range**: it includes index `low` and excludes `high`. So `primes[1:4]` has the elements at indexes 1, 2 and 3, and its length is always `high - low`.

Under the hood a slice is a small struct of three words, often called the **slice header**:

```text
slice s = primes[1:4]

  +-----+-----+-----+
  | ptr | len | cap |      ptr -> primes[1], len = 3, cap = 5
  +-----+-----+-----+
     |
     v
primes: [ 2 | 3 | 5 | 7 | 11 | 13 ]
              ^---------^           <- visible elements (len 3)
              ^---------------------^ <- room to grow (cap 5)
```

Passing a slice to a function copies only this header — cheap, regardless of how many elements the slice has.

### 3.8 Slices are like references to arrays

A slice stores no data of its own. It describes a section of an underlying array, and **changing an element through a slice changes the array**. Other slices that share the same array see the change.

```go
names := [4]string{"John", "Paul", "George", "Ringo"}

a := names[0:2] // [John Paul]
b := names[1:3] // [Paul George]

b[0] = "XXX"
fmt.Println(a, b)  // [John XXX] [XXX George]
fmt.Println(names) // [John XXX George Ringo]
```

This sharing is the source of most slice surprises. When you need an independent copy, make one explicitly with `copy` or `slices.Clone`.

### 3.9 Slice literals

A slice literal is like an array literal without the length. It creates an array and then a slice that references it:

```go
q := []int{2, 3, 5, 7, 11, 13}
r := []bool{true, false, true}

s := []struct {
	i int
	b bool
}{
	{2, true},
	{3, false},
}
```

### 3.10 Slice defaults

When slicing you can omit either bound: the low bound defaults to 0 and the high bound to the slice’s length. For an array `var a [10]int`, these are all the same:

```go
a[0:10]
a[:10]
a[0:]
a[:]
```

`arr[:]` is the common way to turn a whole array into a slice.

### 3.11 Slice length and capacity

- **Length** `len(s)`: the number of elements the slice contains.
- **Capacity** `cap(s)`: the number of elements in the underlying array, counting from the slice’s first element.

```go
s := []int{2, 3, 5, 7, 11, 13}
printSlice(s) // len=6 cap=6 [2 3 5 7 11 13]

s = s[:0]     // len=0 cap=6 []
s = s[:4]     // len=4 cap=6 [2 3 5 7]   -- extend again within capacity
s = s[2:]     // len=2 cap=4 [5 7]       -- dropping the front loses capacity

func printSlice(s []int) {
	fmt.Printf("len=%d cap=%d %v\n", len(s), cap(s), s)
}
```

You can re-extend a slice’s length up to its capacity, but not beyond (that panics with “slice bounds out of range”). Moving the low bound forward permanently cuts off the earlier elements for that slice.

A full slice expression `a[low:high:max]` also limits the capacity to `max - low`, which forces any later `append` to copy instead of overwriting the shared array.

### 3.12 Nil slices

The zero value of a slice is `nil`: length 0, capacity 0, no underlying array.

```go
var s []int
fmt.Println(s, len(s), cap(s)) // [] 0 0
fmt.Println(s == nil)          // true
```

A nil slice works like an empty slice everywhere that matters: `len`, `range` and `append` all handle it. So there is usually no need to initialize a slice before appending to it. To test for emptiness, write `len(s) == 0`, which is correct for both nil and empty slices. (One visible difference: `encoding/json` encodes a nil slice as `null` and an empty one as `[]`.)

### 3.13 Creating a slice with make

The built-in `make` allocates a zeroed array and returns a slice referencing it. This is how you create dynamically sized slices.

```go
a := make([]int, 5)    // len=5 cap=5 [0 0 0 0 0]
b := make([]int, 0, 5) // len=0 cap=5 []
c := b[:2]             // len=2 cap=5 [0 0]
d := c[2:5]            // len=3 cap=3 [0 0 0]
```

If you know roughly how many elements you will append, `make([]T, 0, n)` avoids repeated reallocation.

### 3.14 Slices of slices

Slices can contain any type, including other slices. This is how you build grids and matrices:

```go
board := [][]string{
	{"_", "_", "_"},
	{"_", "_", "_"},
	{"_", "_", "_"},
}
board[0][0] = "X"
board[2][2] = "O"
board[1][2] = "X"

for i := range board {
	fmt.Println(strings.Join(board[i], " "))
}
```

Each inner slice is independent and can have its own length (a “jagged” array). When creating a grid dynamically, you allocate the outer slice and then each row:

```go
grid := make([][]int, rows)
for y := range grid {
	grid[y] = make([]int, cols)
}
```

### 3.15 Appending to a slice

`append(s, vs...)` returns a slice with the new values added:

```go
var s []int  // len=0 cap=0 []
s = append(s, 0)       // len=1 cap=1 [0]
s = append(s, 1)       // len=2 cap=2 [0 1]
s = append(s, 2, 3, 4) // len=5 cap=6 [0 1 2 3 4]
```

How it works: if the underlying array has spare capacity, `append` writes into it and returns a slice with a larger length. If not, it **allocates a bigger array** (roughly doubling for small slices, growing more gently for large ones), copies the elements over, and returns a slice pointing to the new array. The exact capacities you see are an implementation detail.

Two rules follow from this:

1. **Always assign the result**: `s = append(s, x)`. The returned slice may point to a different array than `s` did.
2. **Beware of shared backing arrays.** If two slices share an array with spare capacity, appending to one can overwrite elements visible through the other:

```go
base := make([]int, 3, 10)
a := append(base, 1) // writes into base's array at index 3
b := append(base, 2) // also writes at index 3!
fmt.Println(a[3])    // 2, not 1
```

To append one slice to another, use `...`: `s = append(s, other...)`. The [Go Slices: usage and internals](https://go.dev/blog/slices-intro) blog post is worth reading once you are comfortable here.[^4]

### 3.16 Range

The `range` form of `for` iterates over a slice, array, string, map, channel, integer, or iterator function. For slices it yields two values each iteration: the **index** and a **copy** of the element at that index.

```go
var pow = []int{1, 2, 4, 8, 16, 32, 64, 128}

for i, v := range pow {
	fmt.Printf("2**%d = %d\n", i, v)
}
```

Because `v` is a copy, assigning to it does not change the slice. To modify elements, index into the slice: `pow[i] *= 2`.

Over a **string**, `range` decodes UTF-8 and yields the byte offset and the rune:

```go
for i, r := range "hé!" {
	fmt.Println(i, string(r)) // 0 h, 1 é, 3 !
}
```

### 3.17 Range, continued

Skip the index or the value with `_`, or drop the value entirely:

```go
for i, _ := range pow  // same as: for i := range pow
for _, value := range pow
```

```go
pow := make([]int, 10)
for i := range pow {
	pow[i] = 1 << uint(i) // == 2**i
}
for _, value := range pow {
	fmt.Printf("%d\n", value)
}
```

### 3.18 Exercise: Slices — drawing a picture

**Task.** Implement `Pic(dx, dy int) [][]uint8`. It must return a slice of `dy` rows, each a slice of `dx` 8-bit unsigned integers. The Tour’s `pic.Show` renders the values as a grayscale image (0 = black, 255 = white).

Try it in the Tour: [Exercise: Slices](https://go.dev/tour/moretypes/18)

### 3.19 Maps

A map associates keys with values; it is Go’s hash table. The type `map[K]V` maps keys of type `K` to values of type `V`. The key type must be **comparable** with `==` (numbers, strings, booleans, pointers, channels, interfaces, and structs or arrays made of these) — so slices, maps and functions cannot be keys.

Interface key types are the trap. `map[any]int` compiles, because interface types are comparable, but the check that the *dynamic* type is comparable only happens at run time:

```go
m := map[any]int{}
m["ok"] = 1      // fine: string is comparable
m[[]int{1}] = 2  // panic: runtime error: hash of unhashable type []int
```

The same rule bites generic code that uses `comparable` (5.1).

The zero value of a map is `nil`. A nil map behaves like an empty map for reading, but **writing to it panics**. Create maps with `make` or a literal.

```go
type Vertex struct {
	Lat, Long float64
}

var m map[string]Vertex

func main() {
	m = make(map[string]Vertex)
	m["Bell Labs"] = Vertex{40.68433, -74.39967}
	fmt.Println(m["Bell Labs"])
}
```

Maps, like slices, are reference-like: passing a map to a function lets it modify the caller’s map. Iteration order with `range` over a map is **unspecified** and may differ from one loop to the next, so never depend on it; sort the keys (`slices.Sorted(maps.Keys(m))`) when you need stable output.

### 3.20 Map literals

Map literals are like struct literals, but keys are required:

```go
var m = map[string]Vertex{
	"Bell Labs": Vertex{40.68433, -74.39967},
	"Google":    Vertex{37.42202, -122.08408},
}
```

### 3.21 Map literals, continued

If the top-level type is just a type name, you can omit it from the elements:

```go
var m = map[string]Vertex{
	"Bell Labs": {40.68433, -74.39967},
	"Google":    {37.42202, -122.08408},
}
```

### 3.22 Mutating maps

```go
m[key] = elem       // insert or update
elem = m[key]       // retrieve (zero value if absent)
delete(m, key)      // remove (no-op if absent)
elem, ok := m[key]  // test for presence
```

The **comma-ok** form is essential because a missing key returns the zero value, which you cannot distinguish from a stored zero:

```go
m := map[string]int{"Answer": 42}

m["Answer"] = 48
delete(m, "Answer")
v, ok := m["Answer"]
fmt.Println(v, ok) // 0 false
```

Returning the zero value is often convenient, though: counting with `counts[word]++` works without first checking whether the key exists.

One restriction: map elements are **not addressable**. If the value is a struct, `m[k].X = 1` does not compile; read the struct, change it and store it back, or store pointers (`map[string]*Vertex`). Maps are also **not safe for concurrent writes** — see Part 6.

### 3.23 Exercise: Maps — word count

**Task.** Implement `WordCount(s string) map[string]int`, returning a count of each “word” in `s`. The Tour’s test harness checks it against several inputs. Hint: `strings.Fields`.

Try it in the Tour: [Exercise: Maps](https://go.dev/tour/moretypes/23)

### 3.24 Function values

Functions are values. You can store them in variables, pass them as arguments and return them as results.

```go
func compute(fn func(float64, float64) float64) float64 {
	return fn(3, 4)
}

func main() {
	hypot := func(x, y float64) float64 {
		return math.Sqrt(x*x + y*y)
	}
	fmt.Println(hypot(5, 12))      // 13
	fmt.Println(compute(hypot))    // 5
	fmt.Println(compute(math.Pow)) // 81
}
```

The type of `hypot` is `func(float64, float64) float64`; any function with the same signature (like `math.Pow`) can be used wherever that type is expected. The standard library uses this pattern constantly: `sort.Slice(s, less)`, `strings.Map(fn, s)`, `http.HandleFunc(path, handler)`.

### 3.25 Function closures

A function literal can refer to variables from the scope where it was defined. It is then a **closure**: it captures those variables themselves (not copies), and keeps them alive as long as the closure exists.

```go
func adder() func(int) int {
	sum := 0
	return func(x int) int {
		sum += x
		return sum
	}
}

func main() {
	pos, neg := adder(), adder()
	for i := 0; i < 10; i++ {
		fmt.Println(pos(i), neg(-2*i))
	}
}
```

Each call to `adder` creates a **new** `sum` variable, so `pos` and `neg` each have their own running total. `sum` outlives `adder`’s return because the returned closure still refers to it; the compiler puts it on the heap.

Closures are how Go code builds callbacks, generators, middleware and small bits of private state without defining a type.

### 3.26 Exercise: Fibonacci closure

**Task.** Implement `fibonacci()` returning a function that returns successive Fibonacci numbers (0, 1, 1, 2, 3, 5, …) each time it is called.

Try it in the Tour: [Exercise: Fibonacci closure](https://go.dev/tour/moretypes/26)

## Part 4 — Methods and interfaces

### 4.1 Methods

Go has no classes, but you can define **methods** on types. A method is a function with a special **receiver** argument, written between `func` and the method name:

```go
type Vertex struct {
	X, Y float64
}

func (v Vertex) Abs() float64 {
	return math.Sqrt(v.X*v.X + v.Y*v.Y)
}

func main() {
	v := Vertex{3, 4}
	fmt.Println(v.Abs()) // 5
}
```

By convention the receiver name is short (one or two letters, usually the type’s initial) and consistent across all methods of the type. Go style avoids `this` and `self`.

### 4.2 Methods are functions

A method is just a function whose first argument is the receiver. This version of `Abs` is a regular function with identical behaviour:

```go
func Abs(v Vertex) float64 {
	return math.Sqrt(v.X*v.X + v.Y*v.Y)
}

fmt.Println(Abs(v))
```

You can even get the function form of a method: `Vertex.Abs` is a function of type `func(Vertex) float64` (a *method expression*), and `v.Abs` is a function value with `v` already bound (a *method value*), handy for callbacks like `button.OnClick(counter.Increment)`.

### 4.3 Methods on non-struct types

You can declare methods on any named type defined **in the same package**, not just structs:

```go
type MyFloat float64

func (f MyFloat) Abs() float64 {
	if f < 0 {
		return float64(-f)
	}
	return float64(f)
}
```

You cannot add methods to types from other packages (including built-in types like `int`). This keeps method sets predictable: you can always find every method of a type in its own package. To extend someone else’s type, define your own type based on it, or wrap it in a struct.

The standard library uses this a lot: `time.Duration` is an `int64` with methods like `Seconds()` and `String()`; `http.HandlerFunc` is a function type with a `ServeHTTP` method.

### 4.4 Pointer receivers

A receiver can be a pointer `*T`. Methods with pointer receivers can **modify** the value they are called on:

```go
func (v *Vertex) Scale(f float64) {
	v.X = v.X * f
	v.Y = v.Y * f
}

func main() {
	v := Vertex{3, 4}
	v.Scale(10)
	fmt.Println(v.Abs()) // 50
}
```

Remove the `*` from `Scale` and the program prints 5: a value receiver works on a **copy**, so `Scale` would scale the copy and throw it away. This is the same pass-by-value rule as for ordinary arguments (3.1). In practice, pointer receivers are more common than value receivers because methods often need to change their receiver.

### 4.5 Pointers and functions

Rewriting the methods as functions makes the difference explicit:

```go
func Scale(v *Vertex, f float64) {
	v.X = v.X * f
	v.Y = v.Y * f
}

v := Vertex{3, 4}
Scale(&v, 10) // must pass a pointer explicitly
```

With a plain function taking `*Vertex`, you must write `&v`; `Scale(v, 10)` is a compile error. Methods are more forgiving, as the next section shows.

### 4.6 Methods and pointer indirection

For method calls Go inserts `&` or `*` automatically:

```go
v := Vertex{3, 4}
v.Scale(5)   // Go rewrites this as (&v).Scale(5)

p := &Vertex{4, 3}
p.Scale(10)  // already a pointer

fmt.Println(v.Abs()) // value receiver, value
fmt.Println(p.Abs()) // value receiver, pointer: Go rewrites as (*p).Abs()
```

The one limit: Go can only take the address of something that is **addressable** — a variable, a slice element, a field of an addressable struct. You cannot call a pointer method on a map element (`m["k"].Scale(2)`) or on the result of a function call (`newVertex().Scale(2)`), because those values have no stable address.

### 4.7 Choosing a value or pointer receiver

Use a **pointer receiver** when:

1. The method needs to modify the receiver.
2. The type is large, so copying it on every call is wasteful.
3. The type contains something that must not be copied, such as a `sync.Mutex`.

Use a **value receiver** for small, immutable-feeling types: `time.Time`, a `Point`, a `Celsius` temperature.

And importantly: **don’t mix**. If any method of a type needs a pointer receiver, give all its methods pointer receivers. Mixing causes confusion about which values satisfy which interfaces (4.8).

### 4.8 Interfaces

An **interface type** is defined as a set of method signatures. A value of interface type can hold **any** value that implements those methods.

```go
type Abser interface {
	Abs() float64
}

func main() {
	var a Abser
	f := MyFloat(-math.Sqrt2)
	v := Vertex{3, 4}

	a = f  // MyFloat implements Abser
	a = &v // *Vertex implements Abser

	// a = v // error: Vertex does not implement Abser
	//        //        (method Abs has pointer receiver)
	fmt.Println(a.Abs())
}
```

That compile error deserves explanation. Assume `Abs` is declared on `*Vertex`. The **method set** of `*Vertex` includes methods with both pointer and value receivers; the method set of `Vertex` includes only value-receiver methods. Why the asymmetry? An interface holds a *copy* of the value you put in it, and that copy is not addressable — so Go cannot take its address to call a pointer method. Calling it would also silently modify a copy instead of your variable.

### 4.9 Interfaces are implemented implicitly

There is no `implements` keyword. A type implements an interface simply by having all its methods. This is sometimes called **structural typing** or “duck typing checked at compile time”.

```go
type I interface {
	M()
}

type T struct {
	S string
}

// This method means type T implements the interface I,
// but we don't need to declare that explicitly.
func (t T) M() {
	fmt.Println(t.S)
}
```

The consequences are significant:

- **Decoupling.** A package can define an interface describing just what it needs, and existing types from other packages satisfy it without knowing it exists. Your own `type Saver interface { Save([]byte) error }` is automatically satisfied by any type with that method.
- **Small interfaces.** Go favours one- or two-method interfaces (`io.Reader`, `io.Writer`, `fmt.Stringer`, `error`). The proverb: *the bigger the interface, the weaker the abstraction*.
- **Define interfaces where they are used**, in the consuming package, not alongside the implementation. Functions should generally *accept interfaces and return concrete types*.

To assert at compile time that a type satisfies an interface, a common trick is `var _ I = T{}` — compilation fails if `T` stops implementing `I`.

### 4.10 Interface values

Under the hood an interface value is a pair: **(concrete type, value)**. Calling a method on the interface calls the method of the same name on the stored concrete value.

```go
type F float64

func (f F) M() { fmt.Println(f) }

func main() {
	var i I

	i = &T{"Hello"}
	describe(i) // (&{Hello}, *main.T)
	i.M()       // Hello

	i = F(math.Pi)
	describe(i) // (3.141592653589793, main.F)
	i.M()       // 3.141592653589793
}

func describe(i I) {
	fmt.Printf("(%v, %T)\n", i, i)
}
```

The same variable `i` holds different concrete types at different times, and the call `i.M()` dispatches to the right method. This **dynamic dispatch** is Go’s form of polymorphism.

### 4.11 Interface values with nil underlying values

If the concrete value inside an interface is a nil pointer, the method is still called — with a nil receiver. That is not an error in Go, and methods can handle it gracefully:

```go
func (t *T) M() {
	if t == nil {
		fmt.Println("<nil>")
		return
	}
	fmt.Println(t.S)
}

var t *T
var i I = t
describe(i) // (<nil>, *main.T)
i.M()       // <nil>
```

**The famous gotcha.** An interface holding a nil pointer is **itself not nil**, because its type half is set. This bites people with errors:

```go
func find() error {
	var p *MyError = nil
	if ok {
		return nil
	}
	return p // returns a non-nil error containing a nil *MyError!
}

if err := find(); err != nil {
	// this branch runs even though p was nil
}
```

The fix: return the literal `nil` for “no error”, and never return a typed nil pointer through an interface.

### 4.12 Nil interface values

A nil interface value has **neither type nor value**. Calling a method on it is a runtime error, because there is no concrete type to tell Go which method to call:

```go
var i I
describe(i) // (<nil>, <nil>)
i.M()       // panic: runtime error: invalid memory address or nil pointer dereference
```

So there are two distinct “nil” states: interface is nil (both halves empty), and interface holds a nil pointer (type set, value nil). `i == nil` is true only for the first.

### 4.13 The empty interface

The interface with no methods, `interface{}`, is satisfied by every type. Since Go 1.18 it has a predeclared alias, **`any`**, which is what modern code writes.

```go
var i any
describe(i) // (<nil>, <nil>)

i = 42
describe(i) // (42, int)

i = "hello"
describe(i) // (hello, string)

func describe(i any) {
	fmt.Printf("(%v, %T)\n", i, i)
}
```

`any` is used by code that must handle values of unknown type: `fmt.Println(a ...any)`, JSON decoding into `map[string]any`, and containers written before generics existed. Use it sparingly — every `any` gives up compile-time type checking. If you know the set of types, generics (Part 5) or a specific interface are usually better.

### 4.14 Type assertions

A **type assertion** extracts the concrete value from an interface:

```go
t := i.(T)
```

This asserts that `i` holds a `T` and gives you that `T`. If `i` holds something else, it **panics**. To test safely, use the comma-ok form, just like map lookups:

```go
var i any = "hello"

s := i.(string)
fmt.Println(s) // hello

s, ok := i.(string)
fmt.Println(s, ok) // hello true

f, ok := i.(float64)
fmt.Println(f, ok) // 0 false (no panic)

f = i.(float64) // panic: interface conversion: interface {} is string, not float64
```

`T` can also be an interface type. Then the assertion checks whether the concrete value *also* implements that interface — a common way to discover optional capabilities, e.g. `if sc, ok := w.(io.StringWriter); ok { sc.WriteString(s) }`.

### 4.15 Type switches

A **type switch** runs several type assertions in sequence. The cases name types, and in each case the variable has that type:

```go
func do(i any) {
	switch v := i.(type) {
	case int:
		fmt.Printf("Twice %v is %v\n", v, v*2) // v is an int
	case string:
		fmt.Printf("%q is %v bytes long\n", v, len(v)) // v is a string
	default:
		fmt.Printf("I don't know about type %T!\n", v) // v has i's type
	}
}

func main() {
	do(21)
	do("hello")
	do(true)
}
```

The special syntax `i.(type)` is only valid inside a switch. In a case listing several types (`case int, int64:`), and in `default`, `v` keeps the interface type because no single concrete type applies. A `case nil:` matches a nil interface.

### 4.16 Stringers

One of the most common interfaces is `fmt.Stringer`:

```go
type Stringer interface {
	String() string
}
```

A type with a `String` method controls how the `fmt` package (and many others) print it:

```go
type Person struct {
	Name string
	Age  int
}

func (p Person) String() string {
	return fmt.Sprintf("%v (%v years)", p.Name, p.Age)
}

func main() {
	a := Person{"Arthur Dent", 42}
	z := Person{"Zaphod Beeblebrox", 9001}
	fmt.Println(a, z) // Arthur Dent (42 years) Zaphod Beeblebrox (9001 years)
}
```

This is implicit interfaces at work: `fmt` checks at runtime whether its argument implements `Stringer` (with a type assertion) and uses it if so. A classic trap: calling `fmt.Sprintf("%v", p)` *inside* `p.String()` recurses forever, because `%v` calls `String` again. Print the fields, not the receiver.

### 4.17 Exercise: Stringers

**Task.** Make `IPAddr` (a `[4]byte`) print as a dotted quad, so `IPAddr{1, 2, 3, 4}` prints as `1.2.3.4`.

Try it in the Tour: [Exercise: Stringers](https://go.dev/tour/methods/18)

### 4.18 Errors

Go programs express errors as **values** of the built-in interface type:

```go
type error interface {
	Error() string
}
```

Functions return an `error` as their last result; callers check whether it is `nil`. `nil` means success.

```go
i, err := strconv.Atoi("42")
if err != nil {
	fmt.Printf("couldn't convert number: %v\n", err)
	return
}
fmt.Println("Converted integer:", i)
```

Custom error types carry structured information:

```go
type MyError struct {
	When time.Time
	What string
}

func (e *MyError) Error() string {
	return fmt.Sprintf("at %v, %s", e.When, e.What)
}

func run() error {
	return &MyError{time.Now(), "it didn't work"}
}
```

Why values rather than exceptions? Error handling becomes ordinary control flow: visible at the call site, impossible to forget silently, and you can program with errors like any other value. The everyday tools:

- `errors.New("msg")` and `fmt.Errorf("...%v", x)` create simple errors.
- `fmt.Errorf("reading config: %w", err)` **wraps** an error, adding context while keeping the original.
- `errors.Is(err, fs.ErrNotExist)` checks whether any error in the chain equals a sentinel value.
- `errors.As(err, &target)` finds the first error in the chain of a given type and extracts it.

Reserve `panic` for truly unrecoverable programmer errors (impossible states, failed invariants); ordinary failures like a missing file are errors.

### 4.19 Exercise: Errors

**Task.** Change your `Sqrt` from 2.8 to return `(float64, error)`. For negative input, return a non-nil error of a new type `ErrNegativeSqrt float64`, whose message is `cannot Sqrt negative number: -2`.

Try it in the Tour: [Exercise: Errors](https://go.dev/tour/methods/20)

### 4.20 Readers

The `io` package defines `io.Reader`, the interface for reading a stream of data. Files, network connections, compressors, ciphers, HTTP bodies and many more implement it.

```go
type Reader interface {
	Read(b []byte) (n int, err error)
}
```

`Read` fills the caller’s byte slice with up to `len(b)` bytes and returns how many it wrote. When the stream ends, it returns the error `io.EOF`.

```go
r := strings.NewReader("Hello, Reader!")

b := make([]byte, 8)
for {
	n, err := r.Read(b)
	fmt.Printf("n = %v err = %v b = %v\n", n, err, b)
	fmt.Printf("b[:n] = %q\n", b[:n])
	if err == io.EOF {
		break
	}
}
```

Rules of the contract that matter in practice:

- Always process the `n` bytes returned **before** looking at the error — a reader may return data and `io.EOF` in the same call.
- `Read` may return fewer bytes than requested even when more are coming. Use `io.ReadFull` or `io.ReadAll` when you need a specific amount or everything.
- The caller owns the buffer, which lets one buffer be reused across calls and keeps allocation low.

The real power is **composition**: readers wrap other readers. `gzip.NewReader(file)` decompresses a file, `bufio.NewReader(conn)` buffers a network stream, and `io.Copy(dst, src)` connects any reader to any writer.

### 4.21 Exercise: Readers

**Task.** Implement a `Reader` type that emits an infinite stream of the ASCII character `'A'`.

Try it in the Tour: [Exercise: Readers](https://go.dev/tour/methods/22)

### 4.22 Exercise: rot13Reader

**Task.** Implement `rot13Reader`, which wraps another `io.Reader` and applies the [ROT13](https://en.wikipedia.org/wiki/ROT13) substitution cipher to every letter it reads.

Try it in the Tour: [Exercise: rot13Reader](https://go.dev/tour/methods/23)

### 4.23 Images

The `image` package defines the `Image` interface:

```go
package image

type Image interface {
	ColorModel() color.Model
	Bounds() Rectangle
	At(x, y int) color.Color
}
```

`Bounds` returns an `image.Rectangle` such as `image.Rect(0, 0, w, h)`; `At` returns the colour of one pixel; `ColorModel` describes the colour space. `color.Color` and `color.Model` are interfaces too, with ready-made implementations such as `color.RGBA` and `color.RGBAModel`.

```go
m := image.NewRGBA(image.Rect(0, 0, 100, 100))
fmt.Println(m.Bounds())         // (0,0)-(100,100)
fmt.Println(m.At(0, 0).RGBA())  // 0 0 0 0
```

Because it is an interface, anything that can answer those three questions is an image — including a value computed from a formula, with no pixel buffer at all. Encoders such as `png.Encode(w, img)` work with any `Image`.

### 4.24 Exercise: Images

**Task.** Go back to your `Pic` generator from 3.18. This time, write a type `Image` implementing `image.Image` and return it instead of a slice of data.

Try it in the Tour: [Exercise: Images](https://go.dev/tour/methods/25)

## Part 5 — Generics

Generics arrived in Go 1.18 (2022). They let you write a function or type once and use it with many types, while keeping full compile-time type checking — something `any` cannot give you.

### 5.1 Type parameters

A generic function declares **type parameters** in square brackets before its ordinary parameters. Each type parameter has a **constraint**, which says what the type must support.

```go
// Index returns the index of x in s, or -1 if not found.
func Index[T comparable](s []T, x T) int {
	for i, v := range s {
		// v and x are type T, which has the comparable
		// constraint, so we can use == here.
		if v == x {
			return i
		}
	}
	return -1
}

func main() {
	si := []int{10, 20, 15, -10}
	fmt.Println(Index(si, 15)) // 2

	ss := []string{"foo", "bar", "baz"}
	fmt.Println(Index(ss, "hello")) // -1
}
```

Read `[T comparable]` as “for any type `T` that supports `==` and `!=`”. A few points:

- **Type inference.** You usually don’t write the type argument: `Index(si, 15)` infers `T = int` from the arguments. You can be explicit with `Index[int](si, 15)`, which is necessary when nothing in the arguments determines `T`.
- **The constraint is a contract in both directions.** Inside `Index` you may only use operations the constraint allows. Without `comparable`, `v == x` would not compile, because `T` might be a slice or a map.
- **Constraints are interfaces.** `comparable` is a predeclared one; `any` allows every type but only operations that all types support (assignment, passing around).
- **`comparable` does not rule out a runtime panic.** Since Go 1.20, interface types such as `any` satisfy `comparable`, so `Index[any]` compiles. Comparing two interface values that hold the same non-comparable dynamic type panics, just like the map key in 3.19: `Index([]any{[]int{1}}, any([]int{1}))` fails with `runtime error: comparing uncomparable type []int`. If the dynamic types differ, `==` just returns `false`.

Constraints can also list **types**, using a union, which unlocks operators like `<` and `+`:

```go
type Number interface {
	~int | ~int64 | ~float64
}

func Sum[T Number](xs []T) T {
	var total T
	for _, x := range xs {
		total += x
	}
	return total
}
```

The `~int` means “`int` or any type whose underlying type is `int`”, so a user-defined `type Celsius int` qualifies too. The standard package `cmp` provides `cmp.Ordered` (all types supporting `<`), and the `slices` and `maps` packages are full of generic helpers: `slices.Index`, `slices.Sort`, `slices.Contains`, `maps.Keys`. Before writing your own generic utility, check those packages.

### 5.2 Generic types

Types can have type parameters too, which is how you write type-safe containers:

```go
// List represents a singly-linked list that holds
// values of any type.
type List[T any] struct {
	next *List[T]
	val  T
}
```

**Exercise.** Add some functionality to this `List` type, for example a method that pushes a value onto the front and one that returns all values in order.

Try it in the Tour: [Generic types](https://go.dev/tour/generics/2)

Things to keep in mind when writing generic types:

- Inside the type, you refer to it with its parameter: `*List[T]`. Methods use a receiver like `(l *List[T])` and may use `T` freely.
- **Methods cannot declare their own type parameters.** A method like `func (l *List[T]) Map[U any](...)` is not allowed; write a top-level function `Map[T, U any](l *List[T], f func(T) U) *List[U]` instead.
- Aim for a useful zero value (1.11), so that `var l List[string]` works without a constructor.
- Unlike a list built on `any`, a `List[string]` refuses an `int` at compile time and needs no type assertions when you read values back.

**When to use generics.** Reach for them when you would otherwise write the same code for several types (containers, algorithms over slices and maps, channels utilities). Don’t use them just to avoid an interface: if the code only calls methods on a value, an ordinary interface parameter is simpler. The Go blog’s [When To Use Generics](https://go.dev/blog/when-generics) has good rules of thumb.[^5]

Since Go 1.23, generic types pair nicely with **range-over-func iterators**: a method returning `iter.Seq[T]` lets callers write `for v := range l.Values() { ... }` without allocating a slice.

## Part 6 — Concurrency

Concurrency is built into Go’s core: goroutines to run things at the same time, channels to communicate between them, and `select` to wait on several channels. The guiding proverb is: **don’t communicate by sharing memory; share memory by communicating.**

A note on words: *concurrency* is structuring a program as independently executing tasks; *parallelism* is actually running them simultaneously on multiple CPU cores. Go gives you the first, and the runtime provides the second when cores are available.

### 6.1 Goroutines

A **goroutine** is a function running concurrently with other goroutines, managed by the Go runtime. Start one by putting `go` before a function call:

```go
func say(s string) {
	for i := 0; i < 5; i++ {
		time.Sleep(100 * time.Millisecond)
		fmt.Println(s)
	}
}

func main() {
	go say("world")
	say("hello")
}
```

`hello` and `world` are printed interleaved, in no guaranteed order. Important details:

- The function’s **arguments are evaluated in the calling goroutine**, right away; only the execution happens concurrently (like `defer`).
- Goroutines are **cheap**: each starts with a few kilobytes of stack that grows as needed, and the runtime multiplexes them onto a small number of OS threads. Running 100,000 goroutines is normal.
- **When `main` returns, the program exits**, killing all other goroutines without waiting. Remove the `say("hello")` line above and nothing is printed. You need some form of synchronization (channels, `sync.WaitGroup`) to wait for work to finish.
- Goroutines share the same address space, so access to shared memory must be synchronized. The [race detector](https://go.dev/doc/articles/race_detector) (`go run -race`, `go test -race`) finds unsynchronized access; use it routinely.[^6]

### 6.2 Channels

A **channel** is a typed conduit through which goroutines send and receive values, using the arrow operator `<-`:

```go
ch := make(chan int) // channels must be created before use
ch <- v              // send v to channel ch
v := <-ch            // receive from ch and assign to v
```

The data flows in the direction of the arrow. By default, sends and receives **block until the other side is ready**: an unbuffered channel is a rendezvous where the sender and receiver meet. This lets goroutines synchronize without explicit locks.

```go
func sum(s []int, c chan int) {
	sum := 0
	for _, v := range s {
		sum += v
	}
	c <- sum // send sum to c
}

func main() {
	s := []int{7, 2, 8, -9, 4, 0}

	c := make(chan int)
	go sum(s[:len(s)/2], c)
	go sum(s[len(s)/2:], c)
	x, y := <-c, <-c // receive from c

	fmt.Println(x, y, x+y) // e.g. -5 17 12 (order varies)
}
```

The work is split across two goroutines, and the two receives in `main` both wait for a result, so `main` cannot proceed until both halves are done. Which half finishes first is not determined, so `x` and `y` may swap.

Channel types can be **directional**, which documents and enforces intent in function signatures: `chan<- int` can only send, `<-chan int` can only receive. A bidirectional channel converts to either automatically.

### 6.3 Buffered channels

Give `make` a second argument to create a **buffered** channel:

```go
ch := make(chan int, 2)
ch <- 1
ch <- 2
fmt.Println(<-ch) // 1
fmt.Println(<-ch) // 2
```

Sends to a buffered channel block only when the **buffer is full**; receives block only when it is **empty**. Add a third `ch <- 3` before the receives and the program deadlocks — and the runtime reports `fatal error: all goroutines are asleep - deadlock!` because no other goroutine could ever receive. Section 6.10 explains this error in detail.

Buffers decouple producer and consumer speed, and a buffered channel of size N can act as a semaphore limiting concurrency to N. But don’t add buffers to “fix” a deadlock or race; it usually just hides the problem until load increases. Unbuffered is the right default.

**Printing the values in a buffered channel.** You can’t look inside a channel without taking values out: there is no peek or index operation, and `fmt.Println(ch)` only prints the channel’s address (something like `0xc000020060`). The only way to see the values is to **receive** them, which removes them from the channel.

If no more values will be sent, close the channel and `range` over it. A closed channel still delivers what is left in its buffer, and `range` stops once the buffer is empty (6.4):

```go
ch := make(chan int, 3)
ch <- 1
ch <- 2
ch <- 3
close(ch)

for v := range ch {
	fmt.Println(v) // 1, 2, 3
}
```

**Common mistake: forgetting `close`.** Leave out `close(ch)` and the loop prints all ten values, then crashes:

```go
ch := make(chan int, 10)
for i := 0; i < 10; i++ {
	ch <- i
}
for v := range ch { // never closed
	fmt.Println(v)
}
// 0 … 9, then:
// fatal error: all goroutines are asleep - deadlock!
```

`range` doesn’t stop when the buffer is empty; it stops when the channel is **closed**. After the tenth value it tries to receive an eleventh. The buffer is empty and the channel is still open, so it waits for another send. But `main` is the only goroutine, and it is stuck in that receive, so nothing can ever send. Every goroutine is blocked, and the runtime aborts (6.10). The stack trace shows `goroutine 1 [chan receive]`: `main` is waiting for a value. Closing doesn’t throw away buffered values, so adding `close(ch)` after the last send fixes it.

If the channel stays open, receive `len(ch)` values. `len(ch)` is the number of values currently buffered, and `cap(ch)` is the buffer size:

```go
n := len(ch)
for i := 0; i < n; i++ {
	fmt.Println(<-ch)
}
```

Save `len(ch)` before the loop. Writing `i < len(ch)` in the condition would re-check a shrinking length and print only about half the values.

If other goroutines might also be receiving, drain without blocking using `select` with a `default` case (6.6). With other goroutines involved, `len(ch)` is only a snapshot that may already be stale when you use it:

```go
for {
	select {
	case v, ok := <-ch:
		if !ok {
			return // closed and drained
		}
		fmt.Println(v)
	default:
		return // buffer empty right now
	}
}
```

Don’t leave out the `ok` check. Once a channel is closed and empty, a receive never blocks; it immediately returns the zero value. The receive case is then always ready, `default` never runs, and the loop prints `0` forever.

If you need the values to stay in the channel, you have to send them back after printing, and only when nothing else is using the channel. Usually that means a channel is the wrong tool: collect the values into a slice as you receive them, and print the slice.

### 6.4 Range and close

The sending side can **close** a channel to say that no more values will come. A receiver can test for this with the comma-ok form:

```go
v, ok := <-ch // ok is false if ch is closed and drained
```

The loop `for i := range c` receives values until the channel is closed:

```go
func fibonacci(n int, c chan int) {
	x, y := 0, 1
	for i := 0; i < n; i++ {
		c <- x
		x, y = y, x+y
	}
	close(c)
}

func main() {
	c := make(chan int, 10)
	go fibonacci(cap(c), c)
	for i := range c {
		fmt.Println(i)
	}
}
```

The rules:

- **Whoever is responsible for the channel’s completion closes it, exactly once, after all sends are finished.** With one sender, that is the sender itself. With several senders, a coordinator waits for all of them and then closes, as in the `WaitGroup` example below. What matters is ownership, not which end of the channel you are on: a receiver may close only if it knows every sender has finished. Closing while a sender might still send is a bug, because sending on a closed channel panics (and so does closing an already closed channel). A receiver that stops reading early should signal cancellation instead, so the senders can finish (6.5). The goroutine responsible for completion can then close the data channel.
- Receiving from a closed channel never blocks: it returns remaining buffered values, then zero values with `ok == false`.
- Closing is **not required** like closing a file. Close a channel only when the receiver must be told there is nothing more — for example to end a `range` loop. Unreferenced channels are garbage collected either way.
- Closing is a **broadcast**: every goroutine waiting on the channel wakes up. That makes `close(done)` a common way to tell many workers to stop (the `context` package builds on this idea).

**What if the channel is never closed?** Then the `range` loop never ends. After the last value has been received, it blocks waiting for one that will never come. What happens next depends on the rest of the program.

- **If every goroutine is now blocked, the runtime aborts.** Remove `close(c)` from the example above. `main` prints ten numbers and then waits for an eleventh, but `fibonacci` has already returned, so nothing can ever send. The program crashes with `fatal error: all goroutines are asleep - deadlock!`.
- **If anything else is still running, nothing is detected.** The deadlock check only fires when *all* goroutines are stuck. In a real program a server, a ticker or a network call is usually still alive. If the stuck loop is in `main`, the program then hangs silently. If it is in another goroutine, that goroutine is **leaked**: it stays blocked for the rest of the program, along with everything it references. This is one of the most common goroutine leaks in Go.

To avoid this, close the channel once all sends are done. With one sender, the sender closes it when it is finished. With several senders, none of them knows it is the last one, so a coordinator counts them with a `sync.WaitGroup` and closes from a separate goroutine once they have all finished:

```go
var wg sync.WaitGroup
for _, job := range jobs {
	wg.Add(1)
	go func() {
		defer wg.Done()
		results <- process(job)
	}()
}
go func() {
	wg.Wait()
	close(results) // safe: all senders have finished
}()
for r := range results {
	fmt.Println(r)
}
```

If you know how many values are coming, receive exactly that many instead of using `range`. If the receiver must be able to give up early, use `select` with a cancellation channel or `ctx.Done()` (6.5), so the loop can exit even when the sender never closes.

**Closing a channel filled by a recursive function.** Close it **once, after the whole recursion has finished**, which means outside the recursive function. The usual pattern is a non-recursive wrapper: the recursive helper only sends, and the wrapper calls it once and then closes.

Say you want to stream every file name from a nested folder structure:

```go
type Folder struct {
	Name    string
	Files   []string
	Folders []*Folder
}

// collect only sends. It never closes.
// dir is the path of f's parent, ending in "/" (or "" for the root).
func collect(f *Folder, dir string, ch chan<- string) {
	dir += f.Name + "/"
	for _, name := range f.Files {
		ch <- dir + name
	}
	for _, sub := range f.Folders {
		collect(sub, dir, ch)
	}
}

// AllFiles owns the channel's lifetime: one call, then close.
func AllFiles(root *Folder) <-chan string {
	ch := make(chan string)
	go func() {
		defer close(ch)
		collect(root, "", ch)
	}()
	return ch
}
```

Each call passes its accumulated path down, so a file `notes.txt` in `root/docs` comes out as `root/docs/notes.txt`, and two `docs` folders under different parents can’t produce the same name. The caller just ranges over the result:

```go
for path := range AllFiles(root) {
	fmt.Println(path)
}
```

Why not close inside the recursive function? Every recursive call would reach the `close`. The first call to finish, usually the deepest subfolder, closes the channel while its callers still have files to send. The next send then panics with `send on closed channel`. Even if no one sent again, the second `close` would panic with `close of closed channel`. A function only knows it is “the last one” when it is the outermost call, and the wrapper makes that explicit. Using `defer close(ch)` in the wrapper also means the channel gets closed if the recursion returns early, so the caller’s `range` still ends. (A panic is different: an unrecovered panic in that goroutine crashes the whole program anyway.)

If the recursion starts goroutines, for example to scan subfolders in parallel, the outermost call returns before the work is done, so nobody can close at the end of a call. Count the goroutines with a `sync.WaitGroup` instead, and close from a separate goroutine once the count reaches zero:

```go
func collectParallel(f *Folder, dir string, ch chan<- string, wg *sync.WaitGroup) {
	defer wg.Done()
	dir += f.Name + "/"
	for _, name := range f.Files {
		ch <- dir + name
	}
	for _, sub := range f.Folders {
		wg.Add(1) // before starting the goroutine
		go collectParallel(sub, dir, ch, wg)
	}
}

func AllFilesParallel(root *Folder) <-chan string {
	ch := make(chan string)
	var wg sync.WaitGroup
	wg.Add(1)
	go collectParallel(root, "", ch, &wg)
	go func() {
		wg.Wait() // every recursive goroutine has finished
		close(ch)
	}()
	return ch
}
```

Call `wg.Add(1)` before each `go`, not inside the new goroutine. Otherwise `Wait` could see a count of zero and close the channel before that goroutine has registered. In the parallel version, files also arrive in no fixed order.

The rule is the same in both cases. Exactly one place closes the channel, and only after every possible sender is finished. The recursive code itself never closes.

Both versions also expect the caller to **read the channel to the end**. The channel is unbuffered, so every send waits for a receiver. If a caller `break`s out of its `range` loop after finding the file it wanted, the next send blocks forever. The sending goroutine leaks, and in the parallel version so do the goroutine waiting in `wg.Wait()` and every recursive goroutine still trying to send. Either document “callers must drain the channel”, or, if stopping early should be allowed, pass a `context.Context` and make every send a `select` that also gives up on `ctx.Done()` (6.5). The caller then cancels the context when it stops reading.

### 6.5 Select

`select` lets a goroutine wait on **several channel operations** at once. It blocks until one of its cases can run, then runs that case. If several are ready at the same moment, it picks one by **uniform pseudo-random selection**. That keeps a `select` from always favoring the first case listed, but it guarantees nothing more: a ready case can lose several times in a row, and Go sets no limit on how long it may wait.

```go
func fibonacci(c, quit chan int) {
	x, y := 0, 1
	for {
		select {
		case c <- x:
			x, y = y, x+y
		case <-quit:
			fmt.Println("quit")
			return
		}
	}
}

func main() {
	c := make(chan int)
	quit := make(chan int)
	go func() {
		for i := 0; i < 10; i++ {
			fmt.Println(<-c)
		}
		quit <- 0
	}()
	fibonacci(c, quit)
}
```

Here the producer keeps offering the next number on `c` until someone signals on `quit`. Cases can be sends or receives. Two useful patterns:

```go
// Timeout
select {
case res := <-results:
	fmt.Println(res)
case <-time.After(2 * time.Second):
	fmt.Println("timed out")
}

// Cancellation with a context
select {
case job := <-jobs:
	process(job)
case <-ctx.Done():
	return ctx.Err()
}
```

A receive from a `nil` channel blocks forever, so setting a channel variable to `nil` is a neat way to disable its case inside a `select` loop.

### 6.6 Default selection

A `default` case runs if no other case is ready, making the `select` **non-blocking**:

```go
select {
case i := <-c:
	// use i
default:
	// receiving from c would block
}
```

The Tour’s example combines tickers and a default branch:

```go
func main() {
	tick := time.Tick(100 * time.Millisecond)
	boom := time.After(500 * time.Millisecond)
	for {
		select {
		case <-tick:
			fmt.Println("tick.")
		case <-boom:
			fmt.Println("BOOM!")
			return
		default:
			fmt.Println("    .")
			time.Sleep(50 * time.Millisecond)
		}
	}
}
```

Be careful with `default` inside a loop: without the `Sleep`, the loop would spin and burn 100% of a CPU core. Non-blocking sends are also handy for dropping a message when a consumer is too slow instead of stalling.

### 6.7 Exercise: Equivalent binary trees

Many different binary search trees can store the same sequence of values. The Tour asks you to check whether two trees store the same values, using the package `golang.org/x/tour/tree`:

```go
type Tree struct {
	Left  *Tree
	Value int
	Right *Tree
}
```

`tree.New(k)` builds a randomly shaped tree holding the values k, 2k, …, 10k.

**Task.** (1) Implement `Walk(t *Tree, ch chan int)`, which sends all values of the tree to `ch` in sorted order. (2) Implement `Same(t1, t2 *Tree) bool` using `Walk`.

Two things to get right, both from 6.4:

- **Say how a walk ends.** The receiver has to know when a tree has no more values, so the producer closes the channel once the *complete* walk has returned. `Walk` is naturally recursive, so do it the same way as `AllFiles`: a recursive helper that only sends, and a wrapper that calls it and then closes the channel.
- **Don’t return in the middle.** If `Same` returns at the first mismatch, one or both walkers are still blocked trying to send their next value, and those goroutines leak. For this exercise the simple fix is to remember the mismatch, keep receiving until both channels are closed, and only then return. The alternative is a cancellation path: a `done` channel or `context.Context` that the walkers check with `select` on every send.

Try it in the Tour: [Exercise: Equivalent Binary Trees](https://go.dev/tour/concurrency/8)

### 6.8 sync.Mutex

Channels are great for communication. But sometimes goroutines just need to take turns accessing a shared variable. That is **mutual exclusion**, provided by `sync.Mutex` with its two methods `Lock` and `Unlock`.

```go
// SafeCounter is safe to use concurrently.
type SafeCounter struct {
	mu sync.Mutex
	v  map[string]int
}

// Inc increments the counter for the given key.
func (c *SafeCounter) Inc(key string) {
	c.mu.Lock()
	// Lock so only one goroutine at a time can access the map c.v.
	c.v[key]++
	c.mu.Unlock()
}

// Value returns the current value of the counter for the given key.
func (c *SafeCounter) Value(key string) int {
	c.mu.Lock()
	// Lock so only one goroutine at a time can access the map c.v.
	defer c.mu.Unlock()
	return c.v[key]
}

func main() {
	c := SafeCounter{v: make(map[string]int)}
	for i := 0; i < 1000; i++ {
		go c.Inc("somekey")
	}
	time.Sleep(time.Second)
	fmt.Println(c.Value("somekey")) // 1000
}
```

Without the mutex, concurrent writes to the map would trigger the runtime error `fatal error: concurrent map writes` or silently lose increments. Notes:

- `defer c.mu.Unlock()` (as in `Value`) guarantees the unlock even if the function returns early or panics.
- The methods have **pointer receivers** and the mutex is a value field. Copying a `SafeCounter` would copy its lock, producing two independent locks guarding one map; `go vet` warns about this.
- `time.Sleep(time.Second)` is only for the demo. Real code waits with a `sync.WaitGroup` (see the next exercise).
- Related tools: `sync.RWMutex` allows many simultaneous readers; `sync/atomic` handles a single counter even more cheaply; `sync.Once` runs initialization exactly once.

**Channels or mutexes?** Use channels when passing ownership of data, distributing work, or signalling events. Use a mutex when protecting a piece of state (a cache, a counter, a map). Both are idiomatic.

### 6.9 Exercise: Web crawler

**Task.** The Tour provides a sequential `Crawl` function and a fake `Fetcher` (keep its `fakeFetcher` data as given in the Tour). Modify `Crawl` to fetch URLs **in parallel**, without fetching the same URL twice.

Try it in the Tour: [Exercise: Web Crawler](https://go.dev/tour/concurrency/10)

### 6.10 Understanding “all goroutines are asleep - deadlock!”

Sooner or later every Go programmer sees this message:

```text
fatal error: all goroutines are asleep - deadlock!
```

It means the runtime found that **every goroutine in your program is blocked, waiting for something no other goroutine can ever do**. Since nothing can wake any of them, the program can never make progress, so the runtime stops it instead of letting it hang forever.

“Asleep” here means “blocked”: waiting to send or receive on a channel, waiting on a mutex or `WaitGroup`, or sitting in a `select` with no ready case.

**Common causes**

1. **Sending on an unbuffered channel with no receiver.** A send waits until someone receives, and here nobody ever will:

```go
ch := make(chan int)
ch <- 1 // blocks forever: main is the only goroutine
```

2. **Receiving when nobody will send.** This includes `range` over a channel that is never closed (6.4).
3. **A full buffered channel.** With `make(chan int, 2)`, the third send blocks if nobody is receiving (6.3).
4. **A `WaitGroup` count that never reaches zero.** You called `Add` more often than `Done`, for example because a goroutine returned early without a deferred `Done`.
5. **Locking a mutex twice in the same goroutine.** Go’s `sync.Mutex` is not reentrant, so a second `Lock` waits for an unlock that will never happen.
6. **`select {}` or a receive from a `nil` channel.** Both block forever by design.

**Reading the output**

After the message, the runtime prints a stack trace for every goroutine, with the reason each one is blocked in square brackets:

```text
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [chan send]:
main.main()
	/tmp/main.go:7 +0x31
```

The bracketed reason (`chan send`, `chan receive`, `sync.WaitGroup.Wait`, `sync.Mutex.Lock`, `select`) and the file and line number show you exactly where each goroutine got stuck. Start there and ask: *who was supposed to do the matching send, receive, `Done` or `Unlock`, and why didn’t they?*

**A caveat: no error doesn’t mean no deadlock.** The check only fires when *all* goroutines are blocked. If even one goroutine is still alive (a web server, a ticker, a network read), a deadlock among the others goes unreported. The program just hangs, or quietly leaks goroutines. In those cases, `go test -timeout` or a goroutine dump (send `SIGQUIT` with Ctrl+\\ on Unix, or use `net/http/pprof`) shows you where everything is blocked.

## Where to go next

You now know the whole language the Tour covers. The next step is writing real programs, and these resources pick up where this guide stops:

- [How to Write Go Code](https://go.dev/doc/code) — modules, packages, `go test`, and how a real project is laid out.
- [Effective Go](https://go.dev/doc/effective_go) — the classic guide to idiomatic style. Read it after you have written a few hundred lines.
- [Go by Example](https://gobyexample.com/) — short, runnable examples of everyday tasks (JSON, files, HTTP, timers).
- [The standard library docs](https://pkg.go.dev/std) — especially `fmt`, `strings`, `io`, `os`, `errors`, `net/http`, `encoding/json`, `context`, `slices` and `maps`.
- [The Go blog](https://go.dev/blog/) — deep dives, including the posts linked throughout this guide.
- [The Go Language Specification](https://go.dev/ref/spec) — short and readable; the final word on any question about how the language behaves.

**Suggested first projects:** a command-line tool that reads a file and prints statistics (uses `os`, `bufio`, maps and errors); a small JSON HTTP API with `net/http` (structs, tags, interfaces); and a concurrent link checker that reuses your web crawler with real HTTP requests, a timeout via `context`, and a limit on parallel fetches.

[^1]: [A Tour of Go](https://go.dev/tour/), The Go Authors. The official tour this guide extends; exercise tasks are paraphrased from it.
[^2]: [Go’s Declaration Syntax](https://go.dev/blog/declaration-syntax), Rob Pike, The Go Blog.
[^3]: [Defer, Panic, and Recover](https://go.dev/blog/defer-panic-and-recover), Andrew Gerrand, The Go Blog.
[^4]: [Go Slices: usage and internals](https://go.dev/blog/slices-intro), Andrew Gerrand, The Go Blog.
[^5]: [When To Use Generics](https://go.dev/blog/when-generics), Ian Lance Taylor, The Go Blog.
[^6]: [Data Race Detector](https://go.dev/doc/articles/race_detector), The Go Authors.
