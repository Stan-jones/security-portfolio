# The Stolen ID 
> Scenario: Reconstructed a five-stage OAuth consent-phishing kill chain in a live Azure tenant through forensic analysis of two linked app registrations.
> 
> Phished User: Carl
> 
> My Permissions: Read-only access on directory applications
> 
## 1. ENTRY. 
  A user (Carl) was phished, completed MFA, and had the resulting session token stolen. That token carried an MFA-satisfied claim, so it passed Conditional Access. 
Carl user was also an Owner on a legacy connector app.

Opening up the Branding & Properties blade and reviewing the internal notes left behind, we were able to see that legacy app the phished user was the Owner of:
<img width="1230" height="1076" alt="image" src="https://github.com/user-attachments/assets/a83cddf6-d31e-4c26-a9eb-208991e1df0b" />
This legacy app is (was) no longer used but had been forgotten about in the organization, and never removed.
___
## 2. ESCALATE. 
<!-- Using those Owner rights, the attacker minted a new client secret on the legacy app. That secret let them authenticate through the client credentials flow as the service principal itself, 
inheriting the app's directory permissions without ever signing in as a human again. Note the expiry date: set nearly a century out. -->
Where I looked next: _API permissions, then Certificates & Secrets_ blades


Reviewing the legacy apps' permissions; under Microsoft Graph, they were granted: **Directory.Read.All** & **User.Read.All** with admin consent

<img width="1641" height="848" alt="image" src="https://github.com/user-attachments/assets/9845c9a1-d918-4374-9a11-b3dc9f96c78a" />

> Microsoft Graph is Microsoft's API that allows applications to access and interact with Microsoft 365 and other Microsoft cloud services; such as Entra ID, Outlook, Teams, OneDrive & SharePoint.
> 
___
Checking the _Certificates & Secrets_ blade showed me a Client Secret with an expiration year of **_2099_**
The attacker used the phished users' Owner rights to add a new Client Secret to the legacy app and set the expiration date to the secret to _very far_ in the future.

<img width="1638" height="893" alt="image" src="https://github.com/user-attachments/assets/5aa509ce-b6f6-44a4-9889-f3bdd253d78a" />
With the secret, the attacker can authenticate as the app’s service principal using the client credentials flow (app ID + secret).

___
## 3. PIVOT 
<!-- A single secret dies when it gets rotated. So the attacker registered their own app (every standard user can do this by default in Entra)
and added its service principal to the legacy app's Owners list. Now they can re-credential the legacy app forever, even after the first secret is caught. -->
The attacker thought ahead; if Carl's access is removed or the secret is rotated, they would lose access.
So using Carl's Owner permissions, the attacker created a new app registration and set its' Service Principal to the Owners list on the legacy app:
<img width="1097" height="820" alt="image" src="https://github.com/user-attachments/assets/af9c925d-b01f-4169-a4db-744d70a28ea7" />
This became a _Pivot_ because now the attacker no longer needs to act as (or access) Carl's account and can simply act as the app they just created.
___
## 4. PERSIST 
<!-- Then the backup plan: a custom scope published on the legacy app's Expose an API blade. This turns the legacy app into a callable backend resource, 
which means the attacker's own app can request delegated access to it. -->
Where I looked next: Expose an API blade

Using Carl’s Owner rights, they added a custom scope to the legacy app’s Expose an API blade:
<img width="1658" height="984" alt="image" src="https://github.com/user-attachments/assets/e72c92ad-313e-4911-a8d7-0dc3c19b590d" />
The scope is a configuration, not a credential, so rotation doesn’t touch it. Together with the rogue app made by the attacker, it gives them a way back in that doesn’t depend on any secret they’ve already used.



___
## 5. LOOT 
<!-- Finally, a redirect URI on the rogue app pointing at attacker-controlled infrastructure. Combining the rogue app's client ID, that redirect URI, 
and the exposed API scope produces a working phishing URL. A victim who is already signed in on a corporate device clicks Accept on a consent prompt, and the authorization code lands on the attacker's server. -->
The picture below shows the attackers rogue app, with two redirect URI's:
<img width="1815" height="904" alt="image" src="https://github.com/user-attachments/assets/a7ff09e2-f773-4fda-9e4a-06b009aa3bcc" />

