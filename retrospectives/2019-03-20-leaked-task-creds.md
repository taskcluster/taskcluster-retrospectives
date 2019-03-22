# Retrospective: Credentials in logs - March 20, 2019
#### Bugzilla Bug: https://bugzilla.mozilla.org/show\_bug.cgi?id=1536927

## Background

For ~40 minutes entire responses of queue.claimTask calls were logged via the TC service logging infrastructure. This included the credentials that workers use to perform action on behalf of tasks. These credentials expire after 20 minutes and carry scopes for anything the task itself can do, as well as scopes for manipulating the status of the specific task (e.g., to mark it resolved).  Some of these credentials, such as for decision and action tasks, did have very powerful sets of scopes. The full list of scopes and tasks is in the bug.  Note that these credentials are not the worker's credentials, and thus do not permit claiming of tasks.

The log data originated on a Heroku application (queue-taskcluster-net) which had a number of historical log destinations configured, sending data to services where we are no longer customers.  One of these was via http.  The two intended log destinations are Papertrail and (via an intermediary host that we control) GCP StackDriver.  The full list of people with access to these logs, and unintended log drains, is in the bug.

Soon after the leak was noticed, the queue's credentials were rolled, immediately invalidating  all of the temporary credentials it had assigned to tasks.  We also terminated all running workers in ec2 as a heavy-handed way of purging their caches, purged level-3 sccache and mbsdiff cache s3 buckets, and verified that hardware workers did not have any caches of concern.

Luckily, this was caught quickly and no releases were happening during the time period of the leak. Since the chance of compromise was low, and difficulty in detecting it was high, we chose to destroy anything that might still be tainted after the credentials were invalidated, rather than search for indications of compromise. This should have contained any fallout from this incident completely, and we don't expect any further issues.


## Timeline (PST)
    - 11:16 ::::: bstack deploys queue with new logging included
    - 11:54 ::::: bstack completes checks that deploy did not compromise authentication/authorization and goes to check new logs. Notices that some include credentials. Rolls back queue deployment.
    - 12:16 ::::: queue creds are rolled, invalidating all leaked creds
    - 1:52 ::::: we are nearly done assessing the damage when we realize that old log drains still existed for taskcluster services. This meant that the leaked credentials had left mozilla and even one of the drains was plain http. Decide to terminate all workers, etc.
    - 3:31 ::::: all cleanup work is complete, trees are reopened. Patch is landed to taskcluster to fix this issue.

## Fallout

Trees were closed for 3+ hours. Multiple engineers spent a large part of their day assessing fallout and fixing things. Otherwise, we believe nothing was affected.

## Action Items

Action items should be achievable in short-medium term.

- [x] [bug 1537907](https://bugzilla.mozilla.org/show_bug.cgi?id=1537907) Reconsider tighter schemas on log messages and/or always eliding fields with certain names
- [x] [bug 1537904](https://bugzilla.mozilla.org/show_bug.cgi?id=1537904) Remove all unused log drains and stop using papertrail so that we have one log destination
- [ ] [bug 1478941](https://bugzilla.mozilla.org/show_bug.cgi?id=1478941) Get worker-manager working in staging so that we can test this flow before going to production
- [x] remove no-longer-employed people from papertrail

## Thanks
tomprince, aki, owlish, dustin, ajvb, apop
