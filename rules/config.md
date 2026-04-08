# Configuration & Registration

## Config File

All GraphQL components are registered in `config/graphql.php`. Publish with:

```bash
php artisan vendor:publish --provider="Rebing\GraphQL\GraphQLServiceProvider"
```

## Schema Configuration

### Single Schema (Simple)

```php
'schemas' => [
    'default' => [
        'query' => [
            PostsQuery::class,
            UserQuery::class,
        ],
        'mutation' => [
            CreatePostMutation::class,
            UpdatePostMutation::class,
        ],
        'middleware' => ['auth:api'],
        'method' => ['GET', 'POST'],
    ],
],
```

### Multiple Schemas

For projects with distinct API surfaces (e.g., public API, admin, plugin):

```php
'schemas' => [
    'default' => [
        'query' => [PublicPostsQuery::class],
        'mutation' => [],
        'middleware' => ['throttle:60,1'],
        'method' => ['GET', 'POST'],
    ],
    'admin' => [
        'query' => [AdminPostsQuery::class],
        'mutation' => [AdminCreatePostMutation::class],
        'middleware' => ['auth:api', 'admin'],
        'method' => ['POST'],
    ],
],
```

### Class-Based Schema Config

For complex schemas, use a config class:

```php
'schemas' => [
    'default' => DefaultSchema::class,
],
```

```php
use Rebing\GraphQL\Support\Contracts\ConfigConvertible;

class DefaultSchema implements ConfigConvertible
{
    public function toConfig(): array
    {
        return [
            'query' => [...],
            'mutation' => [...],
            'middleware' => [...],
        ];
    }
}
```

**Artisan:** `php artisan make:graphql:schemaConfig DefaultSchema`

## Global Types

Types shared across all schemas:

```php
'types' => [
    'User'     => UserType::class,
    'Post'     => PostType::class,
    'Response' => ResponseType::class,
],
```

## Route Configuration

```php
'route' => [
    'prefix' => 'graphql',           // URL prefix
    'controller' => GraphQLController::class . '@query',  // Custom controller
    'middleware' => [],               // Global HTTP middleware
    'group_attributes' => [],         // Route group attributes
],
```

Routes:
- Default schema: `POST /graphql`
- Named schema: `POST /graphql/{schemaName}`

## Execution Middleware

Wraps the full GraphQL execution pipeline (not individual resolvers):

```php
'execution_middleware' => [
    \Rebing\GraphQL\Support\ExecutionMiddleware\ValidateOperationParamsMiddleware::class,
    \Rebing\GraphQL\Support\ExecutionMiddleware\AutomaticPersistedQueriesMiddleware::class,
    \Rebing\GraphQL\Support\ExecutionMiddleware\AddAuthUserContextValueMiddleware::class,
    // Custom execution middleware here
],
```

Per-schema overrides are supported — schema-level config replaces global.

## Resolver Middleware

Wraps individual field resolvers. Register per query/mutation:

```php
class PostsQuery extends Query
{
    protected $middleware = [LogQuery::class];
}
```

Or globally:

```php
'resolver_middleware_append' => [GlobalLogMiddleware::class],
```

**Artisan:** `php artisan make:graphql:middleware ResolvePageMiddleware`
**Artisan:** `php artisan make:graphql:executionMiddleware CustomExecutionMiddleware`

## Security Settings

```php
'security' => [
    'query_max_complexity' => 500,    // Max query complexity
    'query_max_depth' => 13,          // Max nesting depth
    'disable_introspection' => true,  // Disable in production
],
```

## Batching

```php
'batching' => [
    'enable' => false,          // Enable batch queries
    'max_batch_size' => 10,     // Max operations per batch (null = unlimited)
],
```

## Automatic Persisted Queries (APQ)

```php
'apq' => [
    'enable' => false,          // Enable APQ
    'cache_driver' => null,     // Laravel cache driver
    'cache_prefix' => 'apq',   // Cache key prefix
    'cache_ttl' => 300,         // TTL in seconds
],
```

## Custom Pagination Type

Override the default pagination wrapper:

```php
'pagination_type' => \Rebing\GraphQL\Support\PaginationType::class,
```

## Error Handling

```php
'error_formatter' => [\Rebing\GraphQL\GraphQL::class, 'formatError'],
'errors_handler' => [\Rebing\GraphQL\GraphQL::class, 'handleErrors'],
```

## Registration Checklist

When creating a new GraphQL component:

1. **Type** → Add to `config/graphql.php` under `'types'`
2. **Query** → Add to the appropriate schema's `'query'` array
3. **Mutation** → Add to the appropriate schema's `'mutation'` array
4. **Enum** → Add to `'types'` (same as regular types)
5. **Input Type** → Add to `'types'` (same as regular types)
6. **Interface** → Add to `'types'` (same as regular types)
7. **Union** → Add to `'types'` (same as regular types)
8. **Scalar** → Add to `'types'` (same as regular types)

Forgetting to register is the #1 cause of "type not found" errors.

## Available Artisan Commands

```
make:graphql:type              App\GraphQL\Types
make:graphql:query             App\GraphQL\Queries
make:graphql:mutation          App\GraphQL\Mutations
make:graphql:enum              App\GraphQL\Enums
make:graphql:input             App\GraphQL\Inputs
make:graphql:interface         App\GraphQL\Interfaces
make:graphql:union             App\GraphQL\Unions
make:graphql:scalar            App\GraphQL\Scalars
make:graphql:field             App\GraphQL\Fields
make:graphql:middleware        App\GraphQL\Middleware
make:graphql:executionMiddleware  App\GraphQL\Middleware\Execution
make:graphql:schemaConfig      App\GraphQL\Schemas
```
