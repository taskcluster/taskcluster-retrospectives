# Retrospective: Worker Manager Overprovisioning - November 2019
#### Bugzilla Bug: 1599122

## Background

From Wednesday, November 20th to Monday, November 25th we observed significant overprovisioning of the gecko-t/t-linux-metal worker-pool in the FirefoxCI Taskcluster deployment.

The worker-pool was created on the 20th, and had a maximum capacity of 10 configured (with a single instance representing capacity 4). By Monday the worker-pool had 395 c5.metal instances running simultaneously, amounting to 1,580 capacity. The c5.metal instances were run on-demand, and over the four day period roughly $160,000 was spent.

There were several conditions that lined up to create this failure:

* Worker-manager entered a pathological state where it continued to provision workers into the gecko-t/t-linux-metal worker-pool dramatically beyond its configured capacity.
* Worker-manager has disconnected processes for scanning for workers and provisioning worker capacity, which became out of sync.
* It was designed to have scanner run at least once between every provisioning run and that was the case until we started having a very large number of workers across many pools.  We’re not sure exactly why this didn’t show up in load testing but we didn’t really test the case of many pools but rather large single pools
* When the worker-scanner hits issues that causes it to not see all workers, such as slow API responses, rate limiting, or inability to save results to the backing store, the
  provisioner is flying blind, and in some cases will spawn new workers because it believes more workers are needed.
* Observations from the logs during the incident:
  * From time to time runningCapacity was suddenly becoming 0 in estimator logs (provisioning loop). Estimator would then be requesting batches of 10 capacity for a while (there were 86 pending tasks the whole time). Then runningCapacity  would suddenly be back non-zero, and it would be either a much higher number (overall, starting from Wednesday 11:05 to 17:22 same day it went up from 12 to 2000) or a lower but still high number. These fluctuations continued heavily over the course of Thursday, and happened 5 times on Friday (reaching its highest of approx 4000), and eventually on Friday the system entered a stable state with runningCapacity 1508. That runningCapacity corresponded to the number of instances Dustin terminated on Monday.
  * Running capacity comes from the provider object’s state, which is set by worker scanner. Every iteration of the scanning loop that state is reset to 0. So something was happening with worker scanner on Wednesday, Thursday and 5 times on Friday that was the scanning loop leave that state at 0. We are not sure what it was and why it stopped happening after 8:23 am PST Friday. (owlish run out of her time-box investigating this)
  * There are some “Iteration exceeded maximum time allowed” errors in worker scanner logs; however, the times don’t always align with the 0s in the logs.
* The AMI that the worker-pool used was improperly configured to use aws-provisioner instead of aws-provider, with the result that the docker-worker service was never started.
* As a result, docker-worker’s idle shutdown was never triggered, so instances remained running until they were shut down.
* Additionally, because the 86 pending tasks assigned to the worker-pool were never completed, causing worker-manager to continue to provision instances to the worker-pool when
  it believed that capacity was needed.
* AWS AMIs for Taskcluster workers do not properly set hostnames
* When debugging began the overprovisioned instances had already been shut down. The logging in Papertrail is configured by hostname, but because these instances did not set
  their hostnames to match worker-pool IDs, we were unable to find instance logs for them.
* Lack of visibility in the Taskcluster UI of running workers
* If workers have not registered and claimed tasks, it isn’t clear from the UI that there may still be workers running. Worker-manager could make this information available.
* Lack of budget / spending alerts to catch outsized forecasted spending ahead of time
* Spending alerts can be noisy for elastic workloads because forecasting doesn’t take into account the relatively short lifespan of Taskcluster workers. There weren’t any
  configured spending alerts that notified us about higher than normal AWS spend.
* A false sense of security after the FirefoxCI migration, as everything appeared OK
* As part of the migration all production workloads were switched from aws-provisioner to worker-manager and aws-provider - and everything appeared fine. This overprovisioning
  issue was the first sign of bugs in worker-manager, which had only been in production for two weeks.
* When releasing any new piece of software there will always be unexpected bugs that come to light. Unfortunately, we hit an expensive bug.

The worker-manager issues that we saw had not been previously observed and were not reproducible in testing. There were some previous concerns about its architecture, but nothing that implied it shouldn’t be used in production.

The separation of worker scanning and provisioning meant that issues scanning for workers could result in overprovisioning. In production, where thousands of instances run concurrently, and more are requested all the time across many regions and availability zones, we ran into issues with worker scanning that led worker-manager to dramatically overprovision.

## Timeline
  - 2019-11-20 18:06 UTC ::::: Tasks are submitted as part of a try push
  - 2019-11-20 19:09 UTC ::::: The worker-pool is configured to use spot instances, we see this error: "Error calling AWS API: There is no Spot capacity available that matches your request."
  - 2019-11-20, 19:38 UTC ::::: More pending tasks as part of the try push => more requestedCapacity
  - 2019-11-20, 19:48 UTC ::::: A change is landed to allow the worker-pool to use on-demand instances
  - 2019-11-20 19:39 UTC ::::: Another “Error calling AWS API: There is no Spot capacity available that matches your request.” (the last time we see this, indicating that the change to on-demand has happened)
  - 2019-11-20 19:40 UTC ::::: We spawn 12 instances (each instance has capacity 4)
  - 2019-11-20 20:01:15 UTC ::::: We’re at runningCapacity 80
  - 2019-11-20 20:02:32 UTC ::::: We’re back down to 0 runningCapacity
  - 2019-11-20 20:05 UTC ::::: "Error calling AWS API: We currently do not have sufficient c5.metal capacity in the Availability Zone you requested (us-west-2a). Our system will be working on provisioning additional capacity. You can currently get c5.metal capacity by not specifying an Availability Zone in your request or choosing us-west-2b, us-west-2c, us-west-2d." This error continues for quite some time, repeatedly

## Fallout

* Some large amount of $$$
* Loss of trust from other teams and management
* Lots of scramble and stress on our part in the “relaxing” period after our cloudops migration

## Action Items

* [x] The worker-pool was disabled and all workers were terminated as soon as the issue was observed.
* [x] Additional logging and error reporting has been added to worker-manager to call attention to overprovisioning.
  You can use https://gist.github.com/imbstack/b18a697fb3046a90080499a93b0ffe15 to check for overprovisioned worker-pools
* [x] The configuration of worker-runner on generated AMIs has been corrected to use aws-provider.
* [x] Worker-manager should track instances that it has spawned that have not registered and terminate them, as this is indicative of a configuration issue.
* [ ] Additionally, if a worker-pool overwhelmingly spawns failing workers it should be disabled.
* [ ] bug 1604925 - Worker-manager’s running capacity per worker-pool should be made visible in the UI
* [ ] AMIs will be generated for the taskcluster-staging account and tested in development clusters before production and FirefoxCI use
* [x] Adding billing alerts
* [ ] Accounting for orphaned instances
* [x] Worker-manager’s loops (per libiterate) should ensure that only a single scanning loop is ever running at once
* [ ] Workers view
* [ ] All workers need to call registerWorker (worker-runner) Owlish’s requirement gathering document to aid the design  https://docs.google.com/document/d/1BFha7fDTX3uXYdVbmeHCXQK6G6o8MghHmldB7Xodz0U/edit?usp=sharing 
* [ ] bug 1604926 - Check for orphaned instances
* [x] Bug 1587511 - worker-manager: hard kill instances after 96 hours
* [ ] Bug 1587516 - worker-manager: kill idle instances and disable bad worker pool configs



## Thanks
- edunham/bpitts
- tomprince
