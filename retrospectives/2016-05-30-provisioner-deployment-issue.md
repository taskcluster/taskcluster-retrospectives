# Heroku Config caused complication of deployment May 30
* Bugzilla Bug: no bug

## Background

    while doing a deployment, I had to replicate the heroku environment on my local machine.  The heroku toolbelt offers the commmand "heroku config -s" which is supposed to return something that can be eval'd in bash.  The problem is that they do not properly quote the return values and so the environment variables did not match production.  The issue is that the setup things that happened used incomplete values


## Timeline
* May 30 -- deployed code, monitored provisioner log for errors
* 1 hour later, deployment done and issues resolved

## Fallout

The provisioner downtime was extended by 1 hour.  No running tests were impacted, only a lack of new instances booting

## Action Items (should be achievable in short-medium term. immediately actionable):
- Write something to replicate heroku environments locally in a sane way -- just grabbing environment variables and setting a new env with nothing but those
- [jhford] Try setting up a staging environment for Ana's patches
* please use separate DB credentials
* idea: npm check-staging (submits a job with ... 
* tc-diagnostics may have something useful to crib

## Thanks

* jhford (i just dealt with it)
