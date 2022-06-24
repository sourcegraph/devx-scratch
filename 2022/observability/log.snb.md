# 2022 observability log

DevX teammates hacking on Sourcegraph's observability libraries and tooling, both internal and customer-facing.

To add an entry, just add an H2 header starting with the ISO 8601 format, a topic.
**This log should be in reverse chronological order.**

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
