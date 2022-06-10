# 2022 general `sg` hacking log

DevX teammates hacking on `sg` log. To add an entry, just add an H2 header starting with the ISO 8601 format, a topic.
**This log should be in reverse chronological order.**

## 2022-06-09

@bobheadxi Following last Friday's `sg` hack hour, I've been working on a shared `check.Runner` construct that can be applied to both `sg lint` and `sg setup`: https://github.com/sourcegraph/sourcegraph/pull/36556

Most details and goals are in the above PR, but some other thoughts I've run into while working on this:

- The way `output.Output` and `std.Output` are structured makes it very hard to use anything other than just raw output - for example, you can't choose to have them backed by an alternative `output.Writer` implementation like `Pending`. You either get to have nice things like `WriteMarkdown`, or you have the very bare-bones `output.Writer`. I spiked moving things around and it was very complicated, and likely not possible.
- `output.Pending`, `output.Progress`, etc are super nice when they work, but can be quite hard to use correctly. You also can't have multiple of them, which makes sense from a terminal perspective, but I approached it with too much of a frontend perspective and confused myself quite a bit thinking I might be able to nest output and stuff. [`charmbracelet/bubbletea`](https://github.com/charmbracelet/bubbletea) has some nice paradigms for that, but probably way overkill unless "stunning terminal visuals" becomes an Enablement goal (...unless...?????).
- Dealing with requests for input when streaming command output to `std.Output` causes commands to stall, so we must ask for input separately. This happens because we can't stream a line that hasn't ended - most input prompts to not end with a new line.
  - In the `gcloud` install, we helpfully get a flag that disables prompts. In the 1Password CLI `op`, we do not, which was a bit disappointing.
- It was very hard to understand the `sg setup` code, and as I refactored it to use as internals for `check.Runner` I broke things many times. Going auto-fix as a first-class citizen and putting manual fixing on the backburner made things a lot simpler (...hopefully).
  - Related to this, I got some interesting feedback from Michael today, who said that he never used `sg setup`, and set things up manually himself. I wonder if the audience for manual fixing might be mutually exclusive from those that want to use `sg setup`.
  - Metrics would be good to get concrete data about the above.
- Seeing the new unit tests for `check.Runner` work is very satisfying, and the integration tests for `sg setupv2` went pretty smoothly except for Docker and `caddy-trust` and auth-related setup.

It's been a sizeable effort, but I'm relatively optimistic this is a worthwhile investment - `sg setup` can be re-oriented as the mechanism for keeping your environment up to date (away from the deprecated onboarding goal), and unifying the internals of `sg setup` and `sg lint` will helps us maintain both in a more nimble manner (hopefully).

## 2022-05-27

@bobheadxi

