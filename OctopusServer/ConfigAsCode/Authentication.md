# Git Authentication
Octopus has two ways of authenticating with a git repository, **Username and Password/Token**, or **Anonymous**.

## Current Approach
- Use `GIT_ASKPASS` to provide git with credentials (See [Requesting Credentials](https://git-scm.com/docs/gitcredentials#_requesting_credentials))
  - Does not expose credentials in plain text
- Option to authenticate with Username and Password/Token
- Option to authenticate anonymously

### Advantages  
- Credentials cannot be exposed to logs or error messages
- Bundling Password and Token in the same field 
  - Git providers issue Tokens as a substitute for Passwords
  - One size fits all approach for EAP till provider-specific authentication is implemented
  - [Bitbucket](https://confluence.atlassian.com/bitbucketserver/personal-access-tokens-939515499.html) and [GitHub](https://docs.github.com/en/github/authenticating-to-github/creating-a-personal-access-token#using-a-token-on-the-command-line) documentation requires Username must be specified while authenticating with Tokens over the Command Line
- Anonymous authentication
  - Easier setup with git repositories hosted on the local filesystem

### Disadvantages
- Username required when using Tokens
  - When git needs credentials, it will always ask for a username. When using PATs, **the username doesn't need to be correct** (as the PAT is enough to identify a user), but it does need to have _something_.
- Anonymous authentication
  - Anonymous authentication will not work for repositories hosted over HTTPS protocol as anonymous auth only gets read-access to these repositories

## Considered Approach
- Separate Token Authentication method

### Advantages
- Clear naming
- Easier to distinguish between Password and Tokens should the need arise in the future

### Disadvantages
- Made redundant by following the "Other Git" approach  
- Token is the same as Username/Password but without a Username
- _Might_ contradict instructions from git providers
  - Git providers _might_ say "Use this PAT as your password"

## Considered Approach
- SSH Authentication

### Advantages
- Supporting SSH would be important if Octopus were providing git hosting, because SSH is a preferable choice in many cases for clients

### Disadvantages
- The two likely most common git providers, GitHub and AzDO, both recommend personal access token with HTTPS
- As Octopus is a git client, there don't seem to be advantages as a user to enter an SSH key vs entering a PAT


## Data Sources

- [Discussion on modelling UsernamePassword separately from UsernameToken](https://octopusdeploy.slack.com/archives/C01AJE4K3T2/p1612149675169500)
- [Discussion on suuporting SSH](https://octopusdeploy.slack.com/archives/C01AJE4K3T2/p1611880542125600?thread_ts=1611877585.120400&cid=C01AJE4K3T2)  
- [GitHub deprecating UsernamePassword in favour of Tokens](https://developer.github.com/changes/2020-02-14-deprecating-password-auth/)