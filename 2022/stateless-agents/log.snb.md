# 2022 stateless agents

https://github.com/sourcegraph/sourcegraph/issues/30233

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

Applied manifest changes in https://github.com/sourcegraph/infrastructure/pull/3182 and merged https://github.com/sourcegraph/sourcegraph/pull/33107 to switch the 10% stateless rollout to target the new dispatched agents to monitor.

I've also updated links to monitor this:

- [Dispatcher logs](https://cloudlogging.app.goo.gl/pqcDLz9aeFCDtWTU9)
- [Dispatched agents](https://console.cloud.google.com/kubernetes/workload/overview?project=sourcegraph-ci&pageState=(%22savedViews%22:(%22i%22:%22d63788ab9603422da3abba5f06030393%22,%22c%22:%5B%5D,%22n%22:%5B%22buildkite%22%5D),%22workload_list_table%22:(%22f%22:%22%255B%257B_22k_22_3A_22Is%2520system%2520object_22_2C_22t_22_3A11_2C_22v_22_3A_22_5C_22False_~*false_5C_22_22_2C_22i_22_3A_22is_system_22%257D_2C%257B_22k_22_3A_22Name_22_2C_22t_22_3A10_2C_22v_22_3A_22_5C_22buildkite-agent-stateless-_5C_22_22_2C_22i_22_3A_22metadata%252Fname_22%257D%255D%22)))
- [Dispatcher metrics](https://console.cloud.google.com/monitoring/dashboards/builder/a87f3cbb-4d73-476d-8736-f3bc1ca9f234?project=sourcegraph-ci)
