# Traits & Code Reuse

Traits are a powerful architectural pattern for avoiding code duplication across GraphQL queries, mutations, and types.

## Shared Query/Mutation Resolver Traits

**When to use:** Whenever the same endpoint needs to exist as both a Query and Mutation, put ALL logic in a shared trait. Both classes then become one-liners.

This is common when migrating from queries to mutations or when clients need both access patterns.

### Structure

The trait defines all four methods: `attributes()`, `type()`, `args()`, `resolve()`.

```php
<?php

namespace App\GraphQL\Traits;

use App\Services\PostService;
use GraphQL\Type\Definition\Type;
use Rebing\GraphQL\Support\Facades\GraphQL;

trait PostsResolver
{
    public function attributes(): array
    {
        return [
            'name' => 'posts',
            'description' => 'Get paginated posts',
        ];
    }

    public function type(): Type
    {
        return GraphQL::paginate('Post');
    }

    public function args(): array
    {
        return [
            'page' => ['type' => Type::int(), 'defaultValue' => 1],
            'per_page' => ['type' => Type::int(), 'defaultValue' => 15],
            'category_id' => ['type' => Type::int()],
        ];
    }

    public function resolve($root, array $args, $context, PostService $postService)
    {
        return $postService->getPosts($args);
    }
}
```

### Usage

Both Query and Mutation become one-liners:

```php
// app/GraphQL/Queries/PostsQuery.php
class PostsQuery extends Query
{
    use PostsResolver;
}

// app/GraphQL/Mutations/PostsMutation.php
class PostsMutation extends Mutation
{
    use PostsResolver;
}
```

### Naming Convention

- `{Feature}Resolver` — short form (e.g., `PostsResolver`)
- `{Feature}QueryMutationResolver` — explicit form (e.g., `PostsQueryMutationResolver`)

Pick one convention and stick with it across the project.

## Authorization Traits

Centralize auth logic in a reusable trait instead of duplicating `authorize()` across queries/mutations.

```php
<?php

namespace App\GraphQL\Traits;

trait Authorizable
{
    public function authorize($root, array $args, $ctx, $resolveInfo = null): bool
    {
        return !auth()->guest();
    }

    public function getAuthorizationMessage(): string
    {
        return 'You are not authorized to perform this action';
    }
}
```

### Schema-Specific Auth Traits

For projects with multiple schemas (e.g., frontend API vs. plugin API), create separate auth traits:

```php
// For standard API endpoints
trait ApiAuthorizable
{
    public function authorize($root, array $args, $ctx, $resolveInfo = null): bool
    {
        return auth()->check();
    }
}

// For plugin/external endpoints with domain validation
trait PluginAuthorizable
{
    public function authorize($root, array $args, $ctx, $resolveInfo = null): bool
    {
        // Validate user + domain ownership
        $user = User::where('api_key', $args['api_key'])->first();
        return $user && $this->validateDomain($user, $args);
    }
}
```

**Important:** Use the right auth trait for the right schema. Plugin-specific auth (with domain validation) should never be used on public API endpoints.

## Field Resolver Traits (on Types)

Extract shared field resolution logic into traits when multiple types need the same behavior:

```php
trait FormatsDateFields
{
    protected function resolveCreatedAtField($root, array $args)
    {
        return $root->created_at?->toISOString();
    }

    protected function resolveUpdatedAtField($root, array $args)
    {
        return $root->updated_at?->toISOString();
    }
}

trait ResolvesMedia
{
    protected function resolveThumbnailField($root, array $args)
    {
        return $root->getFirstMediaUrl('thumbnail');
    }

    protected function resolveBannerField($root, array $args)
    {
        return $root->getFirstMediaUrl('banner');
    }
}

// Usage on Type classes
class PostType extends GraphQLType
{
    use FormatsDateFields, ResolvesMedia;
    // ...
}
```

## When to Create a Trait

1. **Query/Mutation pair** — Same endpoint in both forms? Always use a trait.
2. **Auth pattern** — Same `authorize()` across 5+ queries/mutations? Extract to a trait.
3. **Field resolution** — Same resolver on 3+ types? Extract to a trait.
4. **One-off logic** — Used in only one place? Keep it inline. Don't over-abstract.

## Best Practices

1. **Always** use a shared trait when both Query and Mutation exist for the same endpoint
2. Check existing traits before creating new ones
3. Auth traits go on Query/Mutation classes (or in shared resolver traits), NOT on Types
4. Field resolver traits go on Type classes only
5. Don't create a trait for logic used in only one place
