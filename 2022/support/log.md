# 2022 support log

DevX support rotation log. To add an entry, just add an H2 header with ISO 8601 format. The first line should be a list of everyone involved in the entry. For ease of use and handing over issues, **this log should be in reverse chronological order**, with the most recent entry at the top.

## 2022-04-26

@jhchabran Another round of issues around the firewall, this time after the merge or @bobheadxi's fix. After Zooming with the person that asked for help, it became apparent that some services weren't caught by the code, which I patched here https://github.com/sourcegraph/sourcegraph/pull/34501.  $DURATION=45m

## 2022-04-22

@jhchabran Observed two builds (here and here) failing due jobs losing their agents: 

```
Node condition FrequentContainerdRestart is now: False, reason: NoFrequentContainerdRestart 	NoFrequentContainerdRestart 	Apr 22, 2022, 5:50:28 PM 	Apr 22, 2022, 5:50:28 PM 	1 	
Node condition FrequentDockerRestart is now: False, reason: NoFrequentDockerRestart 	NoFrequentDockerRestart 	Apr 22, 2022, 5:50:27 PM 	Apr 22, 2022, 5:50:27 PM 	1 	
Node condition FrequentKubeletRestart is now: False, reason: NoFrequentKubeletRestart 	NoFrequentKubeletRestart 	Apr 22, 2022, 5:50:26 PM 	Apr 22, 2022, 5:50:26 PM 	1 	
Node condition FrequentUnregisterNetDevice is now: False, reason: NoFrequentUnregisterNetDevice 	NoFrequentUnregisterNetDevice 	Apr 22, 2022, 5:50:26 PM 	Apr 22, 2022, 5:50:26 PM 	1 	
Node gke-default-buildkite-main-ecd9ee7d-s9r6 status is now: NodeHasNoDiskPressure 	NodeHasNoDiskPressure 	Apr 22, 2022, 5:49:49 PM 	Apr 22, 2022, 5:50:19 PM 	7 	
Node gke-default-buildkite-main-ecd9ee7d-s9r6 status is now: NodeNotReady 	NodeNotReady 	Apr 22, 2022, 5:50:19 PM 	Apr 22, 2022, 5:50:19 PM 	1 	
Node gke-default-buildkite-main-ecd9ee7d-s9r6 status is now: NodeHasSufficientMemory 	NodeHasSufficientMemory 	Apr 22, 2022, 5:49:49 PM 	Apr 22, 2022, 5:50:19 PM 	7 	
Node gke-default-buildkite-main-ecd9ee7d-s9r6 status is now: NodeHasSufficientPID 	NodeHasSufficientPID 	Apr 22, 2022, 5:49:49 PM 	Apr 22, 2022, 5:50:19 PM 	7 	
Started Kubernetes kubelet. 	KubeletStart 	Apr 22, 2022, 4:55:25 PM 	Apr 22, 2022, 5:49:49 PM 	2 	
invalid capacity 0 on image filesystem 	InvalidDiskCapacity 	Apr 22, 2022, 5:49:49 PM 	Apr 22, 2022, 5:49:49 PM 	1 	
Updated Node Allocatable limit across pods 	NodeAllocatableEnforced 	Apr 22, 2022, 5:49:49 PM 	Apr 22, 2022, 5:49:49 PM 	1 	
Starting kubelet. 	Starting 	Apr 22, 2022, 5:49:49 PM 	Apr 22, 2022, 5:49:49 PM 	1 	
Starting Docker Application Container Engine... 	DockerStart 	Apr 22, 2022, 4:55:25 PM 	Apr 22, 2022, 5:49:39 PM 	4 	
Node condition FrequentContainerdRestart is now: Unknown, reason: NoFrequentContainerdRestart 	NoFrequentContainerdRestart 	Apr 22, 2022, 5:48:25 PM 	Apr 22, 2022, 5:48:25 PM 	1 	
Node condition FrequentDockerRestart is now: Unknown, reason: NoFrequentDockerRestart 	NoFrequentDockerRestart 	Apr 22, 2022, 5:47:25 PM 	Apr 22, 2022, 5:47:25 PM 	1 	
Node condition FrequentKubeletRestart is now: Unknown, reason: NoFrequentKubeletRestart 	NoFrequentKubeletRestart 	Apr 22, 2022, 5:46:25 PM 	Apr 22, 2022, 5:46:25 PM 	1 	
Node condition FrequentUnregisterNetDevice is now: Unknown, reason: NoFrequentUnregisterNetDevice 	NoFrequentUnregisterNetDevice 	Apr 22, 2022, 5:46:25 PM 	Apr 22, 2022, 5:46:25 PM 	1 	
Node gke-default-buildkite-main-ecd9ee7d-s9r6 status is now: NodeNotReady 	NodeNotReady 	Apr 22, 2022, 5:42:59 PM 	Apr 22, 2022, 5:42:59 PM 	1 	
```

