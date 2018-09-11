# Retrospective: DNS Outage 2018-08-27
#### Bugzilla Bug: [1486582](https://bugzilla.mozilla.org/show_bug.cgi?id=1486582)

## Background

As part of work to move the "redirect cluster" from its current nameservers to
Route53, registration for `taskcluster.net` was reconfigured to point to
nameservers that were not configured with correct information.  Resolution of
hostnames ending in `taskcluster.net` began to fail as the change propagated
throughout the domain-name system.

Once the issue was identified, the registration change was rolled back, with
the understanding that it would again take some time for the change to
propagate.  It became clear some time later that the TTL (the time for which a
value can be cached without checking for updates) for the incorrect data was
very high -- 24 hours.  Thus the propagation of the rollback would not complete
until 24 hours after the rollback was initiated.

When this became clear, correct configuration was added manually in Route53 so
that the cached data would lead to a successful name resolution.

## Timeline
  - ~19:00 UTC Monday Aug 27 - https://bugzilla.mozilla.org/show_bug.cgi?id=1485795 entailed a migration of the NS records for taskcluster.net (and many other domains) from Akamai to Route53 servers.  The change landed, but the records were not properly configured in Route53.
  - As cached records timed out, resolution of all taskcluster hostnames failed
  - Systems cannot reach taskcluster services, including Treeherder
  - Taskcluster services return 500 errors due to failure to contact auth.taskcluster.n,et etc.
  - 19:26 UTC - sheriffs detect infra issue, close trees
  - 19:39 UTC - infra team reverts registration change
  - 19:52 UTC - trees re-opened
  - 20:46 UTC - trees closed again, as problem is not fixed
  - 21:01 UTC - dustin restarts all TC services
  - 22:41 UTC - camd stops treeherder ingestion
  - .. much investigation by bstack, glandium, dustin
  - 11:14 UTC Tuesday Aug 28 - danielh lands change to add records to Route53
  - 11:57 UTC - pmoore restarts services
  - 12:22 UTC - trees re-open
  - (minor cleanup continues)

## Fallout

All trees were closed for 17 hours.

## Action Items

(As this involved teams other than the Taskcluster team, action items are handled elsewhere, and summarized here)

  - Tune TTLs to short times before making risky DNS changes
  - Ensure resources needed to perform a rollback are available (staff, access)

## Thanks

  - bstack
  - camd
  - danielh
  - dustin
  - ericz
  - glandium
  - joeyk
  - nli
  - pmoore
  - ryanc
