# 2022 DevOps log

Log for DevOps tasks handled by DevX teammates, old and new.
To add an entry, just add an H2 header with ISO 8601 format.
The first line should be a list of everyone involved in the entry.
For ease of use and handing over issues, **this log should be in reverse chronological order**, with the most recent entry at the top.

## 2022-07-04 deploy-sourcegraph-cloud renovate scheduling

@bobheadxi

It was noted that Grafana and Prometheus have not been updated in several weeks.
I think the main issue is that the schedule was provided in a crontab format, when [Renovate scheduling docs make no indication it supports crontab](https://docs.renovatebot.com/key-concepts/scheduling/) - the syntax seems documented here: https://breejs.github.io/later/parsers.html#text

I decided to just push directly to the preprod branch with some fixes (pull requests are a nightmare to merge): https://github.com/sourcegraph/deploy-sourcegraph-cloud/pull/17105/commits/24f9d5efb82d81a2a307b2a2b5ff4c9b2a33d38e , tl;dr

```diff
-     schedule: ["* 15 * * 1-5"],
+     schedule: ["after 3pm on monday"],
```

It also seems the schedule has changed - what used to by "Daily" is actually weekly, and so on, so I've updated the names of the Renvoate jobs as well. I then manually bumped Prometheus and Grafana to `158142_2022-07-04_0ce424594726` by hand in pre-prod.

All this will be merged in https://github.com/sourcegraph/deploy-sourcegraph-cloud/pull/17105

## 2022-07-04 GCP Cost Reduction and CD improvements

@kalanchan

Working with Daniel and Manny to identify areas of improvement to dotcom and preprod deployments

- TLDR; commits in preprod branch get squashed and causes conflicts when more image tags get changed and need to be merged into release(main) again.
- Potential fix wouild be to run Github Actions that creates a brand new branch on run time instead of having renovate make updates to preprod. Could incorporate `sg` usage here instead of writing custom script. Dax has some progress [here](https://github.com/sourcegraph/deploy-sourcegraph-cloud/pull/16390), but it's not perfect due to formatting/linting issues.

Regarding GCP cost reductions, we'll tackle the low hanging fruit first identified by RafG in his [doc](https://docs.google.com/document/d/1FQScUkS6fyBfW__dG0WiHmXH6-Fl_4zWwMTY9IWASSI/edit#heading=h.na988urmj90p).  
[Older Version](https://docs.google.com/document/d/1qEnD-1RQ0tD_C-kKiLngKnWA1-kUStOu4xU0YGlKQrM/edit#heading=h.m2y4u5mmwaiw)

- get understanding of usage vs provisioned resources
- identify peak periods and consumption
- involve targetted service's stakeholders to see if we can reduce infrastructure provisioning
- build foundations for FinOps. [FinOps is to Dollars What DevOps is to Code](https://devops.com/how-finops-can-optimize-cloud-costs-and-drive-innovation/)
- [eventual "observability" into cloud cost](https://cloud.google.com/blog/topics/developers-practitioners/optimizing-your-google-cloud-spend-bigquery-and-looker)

## 2022-07-04 Dotcom crashed due to invalid site config

@sanderginn

Opened a [PR](https://github.com/sourcegraph/deploy-sourcegraph-cloud/pull/17063) addressing invalid JSON and YAML that gets committed and deployed. In this case, it led to brief interruptions of sourcegraph.com.
