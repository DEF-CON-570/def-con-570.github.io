# DEF CON 570 Site Expansion — Design

**Date:** 2026-09-06
**Status:** Approved for planning
**Repo:** `def-con-570.github.io`
**Domain:** `defcon570.org`

## Problem

The site is a single static page: logo, title, and two links (Meetup, Discord). It
answers none of the questions a prospective member has — what the group does, when
and where it meets, whether a beginner is welcome, how to get in touch. It also has
nowhere to put talk recordings, which the group expects to start producing.

## Goals

1. Convert a stranger who searched "DEF CON group near me" into someone who attends a meeting.
2. Serve returning members as a reference: next meeting, past talks, how to present.
3. Keep publishing cheap — adding a recording or updating meeting details is a commit,
   not a redesign.

## Non-Goals (v1)

Blog / news feed. CTF scoreboard. Newsletter signup. Comments. Member directory.
Each is additive later; none blocks launch.

## Audience

Primary: prospective members and current members, weighted toward the first-visit
experience. Speakers and sponsors are served incidentally by the contact page.

## Stack Decisions

| Decision | Choice | Rationale |
|---|---|---|
| Generator | Hugo (>= 0.162.0, extended) | Blowfish minimum; no Node toolchain needed |
| Theme | Blowfish, git submodule pinned to a release tag | Real landing-page layout; active maintenance; pinning prevents unattended breakage |
| Structure | Multi-page | Four distinct jobs; recordings section grows |
| Host | Cloudflare Pages, building from GitHub `main` | Native Hugo support, per-PR preview deploys, DNS already on Cloudflare |
| Content | Markdown in `content/`, hand-edited | No CMS to run; content portable if the theme is ever swapped |

Theme choice is reversible: content lives in plain Markdown, so a future theme swap
costs a day, not a rewrite.

## Repository and Deployment

Hugo source lives at the root of the existing repo, preserving history.

```
config/_default/    hugo.toml, params.toml, menus.en.toml, languages.en.toml, markup.toml
content/            page content (below)
layouts/partials/   next-meetup.html
assets/             logo, next-meetup flyer
static/             static files served as-is
themes/blowfish/    git submodule, HTTPS URL, pinned tag
```

Theme install:

```
git submodule add https://github.com/nunocoracao/blowfish.git themes/blowfish
git -C themes/blowfish checkout <release-tag>
```

The submodule URL must be HTTPS — Cloudflare Pages cannot clone an SSH submodule.

Cloudflare Pages project settings:

| Setting | Value |
|---|---|
| Build command | `hugo --gc --minify` |
| Build output directory | `public` |
| Production branch | `main` |
| Environment variable | `HUGO_VERSION` pinned to a specific >= 0.162.0 release |

Every pull request gets a preview URL; `main` publishes to production.

### Cutover

`defcon570.org` already resolves through Cloudflare nameservers (`perla`/`harley.ns.cloudflare.com`),
proxied in front of GitHub Pages. No registrar or nameserver change is required.

1. Build and verify the site on the Pages preview URL.
2. Attach `defcon570.org` (and `www`) as custom domains on the Pages project; Cloudflare
   updates the proxied records itself.
3. Disable GitHub Pages in repo settings so two systems are not claiming the domain.
4. Delete the `CNAME` file — a GitHub Pages artifact with no meaning to Cloudflare.

The repo name `def-con-570.github.io` becomes a misnomer after the move. Harmless;
rename only if desired.

Local development: `hugo server -D`, with Hugo installed via `winget install Hugo.Hugo.Extended`.

## Content Structure

```
content/
  _index.md              home
  about.md               what we do, who we are, code of conduct
  meetings.md            when/where, first-timer guidance, upcoming events
  contact.md             reach the organizers, propose a talk
  recordings/
    _index.md            archive index, including empty state
    YYYY-MM-slug/
      index.md           one page bundle per talk
      cover.jpg
data/
  links.yaml             Discord, Meetup, GitHub, and other links — single source
  next_meetup.yaml       next meeting: date, title, location, RSVP link, flyer
```

`data/links.yaml` exists so social links are defined once and consumed by the nav,
footer, and home CTAs, rather than being pasted into three templates.

### Home

Blowfish hero layout: logo, one-line identity, next-meeting strip, two primary CTAs
(Discord, Meetup), then three or four cards summarizing what happens at a meeting —
talks, hands-on work, CTF, all skill levels welcome — each linking to the relevant page.
A newcomer gets the full pitch without scrolling into detail.

