---
title: "Mage: Make for Go, Written in Go"
date: 2026-11-21
draft: true
description: "A practical introduction to Mage, the Make-style build tool where targets are plain Go functions: installation, writing a magefile, dependencies, namespaces, the sh helpers, CI usage and how it compares to Make."
summary: "Mage replaces your Makefile with ordinary Go functions. No tabs, no shell quirks, works the same on Linux, macOS and Windows. Here is how to set it up, write targets with dependencies, and run it in CI."
tags: ["go", "mage", "make", "build tools", "ci/cd"]
categories: ["Programming"]
---

Almost every Go repository has a `Makefile`. It usually starts with `build`, `test` and `lint`, and a year later it has thirty targets, three levels of `$(shell ...)`, a `.PHONY` line nobody keeps up to date, and a comment saying "doesn't work on Windows, sorry".

Make is a fine tool, but it was designed in 1976 to rebuild C object files when their sources changed. Most Go projects use it as a task runner instead: a list of named shell scripts. For that job you pay for Make's quirks (tabs are syntax, every line runs in its own shell, variables expand in two different ways) without using its main feature, because `go build` already tracks what needs rebuilding.

**Mage** is a build tool that keeps the good part of Make (`mage build`, `mage test`) and replaces the language with Go. You write ordinary exported Go functions; Mage turns them into commands. This post covers Mage 1.17.2, the current release, which came out in April 2026.[^1]

## What Mage is

Mage reads Go files marked with a `mage` build tag, compiles them into a small binary, and runs the function you name on the command line.[^2] That's the whole idea:

```text
magefile.go  ──►  mage compiles a binary (cached)  ──►  runs target "Build"
```

Some properties fall out of that design:

- **It's just Go.** Your editor, `gopls`, the type checker and the debugger all work on your build scripts. Errors are Go errors, not a silently ignored exit code halfway through a recipe.
- **Cross-platform for free.** `os.RemoveAll` works on Windows; `rm -rf` doesn't.
- **No extra runtime.** Mage itself has no dependencies outside the Go standard library and builds with Go 1.18 or newer.[^2] If you can build your project, you can build your magefile.
- **Cached.** The compiled magefile binary is cached and only rebuilt when the magefile changes, so after the first run there's no compile delay.

## Installing

The usual way is `go install`:[^2]

```bash
go install github.com/magefile/mage@latest
mage -version
```

Pin a version instead of `@latest` if you want reproducible builds across your team, for example `@v1.17.2`. Mage is also on Homebrew (`brew install mage`), MacPorts, Scoop and asdf, and there are prebuilt binaries on the GitHub releases page.[^2]

Since Go 1.24 you can also track Mage as a tool dependency in your project's `go.mod`, so everyone gets the same version without installing anything globally:

```bash
go get -tool github.com/magefile/mage@v1.17.2
go tool mage build
```

### Zero install

If you'd rather not ask contributors to install anything at all, Mage can run through `go run`. Add a file with an `ignore` build tag (so it doesn't end up in the magefile compilation itself):[^3]

```go
//go:build ignore

package main

import (
	"os"

	"github.com/magefile/mage/mage"
)

func main() { os.Exit(mage.Main()) }
```

Then `go run mage.go build` behaves exactly like `mage build`. One caveat: `go run` only ever exits with 0 or 1, so if a target exits with some other status code, that number gets lost.[^3]

## Your first magefile

`mage -init` creates a starter template, but writing one from scratch is short enough. Create `magefile.go` in the root of your repository:

```go
//go:build mage

package main

import (
	"os"

	"github.com/magefile/mage/sh"
)

// Build compiles the binary into ./bin.
func Build() error {
	return sh.RunV("go", "build", "-o", "bin/app", "./cmd/app")
}

// Test runs the unit tests with the race detector.
func Test() error {
	return sh.RunV("go", "test", "-race", "./...")
}

// Clean removes build output.
func Clean() error {
	return os.RemoveAll("bin")
}
```

Three things make this a magefile:

1. The `//go:build mage` tag at the top. It keeps the file out of your normal `go build`, so the magefile's `package main` doesn't clash with your real code.
2. `package main`.
3. Exported functions with a supported signature. Each one becomes a target.

Now run it:

```bash
$ mage -l
Targets:
  build    compiles the binary into ./bin.
  clean    removes build output.
  test     runs the unit tests with the race detector.

$ mage build
$ mage test
```

