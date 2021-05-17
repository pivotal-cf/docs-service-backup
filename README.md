
# About Branches

| Branch name     | Use for|
|-----------------| ------|
| master          | This branch is staging: https://docs-pcf-staging.sc2-04-pcf1-apps.oc.vmware.com/svc-sdk/service-backup/0-n |
| v18.4.x         | live http://docs.pivotal.io/svc-sdk/service-backup/18-4/ |
| v18.3.x         | live http://docs.pivotal.io/svc-sdk/service-backup/18-3/ |
| v18.2.x         | live http://docs.pivotal.io/svc-sdk/service-backup/18-2/ |
| v18.1.x         | live http://docs.pivotal.io/svc-sdk/service-backup/18-1/ |
| v18.0.x         | live http://docs.pivotal.io/svc-sdk/service-backup/18-0/ |
| v17.2.x         | live http://docs.pivotal.io/svc-sdk/service-backup/17-2/ |
| v17.1.x         | live http://docs.pivotal.io/svc-sdk/service-backup/17-1/ |
| v17.0.x         | live http://docs.pivotal.io/svc-sdk/service-backup/17-0/ | 
| v16.0.x         | live http://docs.pivotal.io/svc-sdk/service-backup/16-0/ | 

# Book repo for publishing this content

Book repo: [**docs-book-service-backup**](https://github.com/pivotal-cf/docs-book-service-backup/).

# Process for working in this repo

## Submit changes to the docs through PRs

Instructions on doing a PR: https://docs.google.com/a/pivotal.io/document/d/14Go0uCj20BFMBzL2ddEKsZp-GONhVp0yr2cEFSskWnQ/edit?usp=sharing

## New (unpublished) releases

1. Commit new feature docs to **master** only.

2. When the release is ready to publish, the docs team will create a new live/production branch from master, for example, v19.1.x.

## Fixes and enhancements on master

1. For fixes and enhancement, make PRs to **master**.

2. Tell the docs team in the PR, if it should be applied to other specific branches (or make additional PRs to those branches).

## Fixes on branch only

If the fix or enhancement is only relevant to a particular branch, then apply the change to that branch only.
