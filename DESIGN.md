# Aniketos — Design Document

**A macOS disk-accounting, cleanup, and malware-hygiene tool built to be verifiable rather than persuasive.**

| | |
|---|---|
| **Status** | Draft v1 — pre-implementation |
| **Date** | 2026-07-27 |
| **Target platform** | macOS 26.0+ (Tahoe), Apple Silicon only |
| **Reference machine** | MacBook Pro `Mac15,10`, M3 Max (10P+4E), 36 GB RAM, 926 GB APFS, macOS 26.5.1 (25F80), SIP enabled |
| **Name** | **Aniketos** — Gk. Ἀνίκητος, *"unconquered / invincible"*; an epithet of Heracles and later Mithras, not a distinct deity. ⚠️ Trademark clearance still required before branding spend (see §13). |
| **Repo** | `github.com/boncui/Aniketos` |

---

## 0. Thesis

> **macOS cannot tell you where your disk went, and every product that claims to has an incentive to lie about it. Aniketos is the tool that accounts for 100% of your disk and lets you prove it.**

This is not "another Mac cleaner." The cleaner category is commercially crowded and reputationally toxic (§3, §4). The defensible product is one layer beneath it: **honest byte accounting**. Cleanup is the *payload*; accounting is the *reason to exist*; open, auditable rules are the *moat*.

Three claims from this document, each measured on the reference machine, justify that framing:

1. **macOS's own numbers disagree with each other by 34.2 GB on this machine, right now** (§1.2).
2. **~45 GB of this 352 GB is regenerable developer cache that macOS Storage settings structurally cannot see** (§1.1).
3. **Every naive approach to measuring it is wrong**: `st_size` overstates by 13.1× here, `st_blocks` silently fails on APFS clones, and `du -sh ~/Library` does not finish in 5 minutes (§1.3).

No shipping product reports #1. Apple will not build it, because it would require Apple to admit its own numbers are inconsistent.

---

## 1. Ground truth — measurements from the reference machine

Everything in this section was measured directly, not cited. These numbers are the **v0.1 golden acceptance fixture**: regressions in accounting coverage must be detectable against them.

### 1.1 Where the disk actually went

Volume: 926 GiB total, **352 GiB used**, 533 GiB available, 2,126,943 inodes.

| Path | Size | Regenerable? |
|---|---:|---|
| `~/Library/Developer` (total) | **29 G** | yes |
| ↳ `CoreSimulator` | 21 G | yes — re-download 5–8 GB per runtime |
| ↳ `Xcode/iOS DeviceSupport` | 5.5 G | yes — re-downloads on device attach |
| ↳ `Xcode/DerivedData` | 3.2 G | yes — next build is a clean build |
| `~/Library/Application Support` | **24 G** | mixed — see below |
| ↳ `Google` | 8.5 G | mostly cache |
| ↳ `Claude` | 6.4 G | ⚠️ contains `vm_bundles`, agent session state |
| ↳ `com.apple.wallpaper` | 2.6 G | yes |
| ↳ Notion / Figma / Code / Slack / Spotify | 5.7 G | ⚠️ cache *adjacent to* unsaved buffers |
| `~/Library/Caches` | **17 G** | yes |
| `~/.cache` | 7.7 G | yes |
| `~/Documents` | 11 G | **no** |
| `~/Library/Group Containers/…dev.orbstack` | 1.4 G | **no — live VM data** |
| pnpm / npm / bun / Homebrew caches | ~7.5 G | yes |
| `/opt/homebrew` | 2.2 G | rebuildable, slow |

**Unambiguously safe developer cache: ≈ 45 GB (12.8% of used space).** This is the payload.

Also present and relevant: 25 apps in `/Applications`; 8 user LaunchAgents (Adobe, Google Keystone ×2, watchman, Riot, postgres, dbtunnel); 5 system LaunchDaemons (Adobe, Tailscale, **OrbStack privileged helper**, macsfancontrol SMC-write). Three APFS OS-update snapshots exist.

### 1.2 The purgeable-space gap — the "System Data" mystery, measured

Four APIs, one volume, same instant:

| Source | Bytes free | Δ vs `statfs` |
|---|---:|---:|
| `statfs` / `NSURLVolumeAvailableCapacityKey` | 571,449,319,424 | — |
| `…AvailableCapacityForOpportunisticUsageKey` | 583,741,030,739 | **+12.3 GB** |
| `…AvailableCapacityForImportantUsageKey` | 605,619,940,032 | **+34.2 GB** |

**A 34.2 GB spread between two Apple APIs on an idle machine.** Which one a product displays determines whether users call it a liar. Finder and About This Mac use the optimistic ones; `df` uses the pessimistic one. This gap *is* what users call "System Data."

