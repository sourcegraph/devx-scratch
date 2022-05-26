# 2022 support log

DevX support rotation log. To add an entry, just add an H2 header with ISO 8601 format. The first line should be a list of everyone involved in the entry. For ease of use and handing over issues, **this log should be in reverse chronological order**, with the most recent entry at the top.

## 2022-05-24

@jhchabran

[The same exact thing happend to Joe Chen](https://sourcegraph.slack.com/archives/C01N83PS4TU/p1653375898615149), but this time we noticed very quickly that a CI ref file has not been updated and committing https://github.com/sourcegraph/sourcegraph/commit/a9b15bdc32409bbec9c5522c55d4b1f8b20c299c fixed the issue. As for why we're seeing this pattern of behaviour where `sg lint go` hangs if the `sg generate` commands results in a dirty repo, we don't know yet. What's for sure though is that the buffering of the ouptut which makes it appear only once the command has exited is really unpractical when it comes to debugging, and we need to fix this asap.  DURATION=30m .

@jhchabran 

[Thorsten noticed that the `main` branch is broken due to the 3.40.0 release](https://sourcegraph.slack.com/archives/C032Z79NZQC/p1653399008176829), it went unnoticed because the support handle for the team is set this week on Robert, which at this time is sleeping. After some investigation, we noticed that Alex Ostrikov merged a commit 6 hours ago that introduced a backward incompatible change, albeit a peculiar one: the database change itself is backward compatible, but not the tests, which were accounting for faulty behaviour. The fix was to [port the flake files to the new 3.40.0 version](https://github.com/sourcegraph/sourcegraph/pull/35942) and [we added along the way a word of warning about this edge case](https://github.com/sourcegraph/sourcegraph/pull/35945). DURATION=45m

@jhchabran

[An arm64 image was used as a base for CAdvisor](https://sourcegraph.slack.com/archives/CMBA8F926/p1653411739808069) and broke the main branch. [Reverted the PR](https://github.com/sourcegraph/sourcegraph/pull/35959) and advised the user on how to fix this. Also, the CI pipeline generator isn't detecting changes in the images and their PR did not catch this. DURATTION=20m

## 2022-05-23

@jhchabran

[Eric Fritz had trouble with a PR which added new go generate statements](https://sourcegraph.slack.com/archives/C01N83PS4TU/p1653322924295219) We paired with William on this and found out that the result are inconsistent in local, `sg generate` completes, but sometimes `sg lint go` hangs. On CI `sg lint go` consistently hangs. As it was pretty late for us, we unblocked Eric by giving him a branch with a patched version that outputted things straight to stdout. Strangely, forcing the `sg generate` code to produce output did work consistently, which hints at some buffering issue. DURATION=60m.  

## 2022-05-18 

@jhchabran

[Valery mentioned that GitStart is blocked on their PR](https://sourcegraph.slack.com/archives/C01N83PS4TU/p1652866114007209) with issuse related to node versions. The problem was rather simple, their pipeline was simply not calling `asdf` at all. As we recently rolled out `asdf` installation into a plugin, that was a rather straightforward fix: https://github.com/sourcegraph/eslint-config/pull/249 DURATION=30m

## 2022-05-12

@jhchabran

Investigated [the issue the failing back compat tests](https://sourcegraph.slack.com/archives/C01N83PS4TU/p1652311475381819?thread_ts=1652294119.054949&cid=C01N83PS4TU) and found out that there is an [where the go asdf plugin](https://github.com/kennyp/asdf-golang) reverts to Go 1.18.1 for some obscure reason. 

I first reproduced the error on Buildkite to establish that it's not transient somehow and managed to reproduce it locally by running the back-compat/test.sh script manually with the same arguments that are present in the failing step. I had to tweak things a bit to make it work locally, notably adjusting the `find` command, disabling asdf installation part entirely to circumvent issues with some tools not being available for `arm64`. From there, after triple checking that the issue was indeed coming from the `go test` command, I started to bisect the package list to understand which one was causing the error. After a few tries, I managed to get a zero exit code by excluding the `lib/codeintel/tools` folder and finally isolated `lib/codeintel/tools/lsif-repl` to be the culprit. 

```
# This will show exit code 2
./dev/ci/go-backcompat/test.sh exclude github.com/sourcegraph/sourcegraph/enterprise/internal/codeintel/stores/dbstore github.com/sourcegraph/sourcegraph/enterprise/internal/codeintel/stores/lsifstore github.com/sourcegraph/sourcegraph/enterprise/internal/insights github.com/sourcegraph/sourcegraph/internal/database github.com/sourcegraph/sourcegraph/internal/repos github.com/sourcegraph/sourcegraph/enterprise/internal/batches github.com/sourcegraph/sourcegraph/cmd/frontend github.com/sourcegraph/sourcegraph/enterprise/internal/database github.com/sourcegraph/sourcegraph/enterprise/cmd/frontend/internal/batches/resolvers 


# This will show exit code 0 (noted the final excluded package)
./dev/ci/go-backcompat/test.sh exclude github.com/sourcegraph/sourcegraph/enterprise/internal/codeintel/stores/dbstore github.com/sourcegraph/sourcegraph/enterprise/internal/codeintel/stores/lsifstore github.com/sourcegraph/sourcegraph/enterprise/internal/insights github.com/sourcegraph/sourcegraph/internal/database github.com/sourcegraph/sourcegraph/internal/repos github.com/sourcegraph/sourcegraph/enterprise/internal/batches github.com/sourcegraph/sourcegraph/cmd/frontend github.com/sourcegraph/sourcegraph/enterprise/internal/database github.com/sourcegraph/sourcegraph/enterprise/cmd/frontend/internal/batches/resolvers github.com/sourcegraph/sourcegraph/lib/codeintel/tools/lsif-repl

# Go back to the branch, resetting the local mess
git reset --hard HEAD && rm -Rf migrations && git co . && git co main-dry-run/debug-back-compat
```

Surprisingly, this package has no test and is just a `main.go`. Running a `git blame` on the file showed that the last change there was a fix from Keegan to make it work with Go `1.18.1`. But the back compat tests are supposed to be running Go `1.17.5`. Running `go version` within the `lsif-repl` folder showed for some reason, even if the `/.tool-versions` specifies Go `1.17.5`, it's actually Go `1.18.1` that is currently running. 

This put me on track toward an `asdf` related issue, a few prints in `asdf` code confirmed it. There is an [issue](https://github.com/kennyp/asdf-golang/issues/79) about someone reporting a similar issue. Reading the doc showed that there is some questionable behaviour if the `legacy_version` setting is enable. Disabling it in my `~/.asdfrc` solved the issue and now Go `1.17.1` is the current version, as expected, within the `lsif-repl` folder. 

Ran a quickbuild over https://buildkite.com/sourcegraph/sourcegraph/builds/147181 to confirm that we can just drop the `.nvmrc` file and opened a PR with a new agent image that has that setting disabled.

---

Failures were reported on the deploy-sourcegraph-cloud pipeline, due to `asdf` installation of `helm` failing. It was caused by an HTTP timeout. I submitted two PRs to fix these on the plugin themselves over https://github.com/Antiarchitect/asdf-helm/pull/12 and https://github.com/luizm/asdf-shfmt/pull/7.

## 2022-05-11

@jhchabran

A [PR broke the main branch](https://sourcegraph.slack.com/archives/C07KZF47K/p1652270071646589?thread_ts=1652268753.495069&cid=C07KZF47K) and I had to revert it. I asked the author if he knew what to do in case of being mentioned by Buildchecker, just out of curiosity and he did. That person did not know about the other run types btw (and he's been with us since Nov '21).

I submitted a quick PR along the way for the wording in Buildchecker, trying to foster more autonomy from our users: https://github.com/sourcegraph/sourcegraph/pull/35291

@bobheadxi @davejrt

Major outage with asdf caching `tar` failures. Even with that [fixed](https://github.com/sourcegraph/sourcegraph/pull/35334/commits/232fd75c46a32f3a0e0d12633266190de9b7ee02), [back-compat tests continued to fail](https://sourcegraph.slack.com/archives/C01N83PS4TU/p1652308349109149?thread_ts=1652294119.054949&cid=C01N83PS4TU). Eventually [disabled back-compat tests](https://github.com/sourcegraph/sourcegraph/pull/35334#issuecomment-1124362668). $DURATION=90m

## 2022-05-04

@bobheadxi

Made `pr-auditor` required in `sourcegraph/sourcegraph` on [request](https://sourcegraph.slack.com/archives/C01G34KFYE4/p1651677045477279?thread_ts=1651665256.667259&cid=C01G34KFYE4). Noticed issue where `pr-auditor` checks do not persist, and PRs get stuck on "waiting for status" - we [can't set status by branch](https://github.com/sourcegraph/sourcegraph/pull/34913#issuecomment-1117563565), so we need to register the action to run on the `synchronize` event ([#34914](https://github.com/sourcegraph/sourcegraph/pull/34914)) i.e. when a push happens as well. We'll need to propagate this fix to all repos with a batch change if more repos make this a required check. $DURATION=15m

Hadolint script broke. The shell script was a bit cryptic so I just moved the whole thing into `sg`: [#34926](https://github.com/sourcegraph/sourcegraph/pull/34926) $DURATION=30m

A few teammates reported occasionally running into *kernel* panics with `sg start`. Consensus seems to be that it's probably not directly related to `sg`, but might be worth looking into - for now it appears infrequent and I can't make much sense of it. [Thread](https://sourcegraph.slack.com/archives/C01N83PS4TU/p1651696165060899), [core dump](https://sourcegraph.slack.com/archives/C01N83PS4TU/p1651699647925089?thread_ts=1651696165.060899&cid=C01N83PS4TU) $DURATION=15m

Worked on a dependency version guard for an incident a few weeks ago: [#34929](https://github.com/sourcegraph/sourcegraph/pull/34929) $DURATION=30m

## 2022-05-02

@jhchabran Noticed that the builds were stuck this morning, because a new version of rust protobuf-codegen has been released and the dev/generate.sh script wasn't setting any expectation about which version to get. As the new release was publish, all our builds started to fail. I have put up a PR https://github.com/sourcegraph/sourcegraph/pull/34756 with the fix. $DURATION=20m

## 2022-04-28

@bobheadxi

Spent a lot of time trying to make firewall popups go away. Pursued the firewall command approach for a long time (https://github.com/sourcegraph/sourcegraph/pull/34680), before giving up and finding an infinitely easier solution: https://github.com/sourcegraph/sourcegraph/pull/34714 $DURATION=240m

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
