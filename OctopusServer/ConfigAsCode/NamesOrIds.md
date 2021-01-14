# Name and Id Considerations

This cycle, we had a chance to review our current approach to name/ids with a fresh perspective. We had some new eyes on the approach, previous questions being re-raised and some joint concerns have been raised. The most notable concern is: **How far the current approach is spreading**. But also, there's concerns around the user-experience for consumers when we deliver this feature.

This document is an attempt to capture all of this information to make sure we're still heading in the right direction.

Note: This was converted from a [Google Doc discussion](https://docs.google.com/document/d/1J_Vjw1pH5t9ptUI2F6NLyAlSn7WvGBh9LeL1HrUUPdQ/edit).

Discussion: [See Slack thread](https://octopusdeploy.slack.com/archives/C01AJE4K3T2/p1605061937151100)

---
## Current Approach
- We store Names in OCL to make things human-friendly
  - We all agree this is a must-have for any solution we propose
- We send Names to our API in order to populate that OCL
  - This is where complexity has been introduced
  - This is where we may be making things hard for future-us
  - This is where we may risk confusing customers (see disadvantages section)
- We currently do the conversion of Names to IDs at the point a release is snapshotted
  - From this point forward Octopus is dealing with IDs as usual

### Advantages
- Names in OCL are human-friendly
- Consumers of the API generally work in terms of names (be it from the command line, or a script or Hosted configuration). Using names means they don't have to look up IDs.
- We don't pay any conversion-cost of Name-Ids until we're ready to snapshot
- You can provide a Name that doesn't yet exist (but you know _will_ exist when you go to snapshot things)
- For EAP we want to see what a name based API could look like and our customers reactions to that
- We are already well down this road and have it working
- If we were designing the API today, this might be way we would have chosen to do it (names only)
- It makes the API more approachable and easier to consume (and moves us towards our longer-term API goal of using names everywhere, not IDs)

### Disadvantages
- Short term pain, any software-clients that call our API need to be updated to send Names. E.g.
  - our frontend needs switching logic everywhere
  - various octo.exe tooling _may_ need to be updated accordingly as well (NOTE: We want to move the logic in octo.exe into the server and have a command/action based API. When we do, we'd likely want to expose things like create-release as an API that maps the command line parameters into a JSON object to post. The client won't need to map Names to IDs)
  - any existing customer scripts need to be converted to use Names instead
    - “I can't just use my existing scripts!” -- RagingCustomer
- As engineers, there's mounting complexity we're adding here
  - Switching logic everywhere / harder to reason about things
  - Additional test coverage needed everywhere
  - “It's spreading like a virus” is something I've heard a couple of times now
- As a customer, I can't take my existing JSON or scripts and use them for CaC projects
  - “Wait, I need to figure out the name for all the IDs I have in my scripts?” :sadpanda:

---
## Proposed Alternative 1
- We send Ids to our API (like it is in non-CaC projects), and let our API run the conversion from Id to Name / Name to Id respectively on every commit.

### Advantages
- Our software-clients (frontend) and tooling (octo.exe) don't need to change
- The complexity is all managed centrally at the API layer
  - API => OCL = Id to Name conversion
  - OCL => API = Name to Id conversion
- Customer's existing scripts all continue to work for CaC projects

### Disadvantages
- What happens if we can't resolve a Name from the OCL?
  - I.e. You can't provide a Name that doesn't yet exist
  - I'd think we just send through the Name that didn't resolve and let the client deal with the fallout (like we do currently with our ghost chips)
- The conversion work will happen every time you hit Commit
  - Are we concerned about the performance hit?
  - ^ If we measured this and had happy-results, would this concern be squashed?
- Side by side editing of OCL/UI would be harder to implement
- We will have to make sure the customer doesn't use anything that looks like an id, or might ever possibly be an id
   - How do we handle customers who currently have names that look like IDs? (thought he possibility is slim)

---
## Proposed Alternative 2
- We allow you to send Ids or Names to our API (#WhyNotBoth), and we store whatever you send us in the OCL (you send a name, we store a name, you send an id, we store an id).
  - Then there's no confusion about what to send back to the consumer in the API
- At the point of release snapshot, we resolve all TinyTypes to their ID
  - We'll attempt to resolve by ID, if found, we use it
  - Failing that, we attempt to resolve by Name (so ID takes precedence)

### Advantages
- [Same advances from previous alternative]
- As a consumer of the API, I grow to understand that Octopus has first-class support for either Names or Ids for any document references

### Disadvantages
- [Same disadvantages from previous alternative]
- You're not guaranteed to see names in your OCL, we store whatever you send us (but it _can_ be names)
- If we want first-class support for names in our OCL (which we do), we have to continue id/name resolving on our clients for anything we want version-controlled (this is not necessarily a bad thing, but it is effort to consider)
- What happens if I name an environment “Environments-1”?
  - E.g. Id=Environment-34, Name=Environments-1

---
## Proposed Alternative 3
- We allow you to send Ids and/or Names to our API. For example
`{ “Environment”: { Id: “Environments-1”, “Name”: “Production” } }`
- The API will return as much info as it can. I.e both in a non-CaC project. In a CaC project, the ID would be included if it can be resolved.
- Either ID or Name is required. If both are supplied, the consumer makes the choice. The server will always use the ID if supplied

### Advantages
- More information
- Best of both worlds
- No confusion about whether it's an ID or name

### Disadvantages
- It changes the shape of our models
- Data size increases
- Mapping lookups always have to be done
