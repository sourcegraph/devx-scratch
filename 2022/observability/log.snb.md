# 2022 observability log

DevX teammates hacking on Sourcegraph's observability libraries and tooling, both internal and customer-facing.

To add an entry, just add an H2 header starting with the ISO 8601 format, a topic.
**This log should be in reverse chronological order.**

## 2022-07-04 Safe logging

@bobheadxi

Consider the following:

```go
type Data struct {
    log log.Logger // interface
    // ...
}

func (d *Data) Foobar() { d.log.Debug("foobar") }
```

Because [`log.Logger` is a pointer type (interface)](https://sourcegraph.com/github.com/sourcegraph/log/-/blob/logger.go), improper instantiation of `Data` can be fatal since the zero value of the field will be nil, as happened in https://github.com/sourcegraph/sourcegraph/pull/36457 and https://github.com/sourcegraph/sourcegraph/pull/38185.
I considered the possibility of introducing a "safe zero type" version of `log.Logger`, that one could use like so:

```go
type Data struct {
    log log.SafeLogger // value type
    // ...
}

func (d *Data) Foobar() { d.log.Debug("foobar") } // safe
```

Sketch:

```go
// SafeLogger is a Logger that is safe for use as a zero value.
type SafeLogger struct {
    Scope       string
    Description string

    // logger is the underlying Logger instance instantiated with From.
    logger Logger
    // loggerOnce must be a pointer because sync.Once should never be copied.
    loggerOnce *sync.Once
}

var _ Logger = SafeLogger{}

// From instantiates SafeLogger explicitly from a parent logger.
func (s SafeLogger) From(logger Logger) Logger {
    if s.loggerOnce == nil {
        s.loggerOnce = &sync.Once{}
    }
    s.loggerOnce.Do(func() {
        if logger == nil {
            // Create a new top-level logger
            s.logger = Scoped(s.Scope, s.Description)
        }
        s.logger = logger.Scoped(s.Scope, s.Description)
    })
    return s.logger
}

// Concretely implement log.Logger

func (s SafeLogger) Scoped(scope string, description string) Logger {
    return s.From(nil).Scoped(scope, description)
}

// ...etc
```

Problems:

- effectively the same as always using a new global logger (`log.Scoped`)
- if you rely on `SafeLogger`'s zero value, you might never set `Scope` and `Description`, leading to poorly scoped loggers

Additionally, there are other patterns where pointer types are threaded extensively throughout the codebase, namely `database.DB`, and we get away with that fine, so the `log.Logger` pattern might continue to be acceptable:

https://sourcegraph.com/search?q=context:global+repo:%5Egithub%5C.com/sourcegraph/sourcegraph%24+database.DB

## 2022-06-30 Possibly connected initiative in repo-management

@jhchabran

Alex Ostrikov reached me out this morning with a simple question around the ability to export logs through an API, in the context of this [initiative to build a 1-click exporter](https://github.com/sourcegraph/sourcegraph/discussions/37930).
We had a casual conversation that I then ported it in their GitHub discussion in [that comment](https://github.com/sourcegraph/sourcegraph/discussions/37930#discussioncomment-3055172). I proposed them that we reach them out once we have some otel collector
so they can run some experiment on their own.

@bobheadxi also left some comments (e.g. https://github.com/sourcegraph/sourcegraph/discussions/37930#discussioncomment-3081875 and https://github.com/sourcegraph/sourcegraph/discussions/37930#discussioncomment-3075562)

## 2022-06-29 OpenTelemetry trace export exploration

@bobheadxi

https://github.com/sourcegraph/sourcegraph/issues/27386

- `internal/trace.Tracer` seems duplicative of `internal/tracer` (or vice versa)
- `internal/trace/ot` is a mish-mash of trace policy and OT-specific logic
- Turns out we have lots of code using `internal/trace/ot` directly to start traces, rather than `internal/trace` which wraps the latter

Example migration:

https://sourcegraph.com/search?q=context:global+repo:%5Egithub%5C.com/sourcegraph/sourcegraph%24+ot.StartSpanFromContext%28:%5Bargs%5D%29&patternType=structural

Good news: a package exists that might be able to bridge OpenTracing with OpenTelemetry, https://pkg.go.dev/go.opentelemetry.io/otel/bridge/opentracing

Bad news: we need to consolidate how tracers are set up.

In the meantime, opened some quick PRs for cleanup:

- [internal/trace: extract trace policy to OT-agnostic package](https://github.com/sourcegraph/sourcegraph/pull/37962)
- [internal/trace: reduce direct usages of OpenTracing](https://github.com/sourcegraph/sourcegraph/pull/37963)

The big refactor PR to reduce direct usages just does some nifty Comby replace, as well as a bit of manual fixing:

```sh
comby 'ot.StartSpanFromContext(:[args])' 'trace.New(:[args], "")' .go -in-place
goimports -w .
```

Update - it turns out the above is not that simple, and direct usages of OpenTracing is actually incompatible with `internal/trace`, for example the below is not something `internal/trace` can pick up and extract an `internal/trace.Trace` out of:

https://sourcegraph.com/github.com/sourcegraph/sourcegraph@652f77c01967d979237e88e7fccc36873121d391/-/blob/cmd/frontend/graphqlbackend/graphqlbackend.go?L50-59

I reverted the second PR above in https://github.com/sourcegraph/sourcegraph/pull/37979 , and in the interest of focusing on codeintel as outlined in https://github.com/sourcegraph/sourcegraph/issues/37778 I'm going to give up efforts on `internal/trace` and focus on our tracer implementations (`internal/trace.Tracer` and `internal/tracer`) to leverage the OTel bridge: https://pkg.go.dev/go.opentelemetry.io/otel/bridge/opentracing - this should cover most codeintel stuff, which uses `internal/observation` (which, in turn, uses all the above).

I went back, banged my head on this a bit more, and I think I understand how this works better now and how to hook into everything correctly: https://github.com/sourcegraph/sourcegraph/pull/37984 - how this works:

1. `internal/tracer.Init()` -> sets a `switchableTracer` to `opentracing.GlobalTracer()`
2. Everyone else uses `opentracing.GlobalTracer` -> `switchableTracer`
3. We hot-swap the `opentracing.Tracer` _inside_ `switchableTracer` with whatever is in site config
4. Instead of using our default OpenTracing tracer, Jaeger, we now slot in an OpenTelemetry tracer bridged using the previously mentioned OpenTracing bridge library that sends stuff to an OpenTelemetry collector.

Will test all this tomorrow.

## 2022-06-28 Sentry notes

@jhchabran

While taking a look at our Sentry backlog, I noticed that GitServer is kinda light on its use of logging scopes. Opened a [very small PR as a draft](https://github.com/sourcegraph/sourcegraph/pull/37830) to bring this forward to the team as a low prio item. 

## 2022-06-23 OpenTelemetry trace export exploration

@bobheadxi

I've been spending a bit of time reading through our tracing code, assessing what it might take to [migrate to OpenTelemetry](https://github.com/sourcegraph/sourcegraph/issues/27386) - there's no opposition to using OpenTelemetry tracing so I think we should just go ahead and do the necessary implementation.

What I've done so far:

- [internal/trace: reorganize into multiple files](https://github.com/sourcegraph/sourcegraph/pull/37587)
- [trace, profiler, router: remove datadog](https://github.com/sourcegraph/sourcegraph/pull/37654)

Packages of note for tracing:

- [`internal/trace`](https://sourcegraph.com/github.com/sourcegraph/sourcegraph/-/tree/internal/trace)
- [`internal/tracer`](https://sourcegraph.com/github.com/sourcegraph/sourcegraph/-/tree/internal/tracer)

Some initial thoughts:

- We probably want to keep OpenTracing for back-compat
- We might want to try and export OpenTelemetry with minimal API changes - this is a bit unfortunate because we will have OpenTracing imports everywhere still, namely because of `"github.com/opentracing/opentracing-go/log"`:

https://sourcegraph.com/search?q=context:global+repo:%5Egithub%5C.com/sourcegraph/sourcegraph%24+lang:go+github.com/opentracing/opentracing-go/log+count:all&patternType=lucky
