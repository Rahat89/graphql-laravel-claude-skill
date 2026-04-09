# Authorization & Privacy

## Two Levels of Access Control

| Mechanism | Scope | On Failure |
|-----------|-------|------------|
| `authorize()` | Entire query/mutation | Returns error, blocks execution |
| `privacy` | Individual type field | Returns `null` for that field |

## Operation-Level Authorization

Override `authorize()` on Query or Mutation. Must return exactly `true` (strict check).

```php
public function authorize($root, array $args, $ctx, $resolveInfo = null): bool
{
    return auth()->check();
}

public function getAuthorizationMessage(): string
{
    return 'You must be logged in to access this resource';
}
```

### Authorization Traits

For consistent auth across many queries/mutations, use a trait:

```php
trait Authorizable
{
    public function authorize($root, array $args, $ctx, $resolveInfo = null): bool
    {
        return auth()->check();
    }
}

// Usage
class MyPostsQuery extends Query
{
    use Authorizable;
    // ...
}
```

### Token-Based Authorization

For stateless APIs using Bearer tokens or Laravel Sanctum:

```php
trait TokenAuthorizable
{
    public $user;

    public function authorize($root, array $args, $ctx, $resolveInfo = null): bool
    {
        // Use Laravel's built-in auth (works with Sanctum, Passport, etc.)
        $this->user = auth()->user();

        if (!$this->user) {
            throw new \Rebing\GraphQL\Error\AuthorizationError('Invalid credentials');
        }

        return true;
    }
}
```

Then in resolve, `$this->user` is available:
```php
public function resolve($root, array $args, $context, PostService $postService)
{
    return $postService->getUserPosts($this->user, $args);
}
```

If your API needs to resolve users from custom headers or GraphQL args (e.g., for external clients that can't use standard auth middleware), override the user resolution logic in the trait to check those sources.

> **Note:** Prefer header-based auth (Bearer token, Sanctum) over passing tokens in GraphQL args. Headers keep credentials out of query logs and caches.

### Schema-Specific Auth

Different schemas may need different auth strategies:

- **Public API schema** — Token-based auth (Bearer header or args)
- **Admin schema** — Session auth + role check
- **External/partner schema** — API key auth with additional validation

Create separate traits for each and use the right one per schema. Don't mix auth strategies across schemas.

## Field-Level Privacy

Set on individual Type fields. Returns `null` when access is denied — use nullable types.

### Closure Form

```php
'email' => [
    'type' => Type::string(),  // Must be nullable!
    'privacy' => function ($root, array $args, $ctx): bool {
        return $root->id === auth()->id();
    },
],
```

### Class Form

```php
use Rebing\GraphQL\Support\Privacy;

class MePrivacy extends Privacy
{
    public function validate($root, array $fieldArgs, $ctx = null, ?ResolveInfo $resolveInfo = null): bool
    {
        return $root->id === auth()->id();
    }
}

// Usage on field
'email' => [
    'type' => Type::string(),
    'privacy' => MePrivacy::class,
],
```

## When to Use What

| Scenario | Use |
|----------|-----|
| Endpoint requires login | `authorize()` on Query/Mutation |
| Endpoint requires specific role | `authorize()` with role check |
| Hide sensitive field from other users | `privacy` on Type field |
| Same auth across many endpoints | Auth trait |
| Different auth per schema | Schema-specific auth traits |

## Best Practices

1. **Only** add `authorize()` when the endpoint explicitly requires authentication
2. Don't add auth "just to be safe" — unauthenticated endpoints have legitimate use cases
3. Keep auth logic in reusable traits, not duplicated per query/mutation
4. Use `privacy` for field-level access, `authorize()` for operation-level
5. Privacy fields must be nullable — they return `null` on denial, not an error
6. Create separate auth traits for different schemas — don't mix strategies
