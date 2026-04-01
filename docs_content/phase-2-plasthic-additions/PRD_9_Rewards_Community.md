# PRD 9: Reward System & Community

## Overview

| Field | Value |
|-------|-------|
| **Module** | 12 — Community (+ Rewards from Module 11) |
| **System** | System 2: Phlastic Cabinet Frontend + Patient Service |
| **Owner** | Iga Narendra (CV Diantha) |
| **Estimate** | 4–5 days (Community) + included in PRD 8 (Rewards) |
| **Dependencies** | Modules 7 (auth), 9 (dashboard) |
| **Priority** | P2 — Engagement and retention features |

## Goal

Drive patient engagement through a **points-based reward system** that automatically awards points for key actions, and a **community platform** where patients can join groups, share experiences, and interact — all within the Phlastic Cabinet.

---

## Reward System (`/rewards`)

### Reward Points Engine

Points are awarded **automatically by the backend** when specific events occur:

| Event | Points | Trigger |
|-------|--------|---------|
| Complete onboarding wizard | +50 | `onboarding_completed` event |
| Submit a review | +25 | `review_submitted` event |
| Refer a friend | +100 | (Referral tracking — to be defined) |
| Complete a procedure | +200 | Service `status → completed` |

Points can be negative (spent/redeemed). Redemption logic is tracked now — full redemption feature is deferred.

### Rewards Page (`/rewards`)

**Layout:**

```
Phlastic Rewards

┌─────────────────────────────────┐
│     🏆 Your Balance             │
│         150 points              │
└─────────────────────────────────┘

How to earn points:
• Complete onboarding — 50 points
• Leave a review — 25 points
• Refer a friend — 100 points
• Complete a procedure — 200 points

History:
┌─────────────────────────────────────────┐
│ +50   Completed onboarding    March 30  │
│ +25   Left a review           March 31  │
│ +100  Referred Sarah Johnson  April 2   │
│ -75   Redeemed: Free consult  April 5   │
└─────────────────────────────────────────┘
```

**API calls:**
- `GET /cabinet/rewards` — current points balance
- `GET /cabinet/rewards/history` — paginated list of reward transactions

### Rewards DB Schema (`rewards` table)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| id | UUID | yes | Primary key |
| patient_id | UUID | yes | FK → patients.id |
| points | integer | yes | Positive = earned, negative = spent |
| reason | string | yes | Human-readable reason |
| reference_type | string | no | `review` / `referral` / `service` / `onboarding` |
| reference_id | UUID | no | ID of the related entity |
| created_at | datetime | yes | |

**`patients.points`** is always the current running balance (updated atomically with each reward transaction).

### Rewards Acceptance Tests

- [ ] Points balance shows correct number on dashboard and `/rewards` page
- [ ] 50 points awarded automatically when onboarding is completed
- [ ] 25 points awarded automatically when a review is submitted
- [ ] 200 points awarded automatically when a service is marked `completed`
- [ ] Rewards history shows all transactions in reverse chronological order (newest first)
- [ ] History paginates correctly
- [ ] Points balance updates immediately after each reward event
- [ ] `reward_earned` event logged in `patient_events` for each award
- [ ] `rewards` table entry created for each transaction with correct `reference_type` and `reference_id`

---

## Community Feature (`/community`)

### Goal

Social feature where patients can join groups, share experiences, and interact with others on a similar medical journey.

### 12.1 — Community Groups (`/community`)

**Groups list:**

```
Community

Join groups to connect with other patients

┌───────────────────────┐  ┌───────────────────────┐
│ [Cover image]          │  │ [Cover image]          │
│ Rhinoplasty Support   │  │ Recovery Tips          │
│ 45 members            │  │ 128 members            │
│ [Join]                │  │ [Joined ✓]             │
└───────────────────────┘  └───────────────────────┘
```

**API:** `GET /cabinet/community/groups`

**Create Group Flow (2-step modal):**

- Step 1: Group name + description + cover image upload + public/private toggle
- Step 2: Invite members (optional) → Create

**Join / Leave:**
- `POST /cabinet/community/groups/:id/join`
- `POST /cabinet/community/groups/:id/leave`
- `member_count` on group is updated accordingly

