# Structured Configuration Variables

Structured Configuration Variables are a feature of Octopus Server and Calamari that allow for values in files to be replaced with values from Octopus variables/Calamari properties.

- Supports XML, JSON, YAML and .properties files.
- Uses the `Octopus.Action.Package.JsonConfigurationVariablesTargets` property for reasons discussed below.
- Determines file format based on file extension, for reasons discussed below.

# History/decisions

The rest of this documents outlines 3 key technical decisions made in the implementation of the Configuration Features for Java pitch:

- **Decision 1**: Whether to extend or deprecate the related, existing JSON Configuration Variables feature
- **Decision 2**: How to support writing transformed configuration files to different paths
- **Decision 3**: How to determine the format (JSON, vs YAML, vs XML, etc) of input configuration files

# Decision 1: Extend or deprecate existing feature
For context, the data in the existing property (`Octopus.Action.Package.JsonConfigurationVariablesTargets`) can contain multiple filenames, newline delimited, with globs and Octostache extended syntax. Examples:

```
config.json
```

```
*.config
**\*.json
```

```
#{each configFile}
#{configFile}
#{/each}
```


## Option 1: Migrate data into new properties
In this option, we would migrate data from the `Octopus.Action.Package.JsonConfigurationVariablesTargets` property into `Octopus.Action.StructuredConfigurationVariablesTargets`, and store the data in JSON.

As an example, JSON configuration that looks like this:

```
config.json
```

Would get migrated to something like this:

```json
{
    "format": "json",
    "target": "config.json",
    "destination": null
}
```

Pros:

- Migrating to a JSON structure gives us scope for future extension. 
- The property/variable names more clearly reflect intent.

Cons:

- Library steps are not versioned. There is no mechanism in Octopus to restrict library steps to certain versions of Octopus. Old versions of Octopus wouldn't understand this new property, so our only option would be to ensure that library steps support the lowest common denominator, and are migrated forwards upon ingestion.
- Supporting 3rd party tooling is difficult. Without knowing whether a client posting a deployment process to Server knows about the new property, we can't reliably map data from the old structure to the new structure.

## Option 2: Add a new property, deprecate the old one.
Similar to option 1, but don't reuse the existing properties/variables, and instead add new ones. Deprecate the old feature in the UI.

Pros:

- Same as Option 1.

Cons:

- We can put a `deprecated` warning on a feature in the UI, but we don't have a good strategy for actually removing the feature later.
  - Risk of the deprecated feature hanging around for years.
- Library steps are still an issue. Should we accept library step submissions that use the new property/variable?
  - The step won't work correctly on older versions of Octopus Server, and the users will get no warning/explanation as to why. The step will just silently misbehave or fail late.

## Option 3: Keep existing format and properties
In this option, we keep the existing property, and keep the same format.

Pros:

- 3rd party tools (and our own, like the Terraform provider) continue to work as normal.
- Library step templates continue to work as normal.

Cons:

- Limits how we can add capabilities to the feature (e.g. file copying, format specification discussed below)

## Decision
We have gone with option 3. Without API versioning, a deprecation strategy, or a strategy for library step template versioning, the other 2 options impose are high risk.

# Decision 2: File copying
One of the goals of the Configuration Features for Java pitch was to enable not just the replacement of variables in a file, but also the (optional) copying of the file (post-replacement) to a new path.

## Option 1: Implement a new syntax for file destinations
This was the idea proposed in the pitch. It would like look this:

```
logging.json
config.prod.json => config.json
```

Pros:

- The syntax is simple to parse

Cons:

- '`=`', '`>`' and '` `' are all valid characters in Linux filenames. Unlikely to interfere with customers, but possible.
- '`=`' and '` `' are both valid characters in Windows filenames. Unlikely to interfere with customers, but possible.
- The globbing syntax is used by other components in Calamari. This new syntax would effectively be mixing Octostache, globbing and our new custom syntax into one string. There is a risk that future work on any of these individual syntaxes could result in ambiguous parses that wouldn't be discovered until late in development.
- File copying is a common requirement. This option only solves it in one context. It would be nice if we had something reusable.
- Library step templates aren't versioned, so this field would be misinterpreted by old versions of Octopus.

