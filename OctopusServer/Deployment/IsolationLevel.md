# Isolation Level

When a action runs on worker or target it either has a `ScriptIsolationLevel` of `NoIsolation` or `FullIsolation`. These isolation levels control whether other 
scripts can run at the same time on that particular worker or target. 

By default, when a script runs on a worker it has `NoIsolation`. i.e Workers are shared by default as the action should not be modifying the machine state.

On a target the default is `FullIsolation` as the action may modify machine state (e.g. modify IIS, registry, filesystem, etc).
i.e Only one action can execute on a target at a time.

## OctopusBypassDeploymentMutex

The `OctopusBypassDeploymentMutex` variable changes this default for the action. If it is set to `True`, the action will run with `NoIsolation`. 
If `False` it will run with `FullIsolation`.

## Waiting

When one script is waiting on another, it will print out a message in the form:
```
Waiting for the script in task [Tasks] to finish as this/that script requires that no other Octopus scripts are executing on this target at the same time.
Waiting on another script in this task to finish as thi/another task requires that no other Octopus scripts are executing on this target at the same time.
Waiting on scripts in tasks [Tasks] to finish. This script requires that no other Octopus scripts are executing on this target at the same time.
```

If the script requires `FullIsolation` it waits for all other scripts to finish so that it has exclusive access. The wait message will use the the words `this script`.

If the script only requires `NoIsolation` it only waits if another script with `FullIsolation` is running. The wait message will use the words `another task` or `that script`.
