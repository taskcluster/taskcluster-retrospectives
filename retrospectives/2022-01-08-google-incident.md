# Retrospective: Google Incident 2022-01-08
#### Bugzilla Bug: [bug 1749175](https://bugzilla.mozilla.org/show_bug.cgi?id=1749175)

## Background

A [Google incident](https://status.cloud.google.com/incidents/NMcnk6aE8xMHHwRGmyry) caused multiple services to break. The timing, in conjunction with the previous bustages in H2 2021, led to multiple conversations about how we can better respond to future production issues.

## Timeline

  - 2022-01-08 2233 ::::: spotted-azure-bat-02 (one of the cloudamqp nodes for Pulse) stops reporting telemetry to cloudamqp (although there's no other evidence of a Pulse outage or degradation)
  - 2022-01-08 23:14 ::::: taskcluster-queue-web has a networking-looking error:

    ```
    Error: PulsePublisher.sendDeadline exceeded
    at failAtDeadline (/app/libraries/pulse/src/publisher.js:343:32)
    at async PulsePublisher._send (/app/libraries/pulse/src/publisher.js:394:5)
    at async PulsePublisher.<computed> [as taskRunning] (/app/libraries/pulse/src/publisher.js:316:9)
    at async WorkClaimer.claimTask (/app/services/queue/src/workclaimer.js:256:5)
    ```

	Also some postgres(?) connection errors at the same time:

    ```
	error: terminating connection due to idle-in-transaction timeout
	CODE: 25P03
    at Parser.parseErrorMessage (/app/node_modules/pg-protocol/dist/parser.js:287:98)
    at Parser.handlePacket (/app/node_modules/pg-protocol/dist/parser.js:126:29)
    at Parser.parse (/app/node_modules/pg-protocol/dist/parser.js:39:38)
    at TLSSocket.<anonymous> (/app/node_modules/pg-protocol/dist/index.js:11:42)
    at TLSSocket.emit (events.js:315:20)
    at TLSSocket.EventEmitter.emit (domain.js:467:12)
    at addChunk (internal/streams/readable.js:309:12)
    at readableAddChunk (internal/streams/readable.js:284:9)
    at TLSSocket.Readable.push (internal/streams/readable.js:223:10)
    at TLSWrap.onStreamRead (internal/stream_base_commons.js:188:23)
    ```

    All those connection errors tail off by 0200

  - 2022-01-08 23:17 ::::: taskcluster-auth-web starts reporting "Invalid token" errors regarding Sentry?

    ```
	Error: Invalid token
    at Client.<anonymous> (/app/node_modules/sentry-api/lib/client.js:138:19)
    at Request.self.callback (/app/node_modules/request/request.js:185:22)
    at Request.emit (events.js:315:20)
    at Request.EventEmitter.emit (domain.js:467:12)
    at Request.<anonymous> (/app/node_modules/request/request.js:1154:10)
    at Request.emit (events.js:315:20)
    at Request.EventEmitter.emit (domain.js:467:12)
    at IncomingMessage.<anonymous> (/app/node_modules/request/request.js:1076:12)
    at Object.onceWrapper (events.js:421:28)
    at IncomingMessage.emit (events.js:327:22)
    at IncomingMessage.EventEmitter.emit (domain.js:467:12)
    at endReadableNT (internal/streams/readable.js:1327:12)
    at processTicksAndRejections (internal/process/task_queues.js:80:21)
    ```

  - 2022-01-09 00:54 ::::: [Google incident](https://status.cloud.google.com/incidents/NMcnk6aE8xMHHwRGmyry) regarding packet loss in US-WEST1 posted

  - 2022-01-09 01:16 ::::: taskcluster-web-server-web emits an amqp connection error:

    ```
    IllegalOperationError: Connection closed (Error: Heartbeat timeout)
    at Connection.<anonymous> (/app/node_modules/amqplib/lib/connection.js:361:11)
    at Channel.C.sendImmediately (/app/node_modules/amqplib/lib/channel.js:70:26)
    at Channel.C.sendOrEnqueue (/app/node_modules/amqplib/lib/channel.js:81:10)
    at Channel.C._rpc (/app/node_modules/amqplib/lib/channel.js:142:8)
    at /app/node_modules/amqplib/lib/channel_model.js:59:17
    at tryCatcher (/app/node_modules/bluebird/js/release/util.js:16:23)
    at Function.Promise.fromNode.Promise.fromCallback (/app/node_modules/bluebird/js/release/promise.js:209:30)
    at Channel.C.rpc (/app/node_modules/amqplib/lib/channel_model.js:58:18)
    at /app/node_modules/amqplib/lib/channel_model.js:70:17
    at tryCatcher (/app/node_modules/bluebird/js/release/util.js:16:23)
    at Promise._settlePromiseFromHandler (/app/node_modules/bluebird/js/release/promise.js:547:31)
    at Promise._settlePromise (/app/node_modules/bluebird/js/release/promise.js:604:18)
    at Promise._settlePromiseCtx (/app/node_modules/bluebird/js/release/promise.js:641:10)
    at _drainQueueStep (/app/node_modules/bluebird/js/release/async.js:97:12)
    at _drainQueue (/app/node_modules/bluebird/js/release/async.js:86:9)
    ```

  - 2022-01-09 01:36 ::::: one more

  - 2021-01-09 00:16 ::::: code sheriffs report issues when they try to retrigger tasks
  - 2022-01-09 02:42 ::::: google resolves incident
  - 2022-01-09 03:30 ::::: cvalaas becomes aware of a problem with taskcluster/treeherder via slack
  - 2022-01-09 06:30 ::::: cvalaas restarted the non-reporting rabbitmq node
  - 2022-01-09 18:47 ::::: aki looks to see if ci-admin needs to run to fix the broken hook
  - 2022-01-09 18:52 ::::: jbuck force-runs ci-admin on cloudops-jenkins
  - 2022-01-09 18:54 ::::: aki notices that the hook service is hanging
  - 2022-01-09 18:57 ::::: jbuck restarts all deployments with hook in the name
  - 2022-01-09 18:59 ::::: aryx is able to add new jobs via hooks
  - 2022-01-09 19:02 ::::: sheriffs hitting missing indexes when trying to retrigger tasks
  - 2022-01-09 19:18 ::::: aki manually creates missing index, fixes retriggers on that push
  - 2022-01-09 19:21 ::::: aryx points out new pushes are also missing indexes
  - 2022-01-09 19:22 ::::: aki decides this is a multi-service outage closing autoland, and this may need to wait for working hours; notifies the slack thread about the index service
  - 2022-01-09 19:41 ::::: cvalaas restarts the index service
  - 2022-01-09 20:04 ::::: sheriffs verify retriggers are fixed
  - 2022-01-09 20:24 ::::: cvalaas redeploys firefox-tc
  - 2022-01-09 20:33 ::::: firefox-tc redeploy completes


## Fallout

Rerunning and adding tasks to pushes in CI were broken: identifying which change started an issue was not possible.

Treestatus access was intermittently broken; trees could not be opened or closed to let or prevent new changes landing.

## Action Items

- [ ] ramp Taskcluster and CloudOps teams back up on the platform during the H1 sprints
- [ ] start a Taskcluster production runbook in H1
- [ ] add heartbeat monitoring during the H1 sprints
- [ ] bring sheriffs, releng, relops, cloudops, taskcluster together in cross-team management-level meetings in H1

## Thanks
- @archaeopteryx
- @blewa
- @cvalaas
- imoraru
- @jbuck
- @mostlygeek
- nataliaCs
- smolnar
