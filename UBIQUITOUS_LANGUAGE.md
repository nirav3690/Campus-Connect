# Campus-Connect — Ubiquitous Language

> **Project:** Campus-Connect — Centralized College Club Notification Platform  
> **Repository:** https://github.com/nirav3690/Campus-Connect  
> **Course / Context:** Design & Technology (D&T) Project  
> **Domain:** Campus Communication, Multi-Tenant Notification Distribution, Club Lifecycle Management  
> **Target Delivery Date:** 30 Oct 2026, 11:59 PM  

---

## 1. Domain Vision & Purpose

Colleges host dozens of diverse student clubs (technical, cultural, sports, societal, and special-interest). Currently, announcements are scattered haphazardly across unstandardized social media handles, cluttered email inboxes, and informal messaging groups.

This platform provides a **lightweight, centralized notification broadcast system** rather than a content-hosting or social media platform. Existing club websites and social platforms remain external destinations. The platform acts as a single, multi-tenant broadcast hub where notifications are partitioned into **Public** (open to the entire campus) and **Members-Only** (restricted to verified club rosters managed by authorized club conveners).

---

## 2. Core Glossary & Terminology

### 2.1 User Roles & Personas

| Term | Domain Definition | Business Rules & Permissions | What it is NOT |
| :--- | :--- | :--- | :--- |
| **Student** | Any authenticated member of the college community possessing a valid institutional email domain (`@college.edu`). | Can view the entire centralized Public Feed, bookmark notifications, search/filter by club and tag, and view personalized Members-Only Feeds for clubs they are enrolled in. | Not an unauthenticated guest; not automatically a member of any club. |
| **Club Member** | A Student whose institutional email has been explicitly added to an active Club's membership roster by an authorized Club Convener or Deputy Convener. | Inherits all Student permissions plus access to view internal Members-Only notifications for that specific club. | Not a self-subscribed follower; membership cannot be claimed without admin authorization. |
| **Club Convener (Admin)** | The primary student or faculty lead officially designated in charge of a recognized club. | Holds full administrative control over the club’s tenant profile, social media link-outs, member roster additions/deletions, and publication/archival of public and members-only notifications. | Not a global system administrator; authority is strictly isolated to their specific Club tenant. |
| **Deputy Convener** | Co-lead or vice-lead authorized by the Convener to assist in club administrative duties. | Holds delegated administrative rights to compose notifications and manage members within their club. | Not superior to Convener; cannot reassign ownership or decommission the club. |
| **Platform Administrator** | Institutional system administrator (e.g., student affairs / IT deanery). | Provisions new Club Tenants, handles convener assignment, enforces platform compliance, and audits system health. | Does not curate or approve daily notifications. |

---

### 2.2 Domain Entities & Aggregates

| Term | Domain Definition | Invariants & Attributes |
| :--- | :--- | :--- |
| **Club (Tenant)** | An autonomous, college-recognized student organization operating as an isolated tenant within the shared application. | **Attributes:** `clubId`, `name`, `category` (Technical, Cultural, Sports, Special-Interest), `description`, `logoUrl`, `externalLinks` (website, Instagram, LinkedIn, Discord), `convenerEmail`, `isActive`.<br>**Invariant:** Must be uniquely identifiable. Multi-tenant isolation ensures data belonging to Club A cannot be modified by Club B. |
| **Notification** | A concise, structured broadcast message published by a club to convey actionable announcements, deadlines, drives, or tasks. | **Attributes:** `notificationId`, `clubId`, `title`, `summary`, `bodyMarkdown`, `audienceScope` (`PUBLIC` \| `MEMBERS_ONLY`), `categoryTag`, `externalLinkUrl`, `publishedAt`, `expiresAt`, `authorEmail`.<br>**Invariant:** Must belong to exactly one Club. Cannot be altered into a discussion thread. |
| **Audience Scope** | The security and visibility classification determining who can read a given notification. | Permitted values: `PUBLIC` (campus-wide) or `MEMBERS_ONLY` (restricted to the club roster). |
| **Public Notification** | An announcement intended for campus-wide dissemination (e.g., recruitment drives, hackathons, open workshops, cultural fest auditions). | Visible to all logged-in Students on the global campus feed. |
| **Members-Only Notification** | An internal operational directive (e.g., rehearsal schedules, core team meetings, internal task requests, inventory/logistics forms). | Strictly restricted: Queryable and readable only by Students currently on that Club’s Member Roster. |
| **Member Roster** | The authoritative list of college email addresses representing confirmed students belonging to a club. | **Attributes:** `rosterId`, `clubId`, `studentEmail`, `addedAt`, `addedBy`.<br>**Invariant:** Maintained solely by Convener/Deputy Convener via direct email insertion or CSV batch roster import. |
| **Category Tag** | A taxonomy label attached to a notification to aid filtering (e.g., `Recruitment`, `Workshop`, `Meeting`, `Competition`, `General Update`). | Standardized domain tags defined by platform to prevent noise and ensure consistent multi-club indexing. |
| **External Destination (Link-Out)** | A reference URL directing users to the club's external asset (e.g., external application form, club portfolio, GitHub repo, Discord server). | Preserves the rule that the platform is a notification distributor, not a full CMS or social network. |

