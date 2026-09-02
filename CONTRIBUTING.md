# Contributing to cluster-bootstrap

Thank you for contributing to the OpenShift cluster-bootstrap repository. This document outlines the process for submitting changes, coding conventions, and review expectations.

## How to Contribute

### Reporting Issues

- **Bugs**: Open an issue in this repository with a clear description, steps to reproduce, and expected vs. actual behavior
- **Feature Requests**: Propose enhancements by opening an issue and describing the use case, benefit, and proposed implementation

### Submitting Pull Requests

1. **Fork and clone** the repository
2. **Create a branch** for your change:
   ```bash
   git checkout -b your-feature-branch
   ```
3. **Make your changes** following the coding conventions below
4. **Test your changes** — run `make test-unit` to ensure tests pass
5. **Commit your changes** with a clear, descriptive commit message:
   ```text
   Brief summary of the change (50 chars or less)
   
   More detailed explanation if needed. Explain what changed and why,
   not just what you did. Reference related issues with "Fixes #123"
   or "Related to #456".
   ```
6. **Push your branch** and open a pull request against the `main` branch
7. **Respond to review feedback** — maintainers will review your PR and may request changes

## Code Review Standards

### What Reviewers Look For

- **Correctness**: Does the code do what it's supposed to do? Are edge cases handled?
- **Testing**: Are there tests for new functionality? Do existing tests still pass?
- **Code quality**: Is the code readable, maintainable, and consistent with the rest of the codebase?
- **Documentation**: Are user-facing changes documented? Are comments added where necessary?
- **Scope**: Does the PR stay focused on a single issue or feature?

### Review Turnaround

- **Initial review**: Expect feedback within 2-3 business days
- **Approval requirements**: At least one approval from a repository maintainer (see [OWNERS](./OWNERS))
- **Automated checks**: CI tests must pass before merge

### Approval and Merge

- PRs require at least **one approval** from an [OWNERS](./OWNERS) file maintainer
- All CI checks must pass
- Once approved, a maintainer will merge your PR

## Coding Conventions

### Go Style

- Follow the [Effective Go](https://golang.org/doc/effective_go.html) style guide
- Format code before committing — run `make update-gofmt` to apply `gofmt`, and `make verify-gofmt` to check formatting
- Run `make verify` to run all checks (`verify-gofmt`, `verify-govet`, `verify-golang-versions`)

### Naming

- **Packages**: Short, lowercase, single-word names (e.g., `start`, not `start_command`)
- **Variables**: Use camelCase for local variables, PascalCase for exported names
- **Functions**: Descriptive names — prefer `waitForSelfHostedControlPlane` over `wait`

### Error Handling

- Always check and handle errors — don't ignore them
- Wrap errors with context using `fmt.Errorf("descriptive message: %w", err)`
- Use `UserOutput()` for messages intended for humans watching the bootstrap process
- Log diagnostic information to stderr, not stdout

### Testing

- **Unit tests**: Place tests in `*_test.go` files alongside the code they test
- **Table-driven tests**: Use table-driven tests for testing multiple cases
- **Test naming**: Name tests `TestFunctionName` or `TestFunctionName_Scenario`
- **Coverage**: Aim for meaningful test coverage, especially for critical paths

### Comments

- **Public APIs**: Document all exported functions, types, and constants with godoc-style comments
- **Complex logic**: Add comments explaining *why* something is done, not just *what* is done
- **TODOs**: Use `// TODO: description` for future improvements, include an issue number if available

## What to Test

Since cluster-bootstrap runs during OpenShift installation, end-to-end testing requires a full cluster install. For local development:

- **Unit tests**: Test individual functions and logic in isolation. Existing tests simulate the API with an `httptest` TLS server (see `pkg/start/bootstrap_test.go`) rather than a fake clientset — follow that pattern for new API interactions

### Running Tests

```bash
# Run all unit tests
make test-unit

# Run tests for a specific package
go test ./pkg/start/...

# Run a specific test (e.g. one of the existing tests)
go test ./pkg/start -run TestBootstrapControlPlane
```

## Common Development Tasks

### Building

```bash
# Build the binary
make

# Build container images
make images
```

### Updating Dependencies

```bash
# Update a specific dependency
go get github.com/some/dependency@v1.2.3

# Tidy and vendor
go mod tidy && go mod vendor

# Verify the vendored dependencies are consistent
make verify-deps

# If you need to pin/override versions, edit the overrides and run
make update-deps-overrides
```

### Verifying Changes

```bash
# Run all verification checks (formatting, linting, generated code)
make verify
```

## Backporting

If your change needs to be backported to a previous OpenShift release:

1. Wait for the PR to merge to `main`
2. Create a new PR targeting the release branch (e.g., `release-4.17`)
3. Include `cherry-pick-of: #<PR number>` in the commit message
4. Follow the same review process

## Getting Help

- **Questions**: Open an issue or ask in the OpenShift Slack (`#forum-installer`)
- **Review stalled?**: Ping the reviewers/approvers listed in [OWNERS](./OWNERS) (the `control-plane-approvers` alias, defined in [OWNERS_ALIASES](./OWNERS_ALIASES))
- **Not sure where to start?**: Look for issues labeled `good-first-issue` or `help-wanted`

Thank you for contributing to OpenShift!
