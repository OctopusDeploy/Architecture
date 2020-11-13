# Config-as-Code: Project Settings API
As part of the configuration as code, we would like to move deployment based settings into version control.
The Problem
Currently the deployment based settings exist as part of the project resource and consequently live at the endpoint /api/projects/{id}. The main issue here is that for VCS based projects, this endpoint becomes ambiguous. We can’t specify the branch, so the best we can possibly do is assume the default branch. This may very well become problematic to consumers of our API if they aren’t careful as the settings aren't shared across all branches.

---
## Proposed Solution 1
- We keep the existing project endpoint the same
  - For VCS projects we polyfill with the default branch data
- We add a new endpoint for VCS based projects which is branch aware
   - /api/projects/1/{gitRef}

**Advantages**
- Things keep working as is
- Backwards compatible
- Simple

**Disadvantages**
- We are polyfilling data from the default branch, this could be dangerous

---

## Proposed Solution 2
- We keep the existing project endpoint the same
  - For VCS projects we return a 404 response
- We add a new endpoint for VCS based projects which is branch aware
  - /api/projects/1/{gitRef}

**Advantages**
- Explicit
- Simple
- Backwards compatible up until VCS projects are added

**Disadvantages**
- As soon as you have a VCS based project, you may be surprised that hitting the existing endpoint won’t work for you anymore.

---

## Proposed Solution 3
- Introduce a new endpoint for Project Settings for both non vcs projects and vcs projects /api/projects/{id}/deploymentsettings and /api/projects/{id}/{gitRef}/deploymentsettings respectively
- Remove deployment based settings from the Project Resource
- Projects all and index endpoints would return either sort of project

### Advantages**Advantages**- Very explicit with no surprising behaviour
- Settings are completely separated out and we can add the appropriate link to the project resource as to where to find settings

**Disadvantages**
- Effort involved in doing this, feels a lot higher
- Not backwards compatible
- Unknowns

---

## Proposed Solution 4
- Introduce a new endpoint for Project Settings for both non vcs projects and vcs projects /api/projects/{id}/deploymentsettings and /api/projects/{id}/{gitRef}/deploymentsettings respectively
- Create a v2 of the projects endpoint without the settings
- Polyfill the v1 API for non-VCS Settings


**Advantages**
- Very explicit
- Settings are completely separated out and we can add the appropriate link to the project resource as to where to find settings
- v1 can be retired in the future
- Backwards compatible
- Disadvantages
- Effort involved in doing this, feels a lot higher


Discussion: [See Slack thread](https://octopusdeploy.slack.com/archives/C01AJE4K3T2/p1605158184218100)