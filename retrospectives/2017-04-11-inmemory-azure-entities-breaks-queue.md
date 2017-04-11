# Retrospective: inMemory azure-entities breaks queue 2017-04-11

## Background

We wanted to make the queue use inMemory azure-entities for the tables in taskcluster-queue to avoid entity upgrades in branches fouling up the entities for other branches. Without this change, we need to bump the name of the testing table in config every time a change is made to any entity types. This isn't a hardship, but would be nice if it could be a non-issue.

After merging this change into master, the tests passed in travis and heroku immediately deployed the updated revision. However, we had forgotten to add the `credentials` field to the entities in setup, so in production, they did not have access to the queue's credentials and immediately crashed. This was not detected in testing because inMemory doesn't require credentials to be specified. Patch to add creds is here to show what was missing: https://github.com/taskcluster/taskcluster-queue/pull/163

This would've had nearly 0 fallout if we had noticed the breakage earlier. Because this crashed so quickly, there was no sentry notification and heroku does not create any sort of notification when it has deployed. Due to the 10 minute interval between merging and deployment happening, bstack had started working on another task and only went back to check 2 minutes after the deploy had happened.

## Timeline

- 10:30 ::::: PR is merged into master
- 10:40 ::::: Build succeeds in Heroku and is deployed to master
- 10:42 ::::: Rollback
- 10:48 ::::: aryx notices something is wrong and talks to #taskcluster channel

## Fallout

Trees were closed because they could not download artifacts. Trees were quickly reopened after the rollback.

## Action Items

Action items should be achievable in short-medium term.

- [ ] https://github.com/taskcluster/azure-entities/pull/28
- [ ] consider making services with long test times not deploy by default?
- [ ] consider a tc-queue staging service

## Thanks

garndt for finding the traceback in the logs

aryx for closing trees