### Community Groups DB Schema

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| id | UUID | yes | — | Primary key |
| name | string | yes | — | Group name |
| description | text | no | null | Group description |
| cover_image_url | string | no | null | Cover photo path |
| member_count | integer | no | `0` | Auto-updated count |
| is_public | boolean | yes | `true` | Public or invite-only |
| created_by | UUID | no | null | FK → patients.id or staff.id |
| created_at | datetime | yes | | |

---

### 12.2 — Group Detail + Posts (`/community/groups/:id`)

```
┌─────────────────────────────────────────┐
│ [Cover image]                            │
│ Rhinoplasty Support — 45 members         │
│ [Leave Group]                            │
├─────────────────────────────────────────┤
│ [Write something...]            [Post]  │
├─────────────────────────────────────────┤
│ Sarah J. — 2 hours ago                  │
│ "Just had my consultation and I'm so    │
│  excited! Dr. Smith was amazing..."      │
│ ❤️ 12    💬 3    ⚑ Report              │
├─────────────────────────────────────────┤
│ Mike T. — Yesterday                      │
│ "Day 5 post-op. Swelling is going down" │
│ [Photo]                                  │
│ ❤️ 24    💬 8    ⚑ Report              │
└─────────────────────────────────────────┘
```

**Features:**
- Create post: text + optional image attachment
- Like a post (toggle on/off): `POST /cabinet/community/posts/:id/like`
- Report a post (sends to admin for review): `POST /cabinet/community/posts/:id/report`
- Chronological feed (newest first)
- Paginated — "Load more" on scroll (or infinite scroll)
- Author shown as "First name + Last initial" (e.g., "Sarah J.") for privacy

**API:** `GET /cabinet/community/groups/:id` (group detail + posts)

### Community Posts DB Schema

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| id | UUID | yes | — | Primary key |
| group_id | UUID | yes | — | FK → community_groups.id |
| author_id | UUID | yes | — | FK → patients.id |
| content | text | yes | — | Post text content |
| image_url | string | no | null | Attached image path |
| like_count | integer | no | `0` | Auto-updated |
| comment_count | integer | no | `0` | Auto-updated |
| status | enum | yes | `active` | `active` / `hidden` / `reported` |
| created_at | datetime | yes | | |

---

### 12.3 — Public User Profiles (`/community/users/:id`)

Visible to other community members — **minimal PII exposure**:

```
Sarah J.
Member since March 2026
Groups: Rhinoplasty Support, Recovery Tips
Posts: 12
```

**Shows ONLY:**
- First name + last initial
- Member since date
- Groups they belong to
- Post count

**Never shows:** full last name, email, phone, medical info, contact details, or photos.

Patient controls their community visibility in profile settings.

**API:** `GET /cabinet/community/users/:id`

---

### 12.4 — Moderation

- Reported posts (via Report button) go to a moderation queue
- Visible in Sellrise CRM or Phlastic admin panel
- Admin actions:
  - Approve (dismiss report, keep post active)
  - Hide post (`status → hidden`)
  - Ban user from group (future phase)
- Reported post `status → reported` until reviewed

---

### Community Acceptance Tests

- [ ] Groups list shows available groups with name, cover image, member count
- [ ] "Join" button works — patient added to group, `member_count` incremented
- [ ] "Leave" button works — patient removed from group, `member_count` decremented
- [ ] Already-joined groups show "Joined ✓" state on groups list
- [ ] Create group flow works (2-step modal: details → members → create)
- [ ] Group detail page shows cover image, member count, and posts feed
- [ ] Patient can create text post in a group they've joined
- [ ] Patient can attach an image to a post
- [ ] Like button toggles on/off (can like and unlike)
- [ ] `like_count` updates correctly after like/unlike
- [ ] Report button sends post to moderation queue (`status → reported`)
- [ ] Post author shown as "First name + Last initial" (e.g., "Sarah J.")
- [ ] Posts paginated (infinite scroll or "Load more" button)
- [ ] Public user profile shows ONLY: first + last initial, member since, groups, post count
- [ ] No PII or medical info on public user profiles
- [ ] Reported posts visible in admin/CRM moderation queue
- [ ] Admin can hide a reported post
- [ ] All community pages are mobile responsive
- [ ] Community feed ordered newest first
