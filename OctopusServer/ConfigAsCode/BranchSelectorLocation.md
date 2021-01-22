# Branch Selector Location

Every cycle, we tend to ask _"Why is the branch selector not just on the sidebar so I know which branch I'm working in?"_.

Because ... reasons :) Let's see if we're convinced...

## Current Approach Reasons

### Make it clear what is version-controlled

One of our main objectives from a UX point of view is to make it clear what is version controlled within a project.

Currently only your deployment process is version controlled (and soon deployment settings), which is why the Branch Selector exists in the paper element for the deployment process.

This is why, when you navigate into something that is version controlled, that's our trigger for displaying the branch selector (so contextually, they stay together).

If we lifted the Branch Selector up to the sidebar (for example), we found it makes the user question which things in the project are actually version-controlled. Perhaps when the _majority_ of a project is version-controlled, _that's_ when we should revisit where the Branch Selector lives, but for now, with very few things in version control, it makes the most sense to live where it does (on the paper element of the thing that's actually version-controlled).

### Keep Release Creation flexible

When you navigate into a project and are in the mindset of creating a release, we feel it's important you have the option of selecting a different gitRef/branch at this point.

## Alternative Approach #1 - Move the branch selector into the sidebar

### Advantages

- it's a single and consistent place for the user to learn where branch selecting/switching occurs
- it removes the need for the GitRef field in the Create Release process (one less field to think about)
- it's less maintenance in our codebase (rather than scattering the branch selector throughout various components) = less to test and arguably less risk

### Disadvantages

- the user may be confused about which things are version controlled in the project. The current design is very clear on which things are version-controlled
