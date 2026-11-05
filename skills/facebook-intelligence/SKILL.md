---
name: facebook-intelligence
description: When the user wants to scrape, monitor, or analyze public Facebook content: group posts, page posts, profile posts, photos, post comments, comment replies, or social listening data. Also use when the user says "Facebook group scraper", "Facebook profile scraper", "Facebook comments scraper", "export Facebook comments", "monitor a Facebook group", "collect comments from a post", "Apify Facebook Actor", or "MCP Facebook scraping". Routes to the right spbotdel Actor, runs it through Apify MCP (call-actor, get-actor-run, get-dataset-items, get-key-value-store-record), validates coverage from SUMMARY or OUTPUT, and returns counts, gaps, estimated cost and next steps. Public logged-out content only; no private groups, friends-only profiles, or login-gated threads. For building or debugging Apify Actors, see apify-actor-development. For non-Facebook scraping, see apify-ultimate-scraper.
metadata:
  version: 1.0.0
  author: spbotdel
---

# facebook-intelligence

## When to use

Use this skill when the user wants structured data or a short brief from **public** Facebook content:

- Posts and photos from a public group.
- Posts and photos from a public profile.
- Posts together with their comments from a public group, page or profile.
- Comments and replies from specific public post permalinks.

Do not use it for:

- Private, closed or login-gated content.
- Comments on Reels, photo or Watch URLs. These aren't served to logged-out visitors.
- Any request that needs features not listed below.

If the request falls outside scope, explain the limit and stop.

## Actor routing table

| Request | Actor | ID | Reads |
|---|---|---|---|
| Public group posts and photos | `spbotdel/facebook-group-posts-all-photos-scraper` | `tsjhNki5YJUV5avND` | dataset + SUMMARY |
| Public profile posts and photos, including cursor backfill | `spbotdel/facebook-profile-posts-all-photos-scraper` | `dh91XwP7wQscfKkxU` | dataset + SUMMARY |
| Posts and comments in one run (group, page or profile) | `spbotdel/facebook-group-posts-comments-scraper` | `D4fo5uWHFVEb4FYJ2` | dataset + SUMMARY |
| Comments and replies from known post permalinks | `spbotdel/facebook-comments-scraper` | `Ul5IlSzk7K0Lh8Qd6` | dataset + OUTPUT |

Routing rules:

1. **The user already has post permalinks and wants comments** → Comments Actor.
2. **The user wants posts and comments from one surface** → B (`facebook-group-posts-comments-scraper`). For comments, pass a post permalink rather than only a page feed. A page feed may return 0 comments.
3. **The user wants posts and photos only:**
   - From a group → Group Actor.
   - From a profile → Profile Actor.
4. **The request involves Reels, photo or Watch comments, or private content** → don't run anything. Report the limit.

## Workflow

1. **Choose the Actor** using the routing table. Confirm that every URL is public.
2. **Start the run.** Use `call-actor` on the hosted MCP server https://mcp.apify.com.
   - Pass the minimal input below. Change only the URLs and limits the user asked for.
   - Pass the matching `callOptions`: `memory`, `timeout` and `maxTotalChargeUsd`.
3. **Poll.** If `call-actor` returns `RUNNING`, call `get-actor-run` until the run reaches a terminal state.
4. **Read the rows** with `get-dataset-items` from the run's default dataset.
5. **Read the run report:**
   - Group, Profile and B use **SUMMARY**.
   - Comments uses **OUTPUT**.
   - If the report isn't among the dataset rows, fetch it with `get-key-value-store-record` using the key `SUMMARY` or `OUTPUT`.
6. **Validate** against the checklist below.
7. **Return a compact intelligence summary:**
   - Source URLs and the Actor used.
   - Items requested vs. returned.
   - Coverage status and any early stop.
   - Key themes or notable posts and comments, drawn only from the returned data.
   - Estimated cost.
   - Known gaps and suggested next step, such as a continuation cursor or a permalink retry.

## Minimal inputs

**Group** (callOptions: memory 1024, timeout 300, maxTotalChargeUsd 0.05)

```json
{"groupUrls": ["https://www.facebook.com/groups/<group>"], "maxPostsPerGroup": 5, "expandAllPhotos": false}
```

**B: posts + comments** (callOptions: memory 1024, timeout 600, maxTotalChargeUsd 0.10)

```json
{"startUrls": [{"url": "<public group, page, profile or post permalink URL>"}], "maxPostsPerGroup": 3, "commentsMode": "full", "maxCommentsPerPost": 20, "includeReplies": true}
```

For reply-heavy threads, the verified recipe is a **post permalink** plus `commentsMode: "full"`, `maxCommentsPerPost: 50`, `includeReplies: true`, `unionSorts: true`, `commentsSortOrder: "ALL_COMMENTS"`. In deep smoke this returned 50 comments, including 25 replies.

**Comments** (callOptions: memory 4096, timeout 600, maxTotalChargeUsd 0.25)

```json
{"postUrls": ["<public post permalink>"], "maxCommentsPerPost": 50, "maxPagesPerPost": 5}
```

**Profile** (callOptions: memory 1024, timeout 300, maxTotalChargeUsd 0.05)

```json
{"profileUrls": ["https://www.facebook.com/<profile>"], "maxPostsPerProfile": 10, "expandAllPhotos": false}
```