---

### 2.3 Presentation & Architectural Terms

| Term | Domain Definition | Functional Scope |
| :--- | :--- | :--- |
| **Centralized Feed** | The primary campus-wide aggregation view displaying all chronological `PUBLIC` notifications across all active clubs. | Allows real-time search, club-category filtering, date ranges, and direct link-outs for any student. |
| **Personalized Layer (Member Feed)** | An authenticated student's personalized feed overlaying their joined clubs' `MEMBERS_ONLY` notifications alongside relevant public notices. | Computes dynamically: `Filter(Notifications where audienceScope == PUBLIC OR (audienceScope == MEMBERS_ONLY AND studentEmail IN Club.Roster))`. |
| **Multi-Tenant Dashboard Template** | A single, shared, reusable administrative UI/dashboard architecture shared across all clubs. | When a convener logs in, the platform contextually injects that club's specific profile, roster, and notification telemetry into the unified interface without generating separate codebases or custom deployments per club. |

---

## 3. Domain Events & Operations

| Domain Event / Action | Trigger | Preconditions | Postconditions & Effects |
| :--- | :--- | :--- | :--- |
| `PublishPublicNotification` | Convener / Deputy Convener submits a new announcement with `AudienceScope = PUBLIC`. | Convener is authenticated; Club is active; title and summary are non-empty. | Notification is committed to the datastore; immediately appears in the Global Centralized Feed of all students. |
| `PublishMembersOnlyNotification` | Convener / Deputy Convener publishes internal communication with `AudienceScope = MEMBERS_ONLY`. | Convener is authenticated; targeting valid club tenant. | Notification is visible strictly to authenticated students whose emails exist in the club's Roster. Blocked from public index. |
| `EnrollMember` | Convener adds a student's institutional email to the Club Member Roster. | Email format is valid (`@college.edu`); adder has Convener role for this tenant. | Student is granted access to future and active `MEMBERS_ONLY` notifications for this club. |
| `RevokeMember` | Convener removes a student from the Roster. | Target email exists in roster. | Student loses immediate visibility of `MEMBERS_ONLY` notifications for this club. |
| `SwitchTenantContext` | Convener logs in or switches dashboard context (for individuals heading multiple clubs). | User email matches assigned convener list. | Shared dashboard template hydrates with that specific club's dataset (analytics, roster, drafts, active notices). |
| `ExpireNotification` | Scheduled time passes `expiresAt` or Convener manually archives. | Notification exists and is currently active. | Notification is removed from active feeds and moved to archive state. |

---

## 4. Domain Invariants & Business Rules

1. **Email Domain Guardrail:** Only institutional emails (e.g., `*.college.edu`) are permitted for authentication, roster enrollment, and notification subscriptions.
2. **Strict Roster Boundary:** A student cannot self-subscribe or request membership through self-service clicks; physical club admission procedures are mirrored by requiring manual Convener addition via college email.
3. **No Content Replication (Link-Out Principle):** The platform strictly avoids duplicating rich content, social photo galleries, or blog engines. It broadcasts metadata (Title, Date, Venue, Summary, Tag, Link-Out) and routes students to the club's existing web presence.
4. **Single-Template Tenancy:** All club portals run against the same UI component tree; tenant segregation is managed via `clubId` row-level/tenant-level query scoping, avoiding per-club site fragmentation.
5. **Feed Dual-Stream Guarantee:** Unauthenticated or non-member users cannot decrypt or retrieve payload data of `MEMBERS_ONLY` notifications under any API endpoint.

---

## 5. Ubiquitous Language Anti-Patterns (False Friends & Discouraged Terms)

To avoid ambiguity across developers, designers, and faculty evaluators, the following terms must **NOT** be used:

| Avoid Term | Preferred Ubiquitous Term | Rationale |
| :--- | :--- | :--- |
| *Blog / Article* | **Notification** | We do not host long-form editorial content; notifications are concise, time-sensitive alerts. |
| *Follower / Subscriber* | **Club Member** | Membership implies authoritative vetting by the convener, not an informal social media follow. |
| *Social Network / Community Forum* | **Centralized Notification Platform** | Avoids features like comment sections, friend requests, likes, or messaging channels. |
| *Separate Websites / Sub-sites* | **Multi-Tenant Dashboard Template** | There is one web application serving multiple club tenants, not dozens of standalone web apps. |
| *User Group / Circle* | **Member Roster** | Roster directly reflects official club membership tied to student institutional IDs. |
| *Broadcast Email* | **Public Notification** | Replaces spam emails with an indexed, discoverable, on-demand feed. |

---

## 6. Document Versioning & Sign-Off

- **Version:** 1.0.0
- **Authors:** Project Team (D&T Lab)
- **Approved For:** Sprint Planning, Domain Model Class Design, Database Schema Formulation, and API Contract Definitions.
