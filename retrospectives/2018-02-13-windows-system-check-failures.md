# Retrospective: Windows AMIs failing system checks}
#### Bugzilla Bug: [bug 1372172](https://bugzilla.mozilla.org/show_bug.cgi?id=1372172)

## Summary

A change to the internal, unpublished ssh key naming convention in the aws-provisioner caused OCC processes on new Windows instances to fail when they could not determine their worker type.

## Background

${any information needed to understand the issue. retrospectives are shareable outside of the team, so try to make it understandable to someone with less context than us!}


## Timeline
  - 2018-02-13 16:37:31 UTC ::::: Aryx reports heavy backlog of Windows tests in #taskcluster
  - 2018-02-13 16:48:42 UTC ::::: ebalazs (sheriff) closes inbound, autoland due to backlog
  - 2018-02-13 16:53:54 UTC ::::: dustin, jhford, pmoore begin investigating
  - 2018-02-13 17:07:53 UTC ::::: pmoore tries terminating all instances to no avail
  - 2018-02-13 17:26:56 UTC ::::: jhford discovers that the impaired instance termination script is using the old ssh keypair name to find the worker types
  - 2018-02-13 18:55:27 UTC ::::: jhford posts updated script for review in https://bugzil.la/1437973 and sets it up to run as a hook
  - 2018-02-13 20:19:35 UTC ::::: dustin reports that trees are still closed because the backlog is not going down
  - 2018-02-13 20:53:24 UTC ::::: dustin terminates all Windows workers
  - 2018-02-13 22:04:14 UTC ::::: Aryx reports that the backlog is growing
  - 2018-02-13 22:25:41 UTC ::::: Email alert from papertrail: logging rate 5,614 msgs/second (generic-worker: Checking for C:\dsc\task-claim-state.valid file...)
  - 2018-02-13 23:27:33 UTC ::::: coop pages fubar
  - 2018-02-13 23:36:00 UTC ::::: (approx) coop pages grenade
  - 2018-02-13 23:55:10 UTC ::::: coop pages markco
  - 2018-02-13 23:47:09 UTC ::::: bstack and jonas start digging into OCC code
  - 2018-02-14 00:08:00 UTC ::::: fubar attempts redeploying the Windows AMIs
  - 2018-02-14 02:21:00 UTC ::::: bstack opens support case with AWS (https://console.aws.amazon.com/support/v1#/case/?displayId=4875481371)
  - 2018-02-14 02:49:00 UTC ::::: AWS support replies to open case, suggesting we try creating a new Elastic Neetwork Interface (ENI)
  - 2018-02-14 03:30:00 UTC ::::: (approx) coop tries to connect new ENI to an existing instance to RDP in, but doesn't succeed
  - 2018-02-14 06:46:08 UTC ::::: Q arrives on scene, tries to recover capacity by rebooting some instances (~200) that had been disconnected for >6 hours
  - 2018-02-14 07:18:56 UTC ::::: pmoore arrives on the scene
  - 2018-02-14 07:24:19 UTC ::::: New instances still aren't claiming any tasks
  - 2018-02-14 07:39:13 UTC ::::: grenade arrives on the scene, joins vidyo call with jonas and pmoore to debug
  - 2018-02-14 07:51:20 UTC ::::: workerType mismatch in OCC code (as derived from ssh key name) is fingered as culprit for tester AMIs
  - 2018-02-14 08:06:00 UTC ::::: grenade lands patch to handle changed ssh key naming convention, kicks off deployment of new tester AMIs
  - 2018-02-14 09:39:58 UTC ::::: pmoore bumps the maxCapacity for workers to clear the backlog more quickly. Test backlog begins to drop.
  - 2018-02-14 12:32:07 UTC ::::: Aryx reports that Windows builds are not running - https://bugzilla.mozilla.org/show_bug.cgi?id=1438152
  - 2018-02-14 13:58:08 UTC ::::: grenade finds the corresponding workerType mismatch for builders
  - 2018-02-14 15:03:41 UTC ::::: grenade kicks off new Windows AMI deployment with the workerType fix
  - 2018-02-14 16:09:41 UTC ::::: bhearsum reports that his Windows build job has started, indicating that we're starting to Windows build throughput
  - 2018-02-14 17:41:43 UTC ::::: inbound and autoland reopen

## Fallout

mozilla-inbound and autoland were closed for over 24 hours. While the try repo remained open, no Windows jobs were being being picked up during this time. Developers functionally lost a day of work.

## Action Items

Action items should be achievable in short-medium term.

- re-affirm communication between relops and Taskcluster teams, especially around usuable API endpoints and timing of deployments
- audit OCC code to determine whether other non-published interfaces or side-effects of published interfaces are being used
- move impaired instance cron script to a hook, or at the very least onto managed hardware/instance/dyno
- document the base AMI creation process to (hopefully) shine light on the root cause for bug 1372172

## Thanks

* Brian Stack
* Chris Cooper
* Dustin J. Mitchell
* John Ford
* Jonas Finnemann Jensen
* Kendall Libby
* Mark Cornmesser
* Peter Moore
* Q Fortier
* Rob Thijssen
