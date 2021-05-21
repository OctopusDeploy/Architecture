# Git Schema Migrations

When a user converts their project to a source controlled project, the data that is transferred to the repository is based on the state of the domain model at that point in time. As our data model continues to evolve, it's important that the project stored in this system is still able to be used by Octopus in the same way that their project would continue to work throught upgrades had it been stored in the database.

In many ways this migration process looks similar at first glance to the considerations we have with database upgrades. Both should end up with representations of our data model in reasonably similar shapes, and both may involve upgrading through multiple versions at once as the user installs an up-to-date version of Octopus Server after having  previously run an older version. There are however some key differences and concerns in how these operations are performed.

When dealing with database migrations, we can consider migrating between schema versions as a very clear set of state transitions where the user's database moves between each schema version. Putting aside higher level concerns of application back and forwards compatability, all the information needed to move from version n to n+1 should be available, allowing a consistent and stable migration. For Git Schema migrations however this is not _necessarily_ the case. 

## Deferred Migrations
As the repository that backs the project content is not owned by Octopus, every operation by Octopus that changes or commits to it should be expected by the user. Making unexpected commits could trigger builds, break merges or any number of things that would be unexpected from simply upgrading Octopus Server. Instead, unlike the databases, the data models stored in Git would be left as is, and migrated on the fly and in memory when the user access the project through Octopus itself. From the lense of Octopus we should be able to present this project in the state that it _would_ be in had they left it in the database. **Only when the user performs a modification of an entity on their project will the migrated schema be commited** (including the change initiated by the user). Unless a user triggers the schema to be updated, the Octopus Server needs to be able to read and migrate any possible previous version of the schema.

## Unstable State Changes
When we allow a new entity to be version controlled, we effectively move the entity from the database to the ocl file. For projects that are newly converted, this takes place during the "Convert To Version Controlled" step (in CAC, when version controlling a project, _all_ entities that _can_ be version controlled _are_ version controlled, there is no partial version control). For projects which a user had previously already converted, we will need to account for this new entity during the Git Schema Migration. Note that the nature of this step, moving the entity from the database, means that the state of the entity when it is initially added to the file is entirely based on the state of the database at the time that that step in the migration takes place.

Consider the following scenario where the user is currently running V1 and is upgrading their server to 2022.1.
```
  2021.1      │    2021.4     │      2022.1
              │               │
    V1  ──────┼──►   V2   ────┼────►   V3
              │               │
 Deployment   │   Channels    │  Channels Removed
Added To VCS  │ Added To VCS  │   From DB (& VCS)
```
A few days later when they open their project in the portal, Octopus will see that their schema needs to upgrade to V3 and in doing so try and first migrate through V2. In doing so it will 💥 since the Channels entity no longer exists. 

Although this scenario is a little artificial (since we would expect that V2 step at compile time to complain about the unknown entity type), it does provide a very clear demonstration of how the migration is dependant on the state of the database schema. A less obvious scenario invio

##

Scenarioes
* Deleted Entities
* Removed Properties
* Updating Property Name

Options:
* Strongly Typed
* Serializtion Types

Approach
* Serialization Type
* Chained Version Steps

Takes place underneath transaction so it remains transparent to the consumer

Convert to CVS will be treated as converting Version 0 To Version N.

Chain 
V1 -> V2 -> V3
vs
V1->Vn, V2->Vn, V3->Vn
