# The Friday Deploy (a review)

> **Type:** Pre-production deployment review (a review report, not an incident report)
> 
> **Role:** Cloud Platform Engineer (Reader access)
> 
> **Skills:** Azure Function Apps, managed identities, Blob Storage access levels, App Service access restrictions, baseline comparison, risk prioritization

---

## Overview

A new notification service, `func-intern-notify`, was deployed late on a Friday afternoon into a lab resource group and went unreviewed until Monday. Before promotion to production, 
I reviewed it as the senior engineer on the ticket (with **Reader** access).

## Method

I compared the standard from a healthy production service to the new (intern) deployment. Findings came from noticing what the new service does not do what the working one does do.

I organized the review around five questions that apply to any compute review on any cloud:

| # | Question | Area |
|---|---|---|
| 1 | Is it even in the right place? | Region |
| 2 | How does it authenticate to what it depends on? | Identity |
| 3 | Where does its data actually sit? | Storage exposure |
| 4 | Who can reach it? | Inbound access |
| 5 | What gets fixed today? | Prioritization |

Two techniques made this a review rather than a checklist:

- **Compare against a baseline.** A known-good production system is the best answer key when nobody has written a standard down.
- **Prove it, don't just infer it.** I didn't just read a setting and assumed the risk. I demonstrated the exposure.

---

## The Call: What gets fixed today?

> **Rule: Exposure beats hygiene. Active beats potential.**

| Priority | Finding | Nature | Timeline |
|---|---|---|---|
| 1 | Public storage container | **Active exposure.** Data is readable on the public internet right now (as demonstrated below). | **Today** |
| 2 | No managed identity | A design flaw, needs a code change | This sprint |
| 3 | Wrong region | Ongoing cost and latency, not a security event (but can get expensive) | Next deploy |
| 4 | Open inbound access | Hygiene, on an app nothing routes to yet | Same pull request as the redeploy |

**My reasoning:** All four findings are real, but only one is happening *right now*. The public container is not a risk waiting to materialize. It is an event in progress, 
and I proved it by loading the file in a private browser (signed in as nobody). 

The other three are potential or contained problems that can go through normal tickets: the region costs money every day but is fixed by the next deploy, 
the missing identity is a design flaw that needs a code change, and the open front door is bad hygiene on an app that nothing routes to yet.
Closing the container is fast and easy, but every minute it stays open is another chance for someone to scrape/extract the data.

Ranking is the real deliverable of a review. Anyone can list problems, but a team lead/Manager needs to know what to fix first, and leadership needs risk translated into terms that make sense to non-security people.

**Verdict:** Ship this as blocked until the container is closed.

---

## The Where: Is it even in the right place?

**What I observed:** `func-intern-notify` was deployed in **Australia East**, while the production baseline is in **Central US**.

> <img width="2029" height="1060" alt="image" src="https://github.com/user-attachments/assets/2c1b39d4-0e38-4501-93bf-41f4c56ea88d" />

**How I verified it:** I opened the **Overview** page of `func-intern-notify` and read its location (Australia East), then compared it with the production baseline (Central US).

**Why it matters:** Azure doesn't charge to put data *in* (ingress), but it does charge to move data *out* of a region (egress). With the service in a different region from the resources it works with,
every call crosses a region boundary:

- **Latency:** every invocation waits on a long round trip.
- **Cost:** every byte coming back is billed as egress, on every call, indefinitely...

**What to do:** Redeploy into the platform's standard region (Central US).

---

## The Who: How does it authenticate to what it depends on?

**What I observed:** On the **Identity** blade of `func-intern-notify`, **System assigned was Off** and **User assigned was empty**. The app has no managed identity.

> <img width="2002" height="1013" alt="image" src="https://github.com/user-attachments/assets/d8694f05-bb2f-4c47-badf-d32c11ddad98" />

**How I verified it (baseline):** The production baseline has a **user-assigned managed identity** attached. Paired with a scoped role assignment on the storage account, that is why production has no credential to leak 
(Redactions for hints and flags).

> <img width="1618" height="1008" alt="image" src="https://github.com/user-attachments/assets/7352cc50-00e1-48e0-94ff-5b4d343072bd" />

**Why it matters:** With no identity, anything the app uses to reach storage is a **stored secret** by definition. 
Stored secrets can leak through logs, source control, chat messages, or anyone with read access (like me), and they require manual rotation.
A managed identity is issued and rotated by the platform, so there is nothing to steal.