### Meetings

Cadence, venue with map, cost, age policy, and an explicit "what to expect your first
time" section. This page carries the heaviest conversion load; it should answer the
anxieties a first-timer will not ask about out loud.

### Recordings

One page bundle per talk. Front matter carries the metadata:

```yaml
title: "Dumping Firmware Off Cheap IoT Junk"
date: 2026-10-15
speaker: "Jane Doe"
youtube: "dQw4w9WgXcQ"
slides: "https://..."
tags: ["hardware", "reversing"]
summary: "One or two sentences for the archive listing."
```

The body embeds the video with Hugo's built-in `youtube` shortcode. Tags produce
filtered index pages at no cost. Adding a talk means adding a folder and committing.

Until the first recording exists, the index renders an empty state directing readers
to a meeting rather than showing a blank list.

### Contact

Organizer contact plus a short "want to present?" pitch: what topics the group wants,
what a talk slot looks like, and how to propose one.

## Meetup Integration

Meetup event details are maintained by hand in a Hugo data file and rendered by the
site's own partial. No iframe, no API, no third-party script — the block matches the
theme exactly because it is the site's own markup.

```yaml
# data/next_meetup.yaml
date: 2026-10-07T18:00:00-04:00
title: "DEF CON 570 Monthly Meetup"
location: "Venue name, City"
rsvp: "https://www.meetup.com/def-con-570/events/314977882/"
flyer: "next-meetup.jpg"   # optional; lives in assets/, not hotlinked
```

`layouts/partials/next-meetup.html` renders this on both the home and meetings pages.
Updating the site for a new month means editing the date and RSVP link and committing.

The partial must guard its own staleness: if `data/next_meetup.yaml` is missing, empty,
or its `date` is in the past, it renders a "no meeting scheduled yet — check Meetup"
state rather than a stale card. Without that guard, a forgotten update silently
advertises a meeting that already happened.

The flyer image is committed to `assets/` rather than hotlinked from
`secure.meetupstatic.com`. A hotlink breaks silently whenever Meetup rotates the URL.

### Rejected alternatives

**Official Meetup embed widget** (group Settings -> Promote). Zero upkeep, but it is a
third-party iframe whose styling cannot be made to match the dark green theme. Theme
consistency was judged more valuable than the saved monthly edit.

**GraphQL API at build time.** Meetup documents API access as a Meetup Pro feature, and
DC570 is not on Pro. Unavailable regardless of effort. For the record, the shape would
have been: JWT flow (the only unattended one; RSA-signed, 1-hour access tokens,
single-use refresh tokens) driving a scheduled GitHub Action that writes
`data/events.json`.

**Public iCal feed** at `https://www.meetup.com/def-con-570/events/ical/`. Verified live
and unauthenticated, carrying UID, DTSTART/DTEND with timezone, SUMMARY, DESCRIPTION,
URL and STATUS per event — everything except venue and flyer image. A scheduled Action
could parse it into `data/events.json` and remove the manual step. Rejected for v1 as
more machinery than one meeting a month justifies. This is the documented upgrade path
if hand-updating ever becomes a nuisance; the data file's shape is deliberately close to
what such an Action would emit, so the partial would not need rewriting.

**Third-party aggregators** (Elfsight, SociableKit). Inject their own scripts and gate
features behind separate pricing.

## Visual Design

Preserve the existing identity — green on dark — through Blowfish's color scheme
configuration rather than by reproducing the current Courier styling. Dark appearance
is the default; monospace headings retain the DEF CON read with better typography and
responsive behavior than the current page.

## Risks

| Risk | Mitigation |
|---|---|
| Theme upgrade breaks the build | Submodule pinned to a tag; upgrade deliberately, verify on a preview deploy |
| Stale meeting card after a forgotten update | Partial renders an empty state when the data file's date is in the past |
| Two hosts claim the domain during cutover | Disable GitHub Pages as an explicit cutover step |
| Recordings section launches empty | Deliberate empty state that redirects the reader to a meeting |

## Open Content Questions

These need answers from the organizers before the pages can be written; none affect
the architecture.

- Meeting cadence, venue, and start time
- Contact address for organizers (shared email, or Discord only)
- Whether a code of conduct exists or needs drafting
- Group tagline and the two-sentence "what we do" summary
