# The Stolen ID 


## 1. ENTRY. 
<!-- A user was phished, completed MFA, and had the resulting session token stolen. That token carried an MFA-satisfied claim, so it sailed past Conditional Access. 
That user was also, through years of drift, still an Owner on a legacy connector app. -->
___
## 2. ESCALATE. 
<!-- Using those Owner rights, the attacker minted a new client secret on the legacy app. That secret let them authenticate through the client credentials flow as the service principal itself, 
inheriting the app's directory permissions without ever signing in as a human again. Note the expiry date: set nearly a century out. -->
___
## 3. PIVOT 
<!-- A single secret dies when it gets rotated. So the attacker registered their own app (every standard user can do this by default in Entra)
and added its service principal to the legacy app's Owners list. Now they can re-credential the legacy app forever, even after the first secret is caught. -->
___
## 4. PERSIST 
<!-- Then the backup plan: a custom scope published on the legacy app's Expose an API blade. This turns the legacy app into a callable backend resource, 
which means the attacker's own app can request delegated access to it. -->
___
## 5. LOOT 
<!-- Finally, a redirect URI on the rogue app pointing at attacker-controlled infrastructure. Combining the rogue app's client ID, that redirect URI, 
and the exposed API scope produces a working phishing URL. A victim who is already signed in on a corporate device clicks Accept on a consent prompt, and the authorization code lands on the attacker's server. -->

___
## The detail that shows you understood it, 
<!-- and the single most valuable paragraph you will write: explain WHY this path exists at all instead of just phishing credentials again. Ordinary credential phishing runs into device compliance, 
location rules, and MFA. Consent phishing sidesteps all of it, because the victim is already authenticated on a trusted device. And the resulting OAuth2PermissionGrant is not removed by a password reset, 
not removed by revoking sessions, and not removed by enforcing MFA. Most standard containment playbooks leaves it in place. Say that in your own words and you sound like someone who has actually run an incident. -->

___
## Name the pattern: 
<!-- this is a confused deputy attack. Read more about these types of attacks if you want to further familiarize yourself. The legacy app is a trusted service that will act on a request it should never have authorized. -->

___
## What broke / what surprised me:
<!-- Most people are surprised that any standard user can register an app by default, and that owning an app registration is effectively an unlogged privilege path that a review of Global Admins would completely miss. -->

___
## Findings and recommendations:
<!-- written like a real report: revoke the client secret · remove the rogue service principal from Owners · delete the custom exposed API scope ·
revoke the OAuth2PermissionGrant explicitly, because containment does not remove it · remove the attacker redirect URI · review and reduce the Graph application permissions · 
disable default user app registration · audit every app registration's Owners list the same way you audit directory role membership · alert on new client secrets and new redirect URIs. -->
