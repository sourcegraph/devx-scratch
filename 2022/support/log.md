# 2022 support log

DevX support rotation log. To add an entry, just add an H2 header with ISO 8601 format. The first line should be a list of everyone involved in the entry.

## 2022-03-22

@bobheadxi

- Node installation failures in deploy-sourcegraph-cloud pipeline [thread](https://sourcegraph.slack.com/archives/C02KX975BDG/p1647971014157899?thread_ts=1647957942.367989&cid=C02KX975BDG)
  - @daxmc99 deletes a bunch of stuff propagated from upstream depoy-sourcegraph, as well as the prettier step entirely: https://github.com/sourcegraph/deploy-sourcegraph-cloud/pull/15935

## 2022-03-21

@bobheadxi, @jhchabran

We have been running into sporadic, severe issues with stateless agent availability since we [rolled out stateless builds to 75% of `sourcegraph/sourcegraph` builds](https://github.com/sourcegraph/sourcegraph/pull/32751). See [thread](https://sourcegraph.slack.com/archives/C02MWRMAFR8/p1647838869486089). Caused some severe downtime in the morning, with [incident 92](https://app.incident.io/incidents/92)

- Increased `backoffLimit` to max: https://github.com/sourcegraph/infrastructure/pull/3176
- Discussed autoscaling implementation, landed on some misunderstandings: https://github.com/sourcegraph/infrastructure/pull/3177
- Scaled down agent rollout top 10%: https://github.com/sourcegraph/sourcegraph/pull/32840

Follow-up: potential revamp of autoscaler. https://github.com/sourcegraph/sourcegraph/issues/32843