Target names are case-insensitive, and the doc comment becomes the help text. The first sentence shows up in `mage -l`; the full comment shows up in `mage -h build`.[^2] As of 1.17.2, `mage -l` and `mage -h` no longer need to compile anything, so they're instant, and `mage -install <shell>` sets up tab completion for zsh, bash, fish or PowerShell.[^1]

A target that returns an error makes Mage print the error and exit with status 1.[^4] That's what makes it work properly in CI: no `set -e`, no forgotten `|| exit 1`.

### Keeping magefiles out of the way

In a larger repository you may not want build code in the root. If there's a `magefiles/` directory and no magefile in the current directory, Mage uses that directory automatically.[^2] Files in there don't even need the build tag, since they live in their own directory. Splitting targets across several files in `magefiles/` works the same as splitting any Go package.

## Target signatures and arguments

A target is an exported function that optionally takes a `context.Context` first, optionally takes further arguments, and returns either nothing or an `error`.[^4]

Positional arguments can be `string`, `int`, `float64`, `bool` or `time.Duration`:

```go
// Release tags and pushes the given version, e.g. mage release v1.4.0
func Release(version string) error {
	if err := sh.Run("git", "tag", version); err != nil {
		return err
	}
	return sh.Run("git", "push", "origin", version)
}
```

Since 1.16, pointer types become **optional flags**. They're `nil` when not given, and a `*bool` flag can be passed without a value to mean `true`:[^4]

```go
// Deploy deploys to the given environment.
func Deploy(env string,
	dryrun *bool, // only print what would be deployed
) error {
	if dryrun != nil && *dryrun {
		fmt.Println("would deploy to", env)
		return nil
	}
	return sh.RunV("./scripts/deploy.sh", env)
}
```

```bash
mage deploy staging -dryrun
```

The trailing comment on the parameter is used as the flag's help text.[^1]

## Dependencies

Make's killer feature is declaring that one target needs another. In Mage, you do that with `mg.Deps` as the first line of a function:[^5]

```go
import "github.com/magefile/mage/mg"

func Build() error {
	mg.Deps(Generate, Vendor)
	return sh.RunV("go", "build", "-o", "bin/app", "./cmd/app")
}

func Generate() error { return sh.Run("go", "generate", "./...") }
func Vendor() error   { return sh.Run("go", "mod", "vendor") }
```

The rules are worth knowing because they're what makes this better than calling functions directly:[^5]

- **Each dependency runs at most once per `mage` invocation**, no matter how many targets depend on it. If `Build` and `Test` both depend on `Generate`, and you run `mage build test`, `Generate` runs once.
- **`mg.Deps` runs its arguments in parallel**, in their own goroutines, as far as their own dependencies allow. `Generate` and `Vendor` above run concurrently.
- **`mg.SerialDeps` runs them one after another**, for when order matters. Each still runs only once.
- **`mg.CtxDeps` and `mg.SerialCtxDeps`** pass along a context you supply, so cancellation and timeouts propagate. The plain variants pass a default context that's cancelled when the `-t` timeout expires.

Dependencies don't have to be exported, so you can keep helper steps out of `mage -l`. For dependencies with arguments, wrap them in `mg.F`:

```go
func All() {
	mg.Deps(
		mg.F(compile, "linux", "amd64"),
		mg.F(compile, "darwin", "arm64"),
		mg.F(compile, "windows", "amd64"),
	)
}

func compile(goos, goarch string) error {
	out := fmt.Sprintf("bin/app-%s-%s", goos, goarch)
	if goos == "windows" {
		out += ".exe"
	}
	return sh.RunWith(map[string]string{"GOOS": goos, "GOARCH": goarch, "CGO_ENABLED": "0"},
		"go", "build", "-o", out, "./cmd/app")
}
```

Arguments count toward identity: `mg.F(compile, "linux", "amd64")` and `mg.F(compile, "darwin", "arm64")` are different dependencies and both run, in parallel, while two identical calls run only once.[^5] That gives you a parallel cross-compile in a dozen lines, which is noticeably more awkward to write in Make.

## The `sh` helpers

You could use `os/exec` directly, but the `sh` package wraps the common cases:[^6]

| Function | What it does |
|---|---|
| `sh.Run(cmd, args...)` | Run a command. Output is only shown in verbose mode (`-v`). |
| `sh.RunV(cmd, args...)` | Run a command and always show its output. |
| `sh.RunWith(env, cmd, args...)` | Run with extra environment variables. |
| `sh.Output(cmd, args...)` | Run and return stdout as a string. |
| `sh.RunCmd(cmd, args...)` | Return a reusable function with some arguments pre-filled. |
| `sh.Rm(path)` | Delete a file or directory tree; a missing path isn't an error. |
| `sh.Copy(dst, src)` | Copy a file. |

