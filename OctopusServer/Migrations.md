# Migration Scripts

## Overview

When making changes that affect the state/shape of persisted objects, we should always write a migration script to upgrade existing objects in the database. This is the case even if only pre-release versions of Octopus are affected, as our internal instances as well as Octopus Cloud may have been exposed to that version.

The other benefit is reduced ambiguity, where we don’t rely on:
- Serialization behaviour to remove old properties
- Mainline code to add default values for new properties.

## Problem

Currently it's not clear when we need to write migration scripts when the shape of a persisted object changes, since it’s possible to lean on mainline code to fill in default values for new properties, and serialization to remove old properties.

There are also cases where a bug prevented a property from being filled out, should we go to the trouble of writing a migration script if we are just fixing that bug?

## Proposed Solution

If objects created/manipulated before and after the change look different when persisted, then write a migration script.

This ensures that:
- Objects created/manipulated before and after the change look identical
- We're not relying on mainline code to fill in missing property values (That code might be removed in a future release which will cause a lot of confusion)
- There's no confusion about whether a property exists in a future migration script (whether a removed property got cleared from the JSON on the next save etc.)


## Data Sources

This is an example of when a bug fix didn't add migration scripts:

https://github.com/OctopusDeploy/OctopusDeploy/pull/6449 ([Trello](https://trello.com/c/F0vpvQr6/3495-enabled-features-and-importing-step-templates))