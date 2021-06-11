# Octopus Data Schema Migrations
The following provides some helpful tests, classes and considerations when changing the schema by adding entities or properties. Check some of the existing implimentations for each of these areas for further ideas on how it all fits together.

## Core
### Model
New entities should be created in `Octopus.Core.Model` and should impliment `IDocument` to be used by the `IDocumentStore<>` queriables.
Create or update the `DocumentMap<>` [document map class](https://github.com/OctopusDeploy/Nevermore/wiki/Documents#a-simple-document-map) for the new entity or add the new property if it is bound to a column. Any properties on the entity that are not defined in this mapper class will be combined into the table's `JSON` column.
### Permissions
The impliment an instance of the  `IDocumentRestrictionDefinition` class to allow automatic permissions to be applied to the retrieval of this entity. The `SystemDocumentRestrictionDefinition` and `SpaceDocumentRestrictionDefinition` base classes provide the underlying behavoural access for entities that are non-space defined and space defined respectively. 
### Tests
* [SchemaHasNotChangedFixture](https://github.com/OctopusDeploy/OctopusDeploy/blob/master/source/Octopus.IntegrationTests/Server/Orchestration/SystemIntegrityCheck/SchemaHasNotChangedFixture.cs) - Compares the full list of approved DB objects such as indexes, tables and views
* [IndexesHaveNotChangedFixture](https://github.com/OctopusDeploy/OctopusDeploy/blob/master/source/Octopus.IntegrationTests/Server/Orchestration/SystemIntegrityCheck/IndexesHaveNotChangedFixture.cs) - Checks the validity of all historically _possible_ indexes that may have existed on an instance. 
* [LostMasterKeyCommandFixture](https://github.com/OctopusDeploy/OctopusDeploy/blob/master/source/Octopus.Tests/Server/Commands/LostMasterKeyCommandFixture.AllTypesRequiringEncryption_AreProcessedForEncryptedData.approved.txt) - Ensures that all entities which could have encrypted data are either explicitly exluded or the appropriate properties are marked for encryption.

## Upgrade\Migration Scripts

### Legacy Upgrade scripts
Previously this was done in [Octopus.Core](https://github.com/OctopusDeploy/OctopusDeploy/tree/master/source/Octopus.Core/UpgradeScripts) however to allow migrations to work with Bento the upgades have now moved to a seperate project with enhanced capabilities to cater for running migrations on data being imported.

### Upgrade scripts
Add your new upgrade script with an incrimenting prefix in the name to the [UpgradeScripts](https://github.com/OctopusDeploy/OctopusDeploy/tree/master/source/Octopus.Upgraders/UpgradeScripts) directory in `Octopus.Upgraders`. When a new instance is created, an instance is updated or an import via Bento takes place, these migration steps will run in the prefix order. 
Typically two methods should be implimented
* `UpgradeSql` - The SQL statements to provide that actually upgrades the database and performs any data migrations. 
* `UpgradeExportSet` - C# logic to perform when Bento imports entities. Since this could be taking place on external data from any database version, your code should be reasonably permissive of the data coming in. Since it could be running on an Octopus Server of any version (as these migration steps in theory will continue to run in future Octopus Server instances), you should not rely on any data models for serialization or deserialization. Using raw JObjects is the safest approach to load data.

### Tests
For most scenarios, the previously added class should warrant a unit test. These are added to `Octopus.IntegrationTests.Core.UpgradeScripts` and should cover both SQL and Document entity upgrades. As an example take a look at [Script0274FixReleaseVersionsFixture.cs]`https://github.com/OctopusDeploy/OctopusDeploy/blob/master/source/Octopus.IntegrationTests/Core/UpgradeScripts/Script0274FixReleaseVersions/Script0274FixReleaseVersionsFixture.cs`. There is a setup and verification method that should be implimented for both the SQL upgrade (emulating an Octopus instance with data migrating to this version for the first time) and the ExportSet upgrade (emulating entities from a version of Octopus _prior_ to this version being imported into an Octopus instance where the schema is expected to be in the new shape). Obviously for some scenarios, such as adding an entity & table for the first time, these tests may not be as relevant.


## Import\Export (Bento)
The [DocumentSourceFixture](https://github.com/OctopusDeploy/OctopusDeploy/blob/master/source/Octopus.Tests/ImportExport/DocumentSourceFixture.cs) class verifies that a DocumentSource has been defined for every entity that impliments `IId` (all `IDocument` entities). If this class should not have an exporter, an exception for the new class will need to be added.

## Git Schema
_comming soon..._
