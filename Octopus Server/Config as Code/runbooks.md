# Version-controlled (VCS) runbooks

## No database record

**There is no runbooks record in the database for VCS runbooks. All configuration is stored in git.**

- [Slack summary of decision](https://octopusdeploy.slack.com/archives/C01A1E8K9J5/p1599541752127300)
- [Zoom recording of discussion](https://octopus.zoom.us/rec/share/jsXmvkBwoTNPgsHVqdR1K95aaOmcEFdWTdwEuyPB5PaU04FONu8_PVOMg1yA6DE-.M3itLGGF1DzIRbRk) Password: `HcNk!3%0`

## Runbook Identity

**Version controlled runbooks are identified by their file name. If two runbooks on different branches have the same file name, then they have the same identity.**

Users may want to change the runbook name that appears in the UI, but still want to keep its identity. This design allows us to correlate the runs before and after the name change, and switch between branches before and after the name change to compare other changes to the runbook.

Having the file name be the identity for the runbook allows users to achieve this rename without affecting its identity, provided that the filename remains the same.

- [Slack discussion](https://octopusdeploy.slack.com/archives/C01A1E8K9J5/p1599535985120300)
- [Slack summary of decision](https://octopusdeploy.slack.com/archives/C01A1E8K9J5/p1599541752127300)
- [Discussion document, showing some other options](https://docs.google.com/document/d/1PKPCrkcIjWsFywDsbKIvsvEtAbmEPatpAbhEv8zl1PY/edit?usp=drive_web)
- [Zoom recording of discussion](https://octopus.zoom.us/rec/share/jsXmvkBwoTNPgsHVqdR1K95aaOmcEFdWTdwEuyPB5PaU04FONu8_PVOMg1yA6DE-.M3itLGGF1DzIRbRk) Password: `HcNk!3%0`

## Runbook location

**Within Git, runbooks will be located in the `.octopus/runbooks` directory**

- [Discussion and decision](https://octopusdeploy.slack.com/archives/C01A1E8K9J5/p1600236549260500)

## Single aggregate for Runbook and Process

**The runbook settings and the runbook process form an aggregate root**

Unlike non-VCS runbooks, our model for VCS runbooks contains the settings (name, description, multi-tenancy mode, connectivity policy, etc.) and the process, combined into a single aggregate root. 

This inforamtion is persisted together in the one .octo file in git, and is also served together from the same endpoints in our API.

- [Initial discussion](https://octopusdeploy.slack.com/archives/C01A1E8K9J5/p1600122165220800)
- [Decision](https://octopusdeploy.slack.com/archives/C01A1E8K9J5/p1600389911004900)
- [Zoom recording of discussion](https://octopus.zoom.us/rec/share/NG_q30NgU_1RdW6bocmMtB1GdP_ag576GTFjbWDDYxnfwByY6HbkX-QEIaVDP4o.jA25Dobjw0dQIbQo?startTime=1600388416000) Password: `%KYXgk^9`

## Different domain and resource types

**The domain and resource types used for VCS runbooks are distinct from the types used for non-VCS runbooks**

There are enough differences between VCS and non-VCS runbooks, such that it doesn't feel natural for them to share the same model. We therefore have distinct models and resources for these two different types of runbooks.

- [Slack summary of decision](https://octopusdeploy.slack.com/archives/C01AJE4K3T2/p1600058089113200?thread_ts=1600051433.112900&cid=C01AJE4K3T2)

## Runbook Snapshots

**All properties on a runbook should be snapshotted in the runbook snapshot.**

One implication of having no database record is that when you create a runbook snapshot, _all_ of the information needed must be stored in the snapshot, since there is no database entry that can store other information. There is also no guarantee that the gitref from which the snapshot is created will always exist in git, so we should save this information while we still have access to it. 

This means properties such as a runbook's name, description, multi-tenancy mode, connectivity policy, etc. should appear on the runbook's snapshot.

For consistency and simplicity, runbook snapshots should be the same shape for VCS and non-VCS runbooks.

- [Slack summary of decision](https://octopusdeploy.slack.com/archives/C01A1E8K9J5/p1599541752127300)
- [Zoom recording of discussion](https://octopus.zoom.us/rec/share/jsXmvkBwoTNPgsHVqdR1K95aaOmcEFdWTdwEuyPB5PaU04FONu8_PVOMg1yA6DE-.M3itLGGF1DzIRbRk) Password: `HcNk!3%0`


## Published runbooks

**The set of published runbooks should be stored in their own database table, with their own API endpoints.**

Publishing runbooks is conceptually simple - for a given runbook Id, we just point to a specific runbook snapshot and declare that as the single published snapshot.

We can't store that information on the runbook document, because there is no runbook document for VCS runbooks. We also don't want to store it against the runbook snapshots because this way of modelling the data doesn't constrain us to a single published snapshot per runbook id.

Instead, the runbook snapshots should live in their own separate table (SpaceId, ProjectId, RunbookId, PublishedSnapshotId) and have their own set of endpoints from which to expose and modify this information.

For consistency and simplicity, the way we publish snapshots for VCS and non-VCS runbooks should be the same.

- [Slack summary of decision and sample API](https://octopusdeploy.slack.com/archives/C01A1E8K9J5/p1600911060027400)