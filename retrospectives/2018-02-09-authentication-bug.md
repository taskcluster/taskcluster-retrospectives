# Retrospective: Authentication Bug Allowing Side Effects with Bad Credentials
#### Bugzilla Bug: [bug 1437131](https://bugzilla.mozilla.org/show_bug.cgi?id=1437131)

## Summary

As a result of a software bug, HTTP requests to many Taskcluster services that resulted a 401 response *achieved their side-effect anyway*.

### Background

HTTP requests to Taskcluster APIs are authenticated using [Hawk](https://github.com/hueniverse/hawk),
The authentication process determines the [client](https://docs.taskcluster.net/manual/design/apis/hawk/clients) making the request, and that client's [scopes](https://docs.taskcluster.net/manual/design/apis/hawk/scopes) are uesd to determine whether the request can proceed.
There are some additional details, but in any case there are two phases to the access-control process: authenticate the client, then determine whether the client can make the request.
If the first fails, the API returns a 401 response; for the second, a 403.

Taskcluster allows un-authenticated requests, with no Hawk headers at all.
Such a request is treated as a valid authentication as a null client, with no scopes.
Read-only methods that return public data require no scopes, so this mode is used frequently.

In the past few weeks, the Taskcluster team has been deploying an improvement to the implementation of the scope checks, called "[scope expressions](https://github.com/taskcluster/taskcluster-rfcs/issues/23)".
One of the goals of this project is to make required scopes more declarative, and thus prevent ambiguity and the possibility of bugs in the implementation of API methods.

Part of the effort to reduce the possibility of errors involved using exceptions, instead of return values, to handle failures.
Before scope expressions, API method implementations would call `req.satisfies(..)` and, if the method returned false, return early without performing the action.
The new design requires API methods to call `req.authorize(..)`, but the method raises exceptions on error and thus does not return any value.

### Authentication Bug

The implementation of `req.authorize(..)` handles the second phase (scope validation) correctly, but the code for the first phase (client authentication) was not modified.
On authentication failure, the unmodified form sent a 401 HTTP response directly and then returned false.

A typical API method looks like this:

```
api.declare({..}, (req, res) => {
  // ..                 extract parameter from request
  req.authorize(..); // check authentication and authorization
  // ..                 perform the requested operation
  res.reply({..});   // return the result
});
```

The return value of `req.authorize(..)` is not used, so after an authentication failure the requested operation is performed anyway.
Due to a quirk of [Express](https://expressjs.com/), however, the final `res.reply` (which calls Express's `res.status(200).json(..)`) does nothing because `req.authorize` has already sent an HTTP response.

### Ramifications

As a result of this bug, HTTP requests containing an invalid Hawk header resulted a 401 response *but achieved their side-effect anyway*.
This side-effect occurred *without any scope checks* -- equivalent to a request with a `*` scope.
Due to the Express quirk mentioned above, any response payload from the method was not sent.

While any method that modifies stored state was at risk from this bug, the most serious is `queue.createTask`.
The caller of this method specifies the scopes that should be available to the executing task.
These scopes are ordinariliy verified against the caller's scopes, but this bug bypassed such a check.
A malicious user could invoke `queue.createTask({scopes: ['*'], ..})` and the resulting task would execute with unlimited access.

### Detection

The bug was revealed when Marco Castelluccio noticed that the `notify.email()` method sent him an email even after responding with a 401 code due to expired credentials.
He brought the issue to one of the Taskcluster team members privately, quickly initiating an emergency response.

The bug was in production on some "less important" services starting January 24, and on the majority of services for 2-3 days.
Details are in the timeline section, below.

### Fix

The bug was tracked down quickly and fixed with a [short patch](https://github.com/taskcluster/taskcluster-lib-api/commit/488d9fd7e9569bb464039778409d50ac88a13f8e).
To avoid drawing attention to the issue, we chose to release the fix as a package on npmjs.com until all services were patched, only pushing to Github once the danger had passed.
The fix was quickly deployed to all of the affected services.

The bug was introduced in `taskcluster-lib-api` version 5.0.0 and fixed in version 6.0.4.
Version 5.0.0 was not widely deployed, and version 6.0.0 was released soon after.

### Remediation

We have no reason to think that this bug was exploited.
We used two approaches to verify this.

First, we scanned the Heroku service logs for 401 responses to non-GET methods and examined each one.
We excluded GET because the bug only allows side-effects and GET requests do not have side effects.
This dramatically reduced the number of log messages to examine.
With the exception of Hooks, logs for all "important" services were still available at the time.

The resulting list of requests contained the following:

 * Creation of Gecko Action tasks
 * Reruns and cancellations of Gecko tasks
 * A test task created with the Task Creator tool
 * A claim of a Gecko task by an address not in EC2

In general, 401's occur normally when users from Treeherder use expired credentials (until Feb 8th, Treeherder did not indicate this to users) or when users utilize command-line scripts with expired credentials.
We determined that the action tasks were made through Treeherder, and the resulting action tasks are well-formed and of no concern.
The re-runs and cancellations were also likely made via Treeherder, and at any rate their side effects are only denial of service or wasted work.
The test task and claim were traced down to known users who agreed that they had made the request.

The second approach was to compare, using backups, the run-time configuration of clients, roles, hooks, and secrets.
Here we found nothing unexpected.

## Timeline

  - 2018-01-24 21:42:43 UTC ::::: Bug deployed to Taskcluster-Github
  - 2018-01-24 23:16:40 UTC ::::: Bug deployed to Taskcluster-Hooks
  - 2018-02-06 23:35:04 UTC ::::: Bug deployed to Taskcluster-Auth
  - 2018-02-07 19:20:23 UTC ::::: Bug deployed to Taskcluster-Index
  - 2018-02-07 19:36:20 UTC ::::: Bug deployed to Taskcluster-Notify
  - 2018-02-07 19:43:32 UTC ::::: Bug deployed to Taskcluster-Purge-Cache
  - 2018-02-07 19:53:14 UTC ::::: Bug deployed to Taskcluster-Secrets
  - 2018-02-08 00:14:18 UTC ::::: Bug deployed to Taskcluster-Queue
  - 2018-02-09 07:45:38 UTC ::::: Marco alerts Taskcluster team to unexpected behavior of the `notify.email()` method.
  - 2018-02-09 19:00:00 UTC ::::: (approx) Fix deployed to all affected services

## Fallout

While this was an extremely serious bug, ultimately it caused no harm.
Taskcluster operations were not impacted, and trees remained open at all times.

## Action Items

 * Testing
    * regression test for exactly this issue in `taskcluster-lib-api`
    * integration test some critical API endpoints for other potential access-control-related errors
    * consider bundling access-control tests a pre-packaged suite in `taskcluster-lib-testing`
    * add more tests in Taskcluster-Auth's checkStaging script
 * QA
    * look for other early-return bugs like this
    * establish a list of things to check manually in auth, taskcluster-lib-api, taskcluster-lib-scopes, etc. when making critical changes
    * move signature validation to its own library which can be given much greater review scrutiny
 * Fixes
    * [Use an exception](https://bugzilla.mozilla.org/show_bug.cgi?id=1437191) to ensure execution cannot continue after `res.reportError`
    * [Disallow calling `res.send` multiple times](https://bugzilla.mozilla.org/show_bug.cgi?id=1437193) (working around the Express quirk)

## Thanks

* Marco Castelluccio
* Brian Stack
* Jonas Finnemann Jensen
* Dustin J. Mitchell
* Julien Vehent
