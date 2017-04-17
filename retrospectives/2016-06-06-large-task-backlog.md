# Large backlog on multiple worker types Monday June 6, 2016 2-3pm
* Bugzilla Bug: n/a (conveyed via irc)

## Background

Around 2pm on 6/6 Callek brought up in #taskcluster that we had a large backlog on desktop-test. Selena had a look and then wanted to know if this was typical, or abnormal. jmaher weighed in that we uplifted, so mozilla-beta branch might be introducing more tests than typical.  For the first time, jmaher was waiting on taskcluster jobs to complete instead of buildbot.


## Timeline: (in PT)
    - 2pm Callek pings, jmaher concurs
    - 2:08pm jhford weighs in about what instances means
    - 2:15pm selena asks jhford to look into trends
    - 2:30pm jhford sends data from influx queries
    - 2:35pm selena updates workerTypes, incrementing by 500 every iteraction for desktop-test, 50+ for build types with any backlog
    - 3pm ish 

## Fallout

ateam + releng noticed the backlog and mentioned: inability to prioritize jobs, lack of builders compounding test backlog, and suggesting that uplift was causing confusion


## Action Items (should be achievable in short-medium term. immediately actionable):
* create graphs of pending backlog so that we can stay on top of bad trends - selena made some grafana graphs but is not sure how to share them: https://cloud.influxdata.com/grafana#/dashboard/db/grafana
* get pending data into signalfx so that we can easily share dashboards and create alerts
* possibly change to the "auto-scale" option available in aws provisioner - we'll try this out in June

## Thanks

* jhford
