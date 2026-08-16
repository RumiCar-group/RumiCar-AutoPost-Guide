# AutoPost Guide — write the post; everything after that is automatic

A **methods document** for community and activity websites: publish one
article in WordPress, and both the "What's New" section on your front page
and a post on your Facebook Page happen **automatically**. This guide
records how the pipeline is built, how to verify it actually ran, and the
pitfalls we hit in real operation.

[日本語版 README はこちら](README.ja.md) /
This is the record of a setup built and verified on
[www.rumicar.com](https://www.rumicar.com/) in August 2026. Where our
sibling projects
[PhotoStrip](https://github.com/RumiCar-group/RumiCar-PhotoStrip) and
[VideoLoop](https://github.com/RumiCar-group/RumiCar-VideoLoop) published
code, this one publishes **a way of doing things** — there is almost no
code in it. A companion in the same spirit,
[ReleaseNotes-Guide](https://github.com/RumiCar-group/RumiCar-ReleaseNotes-Guide),
automates the leg before this one: publishing a GitHub Release becomes a
paired JA/EN article — which then rides the pipeline described here.

> **How to read this document**: it is the record of one setup that worked
> as of August 2026, not a guarantee that it will work for you. The
> services involved change their UIs, APIs and pricing; checking their
> current documentation is part of any implementation. If it looks useful,
> help yourself.

## The problem we wanted to solve

Our club's website slowly accumulates event reports and technical
articles. Every publish comes with the same manual chores: updating the
front page, and posting an announcement to the Facebook Page. The
motivation was to remove that post-publish work as a category, and keep
only the writing.

The requirements were simple:

1. From the moment a post is published, zero human steps.
2. It appears on the front page as a new item.
3. It gets posted to the Facebook Page, with a link card.
4. Whether it really ran can be confirmed **from logs, not guesses**.

## The big picture

```
Publish a post (WordPress)
   │
   ├─→ appears in "What's New" …… on-site; a newest-N query on the front page
   │
   ├─→ OGP meta output ……………… the material for share link cards
   │
   └─→ RSS feed (/feed/) …………… WordPress standard; it's already there
          │
          ▼
      dlvr.it polls the RSS feed (every few minutes to ~30 min)
          │
          ▼
      auto-post to the Facebook Page (title + link card)
```

The first three live inside WordPress; the rest is an external RSS-to-SNS
relay. **The only thing WordPress hands to the relay is the RSS feed**
(after a post goes live, Facebook itself also fetches the article URL
directly to build the link card — see pitfall #3). That loose coupling is
the useful part: the WordPress side and the relay side can be verified —
and replaced — independently.

## Prerequisites — what needs to be in place

**An RSS feed.** A standard WordPress feature, served at
`https://your-site/feed/` out of the box. Unless a theme or plugin has
disabled it, there is nothing to do. On multilingual sites (Polylang and
similar), feeds are split per language (`/feed/` = default language,
`/en/feed/` = English). rumicar.com connects only the Japanese feed; the
English feed is left unconnected, deliberately.

**OGP meta tags.** The link card on a Facebook post (image + title +
description) is built from the article URL's OGP meta (`og:title`,
`og:image`, …). Output from an SEO plugin or your own code is sufficient.
Pasting one article URL into
[Meta's Sharing Debugger](https://developers.facebook.com/tools/debug/)
shows whether it is in order.

**A "What's New" section.** Mechanically it is just a query for "published
posts, newest first, N items" placed on the front page. A theme's
recent-posts widget, a plugin, or a few lines of your own code all work.
rumicar.com uses its own; conceptually:

```php
// concept: five most recent published posts
$q = new WP_Query([ 'post_status' => 'publish',
                    'posts_per_page' => 5, 'orderby' => 'date' ]);
```

One implementation judgement worth sharing for multilingual sites: when
filtering by the current language yields zero items, rumicar.com falls
back to showing another language's newest posts with a small note, rather
than an empty box.

**A Facebook Page.** The destination of the auto-posts is a Facebook Page
(not a personal profile), and connecting it requires authorizing as an
**admin** of that Page. If the Page does not exist yet, it has to be
created first.

## Procedure — connecting dlvr.it (UI as of 2026-08-11)

We used [dlvr.it](https://dlvr.it/) as the RSS-to-SNS relay. The screens
below are as they were at setup time and may have changed.

1. Sign up at dlvr.it (14-day free trial; plans after that need checking —
   noting the trial end date beforehand avoids finding out the hard way).
2. Automate → New Automation → **Step 1 Input**: choose "RSS Feed" and
   enter the feed URL (`https://your-site/feed/`).
3. **Step 2 Outputs**: choose "Facebook (Page)", authorize via OAuth as an
   admin of the Page, then select the target Page.
4. **Step 3 Review** → Start.

That is the whole setup. From then on, every new item in the feed becomes
a Page post with a title and link card.

Comparable RSS-to-SNS relays exist (Buffer, IFTTT, Zapier, among others).
This guide records a build on dlvr.it; it is not an endorsement of any
particular service.

### Why not the official Meta API

Auto-posting to Facebook can also be self-implemented against Meta's
Graph API. Having compared the two, we chose the external-relay approach
because the official route carries **ongoing maintenance costs**: app
review, plus periodic access-token renewal. The relay approach needs one
OAuth authorization as the Page admin, once. For a small site run by a
few people, that difference dominates. (An organization with review
and ops capacity gets more freedom from the official API — it is a
trade-off, not a verdict.)

## Pitfalls and verification — what only real operation taught us

This section is the reason this document exists. The setup steps are in
the sections above; the following only showed up when the pipeline
actually ran.

### 1. The moment you enable it, the newest article gets posted once

Starting the Automation immediately posts the feed's most recent item. If
that article had already been shared to the Page by hand, **the Page now
shows it twice**. It happened on rumicar.com; deleting the automated copy
resolved it. A quick look at the Page right after enabling catches this
duplication early.

### 2. The "Publisher: dlvr.it" label is visible to Page admins only

Automated posts carry a publisher attribution — but it is shown **only in
the Page admin's view**. Ordinary visitors see a normal post. That is the
measured answer to the worry "will automated posts look robotic?".

### 3. Whether it really ran can be read from your server logs

Beyond trusting the relay's dashboard, your own **web server access log**
shows the whole sequence (this assumes an environment where the raw access
log is readable). Two kinds of visits appear:

```
dlvr.it/1.0 (+http://dlvr.it/fetcher)     ← fetching /feed/ (polling)
facebookexternalhit/1.1                    ← fetching the article URL after
                                             the post succeeds (to build
                                             the link card)
```

The decisive detail is in the second one: the requested URL carries
`?utm_source=dlvr.it&utm_medium=facebook`. dlvr.it appends those
parameters to its posts, so **a Facebook-crawler visit with
`utm_source=dlvr.it` is evidence that the Facebook post went live** — the
success of the pipeline can be confirmed from logs alone, without opening
any dashboard.

Measured example (2026-08-11, times UTC): dlvr.it fetcher reads /feed/ at
09:54–55 → facebookexternalhit fetches the article with the utm parameters
at 09:55–57 → dlvr.it's "Your First Post is Live!" email at 10:02.

### 4. The utm parameters separate this traffic in analytics

As a by-product of #3, every visit arriving via the automated post carries
`utm_source=dlvr.it`, so analytics can count "came from the FB auto-post"
separately from manual shares.

### 5. Cache layers delay everything

With a full-page cache in front (rumicar.com: FastCGI cache, 30 minutes),
both the What's New section and the RSS feed can lag behind a newly
published post until the cache expires. "I published it but it's not in
the feed" — the first suspect is the cache. Add the relay's polling
interval (minutes to ~30 min) and the worst-case delay from publish to FB
post is roughly *cache lifetime + polling period*. In that light,
requirement 1's "from the moment of publish" really reads "within a few
tens of minutes of publish". For announcements that are genuinely
time-critical, posting manually is the practical fallback.

## Disclaimer

- This document records one setup that worked as of August 2026; it is not
  a guarantee of operation.
- dlvr.it, Facebook and WordPress change their behavior, UIs, APIs and
  pricing. Implementation includes checking their current documentation.
- No specific service is being endorsed. The same shape — loose coupling
  through an RSS feed — can be built on comparable services (Buffer,
  IFTTT, Zapier, and others).

## License

MIT. Quote it, republish it, adapt it. If it helps your community's
activity reach people, that is what this document is for.
