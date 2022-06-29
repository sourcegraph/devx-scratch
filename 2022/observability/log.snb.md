# 2022 observability log

DevX teammates hacking on Sourcegraph's observability libraries and tooling, both internal and customer-facing.

To add an entry, just add an H2 header starting with the ISO 8601 format, a topic.
**This log should be in reverse chronological order.**

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
