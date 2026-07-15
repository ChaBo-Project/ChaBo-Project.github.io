# Contributing

ChaBo uses GitHub issues, branches, and pull requests to coordinate work.

## Workflow

1. Pick or create an issue.
2. Create a branch named for the issue and type of work.
3. Keep the branch focused on one issue or closely related issues.
4. Open a pull request into `development` for UAT validation.
5. Link the issue in the pull request description.
6. Promote `development` to `main` after review and UAT validation.
7. Keep reusable technical development documented.

Merged pull request branches from this repository are deleted automatically.
Branches from forks are left untouched.

## Branch Naming

Examples:

```text
docs/mkdocs-github-pages-bootstrap
research/whatsapp-channel-meta-setup
feat/whatsapp-channel-service
docs/management-backend-spec
```

## Pull Request Checklist

- The PR references the relevant issue.
- Documentation is included for new services.
- Secrets are not committed.
- Local setup and deployment steps are documented.
- Tests or verification steps are included when applicable.
