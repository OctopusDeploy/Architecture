# Step Templates in Configuration as Code

## Problem
Previously, when actions based on Action Templates (refferred to as Step Templates in the UI) were serialized as OCL for version controlled projects, all properties defined in the template would be inserted directly into the action, along with a reference to the original template.

Example:
```hcl
action_type = "Script"
properties = {
    Octopus.Action.Template.Id = "ActionTemplates-1"
    Octopus.Action.Template.Version = "12"
    
    // Properties by Template

    // Properties defined by user
}
```

This exposes the pre-defined properties from the template, and allows for them to be edited by the user. This has the potential to cause unintended/undocumented behaviour when using templated deployment actions.

---

## Chosen Solution
Octopus Server now **only** serializes the **user defined parameters** along with a **reference to the template** for version controlled projects.
Octopus Server will resolve this reference at runtime.

Example:
```hcl
properties = {
    Octopus.Action.Template.Id = "ActionTemplates-1"
    Octopus.Action.Template.Version = "12"
    
    // Properties defined by user
}
```

- After fetching the deployment process using the `CurrentDeploymentProcessVersionControlDocumentStore`, Octopus Server will fetch any templates referenced by the deployment actions and insert their pre-defined values
- If any value has been defined in both the action and it's template, the templates values will overwrite the ones already defined

### Pros
- Properties and packages defined in the template are no longer serialised
- Users cannot unintentionally modify the behavour of a templated deployment action

### Cons
- The structure of deployment actions is now inconsistent with what may be seen in the database
### Todos
- Once the generic `IDocumentStore<TDocument>` has been merged, the logic for hydrating and dehydrating deployment actions can be moved into there.

---

## Desired Solution
The desired solution here would be to split the Deployment Action into two separate classes. E.g:
```cs
public abstract class DeploymentActionBase
{
    public PropertiesDictionary Properties { get; } = new PropertiesDictionary();

    // etc.
}

public class DeploymentAction : DeploymentActionBase
{
    public string ActionType { get; }
}

public class TemplatedDeploymentAction : DeploymentActionBase
{
    public string TemplateId { get; }
    public int TemplateVersion { get; }
}
```

This way, we can better differentiate between a standard deployment action, and a deployment action based on a template.
Unfortunately, the current architecture of Octopus Server is not ready for such a change.

---

## Considered Solution
Custom OCL converter

### Pros
- No additions to existing pipeline

### Cons
- Converter must depend on the `IActionTemplateStore`
  - Converter is no longer _just_ a converter, as it's now making database calls
- Unable to take a dependency on `IActionTemplateStore` as JSON and OCL converters are instantiated in the global scope