Reading through the Buildkite docs, we can specifically address this case and force a retry: https://buildkite.com/docs/pipelines/command-step#automatic-retry-attributes, by watching for the -1 exit status: https://github.com/sourcegraph/sourcegraph/pull/34370

@jhchabran Depguard has [failed again, off a fresh branch from main](https://sourcegraph.slack.com/archives/C01N83PS4TU/p1650638888337509), so I had to disable it again. 

## 2022-04-19 

@jhchabran

Got asked to [help with a CD issue](https://sourcegraph.slack.com/archives/CMBA8F926/p1650372090576579), Renovate had not picked a new PR in 8h. We uncovered weird behaviour where Renovate was updating the same PR over and over, but had a failed check dating from when we removed `fd` from being installed in every agent. This had been fixed, but the PR having been opened for a month, it was still there. Closing it apparently got Renovate to open a new PR. Coincidentally, Cloudflare had issues in Germany has well, and Renovate seems to be hosted in Germany too. We have no clue on what the exact problem. $DURATION=45m 

## 2022-04-13

@davejrt

`Sourcegraph Cluster (deploy-sourcegraph) QA` tests failing on main with [this build](https://buildkite.com/sourcegraph/sourcegraph/builds/142212#4f3be801-87b7-4f74-baae-e68b54d3fc14/366-368). Some investigation showed [this commit](https://k8s.sgdev.org/github.com/sourcegraph/deploy-sourcegraph/-/commit/f0d13887f6a3f98bb4f518c29ce1d9afc37fc093?visible=3) broke the `low-resource` overlay. I fixed the the `low-resource` overlay with [this commit](https://k8s.sgdev.org/github.com/sourcegraph/deploy-sourcegraph/-/commit/0a8d240f889a8b0b2b0f4ee84fb63e7f21eced82?visible=3) and a [follow up commit](https://k8s.sgdev.org/github.com/sourcegraph/deploy-sourcegraph/-/commit/0a8d240f889a8b0b2b0f4ee84fb63e7f21eced82?visible=3) that adds a test to ensure that all overlays can be generated without error. 

## 2022-04-12 

@jhchabran

Valery reached me out in DM about [this build](https://buildkite.com/sourcegraph/sourcegraph/builds/141714#aa7664f3-69d2-4c8e-a6e5-fe8c8d5dad6d) and [its PR](https://github.com/sourcegraph/sourcegraph/pull/33701/files) that was behaving strangely. I mentioned him that it's better to reach out over our Slack channel instead btw. We're getting a cache hit as expected, but somehow yarn still wants to download things, ending up taking about 2/3 minutes whereas it should have been less than ten seconds like it does with the normal caching. We tried to reproduce it locally, but stopped when Valery saw that rebasing the `main` branch fixed the problem. We still took a look at what was happening in the PRs but did not find any relevant reason. This means that we have an edge case where it's possible to hit the cache but still have a slow build. The FP Team is aware of it, and they'll monitor it. I also showed Valery how to search in Grafana to find out occurences of Percy (the pupetteer tests) failing. $DURATION=30m

I inquired about the flakinness of those tests and what they plan to do about it. Valery told me that in Q3 they plan to swap Percy to something else. Due to the immediate impact of the flakiness, I suggested him that we should look into how often those tests are actually bringing forward regressions, which, if not that frequent, means we could turn those tests into soft failures, at least on the `main` branch for the time being, possibly adding a Slack notifications so their team can manually review is something is broken or not. That would bring down the flakes ratio down significantly. They're going to talk about it during they team sync today. $DURATION=10m

## 2022-04-11

@jhchabran

[No healthy agents running](https://sourcegraph.slack.com/archives/C01N83PS4TU/p1649658351891759). Agents are failing to start because of missing volume: `buildkite-git...`. It seems it cannot find the disk attached to the volume. I cannot launch again the pipepline to start fresh because no agents are available. I ended up creating an incident, because of the total disruption of the CI, see: [INC-94](https://sourcegraph.slack.com/archives/C03AHJL049M), which mentions with more details how I fixed it. $DURATION=60m

[Blocked main builds because of mismatch in generated files](https://sourcegraph.slack.com/archives/C02FLQDD3TQ/p1649669568010659). Joe thought it was a flake due to the log uploading failures and missed that there was an error in the generated files. He's opening a PR. $DURATION=5m

## 2022-04-06

@bobheadxi

[Thread discussing slow job setup times](https://sourcegraph.slack.com/archives/C01N83PS4TU/p1649240294407269). We need to mitigate the impact of slow cloning on stateless agents, since it's a huge contributor to slowness. [#30237](https://github.com/sourcegraph/sourcegraph/issues/30237)

[Thread about how draft => ready for review triggers a rebuild](https://sourcegraph.slack.com/archives/C01N83PS4TU/p1649254250381289). Side effect of frontend platform request for ready-for-review-only steps.

Agent on `default` queue [stuck for 14 minutes](https://sourcegraph.slack.com/archives/C02KX975BDG/p1649253119257889?thread_ts=1649252899.879369&cid=C02KX975BDG). Fix: add a configurable secondary queue to job dispatcher, which is used to look for scheduled jobs, and set `queue=default` [infrastructure#3208](https://github.com/sourcegraph/infrastructure/pull/3208) $DURATION=30m

[Discussion about `queue` usage](https://sourcegraph.slack.com/archives/C07KZF47K/p1649272455134259?thread_ts=1648590958.956749&cid=C07KZF47K). Follow-ups: [create pipeline creation guide](https://github.com/sourcegraph/handbook/issues/2993), [revert `job` and `stateless` queues to `standard`](https://github.com/sourcegraph/sourcegraph/issues/33238#issuecomment-1090779452)

## 2022-04-05

@bobheadxi

CI incident raised due to:

- Checkov checks breaking (dependency not found)
- Pupetteer finalize flaking
- Newly introduced Pupetter check flaking (chrome extension)
- npm install flake (503)

@bobheadxi mostly reading logs and pinging teams and asking he flakey steps be disabled. Nothing we can do about the infra flake for now, though [#26257](https://github.com/sourcegraph/sourcegraph/issues/26257) could mitigate. $DURATION=30m

We hit GitHub git-lfs bandwidth limits today. Turns out this was due to `git clone` implicitly downloading LFS assets, instead of only fetching LFS assets on `git-lfs fetch`, causing jobs that don't need LFS assets to download LFS assets - we need to explicitly set `GIT_LFS_SKIP_SMUDGE=1` to disable this behaviour. [#3206](https://github.com/sourcegraph/infrastructure/pull/3206)

Had to disable browser extension tests anyway because LFS ended up causing issues again later.

## 2022-04-04

@bobheadxi

Spotted two flakes. Didn't do anything except note their occurrence.

[build 140335](https://buildkite.com/sourcegraph/sourcegraph/builds/140335#803f8727-c5c0-484d-80fe-96f5b1f3c45f/181-2772), isolated occurrence according to [grafana query](https://sourcegraph.grafana.net/goto/7494m3snz?orgId=1).

```none
START| IndexStatus
     |     dbtest.go:111: testdb: postgres://postgres:sourcegraph@127.0.0.1:5432/sourcegraph-test-3601185618347116340?sslmode=disable&timezone=UTC
     |     store_test.go:524: expected index status
     |     dbtest.go:121: DATABASE sourcegraph-test-3601185618347116340 left intact for inspection
FAIL | IndexStatus (2.04s)
```

[build 140327](https://buildkite.com/sourcegraph/sourcegraph/builds/140327#70b8c0bd-a53c-48e5-a90c-0a3ec2bf704e/216-294), nothing we can do here.

```none
error An unexpected error occurred: "https://registry.npmjs.org/@xstate/fsm/-/fsm-1.4.0.tgz: Request failed \"522 undefined\"".
```

Check output and splitting steps so that the checks they run are more granular (most notably prettier, yarn). https://github.com/sourcegraph/sourcegraph/issues/33363

[Thread about submodule cloning](https://sourcegraph.slack.com/archives/C01N83PS4TU/p1649074147208429). [Buildkite discussion about disabling submodule fetch by default](https://github.com/buildkite/agent/issues/1053#issuecomment-989784531), potential workaround, though not implementing in favour of removing submodules entirely: [#33384](https://github.com/sourcegraph/sourcegraph/issues/33384)

Not part of devx-support, but @bobheadxi investigated the `prom-wrapper` alerting configuration issue raised in [INC-93](https://app.incident.io/incidents/93) ([writeup](https://github.com/sourcegraph/sourcegraph/issues/33394)) with the help of @michaellzc, the root cause of which turned out to be some [DevX work to refactor the `conf` package's `init` behaviour](https://github.com/sourcegraph/sourcegraph/issues/29222). The fix: https://github.com/sourcegraph/sourcegraph/pull/33398 , Delivery is doing the remainder of the investigation and follow-up. $DURATION=90m

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

@bobheadxi set up an alert on `seconds waited for expected dispach` (`secondsWaitedForExpectedDispatch`), which will notify us in #dev-experience-internal if the jobs dispatched by the dispatcher do not roll out sufficiently within the expected time frame. Visible in the [dispatcher dashboard](https://console.cloud.google.com/monitoring/dashboards/builder/a87f3cbb-4d73-476d-8736-f3bc1ca9f234?folder=true&organizationId=true&project=sourcegraph-ci&dashboardBuilderState=%257B%2522editModeEnabled%2522:false%257D&timeDomain=1h).

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
