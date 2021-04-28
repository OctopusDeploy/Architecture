# Getting Started

> Our intent is to create a Template Repository that can be leveraged within Github for creating new Step Package repositories. Until then, the below is how to go about creating a new one!

# Repositories

- [Example Step: Azure Storage](https://github.com/OctopusDeploy/step-package-azurestorage)
- [Step API](https://github.com/OctopusDeploy/step-api)
- [Step Package CLI](https://github.com/OctopusDeploy/step-package-cli)
- [Step Bootstrapper](https://github.com/OctopusDeploy/step-bootstrapper)

# Requirements

## Workstation

Step Packages are written in TypeScript, and run in nodejs.

Ensure you have the [latest LTS version](https://nodejs.org/en/download/) of node installed.

[Step API](https://github.com/OctopusDeploy/step-api) and [Step Package CLI](https://github.com/OctopusDeploy/step-package-cli) npm packages are published to a private feed in Github. To access these feeds from your local environment so that you can `npm install` these dependencies within your Step Package, you will need to create a Github [Personal Access Token](https://docs.github.com/en/github/authenticating-to-github/creating-a-personal-access-token) and set up an [.npmrc](https://docs.npmjs.com/cli/v7/configuring-npm/npmrc) file like so:

```
@octopusdeploy:registry=https://npm.pkg.github.com
//npm.pkg.github.com/:_authToken=YOUR_OWN_GH_PAT
```

# Workflow

> Note: API surfaces are still moving. The UI API is still being actively developed in server, and then replicated out to the [Step API](https://github.com/OctopusDeploy/step-api) package periodically. The new step may not work "Out of the box" if there is a mismatch between what is in Server, and what is in the `@octopus/step-api` package.

- Clone an existing [Example Step](https://github.com/OctopusDeploy/step-package-azurestorage)
- Open `%RepositoryRoot%/.github/workflows/main.yml` and update the produced artifact names from `step-package-azurestorage` to `step-package-YOURSTEPPACKAGE`
- Copy all of this sans the `.git` folder to a new folder and `git init && git add -A && git commit -m "initial commit"`.
- Create a new respository and push the content to the new repository.

You are now ready to write some new code!

- Write your step executor code and tests
- Write your step UI code
- Push it to Github and await a green build.
- Grab the `step-package-YOURSTEPPACKAGE.0.0.1.zip` artifact from the green build.
- Place this into `%OctopusServer%/source/Octopus.Server/steps`
- F5 and give it a spin!
