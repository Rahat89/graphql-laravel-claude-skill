# GraphQL Laravel Skill for Claude Code

A [Claude Code skill](https://docs.anthropic.com/en/docs/claude-code) that teaches Claude architecture patterns and best practices for building GraphQL APIs with [`rebing/graphql-laravel`](https://github.com/rebing/graphql-laravel).

## What This Does

This skill makes Claude Code understand how to properly structure a production GraphQL API in Laravel. Instead of generating generic boilerplate, Claude will follow battle-tested patterns for:

- **Thin resolvers** — Business logic delegated to Service classes
- **DRY code via traits** — Shared resolver logic for Query/Mutation pairs
- **Consistent patterns** — Centralized repetitive args, structured error responses
- **Proper authorization** — Auth traits, field-level privacy, schema-specific strategies
- **Validation** — Inline rules, custom messages, custom closures
- **Multiple schemas** — Separate schemas for different API surfaces (public, admin, partner)
- **Testing** — Integration tests for queries, mutations, auth, and validation

## Installation

Copy the `SKILL.md` and `rules/` directory into your project's `.claude/skills/graphql-development/` directory:

```bash
# From your Laravel project root
mkdir -p .claude/skills/graphql-development

# Clone and copy
git clone https://github.com/Rahat89/graphql-laravel-claude-skill.git /tmp/graphql-skill
cp /tmp/graphql-skill/SKILL.md .claude/skills/graphql-development/
cp -r /tmp/graphql-skill/rules .claude/skills/graphql-development/
rm -rf /tmp/graphql-skill
```

Or add as a git submodule:

```bash
git submodule add https://github.com/Rahat89/graphql-laravel-claude-skill.git .claude/skills/graphql-development
```

## Structure

```
SKILL.md                      # Main skill file — quick reference & core principles
rules/
├── types.md                  # Type definitions, field attributes, custom resolvers
├── queries.md                # Query patterns, auth, pagination, SelectFields
├── mutations.md              # Mutation patterns, validation, error handling
├── traits.md                 # Shared resolver traits, auth traits, field resolver traits
├── enums-inputs.md           # Enums, Input Types, Interfaces, Unions, Scalars
├── config.md                 # Schema config, registration checklist, middleware
├── error-handling.md         # Error layers, response helpers, validation errors
├── pagination.md             # Pagination types, custom PaginationType, helpers
├── authorization.md          # authorize(), privacy, auth traits, schema-specific auth
└── testing.md                # Integration testing for GraphQL endpoints
```

## Core Principles

1. **Thin resolvers, fat services** — `resolve()` is glue code. Business logic lives in Service classes.
2. **DRY via traits** — Same endpoint as both Query and Mutation? All logic goes in a shared trait.
3. **Consistent arguments** — Pagination and auth args defined once in helpers, reused everywhere.
4. **Auth only when needed** — Don't add `authorize()` by default. Only when explicitly required.
5. **Register everything** — All types/queries/mutations go in `config/graphql.php`.

## Requirements

- PHP 8.2+
- Laravel 12+
- [rebing/graphql-laravel](https://github.com/rebing/graphql-laravel) v9+
- [Claude Code](https://docs.anthropic.com/en/docs/claude-code)

## Customization

This skill is designed to be customized for your project. Common modifications:

- **Add your helper methods** — If you centralize pagination or other args in a helper class, reference it in the rules
- **Add your auth traits** — Replace generic examples with your project's actual auth trait names
- **Add your response types** — If you use a `ResponseType`, document it in error-handling.md
- **Add project conventions** — Base classes, naming patterns, schema names
- **Add your auth strategy** — Token via headers, args, or both — update authorization.md to match

## Contributing

Contributions welcome! If you've found patterns that work well in production GraphQL APIs with `rebing/graphql-laravel`, open a PR.

## License

MIT
