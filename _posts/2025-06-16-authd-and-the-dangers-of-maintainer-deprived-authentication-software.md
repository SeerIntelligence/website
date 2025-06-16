---
layout: post
title: "authd and the dangers of maintainer-deprived authentication software"
category: security
--- 
> This post was initially going to be called "authd, and WHY ARE WE LETTING CANONICAL SHIP AUTH MODULES? LOOK AT WHAT THEY DID TO GNOME", but that was too long and too yell-y.

in september 2024, canonical shipped `authd`, their new authentication daemon. it’s an nss/pam component intended to handle ephemeral user creation and ssh logins on ubuntu systems—especially managed environments like cloud or container contexts. basically temporary users spun up for one-off tasks, devops pipelines, that sort of thing. so far, so canonical: reinventing wheels they barely understand, quietly, and deploying them at scale.

the idea wasn’t terrible. ephemeral users are real, and handling their lifecycle cleanly matters, this is very much not about a single developer or company, it is more about the cycle of reinventing the wheel when the required resources are just not there.

### the bug itself

inside this daemon is a slight omission: the temporary user struct returned during ssh pre-auth **didn’t include a `pw_gid` field**. that is, the gid — the group id — was left unset.
a group id is the numeric identifier that defines which unix group a user belongs to, determining the user’s default group permissions and access controls. without an explicit gid, the system defaults it to 0 — the root group — unintentionally granting elevated group privileges for the entire ssh session.

this did not happen because of bad actors or malice, just a simple slip caused by far too few resources. open source does not mean someone is actually watching—here, no one noticed this for **NINE** months, until the maintainer finally caught and fixed it themselves.

### the impact

Now in this industry we like to call giving users automatic root group access "Pretty bad", as it can cause some damage:

- **group-based permissions** are everywhere. services gate access using group ownership of sockets, shared files, devices, etc.  
- **root group** often has write access to critical subsystems, especially in systems with legacy compatibility configs.
- **unix domain sockets** used by daemons like docker, podman, or legacy systemd setups often rely on group-level security.
- **ephemeral users**, by design, don’t show up in traditional auditing pipelines. so detecting this kind of overprivilege is non-trivial.

this is a quiet privilege escalation that doesn't show up in `id`, doesn't scream in logs, and persists for the duration of the session.

### the fix

the patch is as simple as the bug:  
set the gid to match the uid, creating a "user private group" (a common unix convention; see [Debian's writeup](https://wiki.debian.org/UserPrivateGroups#UPGs)).  
check that gid doesn't collide with an existing group.  
done.

### the actual problem

this is not a one-off mistake. this is the predictable result of letting a small team own a security-critical subsystem.

authentication code is infrastructure. nss and pam are some of the most security-sensitive paths on a linux system. this is where identity gets resolved. this is where trust is conferred. you don't roll this kind of system out lightly. you don't write a half-page daemon and replace battle-tested modules with it.

yet that’s exactly what canonical did. no formal audit. no coordination with upstream. and—of course—no bug bounty or disclosure pipeline in place when it turned out their half-written daemon granted root group access to all ssh users.

why are we letting distro vendors write their own authentication stacks with two engineers and no budget?

authd is a textbook case of **auth module rot**: the tendency for smaller projects to reimplement authentication logic in a vacuum, poorly, and with just enough gloss to make it look like it works.

we’ve seen this pattern before:
- security modules reimplemented in language du jour (see: electron password managers)
- glibc behaviors misunderstood or undocumented (see: UID/GID edge cases, environment variable inheritance)
- nss/pam ordering bugs that no one remembers how to debug
- fallback defaults that assume trust where there should be failure

you don’t build login infrastructure casually.  
you don’t touch pam without knowing what pam actually does.  
and you definitely don’t ship broken group logic.

But hey! at least they have 100% unit test coverage :D


### Sources and citations:

- [Security report](https://github.com/ubuntu/authd/security/advisories/GHSA-g8qw-mgjx-rwjr)
- [Git blame](https://github.com/ubuntu/authd/blame/619ce8e55953b970f1765ddaad565081538151ab/internal/services/user/user.go#L198)
- [Debian's default behavior](https://wiki.debian.org/UserPrivateGroups#UPGs)