A user who consents grants the rogue app a delegated token for the legacy app’s API. The token cannot call directory data directly, but the legacy app’s backend, which uses its own admin-consented permissions to run a sync, can be driven to do privileged work on the attacker’s behalf. 
This is a **confused deputy** attack: a trusted component with real authority carries out a request it should not authorize.

This route avoids Conditional Access controls such as managed device, trusted location, and MFA prompts. The victim is already signed in on a corporate device and approves the consent themselves, so the attacker never signs in. Consent creates an OAuth2PermissionGrant, which persists through password changes, session revocation, and MFA enforcement. Only deleting the grant or the rogue app removes it.
___
## The detail that shows you understood it, 
<!-- and the single most valuable paragraph you will write: explain WHY this path exists at all instead of just phishing credentials again. Ordinary credential phishing runs into device compliance, 
location rules, and MFA. Consent phishing sidesteps all of it, because the victim is already authenticated on a trusted device. And the resulting OAuth2PermissionGrant is not removed by a password reset, 
not removed by revoking sessions, and not removed by enforcing MFA. Most standard containment playbooks leaves it in place. Say that in your own words and you sound like someone who has actually run an incident. -->

Two techniques are used together in this attack now. 
> **1: Consent phishing**: an attack where a threat actor tricks someone into granting a malicious application permission to access their Microsoft 365 or other organizational data through Microsoft Entra ID.
> 
> **2: Confused Deputy**: when a legitimate application or service is tricked into using its own authority or permissions to perform an action on behalf of someone who shouldn't have access.
> 
The assailant uses consent phishing to obtain a delegated OAuth token from a genuine user, thereby bypassing the device, location, and MFA verifications that credential phishing would activate. But because that token alone gives minimal access to the legacy application’s exposed API, the attacker has to implement the _confused deputy_ technique to reap its actual utility: by approaching the backend of the legacy application, which is trusted with its own permissions due to having been approved by an admin, and performs privileged operations for the attacker.

___
## What surprised me:
<!-- Most people are surprised that any standard user can register an app by default, and that owning an app registration is effectively an unlogged privilege path that a review of Global Admins would completely miss. -->
I was surprised to find out that standard users can register a new application by default and become its owner automatically without any review or approval. I didn't think that an account of a regular employee (Carl) could be sufficient to create a completely new and totally malicious identity within the tenant; which would not appear when you simply check who is the administrator.

What surprised me most was realizing how much of this attack never touched a password at all. After the initial phishing of Carl's account, every other step, making a client secret, setting up the rogue app, the exposed scope, the consent grant, was built entirely out of ordinary Entra ID configuration.

The legacy app Carl was an Owner of had the keys to the [metaphorical] city, and the attacker used them to establish access and persistence.
___
## Findings and recommendations:
<!-- written like a real report: revoke the client secret · remove the rogue service principal from Owners · delete the custom exposed API scope ·
revoke the OAuth2PermissionGrant explicitly, because containment does not remove it · remove the attacker redirect URI · review and reduce the Graph application permissions · 
disable default user app registration · audit every app registration's Owners list the same way you audit directory role membership · alert on new client secrets and new redirect URIs. -->
**Finding**: A client secret with an approximately 100-year expiration was added to the legacy app.

`Recommendation: Revoke this secret immediately. It functions as a permanent credential.`

**Finding**: Mad-Hat-Labs-App (the rogue application created by the attacker) was added as an Owner of the legacy app.

`Recommendation: Remove this service principal from the legacy app’s Owners list. As an owner, it can re-add a client secret even after the original is revoked.`

**Finding:** A custom scope was exposed on the legacy app pointing to attacker-controlled infrastructure.

` Recommendation: Disable and delete the custom exposed API scope, or delete the rogue app entirely`

**Finding**: The legacy app has Directory.Read.All and User.Read.All with admin consent, giving it tenant-wide read capability.

`Recommendation: Reduce to the minimum scope needed. This directly limits the blast radius of any future compromise of this app. Or remove app entirely if no longer needed.`

 
