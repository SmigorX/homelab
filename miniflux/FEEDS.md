# Feeds

A reading list to subscribe to by hand. Miniflux keeps subscriptions in
Postgres and has no declarative import, so this file persists the *intent*
while the feeds themselves live in the database — see **Storage** in
[README.md](README.md) for why they are treated as data rather than config.

Every URL below was checked on 2026-09-08 and returns a real feed. Sources
that turned out to have no usable feed — Reddit, NCC Group — are written up in
place rather than listed as if they worked.

## Kubernetes and cloud-native

| Source | Feed |
| --- | --- |
| Kubernetes Blog — release notes and deep dives | `https://kubernetes.io/feed.xml` |
| CNCF Blog — ecosystem-wide project news | `https://www.cncf.io/blog/feed/` |
| The New Stack — cloud-native journalism | `https://thenewstack.io/feed/` |

## Homelab and self-hosting

| Source | Feed |
| --- | --- |
| selfh.st — self-hosting news and weekly roundups | `https://selfh.st/rss/` |
| Lobsters, `nix` tag | `https://lobste.rs/t/nix.rss` |

### Reddit

`https://old.reddit.com/r/homelab/.rss` and `.../r/selfhosted/.rss` are the
canonical URLs and they do not work unauthenticated: a plain client gets
`403`, a browser user agent gets the "Welcome to Reddit" interstitial, and
Miniflux gets nothing either way.

Reddit does still publish RSS, but only against a per-account token. Logged in
at `https://old.reddit.com/prefs/feeds/`, each feed listed there carries
`?feed=<token>&user=<username>`; appending that same query string to any
subreddit's `.rss` URL authenticates it. The token is a credential — it grants
read access as you — so a working URL belongs in Vault rather than in this
file.

The two feeds above cover much of the same ground without that.

## Linux and sysadmin

| Source | Feed |
| --- | --- |
| LWN.net — kernel and Linux ecosystem, dense but excellent | `https://lwn.net/headlines/rss` |
| Fedora Magazine | `https://fedoramagazine.org/feed/` |
| NixOS Discourse, Announcements category — releases of Nix tooling | `https://discourse.nixos.org/c/announcements/8.rss` |
| NixOS Announcements — official, low volume | `https://nixos.org/blog/announcements-rss.xml` |
| Determinate Systems — written Nix engineering posts | `https://determinate.systems/rss.xml` |
| Tweag — Nix-heavy engineering blog | `https://www.tweag.io/rss.xml` |

Nix has no single equivalent of Fedora Magazine — there is no curated
distro-news publication. The nearest thing by format is Determinate Systems'
blog (articles rather than release notes), while the Discourse announcements
category is what actually carries the news. `https://discourse.nixos.org/latest.rss`
is the whole forum if you want the firehose instead.

## Security

| Source | Feed |
| --- | --- |
| The Hacker News — daily vulnerability and breach roundup | `https://thehackernews.com/feeds/posts/default` |
| Krebs on Security — investigative, less firehose | `https://krebsonsecurity.com/feed/` |

NCC Group's research blog has no working feed to list. Their old WordPress
endpoints under `research.nccgroup.com` now redirect every path — `/feed/`,
`/rss/`, `/feed/atom/` — to an HTML landing page, so the feed needs finding on
whatever platform the blog moved to.

## General engineering

| Source | Feed |
| --- | --- |
| Hacker News front page | `https://hnrss.org/frontpage` |
| Julia Evans — Linux and networking internals | `https://jvns.ca/atom.xml` |

## GitHub releases

Any GitHub repository exposes its releases as an Atom feed at
`https://github.com/<owner>/<repo>/releases.atom`, with no setup and no token.
It is the cheapest way to track versions of things already running here
instead of checking by hand:

    https://github.com/miniflux/v2/releases.atom
    https://github.com/cloudnative-pg/cloudnative-pg/releases.atom
    https://github.com/argoproj/argo-rollouts/releases.atom
    https://github.com/renovatebot/renovate/releases.atom

The same pattern covers every workload in this repo — swap in the owner and
repo of whatever the Deployment's image comes from.
