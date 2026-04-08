---
name: graphql-laravel
description: "Use this skill whenever creating, modifying, or reviewing GraphQL types, queries, mutations, enums, input types, interfaces, unions, scalars, traits, or middleware using rebing/graphql-laravel. Triggers when working with GraphQL schema config, pagination types, authorization patterns, resolver middleware, or any file under app/GraphQL/. Also activate when the user mentions GraphQL endpoints, GraphQL types, or API schema changes in a Laravel project."
license: MIT
metadata:
  author: developer-jetwing
  package: rebing/graphql-laravel
  requires: PHP 8.2+, Laravel 12+
---

# GraphQL Laravel Development (rebing/graphql-laravel)

Architecture patterns and best practices for building production GraphQL APIs with `rebing/graphql-laravel`. These rules focus on **how to structure your code** — for exact API syntax, check the [official docs](https://github.com/rebing/graphql-laravel).

## Consistency First

Before applying any rule, check what the project already does. Look at sibling files in the same directory. If a pattern exists, follow it — don't introduce a second way. These rules are defaults for when no pattern exists yet, not overrides.

## Core Principles

1. **Thin resolvers, fat services** — Queries and mutations delegate business logic to Service classes. The `resolve()` method is glue code only.
2. **DRY via traits** — When the same endpoint exists as both a Query and Mutation, all logic lives in a shared trait. Never duplicate resolver code.
3. **Consistent arguments** — Use helper methods for repeated argument patterns (pagination, auth). Define them once, reuse everywhere.
4. **Auth only when needed** — Don't add authorization by default. Only use `authorize()` when the endpoint explicitly requires authentication.
5. **Register everything** — All types, queries, mutations must be registered in `config/graphql.php`. Easy to forget, hard to debug.

## Quick Reference

### 1. Types → `rules/types.md`

- Extend `Rebing\GraphQL\Support\Type`
- Define `$attributes` with `name`, `description`, and `model` (for SelectFields)
- Use `alias` for DB column mapping, `selectable: false` for computed fields
- Use `resolve{FieldName}Field()` for custom field resolution
- Extract shared resolution logic into reusable traits
- Register in `config/graphql.php` under `types`

### 2. Queries → `rules/queries.md`

- Extend a project-level base Query (or Rebing's `Query` directly)
- Use helper methods for pagination and auth arguments
- Return paginated types via `GraphQL::paginate('typeName')`
- Delegate to Service classes in resolve
- Use `authorize()` only when auth is explicitly required

### 3. Mutations → `rules/mutations.md`

- Extend a project-level base Mutation (or Rebing's `Mutation` directly)
- Inline validation rules in `args()`, custom messages in `validationErrorMessages()`
- Return consistent response types for success/error
- Wrap service calls in try/catch for error handling
- Delegate to Service classes in resolve

### 4. Traits & Code Reuse → `rules/traits.md`

- Shared Query/Mutation resolver traits define `attributes()`, `type()`, `args()`, `resolve()`
- Auth traits centralize authorization logic
- Field resolver traits share Type field resolution logic
- Always check for an existing trait before creating a new one

### 5. Enums, Inputs, Interfaces & Unions → `rules/enums-inputs.md`

- Enums extend `EnumType`, Input types extend `InputType`
- Interfaces extend `InterfaceType` with `resolveType()`
- Unions extend `UnionType` with `types()` and `resolveType()`
- Register all in `config/graphql.php`

### 6. Configuration & Registration → `rules/config.md`

- All components registered in `config/graphql.php`
- Support for multiple schemas with independent middleware
- Execution middleware wraps the full pipeline
- Resolver middleware wraps individual field resolvers

### 7. Error Handling → `rules/error-handling.md`

- Use consistent response helpers for success/error returns
- Auth errors handled by `authorize()` — don't duplicate in resolve
- Validation errors handled automatically when rules are in args
- Catch exceptions in mutations, return structured error responses

### 8. Pagination → `rules/pagination.md`

- `GraphQL::paginate()`, `simplePaginate()`, `cursorPaginate()` for return types
- Centralize pagination arguments in a helper
- Extend `PaginationType` for custom pagination fields if needed

### 9. Authorization & Privacy → `rules/authorization.md`

- `authorize()` gates entire operations
- `privacy` gates individual type fields
- Keep auth logic in reusable traits
- Only add auth when explicitly required

### 10. Testing → `rules/testing.md`

- Use `postJson('/graphql', ...)` for integration tests
- Test both success and error paths
- Use factories for test data
- Verify GraphQL response structure