Two of these deserve a closer look.

`sh.Output` is how you get values like the current commit into your build:

```go
func Build() error {
	commit, err := sh.Output("git", "rev-parse", "--short", "HEAD")
	if err != nil {
		return err
	}
	ldflags := "-X main.commit=" + commit
	return sh.RunV("go", "build", "-ldflags", ldflags, "-o", "bin/app", "./cmd/app")
}
```

`sh.RunCmd` cuts down on repetition:

```go
var (
	goCmd  = sh.RunCmd("go")
	docker = sh.RunCmd("docker")
)

func Image() error {
	mg.Deps(Build)
	return docker("build", "-t", "example/app:dev", ".")
}
```

Note that `sh` functions **don't go through a shell**. `sh.Run("go", "test", "./...")` executes `go` directly with those arguments, so there's no word splitting, globbing or quoting to worry about. The flip side is that pipes and `&&` don't work; you write that logic in Go, or call `sh.Run("bash", "-c", "...")` when you really need a shell.

## Namespaces, defaults and aliases

As the list of targets grows, group them with a namespace:[^4]

```go
type Docker mg.Namespace

// Build builds the container image.
func (Docker) Build() error {
	return sh.RunV("docker", "build", "-t", "example/app:dev", ".")
}

// Push pushes the image to the registry.
func (Docker) Push() error {
	mg.Deps(Docker.Build)
	return sh.RunV("docker", "push", "example/app:dev")
}
```

These show up as `docker:build` and `docker:push`. Note the method-expression syntax `Docker.Build` when using a namespaced target as a dependency.

A default target runs when you type `mage` with nothing after it, and aliases give targets short names:[^4]

```go
var Default = Build

var Aliases = map[string]interface{}{
	"b": Build,
	"t": Test,
}
```

Without a `Default`, plain `mage` just lists the targets.

## Sharing targets across repositories

If you maintain several Go services, they probably share the same `lint`, `test` and `release` steps. Mage can import targets from an ordinary Go package with the `//mage:import` directive:[^7]

```go
//go:build mage

package main

import (
	//mage:import
	_ "github.com/example/buildtools/golang"
	//mage:import docker
	"github.com/example/buildtools/container"
)
```

Exported targets from `golang` land in the root namespace (`mage lint`), and those from `container` are grouped under `docker:` (`mage docker:push`). The imported package must not be `package main`, and its files must not carry build tags, or Go won't import it. Aliases and defaults in imported packages are ignored, but imported targets can be used as dependencies like any other.[^7] Version the shared package like any Go module and every repository picks up fixes with `go get -u`.

## Skipping work that's already done

`go build` and `go test` have their own caches, so most Go steps are already cheap to repeat. For steps that aren't (code generation from protobuf files, downloading a tool, building a frontend bundle), the `github.com/magefile/mage/target` package gives you Make's timestamp check in Go:

```go
import "github.com/magefile/mage/target"

func Proto() error {
	changed, err := target.Dir("gen/api.pb.go", "proto")
	if err != nil || !changed {
		return err
	}
	return sh.RunV("buf", "generate")
}
```

`target.Path` compares a destination against individual source files, and `target.Dir` walks source directories. If nothing is newer than the output, the step is skipped.

## Running Mage in CI

In CI you want a fast, pinned setup. Two options:

**Install and run.** In GitHub Actions, for example:

```yaml
- uses: actions/setup-go@v5
  with:
    go-version-file: go.mod
- run: go install github.com/magefile/mage@v1.17.2
- run: mage -v lint test build
```

Running several targets in one call is cheaper than separate steps because shared dependencies run only once. `-v` (or `MAGEFILE_VERBOSE=1`) makes `sh.Run` print command output, which you usually want in CI logs.[^8]

**Compile once, ship the binary.** `mage -compile ./bin/mage-ci` produces a static binary containing all your targets, and `-goos`/`-goarch` cross-compile it.[^2] The result runs without Go installed, which is handy for a deploy stage running in a minimal image.

Some other flags and variables that come up:[^2][^8]

- `-t 10m` sets a timeout; contexts passed to targets are cancelled when it expires.
- `-f` forces a rebuild of the cached magefile binary, and `-clean` removes old cached binaries. The cache lives in `~/.magefile` by default and can be moved with `MAGEFILE_CACHE`.
- `-d` sets the magefile directory and `-w` the working directory, for monorepos.
- `MAGEFILE_HASHFAST=1` speeds up the "does the magefile need recompiling?" check, at the cost of sometimes missing changes in imported packages.

