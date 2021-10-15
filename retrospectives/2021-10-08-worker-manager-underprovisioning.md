# Retrospective: 2021-10-08 Taskcluster worker-provisioner outage 2021-10-08 23:01 - 2021-10-09 02:40 UTC
#### Bugzilla Bug: https://bugzilla.mozilla.org/show_bug.cgi?id=1734959

## Background

FirefoxCI taskcluster cluster stopped spinning up new workers.


## Timeline
  - 2021-10-08 23:01 ::::: Firefoxci cluster worker-manager stops spinning up new AWS instances
  - 2021-10-08 23:06 ::::: taskcluster-worker-manager-provisioner (GKE pod) logs a WARNING-level message
  - 2021-10-08 23:06 ::::: taskcluster-worker-manager-provisioner (GKE pod) logs an ERROR-level message
  - 2021-10-08 23:06 ::::: taskcluster-worker-manager-provisioner CPU drops to all-but-idle
  - 2021-10-08 23:06 ::::: taskcluster-worker-manager-provisioner begins logging "loop-interference" messages every ten seconds
  - 2021-10-09 01:03 ::::: NarcisB (sheriff) pings Aki about Gecko Decision + Retrigger tasks not starting
  - 2021-10-09 01:16 ::::: Aki starts #taskcluster-cloudops thread
  - 2021-10-09 01:29 ::::: Aki files https://bugzilla.mozilla.org/show_bug.cgi?id=1734959
  - 2021-10-09 01:34 ::::: Aki pages taskcluster-firefoxci pagerduty
  - 2021-10-09 01:35 ::::: Owlish notes that GCP is having an outage
  - 2021-10-09 01:38 ::::: cvalaas starts investigating
  - 2021-10-09 02:01 ::::: cvalaas kills taskcluster-worker-manager-provisioner pod (causing k8s deployment to start a new one)
  - 2021-10-09 02:09 ::::: worker-manager has spun up 20 gecko-3/decision workers
  - 2021-10-09 02:40 ::::: issue considered resolved

## Fallout

Impact: Without human intervention after-hours, the FirefoxCI taskcluster cluster would have essentially stopped running tasks until human intervention. All Firefox desktop, Firefox for Android, Focus, Addons, and other taskcluster-based projects’ development CI, nightlies, and any release processes would have been halted.

[aki] Aki suspects that the Iterate configs for worker-manager means that we exited the worker manager thread/process when we received too many errors from the GCP API. Cvalaas disagrees, since there was a continued heartbeat.

- What happened?
- Is there something in the configs or code we can change to have the app continue working in this type of event?
- Or can we detect that the worker-manager is non-operational and notify people?

[cvalaas] Based on the continuing "loop-interference" log messages, it looks like the provisioner code was executing, but immediately exiting the loop (intentionally) so only one instance of the loop runs at any one time (https://github.com/taskcluster/taskcluster/blob/66941613e4242d69dd2aff3fb560359eb633d59c/services/worker-manager/src/provisioner.js#L66-L69).
During normal operations, the "loop-interference" message is not seen.
This makes me suspect that the loop died at one point (when it hit the iterate timeout error?) but didn't clean up after itself properly? The process didn't exit completely and the provisioningLoopAlive variable was not reset to false.

Further sleuthing and details by @jwhitlock in the bug: https://bugzilla.mozilla.org/show_bug.cgi?id=1734959


## Action Items

Action items should be achievable in short-medium term.

- [ ] Monitor the number of workers created per [minute/hour]? Alert on low numbers?
  - Is this already a tracked metric? If not, how do we get it?
- [ ] Change the provisioner code to crash the process if loop-interference is seen repeatedly?
  - worker-manager work is tracked here: https://github.com/taskcluster/taskcluster/issues/5003


## Thanks
- @aki / @escapewindow
- @jwhitlock
- @owlish
- @mostlygeek
