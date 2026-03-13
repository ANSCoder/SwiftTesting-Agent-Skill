# Contributing

Thanks for helping improve this Swift Testing skill.

## How to contribute

Submit improvements using a pull request (PR). Contributions are most helpful when they add:

- new patterns in `swift-testing-skill/patterns/`
- new anti-pattern guidance in `swift-testing-skill/anti-patterns/`
- new realistic examples in `swift-testing-skill/examples/`
- migration improvements in `swift-testing-skill/migration/`

## PR expectations

- Keep content focused on Swift Testing and AI agent guidance.
- Use modern Swift Testing syntax in all code snippets.
- Prefer concise, practical explanations.
- Include descriptive assertion messages in examples when behavior is not obvious.
- Ensure examples are realistic and maintainable.

## Suggested PR format

In your PR description, include:

1. What problem the contribution solves
2. Which file(s) were added or updated
3. Why the new pattern/example is useful for AI-generated tests
4. Any before/after snippet if you are replacing an older style

## Content review checklist

Before opening a PR, verify:

- No legacy XCTest-only style is presented as the preferred approach
- Snippets use `@Test` and `#expect` as the default style
- Async examples use `async`/`await` when applicable
- Repeated scenario tests are parameterized when appropriate
- Wording is clear for coding agents and maintainers

## Pull request process

1. Fork the repository.
2. Create a focused branch for your change.
3. Commit your updates with clear messages.
4. Open a PR with the details listed above.
5. Respond to review feedback and update the PR as needed.

Thank you for contributing.
