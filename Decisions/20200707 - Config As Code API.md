 - **Subject**: Public API Surface for Configuration As Code Feature
 - **Decision date**: [when was the decision made]
 - **Decision contributors**: Andrew Best, Paul Gradie, Adam McCoy, Matt Richardson, Andrew Moore, Rob Wagner
 - **Decision maker**: TBA
 - **Decision owner**: TBA

Executive Summary: [Created once a decision is made]

Options & Considerations: https://docs.google.com/document/d/1d9XBGd9akeHWn0B-sTPFV6KvfsR17abpBrHtoBRQadE

Decisions: [what was the decision made]

- Resource Identification:
- Resource Location:
- Resource Modification:
- Retrieving VCS metadata:
- Creating Releases:

Revisions: 
**[8/7/2020]**
- Reviewed the proposed approaches with Rob Wagner
- Confirmed the approach of exposing Hypermedia links for Project-related resources off of Branch resource
- Agreed that default Hypermedia links for resources would still be retained on the Project resource. If a project was version-controlled, these links would point to resources on the default VCS branch for the project
- Discussed the potential of version-controlled resources becoming the default and only way to store resources for projects in the future, and this necessitating the API design accommodate that possibility

**[7/7/2020]**
- Discussed the config-as-code spike carried out by Tribe Red with Matt Richardson and Andrew Moore. 
- Arrived at the conclusion we want to discuss Resource Identification further
- Agreed to an approach on Resource Location that would expose Hypermedia links off of the Branch resource
- Confirmed approaches to modification, VCS metadata and Release creation implemented in the spike were appropriate (just needed productionising)