Recently an issue was discovered where `sg lint go` would hang on the `go generate` check if a diff was found ([#35918](https://github.com/sourcegraph/sourcegraph/issues/35918)). The issue was identified to be the call to `std.Out.WriteMarkdown`, which seemed to block forever in Buildkite - a workaround was made to check for `BUILDKITE=true` and write plain text if so, to allow pipelines to pass again ([#36043](https://github.com/sourcegraph/sourcegraph/pull/36043)).

This workaround was not a great long-term solution because:

- the whole point of adding `std.Out.WriteMarkdown` was to make output more readable in CI as well, which this workaround circumvented
- I wanted to avoid `BUILDKITE=true` checks wherever possible, since that goes directly against our goals of unifying CI and local dev and rectifying the divergence that has built up over time

I did a dive into the issue to see if I could resolve it - I found it super bizarre that the Markdown write could hang indefinitely! Under the hood [Glamour](https://github.com/charmbracelet/glamour) (our Markdown printer) quickly delegates to [Goldmark](https://github.com/yuin/goldmark) which seemed super unlikely to have this sort of bug. My train of thought:

1. I figured it since it appeared to be platform-dependent, it meant it was likely that some sort of attempted terminal/platform detection was occurring - the write function was never exiting, and we write to string buffers in various locations, so it probably had to happen before the write step (string buffer is platform-agnostic so it seemed unlikely it would be while writing)
2. I found the call to [`termenv.HasDarkBackground`](https://sourcegraph.com/github.com/sourcegraph/sourcegraph@e62632dc0ca53261d80f62cbd7e270a9733c3a32/-/blob/lib/output/output.go?L235=) and immediately thought this might be a newly introduced bug, and some print statement debugging confirmed that the code stops exactly at this point. Removing that and assuming a dark background, the diff rendered fine in Buildkite!
3. I looked into reverting to [`WithAutoStyle`](https://github.com/charmbracelet/glamour/blob/d5fb104a74ca0a823562b7d0216f9a5729bede8b/glamour.go#L124) but stepping into that revealed [the same call to `termenv.HasDasrkBackground`](https://github.com/charmbracelet/glamour/blob/d5fb104a74ca0a823562b7d0216f9a5729bede8b/glamour.go#L248), so we needed a workaround
4. I then looked back at how we can avoid this entire thing, read up on the override pattern in `lib/output`, decided that the cleanest way forward would refactor the whole detection thing.

The result was [#36193 dev/sg: fix diff output in Buildkite without using Buildkite condition](https://github.com/sourcegraph/sourcegraph/pull/36193), which:

- Adds a `ForceDarkBackground` output option, set to true in Buildkite
- Refactors output capabilities detection to be lazy (if an override is set, do not do any detection at all)
- Changes usages of `output.NewOutput` to `std.NewOutpu`t for consistency and to avoid the problematic dark background detection

## 2022-05-09

@bobheadxi @jhchabran @danieldides

Last week we had a discussion around scripting in Go. We're opting to introduce a new library, [`github.com/sourcegraph/run`](https://github.com/sourcegraph/run), that provides a builder pattern for executing commands and operating on their output in a programmatic manner - WIP example:

```go
// Export lines
lines, _ := run.Cmd(ctx, "ls").Run().Lines()
for i, l := range lines {
    fmt.Printf("line %d: %q\n", i, l)
}

// Render new README file
var readmeData bytes.Buffer
_ = run.Cmd(ctx, "cat", "README.md").Run().Stream(&readmeData)
replaced := exampleBlockRegexp.ReplaceAll(readmeData.Bytes(), exampleData.Bytes())

// Pipe modified data to command
_ = run.Cmd(ctx, "cp /dev/stdin README.md").Input(bytes.NewReader(replaced)).Run().Wait()

// Or, just pipe Output directly  to another command!
lsOut := run.Cmd(ctx, "ls cmd").Run().
    Filter(func(line []byte) ([]byte, bool) {
        return append([]byte("./cmd/"), line...), false
    })
_ = run.Cmd(ctx, "cat").Input(lsOut).Run().
    Stream(os.Stdout)
```

Also, related to the previous `sg analytics` discussion, just landed [dev/sg: track local sg analytics that can be submitted manually](https://github.com/sourcegraph/sourcegraph/pull/35033) toda. Spoke with OkayHQ, was told these events will show up in UI after a feature update next week. Also confirmed we can allow users to not submit an identity. Maybe we can generate randomized identifiers.

## 2022-05-04

@bobheadxi

I have been experimenting with [`github.com/bitfield/script`](https://github.com/bitfield/script). To date, the only usage that has stuck is [#34926](https://github.com/sourcegraph/sourcegraph/pull/34926), and even that is debatable. Thoughts on why:

1. `run.InRoot`, `run.GitCmd`, etc. are already pretty good 75% of the time.
2. `scipt.Pipe()` semantics are really weird sometimes, especially with `Filter` etc, and you end up really question whether or not it's better to simply run commands one by one. Example: https://github.com/sourcegraph/sourcegraph/pull/34680/files
3. Some things that are kind of important are missing, such as setting the execution context's directory, using `context.Context`, using `Exec` with `[]string`, and so on.
4. Most of the helpers don't really seem that useful when it comes down to it

@jhchabran had some thoughts here as well: https://github.com/sourcegraph/sourcegraph/discussions/33903#discussioncomment-2639015

I think for our use cases it might be simpler to build our own script pipeline thingo.

## 2022-04-05

@bobheadxi @davejrt

Potential integration with OkayHQ, geared towards identifying blockers/points of improvement, potentially useful features that could be exposed better, and/or unused features. Is being done at other companies.

Emphasis on not measuring productivity with `sg` analytics, but about things we can do to improve it.

Spike issue: https://github.com/sourcegraph/sourcegraph/issues/33447

## 2022-03-30

@jhchabran

With Thorsten we completely reworked the installation and update process. It will always use the location of the binary as the source of truth.
The bootstrap process (the `curl ... | sh`) will suggest `~/.sg/sg` as the default location. This also led to the complete deprecation of the `./dev/sg/install.sh` script.

Announcement is here: https://sourcegraph.slack.com/archives/C01N83PS4TU/p1648646383295729

## 2022-03-28

@jhchabran

I merged the PR that replace the updates with a download of the prebuilt binary. https://github.com/sourcegraph/sourcegraph/pull/33132

## 2022-03-22 `sg [...] open`

@bobheadxi

Idea brought up [Scratchpad - Improving engineering onboarding: tooling](https://docs.google.com/document/d/1NS_C3te-59P149LD_rrL1zcflNqqZOoSHVJfv8oPcqY/edit)

> we want to make the tools available discoverable

- added one for `sg ci` (https://github.com/sourcegraph/sourcegraph/pull/32857)
- we don’t have `sg monitoring` yet, and we don’t even know if we will continue using Grafana going forward
- not much else to add?
