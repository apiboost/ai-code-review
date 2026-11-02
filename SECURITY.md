# Security Policy

This workflow runs with secrets next to untrusted pull request content, so security reports matter a lot
to us.

## Reporting a vulnerability

**Do not open a public issue or pull request.** Report vulnerabilities privately through
[GitHub private vulnerability reporting](https://github.com/apiboost/ai-code-review/security/advisories/new),
or via https://www.apiboost.com/contact.

Please include the affected version or commit, a description of the issue, and steps to reproduce. For
example: a caller configuration plus a pull request that gets past the label or permission gate, exposes
a secret, runs PR content, or makes the workflow post unfiltered output.

## Supported versions

Security fixes go into the latest release. Callers pin by commit SHA, so update the pin to pick up a fix.
