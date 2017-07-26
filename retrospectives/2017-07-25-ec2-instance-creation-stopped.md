# Retrospective: EC2 Instance Creation Stopped 2017-07-25
#### Bugzilla Bug: http://bugzil.la/1384226

## Background

The Taskcluster AWS Provisioner is responsible for bidding for AWS EC2 "spot
instances" in response to demand for task execution.  It does so by way of the
EC2 Manager service, which manages the process of creating a bid and waiting
for it to convert into a running instance.

Due to the high task load from Gecko, if this instance-startup process fails,
we quickly fall behind and begin to accumulate pending jobs. The sheriffs react
by closing the trees to reduce or eliminate incoming load.

## Timeline

  - 2017-07-25T17:06 ::::: First report: `<tomprince> Do gecko-images workers not get provisioned automatically? https://tools.taskcluster.net/groups/cgKr8KoeTrCcqfgCP0FTkA/tasks/JwXjrsR7TiuI2hpu8y199A/details has been waiting for ~45m`
  - 2017-07-25T17:14 ::::: Observed with other Gecko workerTypes `<Aryx> hi, yes, seems like e.g. builder instances don't increase`
  - Observation: Provisioner is trying to create instances, EC2 Manager is logging that creation, mostly (exclusively?) in us-east-1
  - Observation: All recently created instances in us-east-1 are terminated; other regions don't have a lot of instances
  - Observation: Failed instances have no "instance startup log" and do not appear in papertrail
  - 2017-07-25T17:36 ::::: Modify AWS Provisioner config `ALLOWED_REGIONS` to omit us-east-1, on the hypothesis that the issue is limited to this region
  - 2017-07-25T17:46 ::::: No new instances created (not even terminated) since 17:36
  - Observation: AWS Provisioner iterations failing with an error in `src/check-for-ami.js` because `ec2` is undefined
  - 2017-07-25T17:57 ::::: Reverted the ALLOWED_REGIONS change since it is breaking the provisioner even worse
  - Observation: From the console for i-00034380ef23f0834, `State transition reason: Server.InternalError` and `State transition reason message: Client.VolumeLimitExceeded: Volume limit exceeded`
  - .. much discussion of volume sizes based on the list of about 6500 active volumes, determining that the limit we are hitting is total size of gp2 volumes
  - 2017-07-25T18:19 ::::: Dustin filed AWS request to increase volume limit from 400TiB to 800TiB in us-east-1 (it was increased from 200TiB to 400TiB in the other regions during the all-hands)
  - 2017-07-25T18:40 ::::: Deleted a bunch of old (2016 and earlier) "available" (unattached) volumes in us-east-1, saving a few hundred GB
  - 2017-07-25T19:03 ::::: deployed https://github.com/taskcluster/aws-provisioner/pull/155 to fix the `ALLOWED_REGIONS` issue
  - 2017-07-25T19:05 ::::: removed us-east-1 from `ALLOWED_REGIONS` again
  - 2017-07-25T19:07 ::::: instances starting in us-west-2; trees re-opened

AWS replied just after midnight UTC:

>  For a limit increase of this size, I will need to collaborate with our
>  Service Team to get approval. Please note that it can take some time for the
>  Service Team to review your request. This is to ensure that we can meet your
>  needs while keeping existing infrastructure safe.

The increase to 800TiB was approved at 2017-07-26T18:43.

## Fallout

Failure to schedule new jobs caused backlogs and thus a tree closure. Note that
since jobs were still occurring, try oculd be left open, so this prevented devs
from landing patches, but not from testing them in try.

## Action Items

* re-enable us-east-1 once the volume limit is raised
* provide a way to measure total volume usage in a region (even just manually - doing mental sums in the AWS console is hard)
* set up some kind of periodic monitoring of volume usage as compared to configured limits, with an alert when we get close
* monitor and log instance termination reasons

## Thanks

bstack jonasfj dustin garndt
