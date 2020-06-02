# Documents
`Documents` are the top level types that get stored in the database (or other data store). They are often called `entities` in other systems.

## Id Passing

When passing the ids of documents either individually or in a list, they should be passed as a `TinyType` instead of `string` in all new code. 