# 2022 support log

DevX support rotation log. To add an entry, just add an H2 header with ISO 8601 format. The first line should be a list of everyone involved in the entry. For ease of use and handing over issues, **this log should be in reverse chronological order**, with the most recent entry at the top.

## 2022-04-01

@bobheadxi

Frontend percy finalize failing consistently with no output:

```none
npm WARN exec The following package was not found and will be installed: @percy/cli
Error: exit status 1
```

- https://buildkite.com/sourcegraph/sourcegraph/builds/140259#7cd6556a-83e3-4c52-bd33-4934fe072385/159-163
- https://buildkite.com/sourcegraph/sourcegraph/builds/140241#025449b8-bc41-43dd-b9ca-1c26d89ce41e/165-169
- Also seen on PR branches, e.g. https://buildkite.com/sourcegraph/sourcegraph/builds/140249#7cfc48e9-637b-468e-b0e8-7483075a2ce5/161-165

Made a request for help to frontend platform: https://sourcegraph.slack.com/archives/C01LTKUHRL3/p1648856877518069 $DURATION=30m

Opened a PR to try and reduce [confusion around unrelated failures](https://sourcegraph.slack.com/archives/C02FLQDD3TQ/p1648851816913459): https://github.com/sourcegraph/sourcegraph/pull/33326

## 2022-03-31

@jhchabran

Spotted this [exception](https://github.com/sourcegraph/sec-pr-audit-trail/issues/144) that should not exist as the base branch is not `main`. Led to create [this bug report](https://github.com/sourcegraph/sourcegraph/issues/33275). $DURATION=5m

## 2022-03-30

@davejrt, @bobheadxi, @jhchabran

[This](https://github.com/sourcegraph/infrastructure/pull/3191) PR caused a small outage when we didn't merge in the correct order, and part of the core job was overwritten by the CI on the infrastructure repo. We had to do some juggling between stateless agent rollout, and merging this PR to unblock a large queue of other jobs that accumulated whilst stateless agents wouldn't start.

@bobheadxi set up an alert on `seconds waited for expected dispach`, which will notify us in #dev-experience-internal if the jobs dispatched by the dispatcher do not roll out sufficiently within the expected time frame. Visible in the [dispatcher dashboard](https://console.cloud.google.com/monitoring/dashboards/builder/a87f3cbb-4d73-476d-8736-f3bc1ca9f234?folder=true&organizationId=true&project=sourcegraph-ci&dashboardBuilderState=%257B%2522editModeEnabled%2522:false%257D&timeDomain=1h).

## 2022-03-29

@davejrt, @bobheadxi, @jhchabran

- `depguard` issues on both stateful and stateless agents with more frequency that is now becoming more disruptive. Initial investigation hasn't come up with anything definitive so for the moment, the tests have been disabled [here](https://github.com/sourcegraph/sourcegraph/pull/33182) and an [issue created](https://github.com/sourcegraph/sourcegraph/issues/33183)

## 2022-03-28

@jhchabran

It was reported that `sg` was stuck in an autoupdate loop. This was caused by
the fact that the user had two `sg` installed: one under `~/.sg/sg` and the other one in the `$GOBIN` folder. Because the detection is based on what's in the repository, the result was an infinite loop.

My hunch here is that we should just pick one, and always stick to that install location. I asked Thorsten his thoughts [here](https://sourcegraph.slack.com/archives/C01N83PS4TU/p1648468340459119?thread_ts=1648467054.249209&cid=C01N83PS4TU).

## 2022-03-25

@bobheadxi

- Sporadic `codeinsights` test failures from yesterday. Pinged again, fixed by @coury-clark via DB conn bump. [thread](https://sourcegraph.slack.com/archives/C02FLQDD3TQ/p1648184742528449)
- Unlinted code was merged: https://github.com/sourcegraph/sourcegraph/pull/33094
- Incident: GraphQL changes breaking main. Fix: https://github.com/sourcegraph/sourcegraph/pull/33092, proposed mitigation by @jhchabran: https://github.com/sourcegraph/sourcegraph/pull/33095
- `depguard` issues on some agents: intermittent, can occur on new-ish agents. causes a load of `depguard` false positives that fail the build. potentially co-incides with `golangci-lint 1.45.0` upgrade, though issues persisted both before and after downgrade. [thread](https://sourcegraph.slack.com/archives/C02FLQDD3TQ/p1648227503962249)
  - @davejrt suggests we wait for a repeat occurrence and deep-dive on the agent where it happened.
  - https://github.com/sourcegraph/sourcegraph/pull/33118 rolls out 50% so we can hopefully get a clearer comparison. See below

Also note deployment of the new stateless job dispatcher. Links to monitor:

- [Dispatcher logs](https://cloudlogging.app.goo.gl/pqcDLz9aeFCDtWTU9)
- [Dispatched agents](https://console.cloud.google.com/kubernetes/workload/overview?project=sourcegraph-ci&pageState=(%22savedViews%22:(%22i%22:%22d63788ab9603422da3abba5f06030393%22,%22c%22:%5B%5D,%22n%22:%5B%22buildkite%22%5D),%22workload_list_table%22:(%22f%22:%22%255B%257B_22k_22_3A_22Is%2520system%2520object_22_2C_22t_22_3A11_2C_22v_22_3A_22_5C_22False_~*false_5C_22_22_2C_22i_22_3A_22is_system_22%257D_2C%257B_22k_22_3A_22Name_22_2C_22t_22_3A10_2C_22v_22_3A_22_5C_22buildkite-agent-stateless-_5C_22_22_2C_22i_22_3A_22metadata%252Fname_22%257D%255D%22)))
- [Dispatcher metrics](https://console.cloud.google.com/monitoring/dashboards/builder/a87f3cbb-4d73-476d-8736-f3bc1ca9f234?project=sourcegraph-ci)

To roll back, revert https://github.com/sourcegraph/sourcegraph/pull/33107 or change the queue to target `stateless` instead of `stateless2`. Configuration changes are in https://github.com/sourcegraph/infrastructure/pull/3182

Most things can be traced using the dispatched ID, which is included in the job name, agent metadata, and log fields - e.g. to trace down why the dispatcher decided to make a dispatch.

## 2022-03-22

@bobheadxi

- Node installation failures in deploy-sourcegraph-cloud pipeline [thread](https://sourcegraph.slack.com/archives/C02KX975BDG/p1647971014157899?thread_ts=1647957942.367989&cid=C02KX975BDG)
  - Might be caused by really old Node version in the pipeline, or just plain bad state. Doesn't seem to be happening on the sourcegraph pipeline
  - @daxmc99 deletes a bunch of stuff propagated from upstream depoy-sourcegraph, as well as the prettier step entirely: https://github.com/sourcegraph/deploy-sourcegraph-cloud/pull/15935

## 2022-03-21

@bobheadxi, @jhchabran

We have been running into sporadic, severe issues with stateless agent availability since we [rolled out stateless builds to 75% of `sourcegraph/sourcegraph` builds](https://github.com/sourcegraph/sourcegraph/pull/32751). See [thread](https://sourcegraph.slack.com/archives/C02MWRMAFR8/p1647838869486089). Caused some severe downtime in the morning, with [incident 92](https://app.incident.io/incidents/92)

- Increased `backoffLimit` to max: https://github.com/sourcegraph/infrastructure/pull/3176
- Discussed autoscaling implementation, landed on some misunderstandings: https://github.com/sourcegraph/infrastructure/pull/3177
- Scaled down agent rollout top 10%: https://github.com/sourcegraph/sourcegraph/pull/32840

Follow-up: potential revamp of autoscaler. https://github.com/sourcegraph/sourcegraph/issues/32843
