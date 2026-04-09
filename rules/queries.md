# GraphQL Queries

## Base Class

If your project defines a base Query class with shared middleware, extend that. Otherwise extend Rebing's `Query` directly.

```php
// If project has a base class:
use App\GraphQL\Queries\Query;

// Otherwise:
use Rebing\GraphQL\Support\Query;
```

## Standard Query Structure

```php
<?php

namespace App\GraphQL\Queries;

use App\Services\PostService;
use GraphQL\Type\Definition\Type;
use Rebing\GraphQL\Support\Facades\GraphQL;
use Rebing\GraphQL\Support\Query;

class PostsQuery extends Query
{
    protected $attributes = [
        'name' => 'posts',
        'description' => 'Get paginated list of posts',
    ];

    public function type(): Type
    {
        return GraphQL::paginate('Post');
    }

    public function args(): array
    {
        return [
            'page' => [
                'type' => Type::int(),
                'defaultValue' => 1,
            ],
            'per_page' => [
                'type' => Type::int(),
                'defaultValue' => 15,
            ],
            'category_id' => [
                'type' => Type::int(),
                'description' => 'Filter by category',
            ],
        ];
    }

    public function resolve($root, array $args, $context, PostService $postService)
    {
        return $postService->getPosts($args);
    }
}
```

**Artisan:** `php artisan make:graphql:query PostsQuery`

## Authenticated Query

Override `authorize()` only when the endpoint requires authentication:

```php
<?php

namespace App\GraphQL\Queries;

use App\Services\UserService;
use GraphQL\Type\Definition\Type;
use Rebing\GraphQL\Support\Facades\GraphQL;
use Rebing\GraphQL\Support\Query;

class MyPostsQuery extends Query
{
    protected $attributes = [
        'name' => 'myPosts',
    ];

    public function authorize($root, array $args, $ctx, $resolveInfo = null): bool
    {
        return !auth()->guest();
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
        ];
    }

    public function resolve($root, array $args, $context, UserService $userService)
    {
        return $userService->getUserPosts(auth()->user(), $args);
    }
}
```

## Single Resource Query

```php
public function type(): Type
{
    return GraphQL::type('Post');  // Single type, not paginated
}

public function args(): array
{
    return [
        'id' => [
            'type' => Type::nonNull(Type::int()),
            'rules' => ['required', 'integer'],
        ],
    ];
}

public function resolve($root, array $args, $context, PostService $postService)
{
    return $postService->findById($args['id']);
}
```

## Resolve Method — Dependency Injection

The `resolve()` method supports DI after the first 3 positional params:

```php
// Positional: $root, $args, $context
// Injected: anything after
public function resolve($root, array $args, $context, PostService $postService, ResolveInfo $info)
{
    return $postService->getPosts($args);
}
```

Available injectable types:
- `ResolveInfo $info` — field resolution info
- `Closure $getSelectFields` — lazy factory for SelectFields
- `SelectFields $fields` — eager-loaded instance
- Any class resolvable from the Laravel container

## SelectFields (Optimized Eager Loading)

Use when you need SQL optimization based on requested fields:

```php
use Closure;
use Rebing\GraphQL\Support\SelectFields;

public function resolve($root, array $args, $context, Closure $getSelectFields)
{
    $fields = $getSelectFields();

    return Post::select($fields->getSelect())
        ->with($fields->getRelations())
        ->paginate($args['per_page']);
}
```

## Resolver Middleware

Apply per-query middleware via the `$middleware` property:

```php
class PostsQuery extends Query
{
    protected $middleware = [
        ResolvePage::class,
        LogQuery::class,
    ];
}
```

## Best Practices

1. **Always** delegate to a Service class — no business logic in resolve
2. **Only** add `authorize()` when authentication is explicitly required
3. **Centralize** repeated argument patterns (pagination, sorting, filtering) in static helper methods
4. When the same endpoint exists as both Query and Mutation, use a shared trait
5. Register all queries in `config/graphql.php` under the appropriate schema
6. Use `GraphQL::paginate()` for list endpoints, bare `GraphQL::type()` for single resources
