# 2022 stateless agents

Notes on https://github.com/sourcegraph/sourcegraph/issues/30233. **This log should be in chronological order.**

## 2022-03-22

@bobheadxi

Hacking on https://github.com/sourcegraph/sourcegraph/issues/32843

> Judging from [the Kubernetes `Job` docs](https://kubernetes.io/docs/concepts/workloads/controllers/job), I think a more best-practice approach might be to dynamically generate `Job`s on the fly:
>
> 1. Query for pending jobs and running jobs
>   a. If running > maxAgentsCount, do nothing
> 2. Create **a new `Job`** for `$count = pending` (subject to maxAgentsCount and minAgentsCount), with `completions: $count` and `parallelism: $count`
>   a. Job template source TBD. Ideas are a "no-op" Job deployed to K8s, simply pull from repo, or just embed within the autoscaler
>   b. Creating Jobs from templates is an officially documented use case: https://kubernetes.io/docs/tasks/job/parallel-processing-expansion/
>   c. This seems aligned with how `completions` should be used:
>    > As pods successfully complete, the Job tracks the successful completions. When a specified number of successful completions is reached, the task (ie, Job) is complete.
> 3. Repeat

Idea for rollout: read from @davejrt's current Job implementation, and dispatch new Jobs with the current Job as a template, mutating fields with the above strategy

https://sourcegraph.com/github.com/sourcegraph/infrastructure@fee5b7b43cc8f743e0dc8b28631267255920f6c4/-/blob/buildkite/kubernetes/buildkite-agent-stateless/buildkite-agent.Job.yaml?L10-13

Then we can scale down the current job *autoscaler* and see how what I'm planning to call the job *dispatcher* behaves.

## 2022-03-23

@bobheadxi

Continued from yesterday. Current approach pulls from our current job implementation, updates its values, and attempts to create a new job. This was an idea raised by @jhchabran and is probably the easiest way to get this started, because we can leverage the existing job. It's also nice in that you still get infra-as-code for the template as well.

Started by copying over @davejrt's existing [`job-autoscaler`](https://sourcegraph.com/github.com/sourcegraph/infrastructure@fee5b7b43cc8f743e0dc8b28631267255920f6c4/-/tree/docker-images/job-autoscaler), which has some of the groundwork for getting started.

Job naming must be unique? Date seems clunky. Hesistant to import dependencies, but oh well - decided to use Google's UUID package https://github.com/google/uuid

Ran into some funny business:

```none
Running tool: /usr/local/bin/go test -timeout 30s -run ^Test_computeJobConfig$ github.com/sourcegraph/infrastructure/docker-images/buildkite-job-dispatcher

panic: mismatching message name: got k8s.io.kubernetes.pkg.watch.versioned.Event, want github.com/ericchiang.k8s.watch.versioned.Event
```

Looks like https://github.com/ericchiang/k8s is deprecated? Tried using the official client, but it doesn't seem to provide a struct type for `Job`. Ended up with a workaround:

```go
// Transient dependency incompatability, similar to https://stackoverflow.com/questions/61251129/how-to-fix-conflicts-with-this-error-message
replace github.com/golang/protobuf => github.com/golang/protobuf v1.3.5
```

Also taking this opportunity to use https://github.com/uber-go/zap. Much nicer usage API, standard of passing around logging objects that can have state attached to them, etc. It is often raised as a potential alternative to `log15` weirdness (like its spelling of `eror`).

Fun: when deployed, we automatically get `jsonPayload` populated in GCP logs with Zap's default JSON format (probably also possible with `log15` - I'm not sure if we do use JSON logs at the moment though). Example:

<img width="200" alt="image" src="https://user-images.githubusercontent.com/23356519/159804967-3b61247c-2d27-4908-845d-d0b187bb3105.png">

Can e.g. add `dispatcher.run` to summary fields, to group logs visually by dispatcher iteration.

