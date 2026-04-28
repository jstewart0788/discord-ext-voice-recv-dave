# Fork rationale

This is a soft fork of [`imayhaveborkedit/discord-ext-voice-recv`](https://github.com/imayhaveborkedit/discord-ext-voice-recv) carrying community PR #54 plus one safety hardening commit on top.

## Why the fork exists

Discord made voice end-to-end encryption (DAVE) mandatory on 2026-03-02. Without DAVE support, voice connections are dropped with WebSocket close code 4017. The upstream library (`imayhaveborkedit/discord-ext-voice-recv`) has had no commits since 2025-06-18 — it is effectively unmaintained through the DAVE rollout.

Two community PRs exist on upstream to add DAVE receive support: [#54](https://github.com/imayhaveborkedit/discord-ext-voice-recv/pull/54) and [#56](https://github.com/imayhaveborkedit/discord-ext-voice-recv/pull/56). Both are open, neither merged.

## What this fork carries

1. **PR #54 cherry-picked verbatim** — 5 commits by `rdphillips7` adding DAVE decryption to `opus.py` and a router safety check to `router.py`. Decryption is delegated to Discord's official `davey` library binding (libdave), not re-implemented.

2. **One additional safety commit** — replaces bare `except:` in `opus.py` with `except Exception:` so `KeyboardInterrupt` and `SystemExit` propagate correctly. Adds `log.debug` calls so consumers can build decrypt-failure-rate and member-resolution-miss metrics. Separates the "no member for SSRC" path from the "davey decrypt failed" path. No behavior change on the happy path.

## What this fork does NOT do

- It does not re-implement any cryptography. All DAVE protocol handling lives in `davey` (and its underlying `libdave` from Discord).
- It does not modify the existing voice-receive packet flow except for the DAVE decrypt insertion point.
- It does not vendor `davey` or `libdave`. They remain external pinned dependencies.

## Pin guidance

Consumers should `pip install` this fork from a **specific commit SHA**, not a branch name. Branches can be force-pushed; SHAs are reproducible. Example:

```
discord-ext-voice-recv-dave @ git+https://github.com/jstewart0788/discord-ext-voice-recv-dave.git@<sha>
```

Pair with `discord.py` master pinned to a SHA (DAVE protocol on the send side) and `davey>=0.1.4` (the Python binding to libdave).

## When this fork retires

The fork retires when EITHER condition holds:

- **Upstream merges PR #54 (or equivalent)** and tags a release. Switch back to `imayhaveborkedit/discord-ext-voice-recv@<release-tag>`.
- **py-cord 2.8.0 ships with PR #3159 merged** (receive-side DAVE in py-cord core). The architecture can move back to py-cord and drop both this fork and the discord.py master pin.

Until then, the maintainer of this fork is responsible for the audio-decryption path. Audit changes accordingly.

## Operational notes for consumers

The author of PR #54 is a first-time contributor (`rdphillips7`, one public repo, no prior PRs). The diff is small (+26/-4) and was reviewed line-by-line before adoption — see commit `4cbbec2` for the safety hardening rationale. Treat the rdphillips7 contribution as a community patch under audit, not as a vetted upstream release.

DAVE protects audio in transit between Discord clients and Discord's servers — it does NOT protect against the bot itself. When the bot joins a voice channel, the MLS group includes the bot. Discord's UI flags the bot as a "non-E2EE participant" visibly to other users. Consent for bot presence in shared voice channels is a UX/policy concern, not handled by this library.
