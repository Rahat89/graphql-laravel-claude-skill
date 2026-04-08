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

### Token/API Key Authorization

For APIs that accept tokens or API keys instead of session auth. Tokens can come from headers, args, or both — check headers first, then fall back to args:

```php
trait TokenAuthorizable
{
    public $user;

    public function authorize($root, array $args, $ctx, $resolveInfo = null): bool
    {
        $user = $this->resolveUser($args);

        if (!$user) {
            throw new \Rebing\GraphQL\Error\AuthorizationError('Invalid credentials');
        }

        $this->user = $user;

        return true;
    }

    protected function resolveUser(array $args): ?User
    {
        // 1. Check Authorization header (Bearer token)
        $bearerToken = request()->bearerToken();
        if ($bearerToken) {
            return User::where('api_token', $bearerToken)->first();
        }

        // 2. Check custom header (e.g., X-API-Key)
        $apiKey = request()->header('X-API-Key');
        if ($apiKey) {
            return User::where('api_key', $apiKey)->first();
        }

        // 3. Fall back to args (for clients that pass credentials in the query)
        if (!empty($args['token'])) {
            return User::where('api_token', $args['token'])->first();
        }

        if (!empty($args['api_key'])) {
            return User::where('api_key', $args['api_key'])->first();
        }

        return null;
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

> **Note:** Prefer header-based auth (Bearer token, custom headers) over passing tokens in GraphQL args. Headers keep credentials out of query logs and caches. Support args as a fallback for clients that can't set headers (e.g., some WordPress plugins).

### Schema-Specific Auth

Different schemas may need different auth strategies:

- **Public API schema** — Token-based auth, validates user exists and is verified
- **Plugin/External schema** — Token + domain validation, validates the requesting domain is authorized
- **Admin schema** — Session auth + role check

Create separate traits for each and use the right one per schema. Never mix auth strategies across schemas.

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
