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

https://sourcegraph.com/github.com/sourcegraph/infrastructure@fee5b7b43cc8f743e0dc8b28631267255920f6c4/-/blob/buildkite/kubernetes/buildkite-agent-stateless/buildkite-agent.Job.yaml?L10-13
