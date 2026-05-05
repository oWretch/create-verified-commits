# Contributing

Thank you for considering a contribution to this project!

## Development setup

```bash
npm install
```

Key commands:

```bash
npm test          # run tests (vitest)
npm run typecheck # TypeScript type-check
npm run lint      # ESLint
npm run format    # Prettier (write)
npm run format:check  # Prettier (check only)
npm run build     # compile to dist/index.js via ncc
```

## Commit messages

All commits must follow [Conventional Commits](https://www.conventionalcommits.org/). This is enforced by `commitlint` in CI. Common prefixes:

| Prefix      | When to use                               |
| ----------- | ----------------------------------------- |
| `feat:`     | New input, output, or behaviour           |
| `fix:`      | Bug fix                                   |
| `docs:`     | Documentation only                        |
| `chore:`    | Tooling, dependencies, config             |
| `test:`     | Test additions or changes                 |
| `refactor:` | Code restructure without behaviour change |

Use `!` for breaking changes: `feat!: rename token input`.

## Pull requests

1. Fork the repo and create a branch from `main`.
2. Make your changes with tests where appropriate.
3. Run `npm test` and `npm run typecheck` locally before pushing.
4. Open a PR against `main` — the PR template will guide you through the checklist.

CI will run lint, typecheck, tests, and a build check on every PR. All checks must pass before merge.

## Releases

Releases are fully automated via [semantic-release](https://semantic-release.gitbook.io/) triggered on push to `main`. The release version is derived from conventional commit prefixes:

- `fix:` → patch
- `feat:` → minor
- `feat!:` or `BREAKING CHANGE:` → major

You do not need to bump versions manually.
