# Security Policy

## Reporting a Vulnerability

If you discover a security vulnerability in this repository, please report it
privately so it can be addressed before public disclosure.

**Preferred channel:** [Open a private security advisory](https://github.com/hijacksecurity/HijackSecurityBlog/security/advisories/new)
on this repository. This keeps the report confidential until a fix is published.

Please include:

- A description of the issue and its impact
- Steps to reproduce
- Affected files, URLs, or versions
- Any suggested mitigations

You can expect an initial response within a few business days. Please do not
open public issues, pull requests, or social-media posts disclosing the
vulnerability before it has been triaged.

## Scope

This repository hosts a Jekyll-based static blog deployed to GitHub Pages.
In-scope issues include:

- Cross-site scripting (XSS) or content-injection in rendered pages
- Build-time supply-chain or workflow vulnerabilities (`Gemfile`, GitHub Actions)
- Information disclosure from the repository or built site

Out of scope: third-party services linked from posts, and findings that
require physical or social-engineering access.
