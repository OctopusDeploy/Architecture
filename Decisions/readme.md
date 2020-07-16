# Architecture Decision Records

Lightweight architecture decision records help us record _why_ we chose to implement something a certain way for posterity. They aid future-us, and others, in understanding why things are the way they are, and therefore help them more quickly make better decisions when it comes to building on top of those decisions.

Ensuring we have key decisions documented in a common format, stored in a durable, versioned store, and available to appropriate stakeholders ensures we have good historical visibility of all of the key decisions made for our products, that have led us to our present reality. These documents can be used to promote accountability with decision makers, and to inform and drive business and technical conversations within engineering and product teams.

These records are _immutable_. Once a decision is recorded, it should remain as-is for posterity. Any changes to the decision should be recorded in a new ADR.

### Method

1. Create a decision file with the file name _yyyyMMdd-SubjectName.md_ when the subject for a decision is raised
2. Grab the template for the decision file from [this repository](template.md)
3. Fill out the details you have on-hand and create a PR in this repo to facilitate discussion around the decision
4. For any data sources generated around the subject (google docs, slack threads, etc), add a Data Source to the decision file, and commit it to the open PR
5. Ensure updates are communicated into the appropriate channels (#project-{yourProject}, #topic-{yourTopic}, etc) to confirm their accuracy with people who gave input or have a stake in the decision
6. Once a decision has been made, create an Executive Summary for the decision that summarises the options and considerations that went into making the decision
7. Ensure the rest of the decision details are recorded, including the date it was made, who made it, and who owns it
8. Once everything is final, merge the decision PR - there is no going back after this point, any changes will need to be recorded as a new decision
