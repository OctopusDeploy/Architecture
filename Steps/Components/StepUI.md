# Step UI

## History

The UI for our deployment and runbook steps has historically been implemented within the Octopus Server portal codebase. They have been implemented as react components. [Here is an example](https://github.com/OctopusDeploy/OctopusDeploy/blob/c360e6e45b5dd882d42ca92e523e4ed74f4850fb/newportal/app/components/Actions/dockerRun/dockerRunAction.tsx).

## Key Constraints

**Independent**

Development of our steps should be a value stream that is independent of our core Octopus Server product. Our steps should live in Step Packages that can be included within Octopus Server. All the step-specific implementation details should live within these Step Packages. This means that the UI for steps can be developed independently of Octopus Server.

**Simple**

We should build steps that are simple and easy to maintain. As development of actions scales up, we may end up with hundreds of actions. Every small unnecessary development cost also scales up with the number of actions, so minimising these costs is critical.

## Solution

We should have a _Step UI Framework_ that decouples our Step UIs from the implementation details of our Portal UI (such as the fact that we use React).

Step Packages implement the API exposed by the Step UI API package. Octopus Server includes these Step Packages, and expects the Step UI component to conform to the Step UI API that it references.

![Step UI Architecture](https://user-images.githubusercontent.com/1892715/107308234-fcfcc600-6ad3-11eb-9416-1495cd203b80.png)

The Step UI API should contain high level concepts (sections, code editors, radio buttons, etc.) that can be combined together to build a UI. The Step Package Editor within the Octopus Server Portal can take this UI representation and turn it into rendered DOM.

## Other solutions

### React

The Step UI could be written in react, as react components.

The key distinction between this option and the selected solution is that

- React components would give our Step UIs the flexibility to define any type of custom UI component that has not already been defined in the Step UI API
- A Step UI Framework would strictly constrain the types of components that can be composed together into a Step UI, allowing the Octopus Server portal to cheaply change its design over time without risk of regression or inconsistencies.

We deem the benefit of being able to _cheaply_ change the design of the Octopus Server portal (i.e. without changing hundreds of steps) much more valuable than Step UIs being able to define their own custom UI components.

### Declarative Document (e.g. JSON)

The Step UI could be declared in a document format, such as a JSON document.

This option does not seem viable because there are many places within the Step UI where we would need to encode some logic. These include

- Validation
- Dynamic form content
- Importing/exporting properties (e.g. as a yaml file)
- Showing or hide form elements based on the state of the action properties
- Setting multiple properties when a field value changes

Given this requirement, we would need either:

- a DSL embedded within the JSON document to represent the required logic. This option is far too complex and can be easily ruled out compared to the alternatives.
- a way to reference typescript code that can implement the required logic. At this point, the declarative JSON document has no value over the proposed framework solution.

## References

- [Decision and discussion document](https://docs.google.com/document/d/1zSEVFaK7ke_UUCn2ql-WEmGetIJmqROdEx-yvy6y44A/edit?usp=sharing)
- [Internal consensus in slack](https://octopusdeploy.slack.com/archives/C01GYCD6VUH/p1612411148247400)
- [Wider consensus gathered in zoom](https://octopusdeploy.slack.com/archives/C01JCARGEDV/p1612494323137900)
- [Early spike](https://github.com/OctopusDeploy/sashimi-ui-spike/blob/Framework/intermediate-representation/src/SampleSashimiAction/RunAScriptAction.ts)
