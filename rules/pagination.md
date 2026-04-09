# Pagination

## Three Pagination Types

### Full Pagination (Most Common)

```php
public function type(): Type
{
    return GraphQL::paginate('Post');
}
```

Returns: `data`, `total`, `per_page`, `current_page`, `from`, `to`, `last_page`

In resolve:
```php
return Post::where('status', 'active')->paginate($args['per_page'], ['*'], 'page', $args['page']);
```

Query: `{ posts(per_page: 10, page: 1) { data { id title } total per_page current_page last_page } }`

### Simple Pagination

```php
public function type(): Type
{
    return GraphQL::simplePaginate('Post');
}
```

Returns: `data`, `per_page`, `current_page`, `from`, `to`, `has_more_pages`

In resolve:
```php
return Post::simplePaginate($args['per_page'], ['*'], 'page', $args['page']);
```

### Cursor Pagination

```php
public function type(): Type
{
    return GraphQL::cursorPaginate('Post');
}
```

Returns: `data`, `per_page`, `previous_cursor`, `next_cursor`

In resolve:
```php
return Post::cursorPaginate($args['per_page'], ['*'], 'cursor', $args['cursor']);
```

## Centralizing Repetitive Arguments

When the same arguments appear across many queries, extract them into a static method to keep definitions consistent and DRY. Pagination is the most common example:

```php
// In a helper class, base query, or a dedicated GraphQL args class — your choice
public static function paginationArgs(): array
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
    ];
}
```

Usage via `array_merge`:
```php
public function args(): array
{
    return array_merge(
        YourArgsClass::paginationArgs(),
        [
            'search' => ['type' => Type::string()],
            // ... other args
        ]
    );
}
```

This pattern works for any repeated arg group — sorting, date range filters, common search params, etc. The key is: define once, reuse everywhere.

## Custom PaginationType

Extend the default pagination type if you need additional fields:

```php
use Rebing\GraphQL\Support\PaginationType as BasePaginationType;

class PaginationType extends BasePaginationType
{
    protected function getPaginationFields($typeName): array
    {
        return array_merge(
            parent::getPaginationFields($typeName),
            [
                'total_pages' => [
                    'type' => Type::nonNull(Type::int()),
                    'resolve' => fn($data) => $data->lastPage(),
                ],
            ]
        );
    }
}
```

Register in config:
```php
'pagination_type' => App\GraphQL\Support\PaginationType::class,
```

## Resolver Middleware for Pagination

Handle pagination normalization in middleware:

```php
class ResolvePerPage extends Middleware
{
    public function handle($root, array $args, $context, ResolveInfo $info, Closure $next)
    {
        // Convert -1 to a large number for "get all"
        if (($args['per_page'] ?? null) === -1) {
            $args['per_page'] = 10000;
        }

        return $next($root, $args, $context, $info);
    }
}
```

## Best Practices

1. Centralize pagination args in a helper — never define page/per_page manually per query
2. Use `GraphQL::paginate()` for list endpoints that need total counts
3. Use `GraphQL::simplePaginate()` for large datasets where total count is expensive
4. Use `GraphQL::cursorPaginate()` for infinite scroll / real-time feeds
5. Delegate actual pagination to the Service layer — resolvers just pass args through
