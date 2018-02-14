# Retrospective: Windows AMIs failing system checks}
#### Bugzilla Bug: [bug 1372172](https://bugzilla.mozilla.org/show_bug.cgi?id=1372172)

## Summary

New Windows AMIs began failing their initial system checks in EC2, preventing the generic-worker from starting. This led to high pending counts for all Windows worker types.

## Background

${any information needed to understand the issue. retrospectives are shareable outside of the team, so try to make it understandable to someone with less context than us!}


## Timeline
  - 2018-02-13 16:37:31 Aryx reports heavy backlog of Windows tests in #taskcluster
  - ${timestamp} ::::: ${event}
  - ${timestamp} ::::: ${event}
  - ${timestamp} ::::: ${event}
  - ${...}

## Fallout

${relatively concise description of how people were affected by this}


## Action Items

Action items should be achievable in short-medium term.

- [ ] ${first item to help make this not happen again}
- [ ] ${second item to help make this not happen again}
- [ ] ${...}


## Thanks

* Brian Stack
* Chris Cooper
* Dustin J. Mitchell
* Jonas Finnemann Jensen
* Kendall Libby
* Mark Cornmesser
* Peter Moore
* Q Fortier
