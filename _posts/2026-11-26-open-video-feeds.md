---
layout: post
title: "What If Video Feeds Worked Like RSS?"
permalink: /open-video-feeds/
description: "Retoob Feed Profile 1 is a draft, JSON Feed-based format for sharing curated video lists independently of the apps that discover or play them."
comments: true
---

I wanted to watch the videos people share on Hacker News and Reddit on my TV.
I also wanted to be able to maintain a list of videos myself, share it with
someone else, and let them subscribe to it. A list could come from a website,
a community, a friend, or a script I wrote.

Why should my TV app need to know anything about Hacker News or Reddit?

RSS solved a version of this problem a long time ago. A reader doesn't have to
understand how every blog works. It only needs to understand the feed. I wanted
the same separation for curated video lists, with enough information for a
reader to play a video and understand why it appeared in the list.

That's the idea behind Retoob and its draft [Feed Profile 1](https://algoplay.com/retoob-feeds/spec/).

## The feed is the interface

Rather than build a TV app with a growing number of integrations, I split the
problem into two parts. A publisher (or connector) understands a source and
produces a standard feed. A reader understands the feed and decides how to
display and play the videos.

```text
Hacker News    Reddit    A friend's list    A website
     |            |             |              |
  Connector    Connector    Publisher      Connector
     |            |             |              |
     +------------+-------------+--------------+
                   Public video feeds
                            |
                    Any compatible reader
                            |
                     Playback provider
```

A publisher can be a Python script, a scheduled job, or just a JSON file someone
maintains by hand. It doesn't need a Retoob account, an API key issued by Retoob,
or a central Retoob service. A reader follows an HTTPS URL.

The video itself stays with its provider. The feed contains links, metadata,
attribution, and playback hints, not a copy of the video.

## Why JSON Feed?

I didn't want to invent another general-purpose syndication format.
[JSON Feed 1.1](https://www.jsonfeed.org/version/1.1/) already describes a list of
items with stable IDs, URLs, titles, content, dates, and images. It also provides
a mechanism for extensions.

Retoob Feed Profile 1 builds on that format using `_retoob` extension objects
at the feed and item levels. A generic JSON Feed reader can still show the
ordinary item data. A Retoob-aware reader gets the additional information
needed for video playback, identity, recommendations, and refresh.

Here is the feed-level extension from a small [hand-curated example](https://algoplay.com/retoob-feeds/tech-night/feed.json)
at the time of writing:

```json
{
  "_retoob": {
    "version": 1,
    "listId": "tech-night",
    "revision": 3,
    "publicationEpoch": "tech-night-20261104",
    "semantics": "snapshot",
    "itemCount": 2,
    "schema": "https://algoplay.com/retoob-feeds/schemas/retoob-feed-v1.schema.json"
  }
}
```

This is an excerpt, not a complete feed. The rest is ordinary JSON Feed plus
the extensions on individual items. The schema and the specification describe
what a complete document looks like.

## One video, multiple recommendations

Suppose someone posts a ten-year-old video on Hacker News today. Another
person adds the same video to a list tomorrow.

There are three separate facts here: the identity of the video, the date it
was originally published, and the dates and context of the recommendations.

A list of videos that collapses these into a single title, URL, and timestamp
loses information. It can also make a metadata refresh look like a new
recommendation, or make an old video someone just discovered look irrelevant.

Within a Retoob feed, one video is represented by one card. Multiple
recommendations for that video are retained on the card, with their own
identities, source URLs, notes, dates, and optional playback offsets.

The specification distinguishes a recommendation from a source mention.
A video appearing in a catalogue or a discussion does not necessarily mean
someone endorsed it. It also distinguishes a known recommendation date from
an import time or an unknown historical date.

## A list is a snapshot

What happens when I remove a video from my list?

A reader that treats every feed as an append-only stream may keep that video
forever. A Retoob feed instead represents the complete current contents of
a list. If a card disappears in a new snapshot, it disappears from that
list in the reader too.

That choice creates some other requirements. If a feed is paginated, the
reader must not combine pages from different revisions. If a refresh fails
halfway through, the last complete snapshot is preferable to a partially
updated list. An explicitly withdrawn publication is different from a
temporary server failure.

Profile 1 defines stable identities, revisions, publication epochs,
conditional HTTP requests, and these refresh behaviors. They are not just
JSON field definitions; they are rules about how two independent systems
agree on what the current list is.

Playback is another boundary. The profile describes external provider
handoff, direct media, web links, and unavailable videos. It does not let
a publisher dictate an installed app, supply credentials, or run code in
the reader. Support for a feed format does not guarantee that every video
provider can be played on every device.

## Trying it out

There are already several different kinds of publishers. The
[Algoplay collection](https://algoplay.com/retoob-feeds/feeds.json) currently
contains 35 feed references, including Hacker News windows, Reddit communities,
HackerTube catalogue categories, and a small hand-maintained list.

You can [browse the feeds in a browser](https://algoplay.com/retoob-feeds/),
subscribe to an individual feed, or subscribe to the collection of feeds.
A collection is simply a document containing feed URLs; it is not a merged
video list.

[Retoob TV](https://github.com/josephgardner/retoob-feeds) is the Android TV /
Google TV reader I have been building to exercise the design. The current
preview can be sideloaded, but the format is deliberately independent of that
app. The public repository contains an offline Python publisher, a Hacker
News connector example, the schemas, and a checker for complete local feeds.

There is a useful tension here. I want publishing a feed to be as simple as
putting a file on GitHub Pages. The draft currently specifies a feed-specific
HTTP media type and particular caching behavior that Pages does not provide.
Retoob TV accepts its ordinary JSON responses, but the specification documents
this as a hosting deviation rather than claiming full HTTP conformance.
That's one of the things I would like to resolve with feedback from other
publishers and readers.

## What's next?

Profile 1 is a working draft, not an IETF standard. The point of publishing
it now is to find out what a second implementation would need, where the
format is unnecessarily specific to Retoob TV, and whether the rules are
practical for people publishing from ordinary websites.

I'm particularly interested in static hosting, portable playback descriptions,
and how other video curators want to represent the relationship between a
video and the people or communities sharing it.

The links to get started:

* [Retoob Feed Profile 1 specification](https://algoplay.com/retoob-feeds/spec/)
* [JSON schema](https://algoplay.com/retoob-feeds/schemas/retoob-feed-v1.schema.json) and [schema reference](https://algoplay.com/retoob-feeds/schemas/retoob-feed-v1.schema.html)
* [Example feed](https://algoplay.com/retoob-feeds/tech-night/feed.json) and [feed collection](https://algoplay.com/retoob-feeds/feeds.json)
* [Publisher examples, TV preview and GitHub issues](https://github.com/josephgardner/retoob-feeds)

I would love to see somebody publish a list from their own website and read
it in an app I didn't write. That would be a much more interesting result
than adding another integration to Retoob TV.
