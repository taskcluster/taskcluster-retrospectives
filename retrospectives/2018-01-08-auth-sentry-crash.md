# Retrospective: Auth Sentry Crash 2018-01-08
#### Bugzilla Bug: [Bug 1428854](https://bugzilla.mozilla.org/show_bug.cgi?id=1428854)

## Background

Early in the investigation, we discovered that Auth had a bug -- from way back
when -- that meant reporting errors to Sentry from *within* the Auth service
caused an exception that crashed the web dyno. We hypothesized that this 

There was a change to `AZURE_ACCOUNT_KEY` for [bug
1422989](https://bugzilla.mozilla.org/show_bug.cgi?id=1422989) about 30 minutes
before the first event occurred, but this appears to be unrelated.  It was
switching from the primary to the secondary account key. The modification
caused a Dyno restart, but Dynos also restart automatically on a daily basis.

On December 7, we landed changes to implement [parameterized
roles](https://github.com/taskcluster/taskcluster-rfcs/issues/48). At the time
we knew that this would have a minor performance impact, but during the month
of December that increased load caused no major issues. Jonas noted that after
the 7th the service's CPU load was elevated on average, and much more "spiky".

Around January 4, we modified the `project-admin:*` role to use parameters.
Until this point, no roles with parameters existed.

We have investigated performance of the Auth service using load-testing
techniques, both in Heroku (on the staging instance) and locally. We found
nothing of note.

In each outage, memory usage goes to an average of 512M per Dyno (even for
Dynos allowing 1GB), CPU load goes to 1.0, and request latency goes to 30s (at
which time Heroku terminates the connection and returns 5xx errors). It's not
clear from the metrics which of these are the leading indicator. There is no
unusual logging output from the web Dynos.

After three outages, this event has not recurred for about 70 hours, although
we are seeing a light "background" of 5xx errors.

## Causes

We do not know the root cause of this issue. Ideas include:

 - Spectre/Meltdown mitigations on Heroku caused the failures
   - Heroku had several other incidents around their logging and metrics support during this timeframe, but did not announce any incidents affecting Dynos themselves
   - A Dyno where someone is attempting to use these exploits could potentially blow out the CPU cache repeatedly and cause massive performance dips in co-hosted Dynos
 - Spikes in load, or Heroku's Spectre/Meltdown mitigations, or both, caused sufficient slowdown to trigger failures due to high load
   - But note that doubling Dyno capacity (2x) and count (10) did not help
 - Some flaw in the parameterized roles implementation only manifests itself for certain requests, causing livelock and preventing the web Dyno from performing further work
   - Possibly related to GC pauses?
 - Some backend service (Sentry, Azure, AWS, Statsum) gets slow, tying up request handlers. Heroku limits the concurrent connections for a web Dyno, so this would eventually exhaust all available requests and cause a backlog.

## Timeline

8 Jan

  - 17:33:50 UTC ::::: dustin lands `AZURE_ACCOUT_KEY` change
  - 18:16:00 UTC ::::: dustin sees high-error-rate alerts from Heroku, alerts #taskcluster
  - 19:11:00 UTC ::::: dustin restarted all dynos on the queue service
  - 19:23:00 UTC ::::: dustin restarted all dynos on the index service
  - 19:25:00 UTC ::::: bstack and dustin restarted all dynos for everything
  - 19:45:00 UTC ::::: bstack and dustin determined that all pushes were handled (decision tasks started; still many tasks failed)

9 Jan

  - 15:09:54 UTC ::::: first 503 for today from auth
  - 15:10:00 UTC ::::: dustin notes 500 errors from queue, traces back to failing auth service
  - 15:15:49 UTC ::::: dustin restarts auth, increases web dynos from 5 to 10
  - 15:19:00 UTC ::::: dustin restarts all JS services -- things seem to return to normal
  - 16:57:00 UTC ::::: bstack changes from 1X to 2X dynos (still 10 of them)
  - 19:41:20 UTC ::::: dustin deployed [a fix](https://github.com/taskcluster/taskcluster-auth/compare/a23602f9...a23602f9) for the Sentry issue
    (the fix was developed on the 8th, but accidentally not deployed at that time)
  - 20:23:00 UTC ::::: issues begin again
  - 20:28:00 UTC ::::: auth dyno restarted, but *problem persists* -- restarting does not fix the issue
  - 20:39:00 UTC ::::: all other services restarted

## Fallout

The visible effect of this was that many API calls to the Auth service failed
with 503 errors, as Heroku could not find a working web dyno to service the
request.

Because every microservice validates every API call with the Auth service, this
in turn caused 500 errors from all services.  This likely caused tasks to fail
as well as causing failures of other API services.

This further caused lots of services to fail to get SAS credentials, meaning
that they were themselves failing with 500 errors.  That took some time to
understand and fix (by restarting all dynos).

Pushes to Mercurial repositories from the beginning of the incident through
19:04 UTC on 8 Jan failed to trigger Decision tasks.  However,
mozilla-taskcluster did start those Decision tasks after the incident resolved.

## Action Items

Action items should be achievable in short-medium term.

- [*] ensure that the Auth service can correctly send exceptions to Sentry

- [ ] ensure that the Auth service does not cause the app to crash if
  `sentryDSN` raisees an exception

- [ ] further optimize scope resolution

## Thanks

 * bastien
 * bhearsum
 * bstack
 * dustin
 * jhford
 * jonasfj
