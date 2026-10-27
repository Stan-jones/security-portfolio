# Azure Fundamentals: Governance forensics and deployment audit trail

## Scenario
An intern was given temporary contributor access and spun up a test environment for an experiment. 
The environment does not meet the company's governance compliance standards and on Monday I investigated the environment and why it was allowed to not comply with our standards.

## Environment
* Reader Access
* Multi-user Azure training tenant

## Investigation
### 1: Locating the incorrect naming convention

I started the investigation by identifying the resource group in this subscription that did not follow the typical naming convention (e.g., rg-workload-environment-region-instance).

_the name has been omitted due to it being a flag answer_

___
### 2: The Inspection of the resource

Inspecting the environment that was spun up, I saw only one resource <img width="1817" height="1169" alt="image" src="https://github.com/user-attachments/assets/0b2cbabb-eb96-4372-912c-af699c0d0bbe" />

Delving into the only resource here, I inspected the Tags blade:<img width="1608" height="733" alt="image" src="https://github.com/user-attachments/assets/78d3089f-01b3-481f-830a-321ce57d1657" />
The tags for this resource connected it to the intern, using an intern specific tag. The tag associating the resource to the Intern was correct, but the incorrect RG name was still allowed through somehow.

___
### 3: Using Deployments to trace provisioning
Locating the resource using the Deployments blade showed one successful deployment:<img width="1347" height="1170" alt="image" src="https://github.com/user-attachments/assets/14665eda-2ae1-4c65-b042-5ac5f7540c43" />
Going into the deployment overview details showed me a record of when it was deployed:<img width="2010" height="560" alt="image" src="https://github.com/user-attachments/assets/3606f217-b328-4e43-bb67-df81a48b2187" />
Thus providing me with a timeline to further associate with the incident at hand and a deeper look into the interns activity.

___
### 4: Checking why a Policy did not automatically prevent this Resource Group name
Locating the Policies:

<img width="824" height="1013" alt="image" src="https://github.com/user-attachments/assets/d7250311-d98c-4b59-b4b2-18add1174f2f" />
<img width="1993" height="635" alt="image" src="https://github.com/user-attachments/assets/3c50757f-a7fd-4fb0-9fda-a1481294a4c4" />

In the picture above there is a name convention policy listed, however, it is one of the non-compliant issues present. This told me that there _is_ a naming convention policy at work but it just isn't being enforced, just reported.

I then reviewed the policy assignment for the naming convention policy that should be enforcing the proper naming of our deployments.<img width="1709" height="590" alt="image" src="https://github.com/user-attachments/assets/fdb3bd6d-3489-4528-ad1c-ccd770b33fff" />

It was here that I noticed the "Parameter Value" set to <b>Audit</b>. The Audit value here isn't enforcing deployments to adhere to a strict naming policy, it is just reporting about the compliance violation while allowing it through.

___
## What surprised me
This lab was my first real interaction with Azure; Clicking through the blades and learning where the Deployments and Policies are located was new to me, I could have searched for them using the search bar but I felt doing that would not give me the most experience thus far with this opportunity. I also learned clicking on a resource inside a blade loads a new screen for the resource and hitting the Back button on my browser did not bring me back to how I got to the resource (e.g., in my first picture I have two blades clicked into and displaying a storage account, if I clicked the storage account, it would change the whole page and I would lose the blades.) This caused me to do a decent amount of going back to the main account page and re-opening the blades for where I wanted to go. I learned to open items like that in tabs.

## Findings and recommendations
1. There was a resource group created that violated company-approved naming convention
2. The resource group in violation was associated to the intern who deployed it
3. Tags were appropriately applied to the resource
4. Policy for naming convention was functioning but set to Audit
5. Would recommend the naming Policy be set to Deny, to not allow further incidents
6. Would recommend custom environments seeks prior approval before being deployed.

## What I learned
1. Navigating blades
2. Discovering locations of Deployments and Polciies
3. Policy inheritence

4. One thing I'd do differently,
   Any Policy that needs strict adherence/enforcement, set to Deny and not Audit. Not allowing items that break company policy through.

