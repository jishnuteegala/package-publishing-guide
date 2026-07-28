# Contributing to Package Publishing Guide

First off, thank you for considering contributing! This guide aims to help maintainers ship to every channel safely, and your input is valuable.

## How Can I Contribute?

### 🐛 Reporting Issues

If you find an error, unclear instruction, or missing information:

1. Check if the issue already exists
2. Open a new issue with:
   - Clear description of the problem
   - The channel it concerns (npm, winget, Chocolatey, etc.)
   - Steps to reproduce (if applicable)
   - Relevant tool versions (npm, GoReleaser, gh)

### 💡 Suggesting Enhancements

Have an idea to improve the guide?

- Open an issue describing your suggestion
- Explain why it would be helpful
- Provide examples if possible

### 📝 Pull Requests

Ready to contribute directly?

1. **Fork** the repository
2. **Create a branch**: `git checkout -b feature/your-improvement`
3. **Make your changes**:
   - Keep language clear and practical
   - Test commands if possible
   - Update the table of contents if adding sections
4. **Commit** using the guidelines below
5. **Push**: `git push origin feature/your-improvement`
6. **Open a Pull Request** with a clear description

### 📋 Style Guidelines

- Use clear, concise language
- Include code blocks with proper syntax highlighting
- Prefer real failure modes and exact error messages over abstract advice
- Keep formatting consistent with existing content
- Never include credential values, even expired ones, in examples

### 📝 Commit Message Guidelines

Use a concise, Conventional Commits–style format tailored for this docs-only guide:

- `docs(scope):` content changes to the guide (preferred for new sections)
- `fix(scope):` correct wrong commands or inaccurate statements
- `style(scope):` formatting-only tweaks (spacing, punctuation, Markdown)
- `refactor(scope):` restructure sections/anchors without changing meaning
- `chore(scope):` repo maintenance (links, badges, metadata)

**Notes:**
- Use a clear `scope` like `npm`, `winget`, `credentials`, or `recovery`.
- Avoid `feat:` for documentation additions; use `docs:` instead. Reserve `feat:` only if adding tooling/automation.
- If renaming anchors/sections (breaking deep links), include `BREAKING CHANGE:` in the footer so the major version bumps.
- Releases are automated with [Release Please](https://github.com/googleapis/release-please): merged Conventional Commits accumulate into a release pull request, and merging that PR tags the version and publishes the GitHub release. No manual version bumps or changelog edits are needed.

**Examples:**
- `docs(npm): add namespace-wide trusted publishing note`
- `fix(winget): correct classic PAT scope`
- `style(readme): normalize fenced bash blocks`

### ✅ Good First Contributions

New to contributing? Start here:

- Fix typos or grammatical errors
- Improve unclear explanations
- Add failure modes you have hit on any channel
- Update outdated links or command syntax
- Add channels not yet covered (Nix, Snap, apt/deb repositories)

## Code of Conduct

### Our Pledge

We are committed to providing a welcoming and inclusive environment for everyone, regardless of:
- Experience level
- Gender identity and expression
- Sexual orientation
- Disability
- Personal appearance
- Body size
- Race, ethnicity, or nationality
- Age
- Religion

### Expected Behavior

- Be respectful and constructive
- Welcome newcomers warmly
- Accept feedback gracefully
- Focus on what's best for the community

### Unacceptable Behavior

- Harassment, trolling, or insulting comments
- Personal attacks
- Publishing others' private information
- Other conduct inappropriate in a professional setting

## Questions?

Don't hesitate to ask! Open an issue with your question or start a discussion.

---

Thank you for helping make multi-channel publishing less painful for everyone! 🙏
