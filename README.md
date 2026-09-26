# reverse-grill

**Review a PR without reading the code.**

It's better to make the right decisions before you write the code, right? But what if the code is already written and you have to review it with no idea how much thought the owner put into it before actually handing it off to an AI? Here's a skill that reconstructs the design decisions behind an existing PR, branch, commit, or dirty working tree and provides a compact report covering all the concerns that should be considered.

## When to use

- Reviewing a PR you did not write, whether a colleague's or an agent's.
- Landing on a branch you have to understand before touching it.
- Writing the ADR that should have existed before the merge.
- Deciding what to fix before a release.
- Warming up before a hard bug hunt: it maps the decisions, it does not find the bug.

Because it's too early to hunt for bugs if the decisions are wrong and have to be reimplemented.

## Use

```
/reverse-grill                    current branch vs its base, or the dirty tree
/reverse-grill feat/login         a branch

# or even like this:
/reverse-grill check last 20 commits, I think it's a fu**ing AI slop
```

## What's the result?

Here's an example of a real run on a two-year-old mobile app brought back to life in one PR:

```
# Reverse grill: feat/web-1.0.17

**Target:** branch feat/web-1.0.17 vs main, 3 commits, 47 files (+1792/-1349)

App sat 2 years on frozen tooling. Web image no longer built, Android target too old for store. This PR does five separate things:

1. 🔴 Web build restored - image builds, web login still broken server-side
2. 🔴 Biometric lock - new feature, not mentioned in the PR
3. 🔴 Chat screen - built, not reachable
4. 🟡 Toolchain + platform upgrade - app builds again
5. 🟡 UI changes - unrecorded tweaks

Three things to rethink before this goes further: how the web build reaches the API (1), a fingerprint-only lock on scanners that mostly have no fingerprint reader (2), and where the chat page is expected to live (3).

---

## 1. Web build restored

**What:** The web image builds again from a pinned Flutter 3.41.9 base; web login stays broken because of server-side host and CORS setup, which the PR records as out of scope.

**Decisions:**

🔴 **Web API host contract**
On web the app calls `core-api.` and `plan-api.` versions of each environment's hosts; native builds call the hosts directly.
**Problem:** no environment has such hosts, and the next step is DNS and certificates per customer to satisfy a client-side naming rule; the infra convention is `api.<env>.<product>` and the core server has CORS disabled anyway.
- **Recommended:** same-origin API on web - the web ingress already proxies the core paths, add the planning path per environment and skip the environment lookup on web. (lib/screen/loginScreen.dart:62-71; infra repo, web ingress)
- **Current:** host rewrite kept from before, listed in the PR as a known blocker.
- **Keep current if:** every customer environment is going to get the two extra API hostnames with DNS and certificates anyway; otherwise same-origin through the ingress that already proxies, one path per environment, nothing built per customer.

🟡 **Signing key in build**
The Android signing keystore and both native trees are copied into the image build stage.
**Problem:** the keystore is tracked in git twice and ends up in build layers and the CI cache; the web build never reads it.
- **Recommended:** exclude the keystore, the certificate folder, android/ and ios/ from the build context and drop the delete step. (.dockerignore:1-10, Dockerfile:6-9)
- **Current:** only tooling folders excluded; native trees copied then deleted.
- **Keep current if:** the image registry and CI cache are private and stay that way; otherwise ten lines of exclusions, no behaviour change.

🟢 **Pinned build image** - exact Flutter 3.41.9 from a third-party registry; the old image stopped updating and the project already pins that floor. (Dockerfile:2)

**Conclusion:** rethink before DNS and certificates get built per customer for a host rule nobody else uses. Then: keystore out of the build.

**Questions:**
- Which web host is live: the one in the app's infra repo or the one in the platform's? Both ingress files exist.

---

## 2. Biometric lock

**What:** On phones and scanners a stored session now needs a fingerprint or face at every app start, and a device without one is signed out; before, a stored session opened straight into the app. Not mentioned in the PR.

**Decisions:**

🔴 **Fingerprint-only lock**
The lock accepts only a fingerprint or face; there is no device PIN fallback.
**Problem:** the rugged scanners rarely have enrolled biometrics, so on the shift hardware the lock means no persistent session at all; once staff are trained to re-login daily, that becomes the operating posture. The fallback branch in the code is dead: both paths ask for biometrics only.
- **Recommended:** allow the device PIN or pattern as fallback. (lib/src/authBio.dart:48,67)
- **Current:** biometric only; failure or no biometrics = signed out.
- **Keep current if:** every device that keeps a session has an enrolled fingerprint or face; otherwise PIN fallback, two lines, before staff get trained to re-login daily.

🟡 **Lock at app start only**
The check runs when the app starts cold, never when it comes back from the background.
**Problem:** a shared device handed over mid-shift stays open for hours, which is the case a session lock exists for.
- **Recommended:** re-lock on return from background after a grace period, using the lifecycle hook the app already has for data refresh. (lib/main.dart:74-78)
- **Current:** cold start only.
- **Keep current if:** devices are never handed over mid-shift; otherwise one lifecycle hook the app already has, plus a grace period to pick.

🟡 **Never-asked treated as failed**
A person who has never been asked for biometrics is signed out at app start.
**Problem:** every existing install is signed out once on first launch of this version, with no warning to operators.
- **Recommended:** never asked -> ask now; explicitly failed -> sign out. (lib/main.dart:82-85)
- **Current:** missing answer counts as failed.
- **Keep current if:** one forced re-login of every install at upgrade is acceptable and announced; otherwise one branch, and the upgrade stays silent.

**Conclusion:** rethink before this version reaches the scanner fleet; a fingerprint-only lock leaves scanners without a persistent session. Then: re-lock on resume, never-asked handling.

**Questions:**
- Which devices in the fleet have enrolled biometrics? Is the lock a customer requirement, and for which devices?
- Was the one-time sign-out of every install at upgrade intended?

....
```

Legend:

- 🔴 rethink - wrong direction; gets harder to undo the longer it ships.
- 🟡 worth changing - a better option exists; fix at your pace.
- 🟢 fine - nothing to do; shown only when users would notice.

## Install

Claude Code, from this repo as a plugin marketplace:

```
/plugin marketplace add asidko/reverse-grill-skill
/plugin install reverse-grill@reverse-grill-skill
```

Any agent that reads Agent Skills, via the skills CLI:

```
npx skills add asidko/reverse-grill-skill -g
```

Codex, as a slash prompt:

```
mkdir -p ~/.codex/prompts && curl -fsSL https://raw.githubusercontent.com/asidko/reverse-grill-skill/main/skills/reverse-grill/SKILL.md -o ~/.codex/prompts/reverse-grill.md
```

Codex has no sub-agents; the skill then plays each expert in turn itself and says so in the report.

## Rules it keeps

- Read-only.
- Decisions, not bugs.
- Never invents intent. Unrecorded reasons are labelled so, and become questions for the author.

## License

Public domain - [The Unlicense](https://unlicense.org).