Corroborating field evidence: a macOS 26.2 user reported 660 GB of "System Data" on a 994 GB drive; Apple Tier-3 support spent three hours and escalated without a fix ([MacRumors 2477125](https://forums.macrumors.com/threads/system-data-huge-700gb-help.2477125/)). A documented 1 TB case showed Finder claiming 742 GB "available" against <100 GB actually free.

### 1.3 Three measurement traps, each empirically confirmed here

**Trap 1 — `st_size` is meaningless on a modern Mac.**
Summing `st_size` over `~/Library` (541,364 files) reports **1080.3 GB**. Real physical usage: **82.4 GB**. A **13.1× overstatement**, caused almost entirely by one file:

```
~/Library/Group Containers/HUAQ24HBR6.dev.orbstack/data/data.img.raw
  st_size   = 994,663,481,856  (994.7 GB — larger than the disk)
  st_blocks =       2,933,536  (  1.50 GB actual)
```

Any size-ranked list built on `st_size` puts *the user's entire Docker/OrbStack VM* — every image, every named volume, every local database — at the top of the "biggest files" screen. One click destroys all of it, and there is no Time Machine coverage for a file that changes every second.

**Trap 2 — `st_blocks` does *not* fix it, and this is where the standard advice is wrong.**
Controlled experiment on this volume: create a 200 MB file, `cp -c` it (APFS `clonefile(2)`), and hardlink it.

```
df delta after deleting BOTH the clone and the hardlink:  0 KB
stat -f %b for all three files:                            409600 (200 MB each)
du -sh:                                                    200M each, 400M total
```

**`st_blocks` reports full size for a clone that occupies zero additional disk.** Since Finder's Duplicate command and Option-drag both create clones, a large share of a real user's byte-identical "duplicates" already share storage. A duplicate finder that hashes them, confirms they match, promises 12 GB, and frees 0 GB has just produced the exact FTC fact pattern from *Restoro/Reimage* ($26M, March 2024).

→ **Design consequence (§7.1): never predict reclaim. Measure it.**

**Trap 3 — naive traversal is architecturally disqualified.**

| Method | Throughput |
|---|---:|
| `du -sh ~/Library` | **timed out at 5 min** |
| `find $HOME -type f` (1,065,669 files) | 580 s = **1,838 files/s** |
| `os.walk` over `~/Library`, cold | 510 s = 1.0K files/s |
| `getattrlistbulk`, 1 thread, warm | 37,464 files/s |
| **`getattrlistbulk`, 8 threads, warm** | **124,055 files/s** ← optimum |
| `getattrlistbulk`, 64 threads | 73,898 files/s (**1.7× *slower***) |

Two corrections to widely-cited advice:
- The famous `dumac` benchmark (409,500 files in 521 ms, "6.4× faster than `du`") used a **synthetic tree with ~100 files/directory**. Real `~/Library` here is **5.86 files/directory** — batching amortizes almost nothing. Measured advantage over `du` single-threaded: **1.16×, not 6.4×.**
- `dumac`'s 64-permit semaphore is **actively harmful** on this hardware. Cap at **8**, or P-core count.

Realistic budget: **~17 s warm** for 2.1M inodes, metadata only. Cold is minutes. Any pass that opens file contents is far more expensive.

### 1.4 Hashing and I/O (measured)

| Operation | Throughput |
|---|---:|
| Sequential read, cache-bypassed | 10.07 GB/s |
| XXH3-64 | 42.9 GB/s |
| **SHA-256** (ARMv8 crypto ext) | **3.10 GB/s** |
| BLAKE3, 1 thread | **2.33 GB/s** ← *25% slower than SHA-256* |
| BLAKE3, all threads | 24.98 GB/s |

**BLAKE3 loses to SHA-256 single-threaded on Apple Silicon** because ARMv8 has dedicated SHA-256 instructions while NEON is only 128-bit. The SSD (10 GB/s) is 3.3× faster than single-threaded SHA-256 — **parallelise hashing across files, don't optimise the primitive.**

Also measured: the classic head/tail sample-hash prefilter is a **net loss** here. It must eliminate >94% of candidate bytes to pay for itself; on real data it eliminated 12.7%. Apply it only to files >64 MB.

---

## 2. What people actually need

Ranked by weight of evidence across MacRumors, Apple Support Communities, Hacker News, App Store reviews, and Eclectic Light Co. *(Reddit was inaccessible to the research crawler — sentiment there is inferred, not read.)*

| # | Need | Evidence |
|---|---|---|
| 1 | **Reconcile the lying numbers.** Show Storage vs Disk Utility vs `df` vs `du` and explain the delta | Most-repeated complaint; Apple Tier-3 couldn't solve a 660 GB case |
| 2 | **APFS snapshot manager** — list, size, age, safely thin | Where the 100 GB+ wins live; corrupted `.asr` snapshots trapped 400 GB–1 TB in documented cases |
| 3 | Large/old file finder, incl. areas normal scans can't reach | DaisyDisk's whole business at $9.99 |
| 4 | **Dry-run before any delete**, with total GB | 4 independent Show HN tools in 6 weeks all converged on preview-first |
| 5 | **Undo / quarantine, not `rm -rf`** | The #1 fear |
| 6 | **Dev cache sweep** (Xcode, Docker, node_modules, brew, cargo, gradle) | "57.6 GB not visible in system storage settings"; "CleanMyMac failed to identify developer tool caches" |
| 7 | Docker/OrbStack image compaction | Docker.raw defaults to 64 GB, grows, never shrinks |
| 8 | Complete app uninstall with leftovers | The one thing everyone concedes free tools already do well |
| 9 | Duplicate finder | Gemini sells this alone at $19.95/yr |
| 10 | Per-category attribution of "System Data" | 27 GB `TabSnapshots`, 104 GB Spotlight metadata found in the wild |
| 11 | Cloud-sync local cache control | One case: 680 GB → 72 GB |
| 12 | **Targeted malware persistence check** — not a full AV engine | See §6 |

### Explicitly rejected as snake oil — we will not build these

RAM cleaning · "speed boosters" / one-click optimize / health scores out of 100 · routine cache flushing as headline value · registry cleaning (a Windows-port tell) · **localization/language stripping** (breaks code signatures; app won't launch on Apple Silicon) · **universal-binary thinning** (same reason) · disk defragmentation (meaningless on APFS SSD) · background daemons and scheduled auto-clean · bundled VPN / identity monitoring · **"Repair Disk Permissions"** (a dead concept since El Capitan — the system volume is cryptographically sealed).

Each is independently named as snake oil by the exact audience that drives word-of-mouth. *"Runs only when launched, no background agents"* is a **marketable feature**, not a limitation.

### The two personas want opposite products

| | **Non-technical** | **Developer** |
|---|---|---|
| Machine | 256 GB Air, ~5 GB free | M-series, 512 GB–1 TB |
| Symptom | "Can't install macOS update" | "Disk pressure mid-build" |
| Vocabulary | "System Data" | Exact paths |
| Wants | Safe one-button path, plain language | Flat checkbox list of paths + sizes |
| Pays | $40/yr without blinking | "$10 for a lifetime license"; jeers at $40/yr |

They should not share one adaptive UI. Ship **two explicit modes**.

---

## 3. Competitive landscape (verified 2026-07-27)

### Cleaners

| Product | Price | Notes |
|---|---|---|
| **CleanMyMac** (MacPaw) v5.5.7 | $39.99–65.99/yr; $89.99 MAS one-time / $119.95 direct | Category leader. **MAS build is missing 21 features** by MacPaw's own KB. ~80% of MacPaw revenue |
| **DaisyDisk** 4.34.2 | **$9.99 one-time**, 5 Macs | Untimed free scan, deletion paywalled. Explicit "no data collection" |
| **Hyperspace** (John Siracusa) | $19.99/yr or $49.99 lifetime | ⭐ Most technically interesting: reclaims space via **APFS clones without deleting anything**. MAS-exclusive — chose a mechanism powerful *inside* the sandbox |
| **DeepClean** | $39 one-time | Differentiates purely on **17 developer-toolchain scanners** |
| **Sparkle** (Every) | $119/yr / $179 lifetime | AI-branded, GitHub-distributed |
| Gemini 2 (MacPaw) | $19.95/yr | Duplicates only; being cannibalised into CleanMyMac 5 |
| Sensei · Hazel · Cocktail · AppZapper | $19–$59 | Perpetual-license survivors |
| OnyX · Maintenance · AppCleaner · OmniDiskSweeper · GrandPerspective | **free** | New build per macOS release. Beloved |
| **Pearcleaner** (14.1k ★) | free, source-available | ⚠️ **Apache 2.0 + Commons Clause — forbids selling. Unusable and un-forkable.** Development halted end-2025 (maintainer lost Mac access) |
| MacKeeper · MacBooster · CCleaner · Avast Cleanup | $30–$90/yr | Trust-damaged. MacBooster's store runs *"Hurry up, 200 left today!"* on a downloadable license |

**Market signal:** MacPaw posted a **$10M+ loss in 2024**, ~7% CAGR 2021–24, cut ~100 staff, killed Setapp Mobile (Feb 2026) and CleanMyMac Business (Jul 2026). The incumbent is not thriving. Published TAM figures range **$425M to $46B** across research firms — all unusable.

### Anti-malware — detection is a commodity

**AV-Comparatives, May 2026** (macOS Tahoe, 1,500 samples): CrowdStrike 100%, Kaspersky 100%, **Avast Free 99.9%, AVG Free 99.9%**, ESET 99.9%, Norton 99.9%, Bitdefender 99.0%, Trellix 99.0%, Intego 96.7% (last, two years running). **Zero false positives across all nine.** AV-TEST March 2026: 8 of 10 scored a perfect 18/18.

**You cannot win on detection rate.** Free products beat paid ones. Note also that **Malwarebytes and MacPaw Moonlock are absent from both labs' 2025 and 2026 macOS lineups** — the two most-marketed consumer products have no current independent validation.

### Apple's built-in stack is stronger than its reputation (read from this machine)

- **XProtect 5352** with **449 live YARA rules** and a 129-entry extension blacklist — readable, free, auto-updating, sitting on every Mac
- **XProtect Remediator 157** with **25 named family scanners** (Adload, Bundlore, Genieo, Pirrit, Trovi, ColdSnap, KeySteal, …)
- **Bastion behavioural engine, 24 signatures** — including `macOS.InfoStealer.Generic` and `macOS.DataExfil.Keychain`. This contradicts the common claim that Apple has no behavioural detection.
- MRT.app is **absent** — retired June 2022. Any tool checking MRT version is checking a ghost.

**The gap is not detection quality — it is visibility.** XProtect never scans on demand, never scans a whole disk, and never reports what it found.

**And a real, verifiable hole**, read from the launchd plists on this machine:

```
com.apple.XProtect.{agent,daemon}.scan.plist
  *.fast.scan   Interval=21600  (6 h)  AllowBattery=true
  *.scan        Interval=86400  (24 h) AllowBattery=FALSE
  *.slow.scan   Interval=604800 (7 d)  AllowBattery=FALSE
```

**A laptop that lives on battery may go weeks without a deep XProtect scan, and macOS never tells you.** No shipping product reports this.

### The actual threat model

Adware ~65% + PUAs ~25% = **90% of real detections**. Stealers are only ~1.7% but carry nearly all the harm, and they are a **monoculture**: Odyssey 62.7% + AMOS 29.8% = >90%, one code lineage.

Delivery is **ClickFix** — 47% of all observed initial access (Microsoft Digital Defense Report 2025). A fake CAPTCHA copies a shell command to the clipboard; the user pastes it into Terminal; it `curl`s a DMG and mounts it. **Nothing is downloaded by a browser, so no `com.apple.quarantine` xattr is set and Gatekeeper never runs.** macOS 26.4 added Terminal paste-blocking, but it has a "Paste Anyway" button and *may be suppressed entirely for users with Xcode installed* — i.e. exactly our ICP.

Also: **>50% of malicious Mach-Os are code-signed**, ~20% carry valid-or-recently-revoked Apple certs. Two 2025 families (ChillyHell, MacSync) were signed *and notarized*.

---

## 4. Why this category is toxic — and the design rules that follow

This is not reputational hand-wringing; it is a set of hard engineering constraints.

| Precedent | Outcome |
|---|---|
| **Yencha v. ZeoBIT** (MacKeeper) | $2M settlement — scans "invariably" reported fabricated problems |
| **FTC v. Office Depot / Support.com** (2019) | **$35M** — the "PC Health Check" scan was **four yes/no questions** |
| **FTC v. Restoro / Reimage** (Mar 2024) | **$26M** — free scan that always found problems |
| **FTC v. Avast** (2024) | **$16.5M** + 20 years of biennial audits — sold AV-harvested browsing data |
| **Adware Doctor** (#4 Top Paid Mac app) | Pulled — zipped Safari/Chrome/Firefox history to `adscan.yelabapp.com`, password `webtool` |
| **7 Trend Micro apps** (Sept 2018) | Pulled — uploaded 24 h browser-history snapshots |

**Microsoft's classification criteria** (updated 2026-01-29) name our exact category as unwanted software: *"exaggerated claims about your device's health"*, *"claims in an alarming manner… and require payment… for fixing the purported problems."* Their **"Poor industry reputation"** PUA category means **one vendor's detection becomes independent grounds for every other vendor's**. MacKeeper is *still* detected as `PUP.MacKeeper` a decade after a $2M settlement, an ownership change, AppEsteem certification, and special Apple notarization review.

### Non-negotiable rules derived from the above

1. **Never read browser profile data.** Not to compute a size. Not ever. Every cleaner pulled from the App Store died on exactly this. Enforced by a compiled-in deny-list at a single audited read chokepoint, with a unit test that fails the build.
2. **AppEsteem ACR-004:** never display a finding we will not fix for free. Monetize scope/convenience, never the unlock.
3. **No scareware affordances:** no threat counts, red badges, health scores, countdown timers, fake scarcity, pre-checked destructive items, or "your Mac is at risk" copy. Banned in code review with a named approver.
4. **Zero network in the scan path**, provably — ship the engine with **no networking entitlement**, so LuLu/Little Snitch users see literally nothing. Publish the entitlements diff.
5. **Reproducible scans:** `--json` with a content hash, so a skeptic can run twice and diff.

---

## 5. Platform reality — what is actually buildable

### Hard blockers (verified)

**Mac App Store is foreclosed.** Not hard — *impossible*. Four guideline clauses independently kill it: 2.4.5(i) sandbox required, 2.4.5(ii) no shared-location installs, **2.4.5(v) no root escalation**, 2.5.2 no reading/writing outside the container. There is **no Full Disk Access entitlement for sandboxed apps**, and MAS review forbids even *telling* users to grant it. MacPaw's own published 21-missing-features matrix is the proof.
→ **Developer ID + hardened runtime + notarized + stapled, distributed as a DMG.** No MAS build flag. Ever. *(Avoid `.pkg` — a macOS 26.3 `spctl --type install` regression rejects notarized, stapled packages.)*

**Full Disk Access is not a master key.** FDA grants TCC *data* access and **zero privilege**. Confirmed from a genuinely unsandboxed shell (`sandbox_check()==0`):

| Needs FDA | Free without FDA |
|---|---|
| `~/Library/Safari`, `Mail`, `Messages`, `Cookies` | `~/Library/Caches`, `Preferences`, `Logs` |
| `…/com.apple.sharedfilelist` | `~/Library/LaunchAgents`, `HTTPStorages` |
| Apple app containers, both `TCC.db` files | **`~/Library/Developer`** ← the payload |
| | `~/Library/Group Containers` |

Cleaning `/Library`, root-owned plists, or `/private/var` **still needs the root helper even with FDA**. Model **three orthogonal axes** — TCC data access, privilege, App Management — not one ladder.

**Real-time protection needs five simultaneous gates.** `es_new_client` returns three distinct failures: `ERR_NOT_ENTITLED` (no entitlement), `ERR_NOT_PERMITTED` (no FDA), `ERR_NOT_PRIVILEGED` (not root). Plus macOS 26 refuses to activate a system extension whose app is not in `/Applications`, plus the granted entitlement arrives as a provisioning-profile template Xcode's automatic signing cannot consume. Community-reported wait: **4–12 months, with a real chance of no response**.
→ **File the System Extensions request on day one as a business dependency. Ship v1 without it.**

**App Management is inconsistent, and bundle editing is dead anyway.** Measured here: `Notion.app` (flags `0x10000`, runtime only) → MODIFY OK; `Slack.app` (`0x12000`, library-validation) and `Chrome.app` (`0x12a00`) → MODIFY DENIED. Same user, all notarized. And on Apple Silicon every executable needs a valid signature, so lipo-ing or stripping `.lproj` **invalidates the code directory and the app refuses to launch**. → Cut universal-binary thinning and language stripping entirely; publish *why* as a trust signal.

### Two APIs named in the source research **do not exist** — both on the root-helper security path

| ❌ Hallucinated | ✅ Real |
|---|---|
| `xpc_connection_set_peer_code_sig()` | **`xpc_connection_set_peer_code_signing_requirement()`** — `xpc/connection.h:803`, macOS 12+. (Swift: `NSXPCConnection.setCodeSigningRequirement(_:)`, macOS 13+.) Programming error to call twice per connection. |
| `SecCodeCreateWithAuditToken` | **`SecCodeCopyGuestWithAttributes(NULL, attrs, …)`** with `kSecGuestAttributeAudit` holding the audit token as `CFData`, then `SecCodeCheckValidity` |

Both symbols were probed and confirmed missing from libSystem / Security.framework. An engineer who cannot find the named function falls back to the PID-based check — which is **racy and is the documented macOS privilege-escalation primitive**. → **Run a `dlsym` audit over every API named in this document before writing code.**

### Other corrections
- Endpoint Security has **45 AUTH / 105 NOTIFY** event types (not "~50 NOTIFY"). Framework is at `MacOSX.sdk/usr/include/EndpointSecurity/`, **not** `/System/Library/Frameworks/`.
- `~/Library/Saved Application State` returns **ENOENT** on macOS 26.5 — a stale entry inherited from Pearcleaner's path table.
- `/Applications/Safari.app` is `restricted,hidden`, symlinked into a SIP-protected cryptex. Only `~/Library/Safari` is addressable.
- `ES_EVENT_TYPE_RESERVED_5/6` (undocumented network events, macOS 26.4+) **exist but must not be shipped against** — "RESERVED" is how Apple names placeholders it intends to change, and a wrong struct offset in a root AUTH handler takes the user's network down.
- **SMAppService has an unfixed macOS 26 bug** (`fullPath is nil`) affecting ~1 in 500 Macs: registration reports success, every XPC connection fails, no workaround. Must be detected and degraded gracefully.
- `FileManager.trashItem` → **Finder's "Put Back" only works for the first item trashed per process.** Bulk restore must be our own journal.

---

## 6. Product decisions

### 6.1 The wedge (ranked, decided)

1. **Honest byte accounting** ← *the hook.* Reconcile all four numbers, name every delta, disclose what we cannot see. Nobody ships it; Apple structurally won't; a shell script cannot credibly do it.
2. **Radical transparency** ← *the moat.* Open versioned rule repo, `--json` + content hash, zero network in the scan path. Not a feature anyone buys — but the incumbent cannot copy it without abandoning scan-gates-fix.
3. **Developer-toolchain reclamation** ← *the payload.* Real (45 GB measured) but low-moat: reproducible in ~200 lines of shell. **Demote from wedge to payload.**
4. Free + OSS core ← a distribution strategy, not differentiation.
5. One-time pricing ← positioning, copyable in an afternoon.

**Rejected:** local-only AI as a pricing lever (MacPaw's own founder rates their AI pivot 4/10); malware detection as a headline (§3 — commodity, and it imports the trust problem).

### 6.2 ICP

**Mac developers running Xcode + Docker + Node on ≤1 TB machines.**

They are the only segment whose pain macOS structurally ignores; they can self-serve FDA and admin elevation without a support burden that would sink a solo maintainer; they convert on a *number* rather than on fear; and critically **they will not call you a scam**, which the Apple Support Communities crowd reliably will.

**Accept the known cost:** this is the worst-paying segment available. HN anchors the fair price at *"$10 for a lifetime license"* and jeers at *"$40/yr for cache deletion."* That is precisely why monetization must be **pay for the app, not for the unlock**.

Explicitly rejected as v1 ICP: normal users (support burden, FDA fear, "macOS already does this", and they are the ones destroyed by false positives); designers (photo dedup is Gemini's turf and needs perceptual-hash quality unreachable in v1); SMB/IT (MacPaw just killed CleanMyMac Business — that's a market signal).

### 6.3 The 60-second experience

| t | Tier | What happens |
|---|---|---|
| 0–2 s | **none** | Four free-space numbers side by side from `statfs` + `diskutil` + both `NSURLVolume…` keys, **each pairwise delta named**. No scan required. On this machine: *"34.2 GB is purgeable — macOS counts it as free, `df` doesn't."* |
| 2–25 s | **none** | Bounded parallel `getattrlistbulk` walk over ~50 known dev/cache roots only. Returns ~45 GB here in seconds. |
| 25 s | — | **Headline:** *"45 GB in developer caches — every byte regenerable."* Each row shows **physical** size, last-modified, and a plain-language **regeneration cost**: *"DerivedData 3.2 GB — next build is a clean build, ~4 min"*; *"CoreSimulator 21 GB — 5 runtimes, re-download 5–8 GB each."* |
| 35 s | — | **The kicker no competitor ships:** *"N GB of your disk is not visible to this scan"* — broken into FDA not granted / other users / 3 APFS snapshots / swap+VM, each quantified. |

**Zero rows pre-checked. Primary button reads "Review", never "Clean".**

The wow is not "look at your junk." It is **"this is the first tool that didn't lie to me about my own disk."**

⚠️ **Never do a full-disk walk in the first-run path.** Cold `~/Library` alone is 510 s.

### 6.4 Pricing

**$29 one-time, 3 Macs, lifetime for the current major version.**
**Free forever:** the complete open-source CLI engine and rule set — all scanning, all accounting, all reporting, all deletion.

You pay for the app, not the unlock. This is the Dash/Sublime model: immune to AppEsteem ACR-004, the FTC "pay to see the fix" pattern, and Microsoft's criteria; it turns the free tier into a distribution channel rather than a crippled trial; and it is the only structure developers respect. **No subscription. No auto-renewal of anything** — 50+ logged complaints against the category leader are specifically about surprise renewals, and that is the actual churn mechanism here.

Merchant of record: **Lemon Squeezy** (~7.5% all-in; ships license-key issuance/validation natively, removing a licensing server from scope).

### 6.5 Explicit non-goals for v1

Real-time protection · Endpoint Security entitlement dependency · Mac App Store SKU · photo/perceptual dedup · **any AI/LLM feature** · privileged root helper *(deferred to v1.1 — v1 ships read-only snapshot inventory via `tmutil listlocalsnapshots`, which needs no root)* · Intel support · background daemons or scheduling · cloud accounts or sync · any telemetry · Setapp channel · localization · Windows/Linux.

---

## 7. The hard part — safety architecture

Three forces in tension: (a) real cleaning needs FDA + a privileged helper; (b) those same powers make us indistinguishable from a PUP unless every deletion is provenanced and reversible; (c) honest security value without the ES entitlement. Resolving this is the doc's central job.

### 7.1 Measure, don't predict — the single highest-leverage decision

**Every space figure shown to a user must be a before/after `statfs` delta, never a sum of file sizes.**

```
pre  = statfs(volume).free
execute(plan)
post = statfs(volume).free
report(post - pre)          ← the authoritative number
```

Pre-action figures are shown only as a **labelled upper bound**: *"up to X GB — actual may be lower if files share storage on APFS."*

This one change simultaneously fixes the sparse-file lie, the clone lie, the hardlink lie, and the snapshot-retention lie — and removes the entire class of FTC-actionable exaggerated-savings claims. **Regression test:** clone a file, run the duplicate flow, assert reported reclaim is within 5% of measured `statfs` delta.

Supporting rules: rank by `st_blocks*512`, never `st_size`; render `st_size / (st_blocks*512) > 4` as *"sparse — N GB logical, M GB on disk"* and make it **non-selectable in bulk**; dedupe hardlinks by `(st_dev, st_ino)` and label `st_nlink > 1` as *"deleting frees 0 bytes"*; check `tmutil listlocalsnapshots` before promising anything.

### 7.2 One audited deletion chokepoint

**No code path may call `unlink`, `removeItem`, or `trashItem` directly.** Enforced by a CI lint rule. Every destructive operation passes through one function enforcing, in order:

1. **Deny-list check** (§7.3)
2. **Volume-boundary check** — `statfs f_fsid` must match the scan root
3. **Dataless/cloud gate** — `st_flags & SF_DATALESS`
4. **Running-process check** — refuse paths owned by a running bundle ID
5. **Symlink-safe resolution** — `openat(O_NOFOLLOW|O_DIRECTORY)` descriptor-walking from a pinned root fd
6. **Re-verify** `(dev, ino, size, mtime)` by `fstat` on the open descriptor; abort the item on mismatch
7. **Journal write with `F_FULLFSYNC`** — *before* the mutation, never after
8. The mutation

This is the difference between a bug in one rule and a bug that destroys a home directory.

### 7.3 The Never-Touch Registry

One consolidated, test-backed artifact. **Glob and substring matching on directory names is banned entirely** — `~/Library/Application Support/Code/Cache` is a rule; `*Cache*` is never a rule.

Why that matters, verified on this machine: `~/Library/Application Support/Code/` contains `Backups` (VS Code **hot-exit buffers — unsaved editor content, never written to disk**), `Local Storage`, `Cookies`, and `User` **as direct siblings of** `Cache`, `CachedData`, `GPUCache`. One over-broad rule silently destroys unsaved work with no recovery path anywhere — not Time Machine, not Trash, because it was never a document.

**Permanently excluded:**
- Cloud/dataless placeholders — `~/Library/CloudStorage/*`, `~/Library/Mobile Documents`, anything with `SF_DATALESS`
- **Opaque app packages** — `*.photoslibrary`, `*.musiclibrary`, `*.fcpbundle`, `*.logicx`, `*.sparsebundle`, `~/Library/Mail`, `~/Library/Messages`
- Browser profile directories, and any `*.db`/`*.sqlite`/`*-wal`/`*-journal` within one
- `~/Library/Keychains`; `.git` at any depth; `Xcode/UserData` (CodeSnippets, KeyBindings — hand-authored, never in VCS) and `Xcode/Archives` (dSYMs for shipped builds, **not regenerable after the fact**)
- VM/container disk images — `*.img`, `*.img.raw`, `*.qcow2`, `Docker.raw`, `~/.orbstack`, `~/.colima`, `~/.lima`
- Time Machine destinations, bootable clones, network volumes (`nfs`/`smbfs`/`afpfs`/`webdav`), read-only and sealed volumes
- Other users' home directories; anything SIP-protected; code-signed bundle interiors
- **`Spotify/PersistentCache/Storage`** — verified to contain `Storage/` and `Users/`, i.e. **offline-downloaded music**. This is the single most-cited horror story in the category, and the structure here confirms it was a naming trap, not user error.

### 7.4 Cloud and dataless files — a launch blocker

On macOS 12.3+ every major cloud provider is a File Provider extension; online-only files are **dataless stubs** (`SF_DATALESS = 0x40000000`, confirmed in the live SDK at `sys/stat.h:359`; `~/Library/CloudStorage` exists on this machine).

`stat()` does not materialize a dataless file. **`open()`/`read()` does.** Two catastrophic outcomes from a content-hashing duplicate finder:

1. It silently **downloads the user's entire multi-terabyte cloud drive** onto a 926 GB SSD — *a cleaner that fills your disk*.
2. Deleting a placeholder is a **sync-visible delete that propagates to the cloud and every other device**. "Reclaiming 40 GB of duplicates" permanently destroys the originals everywhere, with no local Trash to restore from because the bytes were never local.

→ Mandatory pre-open gate in the chokepoint that the hasher **physically cannot bypass**, plus a fixture test asserting **zero materialization bytes** during a full scan of a synthetic dataless tree.

### 7.5 Undo that actually works

**Trash-first is necessary but not sufficient**, and it fails in exactly our headline use case: `~/.Trash` is on the same volume, so trashing frees **zero bytes**. The 256 GB Air user with 5 GB free literally cannot use a trash-first cleaner.

- **Own the restore journal.** Append-only SQLite, one row per item *before* any action: absolute path, `st_size`, `st_blocks`, `st_dev`, `st_ino`, `st_nlink`, SHA-256, mode/uid/gid, xattrs, rule ID, rule version, plan ID, timestamp, resulting trashed URL.
- **Tier by size.** Under 5 GB → trash, 30-day retention. Over → explicit *"this cannot be undone — X GB will be permanently removed"* naming the top 10 items, with a typed confirmation.
- Offer `tmutil localsnapshot` before any large batch.
- Per-plan state machine (`planned → committing → executing → complete`) so a restart detects and reports an interrupted batch. Reconcile journal against quarantine store on launch.
- **No "secure shred". No `unlink()` reachable from the default flow.**

### 7.6 Privileged helper (v1.1) — CVE-2025-54595 as the negative spec

**Pearcleaner — an open-source Mac cleaner in this exact category — shipped CVE-2025-54595 (CVSS 8.4)**: its root helper returned `true` unconditionally from `shouldAcceptNewConnection` and passed client strings to `bash -c` as root. Any unprivileged local process got instant root.

Requirements:
- **The helper never accepts a path from the client.** XPC surface is exactly one method: `execute(planID: UUID)`. The helper reads the plan from the shared SQLite DB itself and **independently re-validates every row** against a compiled-in prefix allowlist.
- Pin the peer with `xpc_connection_set_peer_code_signing_requirement()` (§5) against Team ID + bundle ID + hardened-runtime flags. **Return `false` by default.** Never PID.
- TOCTOU-safe: `openat(O_NOFOLLOW)` descriptor-walking; never re-resolve a string path.
- **No shell, no `NSTask`, no string interpolation anywhere.** Under ~500 lines. Reviewed as a standalone unit by someone who did not write it.
- Registered via `SMAppService.daemon(plistName:)`, plist at `Contents/Library/LaunchDaemons`, binary at `Contents/Library/HelperTools`. Budget UX for the fact that **SMAppService cannot prompt for a password** — the user must enable it manually in Login Items.

**Acceptance test:** an attacker with a local unprivileged shell must not be able to obtain root or delete an arbitrary file through our daemon.

### 7.7 Rule-pack operations

Rules ship **out-of-band** from the binary so a macOS point release can be fixed without a full release cycle — which makes the rule channel **remote instruction delivery to a process running as root with FDA**. CrowdStrike's 2024 channel-file incident is the reference for how fast this scales.

- Signed with a key **distinct** from both the Sparkle EdDSA key and the Developer ID cert, separate custody; public key pinned in the binary.
- **Structural limits at load time:** reject any rule whose path prefix is <4 components or doesn't start with a known root.
- Canary run against a golden macOS image asserting zero matches outside the expected set; staged rollout 1% → 10% → 100% with automatic halt on anomalous deletion volume.
- **Kill switch** that disables all rules newer than a pinned version. **Fail closed** — an unverifiable pack means no cleaning, never fallback to a stale one.
- **Regenerability burden of proof:** no path becomes deletable until an engineer has deleted it on a clean test machine, relaunched the owning app, and documented what came back. Stored as a required rule field. Rules lacking this evidence are a release-blocking metric.

---

## 8. Security scope — honest, entitlement-free

We deliver "scan for viruses" — but scoped to what is *true* rather than what sells. **On-demand scanning and hygiene is the honest ceiling for v1**, and it happens to be the unclaimed territory.

**v1 ships (no entitlement required):**

1. **On-demand full-disk YARA scan using Apple's own 449 XProtect rules** — free, auto-updating, already on disk. XProtect applies them at execution time but *never* to a user-initiated scan of an existing disk. Engine: **YARA-X** (BSD-3-Clause, has a native `macho` module legacy YARA never had), built with the **`pulley` interpreter feature** — the default Cranelift JIT needs `com.apple.security.cs.allow-jit`, and *root daemon + FDA + JIT + signature matching* is the exact behavioural fingerprint of the malware we're detecting.
2. **The XProtect battery-scan gap** (§3) — *"your last deep XProtect scan was N days ago (deferred: on battery power)."* True, verifiable, quantitative, and unreported by anyone.
3. **Persistence diffing** — snapshot LaunchAgents/Daemons, SMAppService login items, shell profiles, cron, configuration profiles; alert on the delta, annotated with `SecStaticCodeCheckValidity`, `spctl` assessment, notarization status, Team ID, and **certificate revocation state** (a revoked Team ID on a persistent item is a near-certain hit).
4. **ClickFix retroactive forensics** — scan `~/.zsh_history`, `~/.bash_history`, `~/.zsh_sessions` for `curl|sh` pipelines, `hdiutil attach` against remote URLs, base64-decoded `osascript`, `xattr -d com.apple.quarantine`. Timestamped, so a user can tell whether they already fell for it.
5. **Apple security stack health panel** — XProtect version vs current, Remediator last-run, SIP, Gatekeeper, FileVault, Lockdown Mode, `systemextensionsctl` count. Detects the exact tampering real macOS malware performs.
6. **Adware/PUA removal** for the families Apple dedicates scanners to (Adload, Bundlore, Genieo, Pirrit, Trovi), plus Safari/Chrome extension auditing cross-referenced against the **129 blacklisted bundle IDs already in `XProtect.meta.plist`**.

**Deferred to v2, gated on the ES entitlement filed on day one:** real-time protection, behavioural detection, ransomware I/O monitoring. Architect the scanner behind an engine interface so an ES-backed engine drops in without a rewrite.

**Never:** bundled VPN, identity monitoring, a competing signature database, or any claim to replace XProtect. State plainly in-product that Aniketos **complements** Gatekeeper/XProtect rather than replacing them — that honesty is itself differentiation in a category full of scareware.

### On ML detection

The current SOTA is real and worth knowing: **Montaruli, Oliveri, Dambra & Balzarotti, "The Role of Domain-Specific Features in Malware Detection: A macOS Case Study," ACM AsiaCCS 2026 (arXiv:2606.03218)** — 41,129-sample Mach-O dataset, 4,578 features, XGBoost at **98.50% TPR @ 1% FPR** (99.50% on fresh samples). Critically, removing macOS-specific features (code-signing certs, entitlements, persistence APIs) costs **15.92%**, and the top two features are *"certificate chain validated"* and *"certificate chain present"* — a cheap `SecStaticCode` query, not a disassembly pass.

**But we are not shipping a model in v1**, for reasons the same literature makes clear:
- **Concept drift is brutal.** LAMDA (arXiv:2505.18551): LightGBM F1 collapses 97.49% → 47.24% as the test window moves forward; false-negative rate rises 1.47% → **64.10%**. Any model needs a sustained retraining pipeline we will not have.
- **Evasion is cheap.** 84% evasion against GBDT with off-the-shelf MAB-Malware; poisoning 0.5% of training data raises evasion from 26% to 92.8%.
- **False positives on developer machines are structural.** Every locally-compiled binary is unsigned or ad-hoc-signed, so the single most important feature *systematically misclassifies our own ICP's build output as malicious*. Verified persistence on this machine — unsigned Homebrew postgres, `com.leona.dbtunnel`, an SMC-writing root daemon — fires every proposed heuristic.
- **LLMs are not detectors.** MalEval (ISSTA 2026): best behaviour-category F1 **32.93%** at ~31 min/sample. CyberSOCEval (arXiv:2509.20166): frontier models at **23–34%** on malware-analysis QA.

→ **Signatures for known families (precise, explainable, cheap to update); nothing else claims to detect.** Never auto-quarantine on a heuristic — require a named YARA family match for any automatic action; everything else goes to a read-only review list. Suppress unsigned-binary signals inside developer-owned trees (`/opt/homebrew`, `/usr/local`, `~/.cargo/bin`, `~/Library/Developer`).

---

## 9. Architecture

**Recommended stack: Swift-first, single language, with surgical AppKit escapes. No Rust core in v1.**

The measured advantage of a Rust core over correctly-written Swift calling `getattrlistbulk` is small (`dumac`'s Go→Rust port saved ~8%; the API choice was the rest — syscalls are ~91% of runtime). The costs are concrete: UniFFI is pre-1.0 and its async generated code does not conform to `Sendable` (issue #2448) — precisely the streaming path we need — plus cross-compilation per triple, `lipo`, xcframework assembly, and re-signing an embedded binary in the notarization pipeline.

```
┌─────────────────────────────────────────────────────────┐
│ SwiftUI shell — sidebar, toolbars, onboarding, settings │
│ AppKit NSTableView (NSViewRepresentable) — results grid │  ← SwiftUI Table
└──────────────────────┬──────────────────────────────────┘     regressed on
                       │ batched snapshots @ ~10 Hz            15.5, still worse
┌──────────────────────┴──────────────────────────────────┐    on macOS 26
│ ScanEngine (Swift)                                       │
│  • getattrlistbulk on a DEDICATED GCD queue, 8 permits   │  ← never the
│  • columnar struct-of-arrays index                       │    cooperative pool:
│  • FSEvents eventID watermark → incremental rescan       │    blocking syscalls
│  • SHA-256 fanned across P-cores (not BLAKE3)            │    starve it
└──────────────────────┬──────────────────────────────────┘
┌──────────────────────┴──────────────────────────────────┐
│ GRDB.swift / SQLite (WAL) — index, Plan, undo journal    │  ← also the IPC
│  keyed on (volumeUUID, fileID), NOT path                 │    substrate: how a
└──────────────────────┬──────────────────────────────────┘    path-free helper
┌──────────────────────┴──────────────────────────────────┐    protocol works
│ Deletion chokepoint (§7.2) — the ONLY mutation path      │
└──────────────────────┬──────────────────────────────────┘
┌──────────────────────┴──────────────────────────────────┐
│ [v1.1] PrivilegedHelper — SMAppService daemon, <500 LOC  │
│  execute(planID:) only. Reads plan from DB. No paths.    │
└──────────────────────────────────────────────────────────┘
```

**Key choices**
- **Traversal:** `getattrlistbulk(2)` requesting name, type, `ATTR_CMN_FILEID`, mtime/atime/btime, `ATTR_FILE_DATALENGTH` **and** `ATTR_FILE_DATAALLOCSIZE` in one call with `FSOPT_PACK_INVAL_ATTRS`. A separate `lstat` per entry costs **29.5×**. Concurrency **8**, sized from P-core count, dropped to 2–4 when `thermalState > .nominal` or on battery.
- **UI:** SwiftUI for chrome; **AppKit `NSTableView` for the grid**, behind one protocol so it can be swapped if Apple fixes SwiftUI `Table`. Ship a hidden debug toggle rendering both, to re-measure each macOS release.
- **Dedup:** whole-file only. Meyer & Bolosky (FAST'11, 857 machines, 162 TB) — 8K Rabin reclaims only 18–20% more than whole-file on live filesystems, and you cannot "delete half a file" anyway. Pipeline: size-bucket with a **64 KB floor** (keeps 87% of bytes, drops 94.9% of files) → **skip the head/tail sample hash** (§1.4) → parallel SHA-256. Aim it at **binaries and dev artifacts**, not photos — duplicate bytes cluster in smaller files; large media rarely has whole-file copies.
- **Storage:** GRDB `DatabasePool` (WAL) — within 1.5–3× raw SQLite, ~20× faster than SwiftData on inserts, one writer + many readers so the scanner writes while the UI reads.
- **Update:** Sparkle (MIT) with EdDSA, key in CI secrets, `SUPublicEDKey` in Info.plist.

**License gate in CI from the first commit.** `cargo-deny`-style allowlist (MIT, Apache-2.0, BSD-2/3, BSL-1.0, ISC, Unlicense, MPL-2.0, Zlib); **hard deny on GPL-*, AGPL, LGPL, Commons Clause, Elastic-2.0.** This catches the four real traps found in research:

| Trap | Why |
|---|---|
| **ClamAV / libclamav** | GPL-2.0, no commercial exception → cannot link |
| **Pearcleaner** | Apache 2.0 **+ Commons Clause** → forbids selling. *Reimplement the algorithm; copy no code.* Also **do not fork PureMac** (MIT, 5.4k★) — its 1:1 structural match to Commons-Clause code is unresolved provenance risk |
| **All 12 Objective-See tools** | GPL-3.0 → read for technique, never link |
| **`ssdeep` crate** | GPL-3.0+ → use `tlsh2` (Apache/BSD) |

Detection content: **SigmaHQ + Neo23x0/signature-base are DRL 1.1, which explicitly permits selling** (keep author attribution + URI). **Avoid Elastic-2.0 rules** — non-sublicensable, so our EULA cannot pass rights to end users. **Never call the VirusTotal Public API** — contractually banned in commercial products, permanent-ban enforcement.

---

## 10. Build order

Each increment independently valuable and verifiable; riskiest assumptions proven first.

**v0.1 — personal use (2–4 weeks).** Swift CLI, unsigned, no GUI.
`getattrlistbulk` walker · rule set as external JSON · **four-number reconciliation** · `--json` output · **dry-run only — prints a Plan, deletes nothing.**
✅ *Acceptance: explains >95% of the 352 GB used on the reference machine, and I actually run it weekly unprompted.*

**v0.2 — destructive, safely (2 weeks).**
Deletion chokepoint · trash-first + SQLite restore journal · physical-size accounting with clone/sparse flagging · dataless gate · app-leftover resolution · **open-source the rules repo publicly.**
✅ *Acceptance: reported bytes within 5% of measured `statfs` delta on a fixture including clones, hardlinks, and sparse files.*

**v1.0 — shippable (3–4 months).**
SwiftUI shell + AppKit grid · three-tier permission onboarding that never shows dead UI · "what I cannot see" disclosure · read-only snapshot inventory · **YARA-X on-demand scan using Apple's XProtect rules** · persistence diff · ClickFix forensics · security-stack health panel · Developer ID + notarytool + Sparkle · reproducible scan hash · Lemon Squeezy at $29.
✅ *Acceptance: clean-VM fixture produces zero findings or the build fails.*

**v1.1** — privileged helper (§7.6) + snapshot thinning.

**v2.0 — gated on explicit traction** (500 paid units or 5k GitHub stars in 6 months; below that, keep it OSS and stop investing). Full-volume indexed scan with FSEvents incremental rescan · ES-backed real-time engine *if the entitlement ever arrives*.

---

## 11. Success criteria

| # | Criterion | Verification |
|---|---|---|
| 1 | Accounts for >95% of 352 GB used on the reference machine | Golden fixture test vs §1.1 |
| 2 | Reported reclaim within 5% of measured `statfs` delta | Automated test on a fixture with clones + hardlinks + sparse files |
| 3 | Clean-VM fixture yields **zero** findings | CI gate; build fails otherwise |
| 4 | **Zero materialization bytes** during a full scan of a dataless tree | Byte-counter fixture test |
| 5 | No code path calls `unlink`/`removeItem`/`trashItem` outside the chokepoint | CI lint rule |
| 6 | No read path resolves inside a browser profile directory | Unit test; build fails otherwise |
| 7 | Byte counts agree with `df` and Disk Utility to within one displayed unit | Automated comparison test |
| 8 | Scan of 2.1M inodes ≤ 30 s warm | Benchmark harness, nightly, fails on >10% regression |
| 9 | Local unprivileged shell cannot obtain root or arbitrary delete via the helper | Security review + exploit attempt (CVE-2025-54595 as the test case) |
| 10 | Uninstall leaves **zero** residue — verified by our own leftover scanner | End-to-end test: install → register helper → uninstall → assert clean |
| 11 | Two identical scans produce byte-identical `--json` + matching content hash | Determinism test |
| 12 | Scan engine binary has **no networking entitlement** | `codesign -d --entitlements` assertion in CI |

---

## 12. What kills this, ranked

1. **No personal pain.** This machine has 533 GB free. The "build it for myself" premise is weak — the project stalls after the fun scanner work. **Mitigation:** reframe the personal goal from *reclaim space* to *account for the disk I cannot explain*, which is a genuinely open question here, and make v0.1's acceptance test "explains >95%", not "frees N GB."
2. **One destructive false positive** → a PUP detection that cascades industry-wide via Microsoft's "poor industry reputation" category and never clears (MacKeeper: still flagged a decade later).
3. **Scope creep into real-time AV** burns the year on a 4–12 month entitlement queue.
4. **FDA grant rate comes in low** and the headline feature doesn't work for most users. *This is the single load-bearing unvalidated number in the plan* — instrument it as metric #1, and define the pivot if it lands under ~40%.
5. **Apple adds a "Developer" section to Storage recommendations** in macOS 27 and deletes the payload for free. (They already ship an "Xcode build files" recommendation — they've conceded the category exists.) **Mitigation:** the moat is the accounting layer, which Apple structurally will not build.
6. **PureMac** (MIT, 5.4k★, weekly releases) fills the free-OSS vacuum before the paid GUI ships.
7. **macOS 27 "Golden Gate"** (September 2026, Apple-silicon-only) breaks path heuristics and the solo maintainer can't keep up — the documented Pearcleaner outcome. Budget the September audit as a **permanent recurring commitment**.
8. **Shipping `st_size` instead of measured delta once, in public** — and being correctly accused of the exact dishonesty the product was built to attack.

**Continuity commitment (publish on the download page):** the rule set is Apache-2.0/MIT from day one in a separate repo; the CLI engine is open source; **if the app goes unmaintained for 12 months the GUI source is released under the same license.** This costs nothing while things go well and converts the biggest objection developers have to paying a solo dev into a reason to trust one.

---

## 13. Open questions

**Blocking — must be resolved before v1.0 code:**
1. **FDA grant rate.** Entirely unvalidated, load-bearing for the whole funnel. Needs a landing-page or beta measurement.
2. **AppEsteem ACR-004 vs the free/paid boundary.** Proposed resolution (§6.4): everything found is free to fix; payment buys the *app*, not the unlock. Must be written down and signed off, or it gets decided by whoever writes the paywall code under deadline.
3. **"Zero network in the scan path" vs licensing activation.** Proposed resolution: activation and any metrics live in a **separate target with their own entitlements**, so the isolation claim stays literally true and verifiable via an entitlements diff.

**Non-blocking, proceeding on stated assumptions:**
4. Personal tool or product? → **Assuming personal-first, productizable later.** Sized as a 3–4 month craft project with optional revenue, not a startup. Model revenue at ~$15k year one; if that doesn't justify the build, ship it as pure OSS.
5. Product name → **provisional**; trademark clearance required before branding spend.
6. Whether reading/shipping Apple's XProtect YARA rules at runtime is within our rights → **needs 20 minutes of counsel**; a headline v1 feature depends on it.

**Deferred but must be in the doc before v1.0** *(each is a known gap, not an oversight)*: accessibility (VoiceOver on a 100k-row destructive table; EAA is enforceable in the EU since June 2025) · localization and **decimal-GB byte formatting to match Finder** (a GiB/GB mismatch makes *us* look like the liar) · Unicode normalization for path comparison · support forensics runbook · multi-user and MDM-managed Macs · standard (non-admin) accounts, which cannot authorize the helper at all · Migration Assistant leftovers (a real feature *and* a stale-index hazard) · energy/thermal policy · CLI + App Intents surface · EULA, privacy policy, EAR 5D002 encryption self-classification · performance validation **on an 8 GB / 256 GB Air at >90% full** — every number in §1 comes from a top-of-line workstation, and the persona we claim to rescue owns the opposite machine.

---

## Appendix — key sources

**Academic**
- Montaruli, Oliveri, Dambra & Balzarotti. *The Role of Domain-Specific Features in Malware Detection: A macOS Case Study.* ACM AsiaCCS 2026. arXiv:2606.03218
- Meyer & Bolosky. *A Study of Practical Deduplication.* USENIX FAST 2011
- Agrawal, Bolosky, Douceur & Lorch. *A Five-Year Study of File-System Metadata.* USENIX FAST 2007
- Xia et al. *FastCDC.* USENIX ATC 2016 · Udayashankar et al. *SeqCDC.* Middleware 2024
- Pendlebury et al. *TESSERACT.* USENIX Security 2019 · Haque et al. *LAMDA.* arXiv:2505.18551
- Joyce et al. *EMBER2024.* KDD 2025 (contains **zero** Mach-O samples)
- Zheng et al. *MalEval.* ISSTA 2026 · Deason et al. *CyberSOCEval.* arXiv:2509.20166
- Casino et al. *Ransomware detection through classification of high-entropy file segments.* J. Cybersecurity 11(1), 2025
- Song et al. *Learning Relaxed Belady.* USENIX NSDI 2020 · Santry et al. *Elephant.* SOSP 1999

**Platform & security**
- Apple App Review Guidelines 2.4.5, 2.5.2, 2.3.1(a), 5.1.1 · Apple Platform Security (SIP, SSV, XProtect)
- Objective-See — *The Mac Malware of 2025*; Pearcleaner CVE-2025-54595 (GHSA-gr2j-65fh-8pvc)
- Eclectic Light Co. (Howard Oakley) — APFS special files, purgeable space, XProtect internals
- AV-Comparatives *Mac Security Test 2026* · AV-TEST macOS March 2026
- Microsoft, *How Microsoft identifies malware and PUA* (2026-01-29) · Malwarebytes PUP criteria · AppEsteem ACR-004
- FTC: Restoro/Reimage ($26M, 2024) · Office Depot/Support.com ($35M, 2019) · Avast ($16.5M, 2024) · Yencha v. ZeoBIT ($2M, 2015)

**Measured on the reference machine** — all §1 figures, XProtect 5352 / Remediator 157 / 449 YARA rules / 24 Bastion signatures / battery-scan gap, APFS clone experiment, `getattrlistbulk` thread scaling, hash throughput, TCC-gated path enumeration, missing-symbol probes.

### Reproduce the §3 security claims yourself

Every Apple-stack figure in this document was independently re-verified on 2026-07-27. All of it is readable without root:

```bash
XP=/Library/Apple/System/Library/CoreServices

# XProtect 5352 · 449 YARA rules · 129 blacklisted extensions
/usr/libexec/PlistBuddy -c "Print :CFBundleShortVersionString" $XP/XProtect.bundle/Contents/Info.plist
grep -c '^rule ' $XP/XProtect.bundle/Contents/Resources/XProtect.yara
python3 -c "import plistlib;print(len(plistlib.load(open('$XP/XProtect.bundle/Contents/Resources/XProtect.meta.plist','rb'))['ExtensionBlacklist']))"

# Remediator 157 · 25 named family scanners
/usr/libexec/PlistBuddy -c "Print :CFBundleShortVersionString" $XP/XProtect.app/Contents/Info.plist
ls $XP/XProtect.app/Contents/MacOS/ | grep -c '^XProtectRemediator'

# 24 Bastion behavioural signatures — incl. macOS.InfoStealer.Generic, macOS.DataExfil.Keychain
python3 -c "import plistlib;[print(e['SignatureName']) for e in plistlib.load(open('$XP/XProtect.app/Contents/Resources/BastionMeta.plist','rb'))['Behaviors']]"

# THE BATTERY GAP — deep scans are AC-power-only
python3 -c "
import plistlib,glob,os
for p in glob.glob('$XP/XProtect.app/Contents/Resources/*scan.plist'):
    for k,v in plistlib.load(open(p,'rb'))['LaunchEvents']['com.apple.xpc.activity'].items():
        print(f\"{k:60} Interval={v['Interval']:>6}s AllowBattery={v['AllowBattery']}\")"

# MRT.app — retired June 2022, absent from macOS 12.3+
[ -e /System/Library/CoreServices/MRT.app ] && echo PRESENT || echo ABSENT
```

Confirmed output for the battery gap on macOS 26.5.1 (identical for both `.agent.` and `.daemon.`):

```
…fast.scan   Interval= 21600s (6 h)  AllowBattery=True
…scan        Interval= 86400s (24 h) AllowBattery=False   ← skipped on battery
…slow.scan   Interval=604800s (7 d)  AllowBattery=False   ← skipped on battery
```

---

*Research base: 266 cited findings across 9 dimensions, plus 69 issues (22 blocker-severity) from 4 adversarial reviews. Where the research and direct measurement disagreed, measurement won — and §1.3, §1.4, and §5 record three cases where widely-repeated advice is simply wrong on this hardware.*
