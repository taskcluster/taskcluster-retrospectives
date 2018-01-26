# Retrospective: Spot Requests Pending Forever - 2018-01-19
#### Bugzilla Bug: https://bugzilla.mozilla.org/show_bug.cgi?id=1431742

## Background

Several things went wrong here.  There may be some connections between them.

 * EC2 had an issue that caused spot instances to not be evaluated, stretching intermittently from 2018-01-16 to 2018-01-19, primarily affecting us 19-1-2018
   * Per AWS, this was AWS throttling us for fraud detection, since we were cycling instances so quickly (OCC issue)
   * Something deployed on the 19th that was under-provisioned for this purpose
   * We have talked with them about it, they have "whitelisted" us
 * Deployment deleted older AMIs that were still configured in workerTypes
   * Accidentally deleted a docker-worker AMI that was in production, (so had to roll forward in a rush)
   * OCC deploy deploy Friday morning pruned AMIs beyond the last N (need more information here); a build to try to restore things started deleting known-working AMIs (so unable / difficult to roll back - deleted regions from workerTypes where couldn't find)
 * An [OpenCloudConfig deployment](https://github.com/mozilla-releng/OpenCloudConfig/compare/4198b97d65e15a067db2c412a1c9ccb815bed414...a3453c1e0cb373a381a53d0f3b3d733668cad41d) created instances that [halted themselves just after startup](https://bugzilla.mozilla.org/show_bug.cgi?id=1431742#c16)
 * A [docker-worker PR to shut down sooner for per-second billing](https://github.com/taskcluster/docker-worker/pull/352) may have caused instances to shut down after only a few minutes
 * A different [docker-worker PR for purging caches](https://github.com/taskcluster/docker-worker/pull/347) caused issues with retries on exit statuses
    * This only really happened because we deployed docker-worker with untested AMIs (because of earlier deletion of production AMIs)

     * Only affected tasks that failed (things that failed, were retried 5 times and resolved exception)


Root Causes:
 * We are bad at deprecating projects (docker-worker in this case)
   * workerType: worker-ci-test  is deployed with a very old AMI -- it was also accidentally deleted in one region
   * bad updates went out without adequate testing
 * No good rollback options for bad docker-worker AMIs
   * Will be helped by github releases support in docker-worker
   * Don't have a good way of tracking AMIs
 * Deleting production data

cleanup:

    - re-enabled regions disabled from windows workerTypes (regions were removed as we rollback AMIs and some regions where missing AMIs)


In TC redeployability:
1. we want services to do versioning of runtime configuration

    - not yet agreed (fair point)

1. we want to use terraform for managing most runtime configuration in services

    - not yet agreed at all

1. we want some locked runtime configuration to burned into the TC as deployment configuration


## Timeline
  - 2018-01-16 16:41:00 UTC ::::: Email on TC internal list identifying an unusual number of instances in the "pending evaluation" state and small backlog, but trees still open

  - 2018-01-17 10:05:00 UTC ::::: AWS support contacted
  - 2018-01-17 19:20:23 UTC ::::: Provisioning looks normal

  - 2018-01-18 06:10:00 UTC ::::: Response from AWS support asking for more information

  - 2018-01-19 10:02:00 UTC ::::: Aryx identifies high pending tasks for `gecko-1-b-linux` and `gecko-t-win7-32-gpu` with few instances running and lots pending
  - 2018-01-19 12:00:00 UTC ::::: OCC Windows AMIs start updating (time may be off by a few hours because arrived in #taskcluster at 14:16 UTC)
  - 2018-01-19 13:07:00 UTC ::::: OCC Windows AMIs finish
  - 2018-01-19 13:56:00 UTC ::::: More information provided to AWS
  - 2018-01-19 14:20:03 UTC ::::: Aryx closes trees for taskcluster worker shortage (jobs don't run or run delayed) due to spot requests not fulfilled
  - 2018-01-19 14:59:03 UTC ::::: Begin cancelling spot instance requests in the pending-evaluation state
  - 2018-01-19 15:56:54 UTC ::::: Disable spot requests in us-west-1 by removing from ALLOWED_REGIONS (AWS claims us-east-1 and us-west-2 are fixed but us-west-1 is not)
  - 2018-01-19 19:27:00 UTC ::::: Restore us-west-1 in ALLOWED_REGIONS
  - 2018-01-19 19:40:00 UTC ::::: Discovered that AMIs for several production workerTypes were deleted by a cleanup operation; wcosta doing a rebuild of docker-worker to produce new AMIs
  - 2018-01-19 20:12:15 UTC ::::: New AMIs deployed, instance provisioning resumes
  - 2018-01-19 20:31:00 UTC ::::: jhford detects instances shutdown after 5 minutes, due to new billing cycle code?
  - 2018-01-19 21:00:00 UTC ::::: AMIs without billing cycle patch deployed
  - 2018-01-19 21:24:28 UTC ::::: gps notes that windows tasks still not running
  - 2018-01-19 22:47:11 UTC ::::: jhford notes that ec2-manager is experiencing memory spikes, shuts down provisioner, switches to performance-M dyno
  - 2018-01-20 ~1:00    UTC ::::: jhford splits out different subsystems of ec2-manager resulting in fix for the memory issue, back to normal dynos

  - 2018-01-20 02:19:38 UTC ::::: cosmin re-opens trees briefly for a merge and to let a few pushes land
  - 2018-01-20 02:29:57 UTC ::::: trees close again to avoid generating a large backlog before state is known good
  - 2018-01-20 04:11:25 UTC ::::: trees reopen
  - 2018-01-20 18:33:12 UTC ::::: philor notes tasks are failing and being retried due to claim-expired
  - 2018-01-20 21:36:49 UTC ::::: philor closes trees for Bug 1431950 - failed jobs being retried as claim-expired (tracked down to an issue with purge statuses)

  - 2018-01-21 05:11:17 UTC ::::: new docker-worker AMIs deployed, old terminated
  - 2018-01-21 05:12:40 UTC ::::: trees reopen
  
  - 2018-01-22 16:00 approx UTC ::::: jhford confirms with EC2 that the ultimate cause of the EC2 outage was a result of the bad windows AMI
  

## Fallout

Trees were closed for most of a three-day period.

## Action Items

- [ ] reconsider work involving ec2 resources when ec2 is not working
- [ ] only delete images which are not in use and haven't been used in a while
- [ ] deletion of anything should be scripted, and the script reviewed
- [ ] better (thorough) tagging of AWS resources
- [ ] all image auto-purging should use time to decide if an image is to be deleted, not the count of previous images.
- [ ] rollback needs to be possible/easy
   - With docker-worker github release, rollback will be easier, even if AMIs are deleted
- [ ] re-land UCOSP project stuff
- [ ] staged roll-out for new AMIs?
- [ ] record configuration changes when they occur
   - provisioner already does this (event stream, not recorded)
   - notification channel would record everything (incl. secrets)
   - use Azure entities
   - in-channel notification of deployments
- [ ] more frequent mass-update of Windows AMIs?
- [ ] queue should track failure rates of worker types and worker ids
    - bad workertype: pendingTasks always 0
    - bad workerId: request provisioner kill it
- [ ] improve OCC knowledge/bandwidth
- [ ] don't let rob near busses

Sample docker-worker release https://github.com/taskcluster/docker-worker/releases/tag/v201801191813


## Thanks

* Aryx
* bstack
* coop
* cosmin
* gps
* grenade
* jhford
* jonasfj
* philor
* pmoore
* tomprince
* wcosta


