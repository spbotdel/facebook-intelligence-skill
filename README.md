# facebook-intelligence

Agent Skill for public Facebook research with four Apify Actors:

| Request | Actor |
|---|---|
| Public group posts and photos | `spbotdel/facebook-group-posts-all-photos-scraper` |
| Public profile posts and photos | `spbotdel/facebook-profile-posts-all-photos-scraper` |
| Posts plus comments in one run | `spbotdel/facebook-group-posts-comments-scraper` |
| Comments and replies from post permalinks | `spbotdel/facebook-comments-scraper` |

The skill routes a request to the right Actor, runs it through the hosted Apify MCP server, reads the dataset and the SUMMARY/OUTPUT record, validates coverage, and returns a compact intelligence summary.

## Install

The skill lives at `skills/facebook-intelligence/SKILL.md`. Copy that folder into your agent's skills directory, or add this repository as a skill source.

For Codex, the resulting path is:

```text
%USERPROFILE%\.codex\skills\facebook-intelligence\SKILL.md
```

The agent needs access to the Apify MCP server (`https://mcp.apify.com`) and an authenticated Apify account.

## What it handles

- public group posts and photos;
- public profile posts and photos;
- posts with public comments and replies;
- comment threads from known post permalinks;
- coverage validation and cost-aware limits.

It does not access private, closed, friends-only, or login-gated content. Reels, photo pages, and `/watch` comment lists are not served to logged-out visitors.

## Provenance

- Drafted by Opus 5.5, reviewed by Muse 1.3.
- Validated on 2026-11-05 deep smoke: B returned 50 comments including 25 replies; Comments returned 400 unique comments across 40 pages; Profile respected three cost caps.
- Passes the Agent Skills validator from `coreyhaines31/marketingskills` (`1/1 valid`, no warnings).