**Profile backfill:** add `"startCursor"` set to the value of `SUMMARY.profiles[].pointer.nextCursor` from the previous run. Only use a cursor whose `pointer.useFor` is `historical_backfill_only`; never use it for latest-post monitoring.

## Validation checklist

- [ ] The run reached a terminal state and was not stopped by `maxTotalChargeUsd`.
- [ ] Terminal status is `SUCCEEDED`. On `FAILED`, `ABORTED` or `TIMED-OUT`, read `statusMessage` and `chargedEventCounts` instead of reporting an empty result.
- [ ] Rows returned are compared with the requested limits.
- [ ] **Group:** check `coverage_status` on rows and `coverageStatus` in SUMMARY. A logged-out feed often stops early, typically at 71–79 items. Report this as an early stop, not as "the group has no more posts."
- [ ] **B:** if a page feed returned 0 comments, retry with post permalinks. Posts are billed even when they have no new comments.
- [ ] **Comments:** check OUTPUT. Group threads may return only teasers. Reels, photo and Watch URLs aren't served.
- [ ] **Profile:** if more history is needed, record `SUMMARY.profiles[].pointer.nextCursor` for the next run.
- [ ] **Profile:** check `skippedProfilesByChargeLimit` and per-profile `stopReason`; the `chargeLimitReached` field may be absent from SUMMARY.
- [ ] The summary uses only returned data. Gaps are stated explicitly.

## Limits

- **Public content only.** No login, private groups or login-gated threads.
- **Group:** the logged-out feed may stop before the requested count.
- **B:** comments need post permalinks. A page feed alone may yield 0 comments.
- **Comments:**
  - Reels, photo and Watch URLs aren't served logged out.
  - Group threads may be teasers.
  - It runs at 4096 MB memory, which raises the start fee.
- **Profile:** public profiles only. Older history is reached through cursor backfill.

## Pricing

| Actor | Per item | Start fee | Min charge |
|---|---|---|---|
| Group | 0.00249 USD/post on FREE/BRONZE; 0.00237 on Silver; 0.00224 on Gold+ | 0.00005 USD | 0.02 USD |
| Profile | 4.99 USD/1000 posts on FREE/BRONZE; 4.74 on Silver; 4.49 on Gold+ | 0.00005 USD | 0.02 USD |
| B | Paid: 3 USD/1000 posts + 1 USD/1000 comments. FREE: 6 USD/1000 posts + 2 USD/1000 comments | 0.005 USD on all plans | 0.02 USD |
| Comments | Paid: 1.20 USD/1000 comments. FREE: 3 USD/1000 comments | 0.003 USD per GB of memory (0.012 USD at 4 GB) | 0.02 USD |

Comments uses `isPPEPlatformUsagePaidByUser = true`: the caller pays compute and proxy usage on top of event charges. The other three Actors have platform usage paid by the developer; owner/POC runs come from the separate $85/month budget.

**Cost accounting:** worked examples below use customer event charges. The deep-smoke total ($0.0323) is owner **platform usage**, not the same number, and must not be compared directly with customer pricing.

## Worked examples

### 1. "What's being posted in this public group lately?"

- **Route:** Group Actor with `maxPostsPerGroup: 5`.
- **Run:**
  1. Call `call-actor`, then poll `get-actor-run`.
  2. Read rows with `get-dataset-items`.
  3. Read SUMMARY.
- **Validate:** check `coverageStatus`. If the run stopped early, say so.
- **Cost on FREE:** 5 × 0.00249 + 0.00005 ≈ 0.0125 USD. The listed minimum charge is 0.02 USD.
- **Summary:** 5 posts with recurring topics, photo-heavy posts and the coverage status.

### 2. "Pull the comments under this page post and tell me the main reactions."

- **Route:** the user has a permalink, so use the Comments Actor with `maxCommentsPerPost: 50`. First confirm the URL isn't a Reel, photo or Watch link.
- **Validate:** read OUTPUT. Flag any teaser-only threads.
- **Cost:**
  - Paid, at 50 comments: 0.06 + 0.012 = 0.072 USD.
  - FREE, at 50 comments: 0.15 + 0.012 = 0.162 USD.
  - Both are within the 0.25 USD cap.
- **Summary:** dominant sentiment and themes, plus comments requested vs. returned.

### 3. "Give me this public profile's recent posts, then go further back."

- **Route:** Profile Actor with `maxPostsPerProfile: 10`.
- **Cost on FREE/BRONZE:** 10 × 0.00499 + 0.00005 ≈ 0.04995 USD. This sits just under the 0.05 USD cap, so use a 0.06 USD cap (or fewer posts) to leave margin.
- **Backfill:** save `SUMMARY.profiles[].pointer.nextCursor` and pass it as `startCursor` in the next run.
- **Summary:** the 10 posts, their posting pattern, the cursor saved for continuation, and any gaps.

> **B cap note:** the default B input can return up to 3 posts and 60 comments if every collected comment counts toward the 20-per-post limit and is billed. That would cost about 0.074 USD on paid plans, which fits the 0.10 USD cap. On FREE it would cost about 0.143 USD, which exceeds the cap. Raise `maxTotalChargeUsd` or lower the limits on FREE.
