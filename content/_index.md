---
title: go-kv
---

`go-kv` (package [`kv`](https://pkg.go.dev/github.com/gomatic/go-kv)) is the gomatic ecosystem's **dependency-light helper for reading and manipulating key/value environment data**. It provides an [`Environment`](https://pkg.go.dev/github.com/gomatic/go-kv#Environment) map, lookups with fallbacks across one or more keys, typed conversions through any `func(string) (T, error)`, loading from JSON/YAML readers and files, and scoped set/restore of the process environment. It depends only on the standard library plus a YAML decoder.

- **Source:** [`gomatic/go-kv`](https://github.com/gomatic/go-kv)
- **API reference:** [pkg.go.dev/github.com/gomatic/go-kv](https://pkg.go.dev/github.com/gomatic/go-kv)

## Install

```sh
go get github.com/gomatic/go-kv
```

Requires Go 1.26+.

## Usage

### Lookups with fallback

The package-level helpers read the process environment directly:

```go
import kv "github.com/gomatic/go-kv"

port := kv.GetOr("PORT", "8080")                          // value of PORT, or "8080"
url := kv.FirstOr("http://localhost", "PRIMARY_URL", "FALLBACK_URL")
value, ok := kv.Lookup("OPTIONAL")                        // ok reports whether it was set
trimmed := kv.GetTrimmed("PADDED")                        // value with surrounding whitespace removed
```

`First` returns the value of the first key that is set (or `""`), and `LookupFirst` adds the "found anything?" boolean. Note the parameter order differs between the two fallback helpers — `GetOr(key, fallback)` takes the key first, while `FirstOr(fallback, keys...)` takes the fallback first.

### The Environment map

[`Environment`](https://pkg.go.dev/github.com/gomatic/go-kv#Environment) is a `map[string]string` carrying the same lookup and load methods, so it works as a standalone store that need not mirror the process environment:

```go
env := kv.New()                 // snapshot of os.Environ()
env = kv.Parse(os.Environ())    // or parse any []string of "KEY=VALUE" entries

got := env.GetOr("PORT", "8080")
first := env.FirstOr("none", "PRIMARY", "SECONDARY")
value, ok := env.Lookup("OPTIONAL")
```

`Set` pushes every entry onto the process environment (and returns the map so you can chain); `Unset` pulls them back out.

### Typed values

Environment values are strings; [`LookupAs`](https://pkg.go.dev/github.com/gomatic/go-kv#LookupAs) and [`GetOrAs`](https://pkg.go.dev/github.com/gomatic/go-kv#GetOrAs) convert them with any [`Convert[T]`](https://pkg.go.dev/github.com/gomatic/go-kv#Convert) — a `func(string) (T, error)`. Standard-library parsers already have that signature, so they pass directly and the type is inferred:

```go
port := kv.GetOrAs("PORT", strconv.Atoi, 8080)            // int
debug := kv.GetOrAs("DEBUG", strconv.ParseBool, false)    // bool
timeout := kv.GetOrAs("TIMEOUT", time.ParseDuration, time.Second)

// Distinguish "unset" from "set but invalid": ok reports presence, err reports a bad parse.
value, ok, err := kv.LookupAs("PORT", strconv.Atoi)
```

### Load from JSON or YAML

Load a fresh `Environment` from a reader or a file:

```go
env, err := kv.LoadFromJSONFile("config.json")
env, err = kv.LoadFromYAMLFile("config.yaml")

// Or from any io.Reader:
env, err = kv.LoadFromJSON(strings.NewReader(`{"PORT":"8080"}`))
env, err = kv.LoadFromYAML(strings.NewReader("PORT: \"8080\""))
```

The reader-and-file constructors are built on [`LoadFromUnmarshaler`](https://pkg.go.dev/github.com/gomatic/go-kv#Environment.LoadFromUnmarshaler), which loads into an existing map with any `func([]byte, any) error`; a `nil` reader yields [`ErrNilReader`](https://pkg.go.dev/github.com/gomatic/go-kv#pkg-constants).

### Load a value from a file

[`SetFromFile`](https://pkg.go.dev/github.com/gomatic/go-kv#SetFromFile) populates a variable from a file named by another variable — handy for large or secret values. It is a no-op when the target is already set or the file variable is unset, and the loaded content is whitespace-trimmed:

```go
// If SECRET is unset and SECRET_FILE names a readable file, load it into SECRET.
err := kv.SetFromFile("SECRET", "SECRET_FILE")
```

A read failure wraps [`ErrFileLoad`](https://pkg.go.dev/github.com/gomatic/go-kv#pkg-constants), recoverable with `errors.Is`.

### Scoped set and restore

Apply a modified environment for the duration of some calls, then restore the original:

```go
// Run fns with the overrides applied, then restore — stops at the first error.
err := kv.WrapCalls(kv.Environment{"DEBUG": "1"}, func() error { return run() })

// Or drive the lifecycle directly.
w := kv.NewWrapper(kv.Environment{"DEBUG": "1"})
err = w.CallWithRestore(run)

// Set a single variable and restore its prior state later.
restore := kv.SetWithRestore("DEBUG", "1", false)
defer restore()
```

`SetWithRestore`'s `allowEmpty` flag decides whether an empty value sets the key to `""` or unsets it; the returned function puts the variable back exactly as it was — set to its previous value, or unset if it never existed.

### Typed names

[`Name`](https://pkg.go.dev/github.com/gomatic/go-kv#Name) is a typed environment-variable name with its own accessors, and [`Names`](https://pkg.go.dev/github.com/gomatic/go-kv#Names) is a name→override map that falls back to the process environment and expands `${VAR}` references:

```go
const Port kv.Name = "PORT"
p := Port.ValueOr("8080")              // value of PORT, or "8080"
v, ok := Port.Lookup()                 // value plus whether it was set

names := kv.Names{"HOST": "localhost", "ADDR": "${HOST}:8080"}
expanded := names.Expand()             // ADDR resolves to "localhost:8080"
args := names.Replace("--url=${ADDR}") // ${VAR} references substituted in a fresh slice
```

`Expand` and `Replace` never mutate the receiver — each returns a fresh map or slice — and `Expand` iterates to a fixed point with a bounded number of passes so self-referential definitions can't loop forever.

## Design

- **Dependency-light** — the package leans on the standard library (`os`, `strconv`, `encoding/json`, `io`) plus a single YAML decoder; there is no configuration framework to adopt.
- **Value types throughout** — `Environment`, `Names`, and `Name` are plain map/string newtypes, safe to copy, and the expansion helpers return fresh copies rather than mutating their receivers.
- **Constant sentinel errors** — every error the package emits is a const of the [`Error`](https://pkg.go.dev/github.com/gomatic/go-kv#Error) string type (`ErrNilReader`, `ErrFileLoad`), matched with `errors.Is` rather than by string comparison, mirroring the rest of the gomatic ecosystem.
- **Mind the parameter order** — `GetOr(key, fallback)` leads with the key, while `FirstOr(fallback, keys...)` leads with the fallback so the variadic key list can trail.

## Who uses it

`go-kv` underpins environment handling across the gomatic Go projects, including [`renderizer`](https://github.com/gomatic/renderizer) and the other [`gomatic/go-*`](https://github.com/orgs/gomatic/repositories?q=go-) libraries.