Cleaning up generated Kubernetes object metadata by fixing + deploying + looking at logs for errors (don't seem to get all the possible errors at once):

- Metadata UID (Kubernetes identifier): https://kubernetes.io/docs/concepts/overview/working-with-objects/names/#uids
- Metadata resource version
- Labels and selectors for both the spec and template spec (controller info is attached here)

Big oops detected at this point - I've threaded configuration to target a custom Buildkite queue, because I want to be able to roll this out alongside our existing jobs implementation. Querying is based on new queue, but agents are created in the old queue.

Result: [dozens of jobs being created](https://sourcegraph.slack.com/archives/CMBA8F926/p1648076224059209), because the dispatcher thinks we aren't creating anything because its view depends on Buildkite.

TODOs: configure agent template to deploy in new queue. look at mitigation for this scenario. if no agents in X iterations, exit as unhealthy?

Draft PR: https://github.com/sourcegraph/infrastructure/pull/3182

## 2022-03-25

@bobheadxi @davejrt @jhchabran

Went over my updates to https://github.com/sourcegraph/infrastructure/pull/3182 yesterday:

1. Buildkite API being nonfunctional - metadata query is unreliable, so we resort to plain search now. [@davejrt](https://github.com/sourcegraph/infrastructure/pull/3182#discussion_r835246040) noted might be related to only one `queue` being made queryable at a time.
2. Migation for jobs taking some time to start up: expect percentage of previous batch to deploy + padding agent counts with `ROLLOUT_CONSEQUENT`
3. Created fork of https://github.com/ericchiang/k8s to fix [Job TTL after done not being available](https://github.com/sourcegraph/infrastructure/pull/3182#issuecomment-1078544735): https://github.com/sourcegraph/k8s
   1. This removes the need for [the protobuf `replace` workaround from the other day](#2022-03-23)
4. Fixed issue with [yarn cache not working](https://github.com/sourcegraph/infrastructure/pull/3182#issuecomment-1078668892) - the caching mechanism was toggled on the `job` queue name. See https://github.com/sourcegraph/sourcegraph/pull/33107

We agreed that the current approach, with Buildkite as agent count source-of-truth (as opposed to Kubernetes), is fine despite overscaling tendencies.

Applied manifest changes in https://github.com/sourcegraph/infrastructure/pull/3182 and merged https://github.com/sourcegraph/sourcegraph/pull/33107 to switch the 10% stateless rollout to target the new dispatched agents to monitor. See PRs for links to observability.

After monitoring for half a day, some changes:

- bumped down dispatch interval, increased buffer - scale-up wasn't happening fast enough
- more Buildkite API woes - updated the scheduled jobs query to work properly. [feedback thread](https://sourcegraph.slack.com/archives/C02RH06NMH6/p1648238136391329)
- more aggressive scale-down strategy by setting `ActiveDeadlineSeconds`:
  - if scaling up to around buffer count, set a long deadline (potential quite period)
  - otherwise set a short deadline

The [logs-based metrics](https://console.cloud.google.com/logs/metrics?folder=true&organizationId=true&project=sourcegraph-ci) + [dashboards](https://console.cloud.google.com/monitoring/dashboards/builder/a87f3cbb-4d73-476d-8736-f3bc1ca9f234?folder=true&organizationId=true&project=sourcegraph-ci&dashboardBuilderState=%257B%2522editModeEnabled%2522:false%257D&timeDomain=1h) + ability to trace a dispatched set of agents back to the logs using the dispatch ID works together quite nicely!

## 2022-03-30

@bobheadxi, @jhchabran, @davejrt

Finalized rollout of stateless agents:

- [x] https://github.com/sourcegraph/infrastructure/pull/3188
- [x] https://github.com/sourcegraph/sourcegraph/pull/33186 (for `sourcegraph/sourcegraph`)
- [x] https://github.com/sourcegraph/sourcegraph/pull/33226
- [x] https://github.com/sourcegraph/infrastructure/pull/3192

Some stateful agents are still running, but the Kubernetes manifests have been removed and we can spin them down after a while.

Ran into some hiccups, but otherwise looks good - for more details see [the support log](../support/log.md#2022-03-30)

The dispatcher is keyed on `queue: stateless`. We might want to migrate [references to `queue: job`](https://sourcegraph.com/search?q=context:%40sourcegraph/all+queue:+job+lang:yaml&patternType=literal) eventually. If we ever switch back to the `standard` queue, we'll want to update all references as well. Rename is required to make the pending builds detection work correctly. [#33238](https://github.com/sourcegraph/sourcegraph/issues/33238)

## 2022-03-31

@jhchabran

The slack token used on CI seems to have some issues, [see this buildkite notification](https://sourcegraph.slack.com/archives/C02FLQDD3TQ/p1648722027703459). It was missing from the stateless agents manifest. As I was alone, I self merged on this one and applied the changes. $DURATION=10m

## 2022-04-04

@bobheadxi, @davejrt

We paired today to investigate recent occurences on the alert @bobheadxi set up for `secondsWaitedForExpectedDispatch`. Findings:

- Right now we fetch the expected (previous) agent dispatch state from Buildkite, where we count agents with the corresponding dispatch ID and wait for a percentage of the expected agents to roll out. However, agents can be "consumed" (finish running their jobs) before (or while) the job dispatcher waits for the expected dispatch to roll out. This is indicated [in the metrics](https://console.cloud.google.com/monitoring/dashboards/builder/a87f3cbb-4d73-476d-8736-f3bc1ca9f234?project=sourcegraph-ci), where the total agents count drop drastically before the expected dispatch has reached the required threshold, likely due to the agents being consumed quickly by short jobs. To mitigate this, we should change the source-of-truth of expected dispatches from Buildkite to Kubernetes, where we can get a more accurate picture by looking at the state of the Job resource.
  - PR: [infrastructure#3200](https://github.com/sourcegraph/infrastructure/pull/3200)
- @davejrt noted that anecdotally he sees many occurences of us hitting our resource quotas for the `sourcegraph-ci` project.
  - Confirmed in the [GCP quotas page](https://console.cloud.google.com/iam-admin/quotas?referrer=search&project=sourcegraph-ci) (notably `N2D CPUs` and `Local SSD (GB)` quotas in the `us-central-1` region)
  - We are not over-provisioning - at least some CI jobs make full use of the machines. We [added a panel to the GCP dashboard for `buildkite-job-dispatcher`](https://console.cloud.google.com/monitoring/dashboards/builder/a87f3cbb-4d73-476d-8736-f3bc1ca9f234?project=sourcegraph-ci) that shows CPU utilization can regularly hit 100%.
  - @davejrt to investigate simplifying our node pools. The next step could be to introduce separate queues that use node pools of differently sized machines to process jobs of varying intensity. See [issue #33390](https://github.com/sourcegraph/sourcegraph/issues/33390) for further investigation 

We will continue observing after merging [infrastructure#3200](https://github.com/sourcegraph/infrastructure/pull/3200)
