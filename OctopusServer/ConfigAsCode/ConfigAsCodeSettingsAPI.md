# Config-as-Code: Project Settings API

As part of the configuration as code, we would like to move deployment based settings into version control.
The Problem
Currently the deployment based settings exist as part of the project resource and consequently live at the endpoint /api/projects/{id}. The main issue here is that for VCS based projects, this endpoint becomes ambiguous. We can’t specify the branch, so the best we can possibly do is assume the default branch. This may very well become problematic to consumers of our API if they aren’t careful as the settings aren't shared across all branches. This also extends to other related endpoints such as the all and index endpoints where trying to load the settings from each remote would be problematic.

---

## Chosen Solution
- We keep the existing project endpoint the same
  - We will omit deployment settings for any VCS based project and never polyfill for this sort of project. This means that get/index/all endpoints will have their values defaulted if the project is version controlled. For example, this would be the response for the all endpoint
- We add a settings endpoint for Vcs based projects as well as non Vcs based projects
- We will polyfill the project settings for existing projects which aren't version controlled to maitnain backwards compatibility

**Advantages**
- Simple
- Backwards Compatible
- We keep the door open for versioning the API in the near future by dropping settings on the project resource
- Tooling i.e. integration plugins don't have to be updated to keep working

**Disadvantages**
- We may end up with some confusion and bugs related to settings being defaulted in the API, mainly because some of these settings are boolean values.

We have specifically chosen this as the current option as we have not finalized our strategy around how we want to version our API, even though we feel that this would be a better approach. Unfortunately, we also have limited amount of time left for the cycle, which forces us to choose a more simplistic option for now. The risks associated with this option feels more tolerable since VCS isn't enabled generally quite yet, so we have some time to think about this a bit more and solve it properly.

___


## Considered Solution 1
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
- We may end up with some confusion and bugs related to settings being defaulted in the API, mainly because some of these settings are boolean values. 

---

## Considered Solution 2
- Introduce a new endpoint for Project Settings for both non vcs projects and vcs projects /api/projects/{id}/deploymentsettings and /api/projects/{id}/{gitRef}/deploymentsettings respectively
- Remove deployment based settings from the Project Resource
- Projects all and index endpoints would return either sort of project

### Advantages
- Very explicit with no surprising behaviour
- Settings are completely separated out and we can add the appropriate link to the project resource as to where to find settings

**Disadvantages**
- Effort involved in doing this, feels a lot higher
- Not backwards compatible
- Unknowns

---

## Considered Solution 3
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