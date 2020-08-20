# Overview

- **Subject**: Decisions regarding Structured Variable Configuration
- **Decision date**: 2020-08-13

# Executive Summary
This document contains details of 3 decisions made about the Structured Configuration Variables feature:

- Why it reuses the existing JSON Configuration Variables properties/variables
- Why it doesn't support file copying, as per the [pitch](https://docs.google.com/document/d/1c2FzUglohWoNSJycs8kCjeYhbFDSkz_vizNtwsbX3pc/edit)
- Why it doesn't support specification of file formats

# Detail

## Considerations

- The JSON Configuration Variables feature stored a newline delimited list of target files, with support for globs and Octostache extended syntax.
  - Note: some of the characters in the glob syntax are valid characters in Linux filenames. This is an unlikely scenario, but possible. There is no syntax for escaping special characters.
  - Note: Octostache extended syntax makes it impossible to reason reliably about target files outside of a deployment context.
- Octopus Server has an existing means to move files, using [custom scripts](https://octopus.com/docs/deployment-examples/package-deployments/package-deployment-feature-ordering).
- Octopus does not have tried and tested strategy for deprecating features and later removing them.
  - When do we remove the old feature? Needs metrics.
  - Customers don't always read the release notes. Can we help customers make early decisions whether to upgrade or not?
  - We want to avoid deprecated things hanging around forever.
- Octopus does not have API versioning.
- Library step templates are not versioned - they must work against all versions of Octopus Server.
- There are plans to overhaul 'features' in the future, so that a user can add a feature multiple times, and reorder them as required.
  - In this context, adding a file copying feature to the Structured Configuration Variables feature may be less appealing - adding a generic 'file copy' feature is more flexible.

## Options & Considerations
These options aren't all mutually exclusive.

### Option 1: Migrate existing data to a JSON format
Example:

```json
{
    "format": "xml",
    "target": "config.prod.xml",
    "destination": "config.xml"
}
```

Issues:

- Library steps would have to stay in the old format in the library but be upgraded on ingestion
  - Is this a good strategy going forward? Consider if there are dozens of these kind of modifications that have to happen. Needs consideration.
- Not backwards compatible (assuming that properties in the property bag are considered part of the API - part of a bigger conversation around API versioning):
  - 3rd party tools that try to `POST` deployment processes would break.
  - The Terraform provider would break. It was only luck that someone told us about it. How many other cases like this are out there?

### Option 2: As option 1, but with a new property
Same as option 1, but don't reuse the existing properties/variables, and instead add new ones. Deprecate the old feature in the UI.

Sidesteps lots of the issues in Option 1, but also presents some new ones.

Issues:

- We can put a `deprecated` warning on a feature in the UI, but we don't have a good strategy for actually removing the feature later.
  - Risk of the deprecated feature hanging around for years.
- Library steps are still an issue. Should we accept library step submissions that use the new property/variable?
  - The step won't work correctly on older versions of Octopus Server, and the users will get no warning/explanation as to why. The step will just silently misbehave or fail late.

### Option 3: Implement file copying using a teamcity like syntax
Unlike options 1 and 2, this doesn't try to use a JSON format, but instead extends the existing syntax.

Example:

```
config.prod.xml => config.xml
```

Issues:

1. '`=`', '` `', and '`>`' are all valid characters in Linux filenames.
2. Given that we would be mixing the globbing syntax in with a custom '`=>`' syntax, we would create an implicit relationship between the globbing syntax and the arrow syntax. This would potentially cause problems for future work on the globbing syntax.
3. This only solves file copying for this feature. Given the future of features, we can do better.

### Option 4: Implement file copying using custom scripts
Use the existing custom script functionality.

Issues:

- The UX isn't as friendly. The fact that we only discovered this weeks into the project is a testament to the fact that it's kind of "out of sight, out of mind".

### Option 5: Implement file format specification using a `<$format$>`-like syntax

Example:

```
<xml> application.config
<yaml> web.conf
```

### Option 6: Auto detect file formats based on file name/extension
Issues:

1. Always has a fallback to JSON, for backwards compatibility
2. Doesn't support unrecognised file extensions

## Decision
We have gone with a combination of options 4 and 6.

- This provides the easiest path to future enhancements once API versioning and/or deprecation strategy pieces are in place.
- No interplay between globbing syntax and file format/file copying syntax.
- No impact on library steps/3rd party tools.

## Data Sources

Source listed below might provide additional context but keep in mind that they can disappear at any time.

- [Pitch](https://docs.google.com/document/d/1c2FzUglohWoNSJycs8kCjeYhbFDSkz_vizNtwsbX3pc/edit)
- [Slack](https://octopusdeploy.slack.com/archives/C016F9TQESH)