## Mage vs Make (and Task, and Just)

Mage isn't the only Make alternative. A quick comparison of the common choices for a Go project:

| | Make | Mage | Task | Just |
|---|---|---|---|---|
| Language | Make + shell | Go | YAML + shell | Own syntax + shell |
| Pre-installed | On most Unix systems | No | No | No |
| Windows | Painful | Native | Good | Good |
| File-based rebuilds | Built in | `target` package | Built in (checksums/timestamps) | No |
| Parallel deps | `make -j` | `mg.Deps` | Built in | No |
| Real logic (loops, errors, data) | Awkward | Full Go | Limited | Limited |

**Make** wins on ubiquity: it's already there, everyone has seen a Makefile, and for five targets that each run one command, a Makefile is shorter than the equivalent magefile.

**Task** and **Just** are both easier to pick up than Mage for anyone who doesn't write Go, and they read like a list of commands, which is what most build scripts are.

**Mage** pulls ahead when build logic stops being "run this command" and starts being a program: building a matrix of platforms, reading a version from a file, calling an API to create a release, deciding what to deploy based on a Git diff. In shell-based tools, that turns into embedded scripts; in Mage, it's Go code you can test, refactor and step through in a debugger. It's also the obvious choice when your team only writes Go and you'd like contributors on Windows to be able to run the build.

The cost is a little more boilerplate per target, an extra tool to install (or the zero-install trick), and a compile step the first time you run it.

## Pitfalls

- **Forgetting the build tag.** Without `//go:build mage`, your magefile's `package main` collides with the rest of the root package and `go build ./...` breaks, or worse, the magefile's imports leak into your module's normal build.
- **`sh.Run` hides output.** If a command fails and you can't see why, either run with `-v` or use `sh.RunV`. This catches out almost everyone once.
- **Calling targets directly instead of via `mg.Deps`.** `Build()` called as a plain function runs every time and isn't parallelised. Use `mg.Deps(Build)` to get the run-once guarantee.
- **Expecting shell syntax.** `sh.Run("go test ./... | tee out.txt")` tries to execute a binary with that whole string as its name. Split arguments, and handle pipes in Go.
- **Magefile dependencies in `go.mod`.** Imports in your magefile (`github.com/magefile/mage/mg` and anything else) become dependencies of your module. That's usually fine, but keep the magefile's imports lean, or give `magefiles/` its own `go.mod` if you'd rather keep them separate.

## Cheat sheet

```bash
mage -init                 # create a starter magefile
mage -l                    # list targets
mage -h <target>           # help for one target
mage <target> [args]       # run a target
mage a b c                 # run several targets, shared deps run once
mage -v <target>           # verbose: show sh.Run output
mage -t 5m <target>        # timeout
mage -f <target>           # force recompiling the magefile
mage -compile ./bin/tool   # build a standalone binary of all targets
mage -install zsh          # shell completion
```

```go
//go:build mage

package main

import (
	"github.com/magefile/mage/mg"
	"github.com/magefile/mage/sh"
)

var Default = Build

// Build compiles the app.
func Build() error {
	mg.Deps(Generate)
	return sh.RunV("go", "build", "./...")
}

func Generate() error { return sh.Run("go", "generate", "./...") }

type Docker mg.Namespace

// Build builds the image.
func (Docker) Build() error {
	mg.Deps(Build)
	return sh.RunV("docker", "build", "-t", "app:dev", ".")
}
```

If your Makefile is ten lines, keep it. If it has grown its own dialect of shell, embedded `if` statements and a list of things that don't work on someone's laptop, try porting a few targets to Mage. It takes about as long as reading this post, and the build becomes one more Go package you already know how to maintain.

## References

[^1]: [Releases](https://github.com/magefile/mage/releases), magefile/mage on GitHub (v1.17.2, April 2026).
[^2]: [Mage](https://magefile.org/), official documentation.
[^3]: [Zero Install Option](https://magefile.org/zeroinstall/), Mage documentation.
[^4]: [Targets](https://magefile.org/targets/), Mage documentation.
[^5]: [Dependencies](https://magefile.org/dependencies/), Mage documentation.
[^6]: [Package sh](https://pkg.go.dev/github.com/magefile/mage/sh), pkg.go.dev.
[^7]: [Importing Targets](https://magefile.org/importing/), Mage documentation.
[^8]: [Environment Variables](https://magefile.org/environment/), Mage documentation.
