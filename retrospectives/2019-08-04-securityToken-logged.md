# Retrospective: AWS-Provisioner securityToken Logged - Aug 4, 2019
#### Bugzilla Bug: [bug 1541672](https://bugzilla.mozilla.org/show_bug.cgi?id=1541672), [bug 1571300](https://bugzilla.mozilla.org/show_bug.cgi?id=1571300)

## Background

The [aws-provisioner service](https://github.com/taskcluster/aws-provisioner) gives Taskcluster credentials to workers so that they can execute tasks.
In order to bootstrap secure identification of workers, the provisioner passes a "securityToken" in each worker's user-data.
The worker passes that security token back to the provisioner in an `awsProvisioner.getSecret` REST API call in exchange for Taskcluster credentials.
Once the credentials are downloaded, the worker calls `awsProvisioner.removeSecret`, after which time the secret can no longer be used to get credentials.

In [bug 1541672](https://bugzilla.mozilla.org/show_bug.cgi?id=1541672), docker-worker was logging its userdata.
In [bug 1571300](https://bugzilla.mozilla.org/show_bug.cgi?id=1571300), aws-provisioner was logging the securityToken as it generated it.
The latter bug also implicates docker-worker for logging the securityToken, but in fact this is the same bug, just not deployed due to difficulties deploying docker-worker.

## Timeline

  - 2014-07-20 ::::: docker-worker logging added to codebase
  - 2015-07-16 ::::: aws-provisioner logging added to codebase
  - 2019-04-03 ::::: docker-worker logging identified
  - 2019-04-15 ::::: docker-worker fix landed in version control (but not deployed)
  - 2019-08-04 ::::: aws-provisioner logging identified
  - 2019-08-05 ::::: aws-provisioner logging fixed and changes deployed
  - 2019-08-06 ::::: unused log drains removed from aws-provisioner configuration in Heroku

## Fallout

The secretToken is usable from the time it is generated (when aws-provisioner logs it) until the time it is removed (moments after docker-worker logs it).
This duration is the time it takes to create a worker instance and for that instance to start up -- on the order of seconds to minutes.

Log data is treated as confidential but is distributed fairly widely.
Worker logs and aws-provisioner logs are sent to papertrail
Logs from aws-provisioner were also, unexpectedly, sent to several log services for which we no longer have active subscriptions, some over http.
This has been addressed in [bug 1571879](https://bugzilla.mozilla.org/show_bug.cgi?id=1571879).

## Mitigations

In newer deployments, TC uses structured logging which documents each field, meaning fields like "securityToken" stand out in review.
The library supporting this also "elides" properties with names like "password" and "credentials" just in case they do get through review.
Logging configuration is also more tightly controlled, with service logs going to a centralized location (Stackdriver in GCP) and managed by the cloud operations team.

## Action Items

Action items should be achievable in short-medium term.

- [ ] ${first item to help make this not happen again}
- [ ] ${second item to help make this not happen again}
- [ ] ${...}


## Thanks
* @tomprince
* @ameihm0912