## Option 2: Use custom scripts
Octopus already provides a way to copy files, using [custom scripts](https://octopus.com/docs/deployment-examples/package-deployments/package-deployment-feature-ordering). Using this, a user can write some bash/powershell to copy a file.

Pros:

- The functionality already exists in Octopus.

Cons:

- The UX isn't as friendly. The fact that we only discovered this weeks into the project is a testament to the fact that it's kind of "out of sight, out of mind".

## Decision
We have gone with option 2. 

- No interplay between globbing syntax, Octostache and file format/file copying syntax.
- No impact on library steps/3rd party tools.

# Decision 3: How to determine file formats
Structured Variable Replacement now supports JSON, XML, YAML and .properties files. It is easy to create files that can parse in multiple formats (YAML is a superset of JSON, and some files can parse as both YAML and .properties), so we need a way to determine file types.

## Option 1: Allow user to specify file formats
In this option, we would extend the target file syntax to something like this:

```
<yaml> app.conf
<xml> logging.configuration
```

Pros:

- Users get absolute control over their file formats.

Cons:

- '`<`', '`>`' and '` `' are all valid characters in Linux filenames.
- '` `' is a valid character in Windows filenames.
- As in Decision 2 (file copying), there is an interplay between the globbing, Octostache and file format syntaxes. Were we to adopt the `=>` syntax in Decision 2, there would be another interplay.
- Library step templates aren't versioned, so this field would be misinterpreted by old versions of Octopus.

## Option 2: Try all parsers
In this option, for each file, we would try all the parsers and see which one worked.

Pros:

- Can be backwards compatible by always trying JSON first.
- Doesn't require users to specify a file format.

Cons:

- Error reporting is difficult. If the file is invalid in all formats, which parser errors should you show? If we end up supporting 20 formats, showing 20 errors to the user would be overwhelming.
- Some formats are extremely permissive. The only way to make an invalid .properties file is a bad unicode escape (e.g. `\u000`). As we add more formats it will become harder to determine what the file format is.
- If it's "first valid parse wins", the order in which we try parsing is critical, and there might not be an order that satisfies everyone. Furthermore, adding a new parser in a future version of Octopus might break existing deployment processes unless it's added as the last parser.
- For a well-known file extension (like a `.yml` file), it may be unexpected for a user to see that we first tried to parse it as a `.xml` file.

## Option 3: Rely on well known file extensions
In this option, we only support well known file extensions. e.g. `.xml`, `.yaml`, `.yml`, `.json`, and `.properties`.

Pros:

- Unlike option 2, we don't have to try all parsers.
- Can be backwards compatible by always trying JSON first.
- Doesn't require users to specify a file format.
- No surprising positive parses.

Cons:

- We won't support different file names. If you need a different file name, you'll have to rename the file in your package, or use custom scripts to rename the file, let it be transformed, and then rename it again.

## Option 4: Rely on well known file extensions, and fall back to trying all parsers

This option uses the strategy in Option 3 first, and if that fails to match a file extension, it fals back to the strategy in Option 2.

The algorithm works like this:

- If a file matches a well known file extension (e.g. `.xml`, `.yaml`, `.yml`, `.json`, and `.properties`), we will only try to use the parser for that file type (after trying JSON first for backwards compatibility).
- If a file does not match a well known file extension (e.g. a `.config` file), we will try all parsers in a hand-crafted order which tries the most restrictive formats first, and the most permissive formats last.

Pros:

- Unlike option 2, the behavior is very predictable for well known file extensions. e.g. It will never try to parse a `.yml` file as a `.xml` file.
- It can work with files of any type without workarounds. A common use case is `.config` files. These tend to be XML files, but this is not guaranteed.
- Can be backwards compatible by always trying JSON first.
- Doesn't require users to specify a file format.

Cons:

- Error reporting is difficult. If the file is invalid in all formats, which parser errors should you show? If we end up supporting 20 formats, showing 20 errors to the user would be overwhelming.
- Some formats are extremely permissive. The only way to make an invalid .properties file is a bad unicode escape (e.g. `\u000`). As we add more formats it will become harder to determine what the file format is.
- If it's "first valid parse wins", the order in which we try parsing is critical, and there might not be an order that satisfies everyone. Furthermore, adding a new parser in a future version of Octopus might break existing deployment processes unless it's added as the last parser.
- There may be edge cases where it parses a file as the wrong type. For example, there may be properties files that sometimes get parsed as yaml files.

## Decision
We have gone with option 4.

# Data Sources

Source listed below might provide additional context but keep in mind that they can disappear at any time.

- [Pitch](https://docs.google.com/document/d/1c2FzUglohWoNSJycs8kCjeYhbFDSkz_vizNtwsbX3pc/edit)
- [Slack](https://octopusdeploy.slack.com/archives/C016F9TQESH)
- [Deprecation pitch](https://docs.google.com/document/d/1Jo8cxYoqiNK1HqgO4jayUTHcXs4etWbunQPA_faSkPw/edit)
- [API Versioning proposal](https://github.com/OctopusDeploy/Architecture/pull/11)
- [Deprecation strategy proposal](https://github.com/OctopusDeploy/Architecture/pull/18)