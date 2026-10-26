# Azure Fundamentals: Operation Dead Deply

## Scenario
<!-- 2 to 3 sentences. What was the situation and what question did the investigation answer? Frame it like a work ticket, not homework. -->
An intern was given temporary contributor access and spun up a test environment for an experiment. 
The environment does not meet the company's governance compliance standards and on Monday I investigated the environment and why it was allowed to not comply with our standards.

## Environment
<!-- One list: platform, services, tools, access level. Honest framing: "live multi-user Azure training tenant, Reader access." -->
* Reader Access
* Multi-user Azure training tenant

## Investigation
<!-- The core. Numbered steps IN YOUR OWN WORDS: what you looked at, what you found, what you concluded at each step. 6 to 12 screenshots of meaningful moments (portal views, query results, before/after). -->
### 1: Locating the incorrect naming convention

I started the investigation by identifying the resource group in this subscription that did not follow the typical naming convention (e.g., rg-workload-environment-region-instance).

_the name has been omitted due to it being a flag answer_

### 2: The Inspection of the resource

Inspecting the environment that was spun up, I saw only one resource <img width="1817" height="1169" alt="image" src="https://github.com/user-attachments/assets/0b2cbabb-eb96-4372-912c-af699c0d0bbe" />

Delving into the only resource here, I inspected the Tags blade:<img width="1608" height="733" alt="image" src="https://github.com/user-attachments/assets/78d3089f-01b3-481f-830a-321ce57d1657" />
The tags for this resource connected it to the intern, using an intern specific tag. The tag associating the resource to the Intern was correct, but the incorrect RG name was still allowed through somehow.

### 3: Using Deployments to trace provisioning
Locating the resource using the Deployments blade showed one successful deployment:<img width="1347" height="1170" alt="image" src="https://github.com/user-attachments/assets/14665eda-2ae1-4c65-b042-5ac5f7540c43" />
Going into the deployment overview details showed me a record of when it was deployed:<img width="2010" height="560" alt="image" src="https://github.com/user-attachments/assets/3606f217-b328-4e43-bb67-df81a48b2187" />
Thus providing me with a timeline to further associate with the incident at hand and a deeper look into the interns activity.

### 4: Checking why a Policy did not automatically prevent this Resource Group name
Locating the Policies:<img width="824" height="1013" alt="image" src="https://github.com/user-attachments/assets/d7250311-d98c-4b59-b4b2-18add1174f2f" />
<img width="1993" height="635" alt="image" src="https://github.com/user-attachments/assets/3c50757f-a7fd-4fb0-9fda-a1481294a4c4" />

In the picture there is a name convention policy listed, however, it is one of the non-compliant issues present. This told me that there _is_ a naming convention policy at work but it just isn't being enforced, just reported.

I then reviewed the policy assignment for the naming convention policy that should be enforcing the proper naming of our deployments.<img width="1709" height="590" alt="image" src="https://github.com/user-attachments/assets/fdb3bd6d-3489-4528-ad1c-ccd770b33fff" />

It was here that I noticed the "Parameter Value" set to <b>Audit</b>. The Audit value here isn't enforcing deployments to adhere to a strict naming policy, it is just reporting about the compliance violation while allowing it through.
<!--
## What broke / what surprised me
 The most credible section in the document. Dead ends, wrong guesses, the thing that took an hour. Employers know real work is messy. This section separates you from certificate collectors. -->


## Findings and recommendations
<!-- What you determined, plus 2 or 3 recommendations as if you were reporting to the resource owner. -->
1. There was a resource group created that violated company-approved naming convention
2. The resource group in violation was associated to the intern who deployed it
3. Tags were appropriately applied to the resource
4. 

## What I learned
<!-- 3 to 5 bullets. At least one technical, one "what I'd do differently." -->