**What to do:** Attach a user-assigned managed identity (like seen in Prod baseline), grant it a role scoped to the storage account (least privilege), and remove any stored credential.

---

## The Leak: Where does its data actually sit?

**What I observed:** In the intern's storage account (`stinternnotify...`), under **Data storage > Containers**, the **Anonymous access level** column showed one container set to **Blob** (anonymous read access for blobs), 
while the others were private.

> <img width="2185" height="877" alt="image" src="https://github.com/user-attachments/assets/1e767e25-5470-4f5d-9f42-20aa5b13147a" />

**How I verified it (prove it, don't infer it):** A setting only shows a *misconfiguration*. To show real *exposure*, I opened the public container, selected the blob inside it,
and copied its URL from the properties page. I pasted that URL into a **private browser** and the file loaded with no account, no token, and no sign-in needed.

> <img width="1738" height="773" alt="image" src="https://github.com/user-attachments/assets/e6f58127-593f-4e65-9f6c-56059d25cd96" />
> <img width="972" height="306" alt="image" src="https://github.com/user-attachments/assets/c5d9e2ee-e535-4b0d-a276-d2d39aecf876" />

**What was exposed:** The blob was the intern's informal deploy notes, readable by anyone. They did not contain a credential, but they came close and revealed a lot:

- The **connection string** was not in the file, but the "notes" said to get it from a chat messages. Meaning the secret is being passed around outside any secret store.
- The **managed identity** was never set up (..yet)
- The **inbound restrictions** were skipped on the assumption that the service has to be reachable to work. (In this case, by anyone and everyone..)
- The file also contained a **lab flag** (omitted here).

The exposed file documented three of the other findings in the intern's own words.

**Why it matters:** This is the difference between "misconfigured" and "exposed." Anyone with the URL can read this *right now*. 
It is an event in progress, not a potential risk, and even without a credential, the notes give an attacker a map of how the service is built and which security steps were skipped.

**What to do:** Set the container to **Private** immediately. Consider disabling anonymous blob access on the storage account so it can't happen again, and remove/delete the notes. 
Because we found out the connection string was shared through chat, rotate it as a precaution, and move to a managed identity so there is no secret to share.

---

## The Door: Who can reach it?

**What I observed:** On `func-intern-notify`, under **Settings > Networking > Inbound traffic configuration**, public access was **Enabled with no access restrictions**. 
There were no rules: **Action: Allow, Source: Any**, and the **unmatched rule action** was **Allow**.

> <img width="1000" height="1165" alt="image" src="https://github.com/user-attachments/assets/1ca6b7fb-4a97-42cb-8a40-630373311071" />
> <img width="1355" height="1077" alt="image" src="https://github.com/user-attachments/assets/b612d813-081e-48cf-8798-18537509e74a" />

**How I verified it (against baseline):** Production has **one explicit deny rule** (`plat..flag redaction`, priority 65500, source `0.0.0.0/0`). Once any rule is added, 
Azure's App Service changes its default catch-all (priority 2147483647) from Allow all to **Deny all**.

> <img width="1049" height="1191" alt="image" src="https://github.com/user-attachments/assets/4427f7d9-f5ce-4fc7-93a2-844eea98bceb" />
> <img width="1359" height="1174" alt="image" src="https://github.com/user-attachments/assets/0727ffc0-f568-41e6-8a4d-9bb565059d11" />

**Why it matters:** The service doesn't need to answer any inbound requests, even though the intern thought it needed to be reachable (it does not since it is a scheduled job). 
Luckily, nothing routes to it right now so this is **hygiene**, not an active exposure, but any endpoint or bug added later would be reachable by default.

**What to do:** Add a single deny rule.

---

## Recommendations

1. **Close the public container immediately**, scrub/delete the exposed notes, and rotate the connection string that was shared over chat.
2. **Add a user-assigned managed identity** with a scoped role on the storage account, and remove any stored credential.
3. **Redeploy into the platform's standard region** (Central US) to eliminate cross-region latency and egress charges.
4. **Add a single deny rule** to close inbound access. One rule flips all unmatched traffic to implicit deny.

---

## Takeaways

- **Compare against a baseline.** It's one of the fastest ways to spot misconfigurations.
- **Verify, don't infer.** Proving an exposure shows the risk is real.
- **Prioritization is the skill.** Listing findings is easy. Ranking them by real impact is what a team lead acts on.
