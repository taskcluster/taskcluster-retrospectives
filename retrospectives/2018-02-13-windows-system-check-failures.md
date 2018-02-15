# Retrospective: Windows AMIs failing system checks
#### Bugzilla Bug: [bug 1372172](https://bugzilla.mozilla.org/show_bug.cgi?id=1372172)

## Summary

A change to the internal, unpublished ssh key naming convention in the aws-provisioner caused OCC processes on new Windows instances to fail when they could not determine their worker type from the ssh key name.

## Background

Taskcluster uses [Open Cloud Config (OCC)](https://github.com/mozilla-releng/OpenCloudConfig) to configure Windows instances before they can accept work from the queue. The [AWS provisioner](https://github.com/taskcluster/aws-provisioner) spins up instances in AWS based on the size and contents of the Taskcluster queue. OCC relies on data from the provisioner to properly configure Windows instances to work as specific Windows worker types.

### The bug

There are many different worker types. Historically the provisioner has used distinct ssh key pairs for each worker type. OCC relied on the name of the ssh key pair to determine which worker type it was setting up. The key pair name was static for a very long time, but there was never a contract or provisioner API endpoint associated specifically with the ssh key pair, although there were other endpoints that could have been used instead to determine the worker type.

As part of adopting the new AWS spot pricing model, jhford landed a change to use a new, single, universal key pair per provisioner that applied to all workers. The new key had a new name, so after the new provisioner code was deployed to production on the morning of February 13, various OCC tasks related to the setup and maintenance of Windows instances in AWS started to fail.

### Detection

Aryx first noticed the Windows test backlog and reported the issue in #taskcluster. The build backlog was not noticed until almost a day later once the test backlog had begun to clear.

### Fix

Because the Taskcluster team is only vaguely aware of how OCC works, and because RelOps resources were not available for large parts of the debugging timeline, the fix came in three parts.

We have had other Windows outages due to the Windows impaired instance culling script not running, the most recent being [bug 1435503](https://bugzilla.mozilla.org/show_bug.cgi?id=1435503). jhford quickly found the first ssh key pair issue in the culling script and posted [a new version](https://bug1437973.bmoattachments.org/attachment.cgi?id=8950682).

When the test job backlog did not go down even after the culling script was run and new instances started appearing, Taskcluster team member started digging into the OCC code and, with grenade's help,  eventually found the [second issue affecting test worker configuration](https://github.com/mozilla-releng/OpenCloudConfig/commit/d7d2df5a174087bad52e7d3636ae92e043f999f0).

Once that change was deployed and the test job backlog started to go down, [Aryx noticed that we still had a build backlog](https://bugzilla.mozilla.org/show_bug.cgi?id=1438152). The number of build relative to the number of tests is very small &mdash; one build triggers many tests &mdash; made this easy to overlook. Now that we knew what to look for, grenade quickly found [the corresponding change for builders](https://github.com/mozilla-releng/OpenCloudConfig/commit/137c8c1b0e4b3927f15cf38ee4f9771894818221) and rolled out new AMIs.

### Remediation

The fixes detailed above have dealt with the proximal issue, but this prolonged outage highlights bigger issues that need to be addressed.

#### Taskcluster / Relops communication and SLAs

These two teams need to have a better working relationship. Mozilla relies on the Taskcluster platform for Firefox CI, and Taskcluster relies on the Relops team to provide the Windows capacity to meet those demands. The major intersection point right now is the provisioner which spins up the Windows capacity. To that end, I've asked jhford to meet with grenade to go over the OCC code to make sure we are providing all the necessary, *supported* endpoints that OCC requires to properly configure Windows workers. This informal code audit should hopefully uncover any other areas where the code currently relies on unpublished behavior or side-effects.

Knowing the interplay between the provisioner and OCC, it's important that we develop a better way to keep *both* teams informed when there are deployments to either tool. There should also be monitoring in place after provisioner deployments to ensure that work is still being scheduled and completed. It took over three hours after the provisioner deployment to even notice that something was wrong, and that's unacceptable.

We also need to establish a defined SLA with the Relops team, especially in response to tree-closing events. The trees were closed for 14 hours before a non-manager from Relops attempted to assist. grenade was ill and should not have been expected to help out, but we are glad he did, otherwise the trees might still be closed. From the Taskcluster side, we need to raise the alarm more quickly when we need assitance, but this has highlighted grenade as a SPoF for supporting our Windows infrastructure.

#### OCC and complexity

One of the reasons our Windows infrastructure is so hard to support is OCC. Windows configuration support has always been challenging, and in this respect OCC is no different. Members of the Taskcluster team found it hard to reason about the code when they tried to debug it in the absence of assistance from grenade.

Windows instances configured via OCC also require up to 30 minutes to complete the configuration process and begin running tasks. This makes any debugging process where we need make Windows configuration changes much longer than necessary.

We should examine whether OCC is meeting our current configuration needs &mdash; reliability, maintainability, debugability, and performance &mdash; and pursue alternatives if required.

pmoore provided the counter-example of the Windows worker configuration for NSS that is configured using a [simple, linear, PowerShell script](https://github.com/taskcluster/generic-worker/blob/master/worker_types/nss-win2012r2/userdata). These workers are available within 5 minutes of starting up. Perhaps there are complexities that prevent us from using such a straightforward system, but we should investigate and make those trade-offs known. If we cannot switch, we should debug the start-up performance issue.

#### Understanding the base AMI creation process

The [bug we used to track this outage](https://bugzilla.mozilla.org/show_bug.cgi?id=1372172) turned out not to be the real issue, but because we've had similar outages recently, it seemed like a good place to start. The root cause of that bug is still unknown. The script we use to recycle impaired Windows instances wallpapers over the symptom, but we need to debug the base AMI creation process to discover why all Windows instances eventually enter this impaired state. None of our non-Windows worker types have this type of issue.

Unfortunately, the base AMI creation process for Windows is completely unknown. More than one person on the Taskcluster and Relops teams has likened the process to "magic."

Q on the Relops team is responsible for creating the base AMIs for Windows. Q needs to meet with grenade and document/automate that process so that non-magicians can debug why these instances eventually become impaired.

## Timeline
  - 2018-02-13 13:18:43 UTC ::::: jhford deploys the single, universal ssh key pair change to ec2-manager
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
  - 2018-02-14 16:09:41 UTC ::::: bhearsum reports that his Windows build job has started, indicating that we were starting to see Windows build throughput
  - 2018-02-14 17:41:43 UTC ::::: inbound and autoland reopen

## Fallout

mozilla-inbound and autoland were closed for over 24 hours. While the try repo remained open, no Windows jobs were being being picked up during this time. Developers functionally lost a day of work.

## Action Items

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
* Peter Moore
* Q Fortier
* Rob Thijssen
* Sebastian Hengst
