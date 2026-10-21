# Security Write-ups

A collection of my write-ups from CTF challenges and hands-on security labs,
documenting the vulnerabilities, my approach (including the dead ends), and
the underlying root cause and fix for each one.

I'm a Master's student in Computer Science (Kiel University) transitioning
into cybersecurity, with a focus on offensive security. This repo is where I
track that progress in public.

## How this repo is organized

```
security-writeups/
├── picoctf/
│   ├── web-exploitation/
└── other/
```

Each challenge gets its own folder with a `README.md` containing:

- **Challenge summary** — category, platform, what was given, the goal
- **Approach** — including what I tried that *didn't* work and why
- **The key step, explained** — not just the payload, but why it works
- **Root cause and fix** — the underlying vulnerability class and how a
  developer would actually prevent it
- **Tools used**

Flags are not included in these write-ups.

## Index

| Challenge | Category | Platform | Key Technique |
|---|---|---|---|
| [Bank-Loan Calculator](./picoCTF/bank-loan-calculator-eval-rce/) | Web / RCE | CyLab Security Academy | Python `eval()` blocklist bypass via `chr()` encoding |

*(updated as new write-ups are added)*

## Why I'm doing this

I'm building practical skills through CTF practice (picoCTF, HTB, CyLab
Academy) alongside the Google Cybersecurity Professional Certificate and
CompTIA Security+. Writing these up properly — not just "here's the flag" —
is how I make sure I actually understood the vulnerability, not just
pattern-matched a solution.
