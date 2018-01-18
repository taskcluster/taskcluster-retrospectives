# Retrospective: Spring Cleaning URL Change 2018-01-18

## Background

We have been performing "spring cleaning" work on each of our services over the
last month or so, upgrading dependencies and modernizing the build process.

In this case, that process upgraded the AWS SDK package, with the new version
generating a different (but still valid) URL for S3 buckets.  The difference is
between `s3-us-west-2` and `s3.us-west-2`.  The resulting URL did not match the
whitelist configured in cloud-mirror, and that service proceeded to respond
with 403 errors to all incoming requests.

## Timeline
  - 2018-01-18 14:57:09 UTC ::::: spring cleaning patch deployed to queue
  - 2018-01-18 14:57:24 UTC ::::: first 403 response from cloud-mirror
  - 2018-01-18 15:00:19 UTC ::::: <Aryx> forbidden to download our own nightly builds: https://treeherder.mozilla.org/logviewer.html#?job_id=157161817&repo=mozilla-central&lineNumber=1146
  - 2018-01-18 15:11:14 UTC ::::: jhford discovers that the error message indicates "URL does not validate" which points to the `allowedPatterns` configuration
  - 2018-01-18 15:19:22 UTC ::::: roll back queue deployment based on coincidence of time
  - 2018-01-18 15:27:42 UTC ::::: trees re-opened

## Fallout

This caused all artifact downloads to fail when made from any EC2 region other
than us-west-2. Trees were closed for the duration of the event.

## Action Items

- [ ] [Add the updated URL](https://github.com/taskcluster/cloud-mirror/pull/38) to the `allowedPatterns` config in cloud-mirror and redeploy
- [ ] Redeploy the queue change.
- [ ] Monitor for spikes of 403 errors from services via Papertrail

## Thanks
* jhford
* Aryx
* imbstack
