# Branch Selector Location

Every cycle, we tend to ask _"Why is the Branch Selector not just on the sidebar so I know which branch I'm working in?"_.

Because ... reasons :) Let's see if we're convinced...

## Current Approach Reasons

### Make it clear what is version-controlled

One of our main objectives from a UX point-of-view is to make it clear what is version-controlled within a project.

Currently only your deployment process is version-controlled (and soon deployment settings), which is why the Branch Selector exists in the paper element for the deployment process page.

When you navigate into something that is version-controlled, that's our trigger for displaying the Branch Selector (so contextually, they stay together).

If we lifted the Branch Selector up to the sidebar (for example), we found it makes the user question which things in the project are actually version-controlled. Perhaps when the _majority_ of a project is version-controlled, _that's_ when we should revisit where the Branch Selector lives? But for now, with very few things in version control, it makes the most sense to live where it does (on the paper element of the thing that's actually version-controlled).

### Keep Release Creation flexible

When you navigate into a project and are in the mindset of creating a release, we feel it's important you have the option of selecting a different gitRef/branch at this point. Moving the Branch Selector up higher could then create confusion when creating releases.


## Alternative Approach #1 - Move the Branch Selector into the sidebar

When you load a project (E.g. just going to the Overview project page), there are various things that still need to know which branch you're working on.

For example:
- Various sidebar links need to embed the branch into their routes
- There are network requests (to the deployment process API) to decide whether we even show the `Create Release` button. So knowing which branch you're on is important to the _whole_ screen, not just select paper layouts where version-control applies.

Moving the Branch Selector up into a central place may simplify a lot of things and is actually consistent with what the user is seeing as a whole.

### Advantages

- It's a single and consistent place for the user to learn where branch selecting/switching occurs = less to think about
- It's less maintenance in our codebase (rather than scattering the Branch Selector throughout various components) = less to test and arguably less risk
- The branch selector is a global concept within a project, it would be nice to have it in a consistent place. Having the selector on individual pages make it looks like an independent idea. Let’s not chop the concept into pieces; it’s good for our user to see the project as a whole.
- Having the selector on mutiple pages may give the impression that they are independent controls, when really they are both representing the same value; this could be unexpected for the user - if they are the same control then that implies that they should be in a single consistent location.
- Users do not have to navigate to a page that contains a branch selector to confirm which branch they are currently working on.
- Having a control in a common area might not be ideal in the immediate term (as we have very few things is version-controlled). But, ideally, we want to design a solution for the ultimate goal when more things are under version control. It would be good to choose a reasonable solution from EAP and all the way through, and not moving it around. Re-educated users about a new UI might cost more confusion down the track.
- Moving the branch selector to a central area in the early stage also minimises the changes we need to make later on the project.
- Let’s not move their cheese; design for where we want to be in the future rather than where we are now

### Disadvantages

- The user may be confused about which things are version-controlled in the project
- It may get confusing when creating a release. E.g. You're on a given branch, but then you see a GitRef selection when creating a release. We want to support selecting more than just branches when creating releases. We will supports tags and commits also.
- The selector is less noticeable sitting in the sidebar. As the user's focus would always be on the paper form, unless they need to navigate to the other area in a project using the sidebar menu.
- Concern about the position of the Branch Selector on a smaller screen side.


**Propose solution to the disadvantages:**

- Show the branch selector in an enabled/ disabled state, depending on whether the current page is under version control.
- If a page is version-controlled (process editor page, and soon the deployment settings page) visually enable the branch selector.
- If a page is not version-controlled, visually disable the branch selector. When users hover on the selector, have a tooltip to explain which area is currently version controlled, and highlight those pages on the sidebar nav.


## Alternative Approach #2 - Move the Branch Selector to the top of the paper element
Wrapping the Branch Selector with component Branch Selector Banner (e.g.) and put this component on the top of the paper element.
The banner will be displayed on every page in a project area.  

The Branch Selector Banner component contains the Branch Selector, a help drawer toggle and will display a short message if the page is not version-controlled yet. 

For a page that will never ever be version-controlled, for example, the release page, replace the Branch Selector with a message something like `Releases is stored in the Octopus database`.

### Advantages
- Most of the time user using Octopus will be focusing on the paper element. A user won't miss it, having the Branch Selector on top of the paper element.
- The banner is displayed on every page. If the page is not CAC ready, will display a message `pageName is not version-controlled yet`. This explains why the banner is disabled, but the page will fall under version-control eventually.
- Next to the unavailable message, we could use the help drawer toggle to give further information on the CaC feature in the help sidebar
- On the `Process Editor` page, move the commit button inside the Branch Selector Banner, it makes sense to have everything version control sit together.

### Disadvantages
- It takes up more space on the paper element, and push the current content down a bit.
