# Contributing

These rules apply to every repository in RSOcc26 unless it has its own
`CONTRIBUTING.md`.
By taking part you agree to the [code of conduct](CODE_OF_CONDUCT.md).

## Workflow

1. Open or pick an issue describing the change.
2. Create a branch from `main`:
   - `feat/<short-description>` — a new feature
   - `fix/<short-description>` — a bug fix
   - `chore/<short-description>` — tooling, dependencies, housekeeping
   - `docs/<short-description>` — documentation only
3. Commit in small, focused steps.
4. Open a pull request and link the issue (`Closes #123`).
5. Get one approving review and green checks, then squash-merge.

`main` is protected: no direct pushes, no force pushes, no deletion. Every
change goes through a pull request with at least one approval, and all
review conversations must be resolved before merging.

## Commit messages

We follow [Conventional Commits](https://www.conventionalcommits.org/):

```
<type>(<optional scope>): <summary in the imperative>

<optional body: what and why>
```

Types: `feat`, `fix`, `docs`, `refactor`, `test`, `chore`, `ci`, `perf`, `build`.
A breaking change adds `!` after the type (`feat!: …`) and explains it in the body.

Pull requests are squash-merged, so the **pull request title** becomes the
commit on `main` and must follow the same format.

## Pull requests

- Keep them small and about one thing.
- Fill in the template: what changed, why, and how it was tested.
- Update `CHANGELOG.md` under `Unreleased` for user-visible changes.
- Update the README or API documentation when behaviour or configuration changes.

## Reviews

- Review within one working day.
- Approve when the change is correct, tested and understandable; request
  changes with a concrete reason.
- New commits dismiss earlier approvals, so re-request a review after changes.

## Secrets

Never commit credentials, keys, tokens or `.env` files. Secret scanning with
push protection is enabled and will block such pushes. Configuration goes in
environment variables; examples go in `.env.example` with placeholder values.
