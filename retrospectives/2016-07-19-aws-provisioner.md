# AWS Provisioner 7-19-2016
* Bugzilla Bug: https://bugzilla.mozilla.org/show_bug.cgi?id=1287923

## Background

We're not sure what exactly caused the provisioner to get stuck, but it did. Jonas and bstack's bug-divining-rods point to something to do with the api error rates in us-east-1, but there's not any real evidence to back that up as of the writing of this document. Whatever the cause, restarting the provisioner allowed it to work eventually. It was caught relatively early, so the new instances came online and worked through the backlog fairly quickly.

## Timeline
* 12:21 PT gps noticed we had 6 minutes go by and had a pending decision task
* 12:31 PT bstack restarted the aws-provisioner to "unstick" it
* 12:50 PT us-east-1 reports Increased API Error Rates
* 13:40 PT selena spoke up and said that backlog was cleared, but she might have been wrong
* 13:45 PT gps' backlogged docker worker job completed

## Logs

 I see a lot of desktop-test: Error: connect ETIMEDOUT 13.93.168.88:443\n    at Object.exports._errnoException (util.js:873:11)\n    at exports._exceptionWithHostPort (util.js:896:20)\n    at TCPConnectWrap.afterConnect [as oncomplete] (net.js:1077:14)","
time":"2016-07-18T15:06:16.514Z","v":0}

It was a timeout talking to the blob store stuff
blob.by4prdstr10a.store.core.windows.net. is the ip in that error... but that should be ignored.

## Fallout

* trees were closed for 1-1.5 hours
* gps was the person who alerted us to the problem instead of an automated system
* selena witnessed distributed systems in action, wherein there was a 5 min gap between aws-provisioner reporting 0 pending and the specific task gps was concerned with was pending
** azure rest docs says queue size is an upper-bound: https://msdn.microsoft.com/en-us/library/azure/dd179384.aspx
** queue.taskcluster.net caches the queue size for 20 seconds: https://github.com/taskcluster/taskcluster-queue/blob/fca8965305a8de87ddfc605efcf6761f05befd5c/src/queueservice.js#L637-L638 


## Action Items (should be achievable in short-medium term. immediately actionable):
* get garndt's alerts from papertrail and signalfx ready to send to the entire list
** what lists do we have ?  taskcluster-internal@ is one -- probably want a separate list for robot-initiated email
* make provisioner able to avoid regions that are marked as unhealthy by aws (already done)
** only considers available availability zones
* make provisioner pick slightly more expensive regions probabilistically to avoid one bad region making everything bad (optional, possible already done)
* discuss whether we need to force use of two different regions, not just AZs
* better error handling for blob-storage failures
* to have out-of-date state on the UI
* we sort of already have that


## thanks

gps for alerting us to the whole thing and checking back in to see if it was resolved.
