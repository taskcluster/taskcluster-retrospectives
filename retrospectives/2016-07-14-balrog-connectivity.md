# Balrog VPN connectivity 7-14-2016
* Bugzilla Bug: 1286824

## Background

Tasks that produce updates meant to be uploaded to Balrog will make use of the balrogVPNProxy which maintains a vpn connection to the datacenter.  Balrog was moved out of the datacenter to AWS on 7-13, the day before.

Prior to doing so, routes were added to the vpn for this new IP along with a new ldap group granting net flows to that IP.  However, because of the restricted nature of the balrog vpn user used by taskcluster, and some configuration in docker-worker, taskcluster tasks could no longer contact balrog once this switch over happened.

Pete updated docker-worker to include the docker-worker specific configuration changes and with the help of jabba and mostlygeek the ldap account was updated with the new net flows required to access balrog.

Complicating this matter was an issue with npm which prevented deploying a new ami for awhile.  This has been resolved as well.

## Timeline

* 12:45 UTC - Sheriffs reported issue with update tasks on mozilla-aurora
* 14:00 UTC - Pete and Ben identify that Balrog's ip address changed when moving Balrog to AWS from scl3.  Fix merged to docker-worker, but AMI couldn't be deployed.
* 14:30 UTC - Greg started working on deploying docker-worker.  Required editing the deploy process to work around ECONNRESET issues in npm.
* 15:30 UTC - AMI deployed, but still connection issues with balrog
* 17:30 UTC - after troubleshooting with Ben, mostlygeek, rail, and jabba, determined that the new vpn group needed to be added to the taskcluster-balrog account.  Tasks started greening up afterwards.

# Fallout

Updates for firefox releases were not being uploaded to the balrog servers causing a delay in the beta process.


# Action Items (should be achievable in short-medium term. immediately actionable):
- [done] deploy new ami with host->ip mapping for balrog
- [done] Add vpn group for taskcluster-balrog

# thanks:

* pmoore
* jabba
* bhearsum
* rail
* mostlygeek
