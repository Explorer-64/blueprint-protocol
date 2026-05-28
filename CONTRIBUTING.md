# Contributing to Blueprint Protocol

Blueprint Protocol is an open standard. Contributions are welcome — whether that's a new example, a spec clarification, a bug report, or a real-world implementation you want to share.

---

## Ways to Contribute

### Add a real-world example
The most valuable contributions are working blueprints from real apps. If you've implemented `blueprint.txt` on your app, open a PR to add it to `/examples`. Use the habit tracker or demo video tool as a reference for format.

### Suggest a spec change
Open an issue describing the problem you hit and what you think should change. Explain the use case — what broke, what you needed, why the current spec didn't cover it. Changes to the spec require a clear real-world motivation.

### Report something that doesn't work
If you followed the spec and something behaved unexpectedly — an agent ignored the blueprint, a step failed, a field was ambiguous — open an issue. Real-world failure reports are how the spec gets better.

### Fix a typo or clarify wording
Open a PR directly. No issue needed for small corrections.

---

## Pull Request Guidelines

- One change per PR
- PRs that add examples should include a working `blueprint.txt` file in `/examples`
- Spec changes should reference the issue that motivated them
- Keep commit messages descriptive — say what changed and why, not just what

---

## Spec Versioning

Blueprint follows [Semantic Versioning](https://semver.org/):

- **PATCH** — typo fixes, clarifications, no behaviour change
- **MINOR** — new optional fields or actions
- **MAJOR** — breaking changes to required fields or the action verb set

When proposing a spec change, indicate which version bump it warrants.

---

## Questions

Open a Discussion — not an issue. Issues are for bugs and concrete proposals. Discussions are for questions, ideas, and implementation help.

---

*Blueprint Protocol is maintained by Jill Mercer. Licensed under MIT.*
