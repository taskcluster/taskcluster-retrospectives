# AWS Provisioner "stuck" 2016-07-21
* Bugzilla Bug: https://bugzilla.mozilla.org/show_bug.cgi?id=1288534

## Background

In responding to a credential leak, we re-generated all of the docker-worker AMIs.  Unfortunately, when deploying them, we neglected to use the PV AMIs for those workerTypes that use PV instances.  Most importantly, the desktop-test workerType.

## Timeline
* 2016-07-21 19:44 UTC ::::: new (incorrect) AMIs deployed via script
* 2016-07-21 20:06 UTC ::::: desktop-test pending >6000, with 783 instances according to aws-provisioner UI
* .. a great deal of confusion over which workerTypes need the PV AMIs, since the existing information had been overwritten ..
* 2016-07-21 20:37 UTC ::::: KWeirso informed of the pending (he was not concerned)
* 2016-07-21 20:51 UTC ::::: desktop-test AMIs set correctly
* 2016-07-21 20:55 UTC ::::: provisioner restarted to stop it from wasting time failing to create spot bids
* 2016-07-21 21:07 UTC ::::: noticed that qa-3-linux-fx-tests also has m1.medium; fixed that
* 2016-07-21 21:23 UTC ::::: gps has been waiting 17 minutes for a taskcluster-images instance
* 2016-07-21 21:34 UTC ::::: desktop-test pending at 3788, still reading 783 running instances in the UI
* 2016-07-21 21:51 UTC ::::: provisioner UI updated, 2217 instances running; desktop-test pending at about 3200
* 2016-07-21 21:58 UTC ::::: desktop-test maxCapacity set to 2200 so it won't try to provision more, and provisioner restarted again
* 2016-07-21 22:05 UTC ::::: taskcluster-images instance starts, gps is happy
* 2016-07-21 22:24 UTC ::::: desktop-test pending at 1926

The existing desktop-test entries were basically up to the task of chewing through the pending -- from 20:06 the pending continued to trend downward.  The issue was that the provisioner got stuck on desktop-test and didn't create the one instance of taskcluster-images that greg's work needed (or anyone else needing a less-used workerType).  The provisioner was effectively "stuck" from 20:06 until 22:05, with no useful status output and not starting instances.  It was doing things, just not *useful* things.

## Fallout

* Long pending time for test tasks
* Less-used workerTypes starved (crater, taskcluster-images, etc.)
* Trees did not close


## Action Items (should be achievable in short-medium term. immediately actionable):
* [garndt?] more clarity on AMI configuration (pv vs hvm)
* [dustn] AMI-sets will solve part of the issues present
* [jhford] dryrun of a configuration before running the provisioner -- john has a patch, but needs some work on it
* [dustin] finish this: https://github.com/taskcluster/aws-provisioner/pull/86/commits/5bcd621d0569553e17d3b7c681986903c823ff96
* [dustin] iteration loop for a single worker type? could we interleave?  suggestion for probability, logorithmic and round-robin approaches? could we make a decision and implement something here :)
* [bstack] could we have a job latency alarm? (the things that users notice)

* maybe use the lag/wait time to inform a lower bound on the minimum # of instances for a workerType

* [selena] made gecko-decision, taskcluster-images minCapacity-> 2

## Thanks
* wcosta
* pmoore
* dustin
* gps
