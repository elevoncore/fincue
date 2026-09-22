# Security Policy

This file covers **how to report a vulnerability**. For the platform's
security architecture (auth, encryption, threat model), see
[`docs/security.md`](./docs/security.md).

## Supported versions

This project is a pre-alpha Final Year Project. There is currently one
actively developed line (`main`). Once a `v1.0` demo build exists, this
table will track which versions receive security fixes.

| Version | Supported |
|---|---|
| `main` (pre-release) | ✅ |

## Reporting a vulnerability

If you believe you've found a security vulnerability in this project
(for example: an auth bypass, a way to read another user's financial data,
an injection vulnerability, or exposed secrets):

1. **Do not** open a public GitHub issue.
2. Use GitHub's **private security advisory** feature on this repository
   (`Security` tab → `Report a vulnerability`), or email the maintainer
   address listed in `README.md` directly.
3. Include: a description of the issue, steps to reproduce, and the
   potential impact. A proof-of-concept is welcome but not required.

## What to expect

This is a small student-run project, not a company with a formal security
team — please calibrate expectations accordingly:

- **Acknowledgement:** within 5 business days.
- **Assessment:** we'll confirm whether it's a genuine issue and its
  severity within 10 business days of acknowledgement.
- **Fix timeline:** depends on severity and academic schedule constraints;
  we will communicate a rough timeline once assessed.

## Scope

In scope: this repository's code, configuration, and documented
infrastructure (Supabase project, `ai-engine` service, deployed web app).

Out of scope: vulnerabilities in third-party services we depend on
(Supabase, Vercel, Groq, Gemini, Plaid, etc.) — please report those directly
to the respective vendor. Social engineering against maintainers is also
out of scope.

## Safe harbor

We will not pursue legal action against good-faith security research that:

- Avoids privacy violations, data destruction, and service disruption.
- Only interacts with test/demo accounts you created yourself (this app
  handles financial data — please do not attempt to access real user data
  during testing).
- Gives us a reasonable opportunity to fix the issue before any public
  disclosure.
