# Error Handling

## Two Error Layers

### Client-Safe Errors (GraphQL Errors)

Extend `GraphQL\Error\Error`. Returned in GraphQL responses. Users see these.

| Class | Category | When Thrown |
|-------|----------|-------------|
| `ValidationError` | `validation` | Validation rules fail |
| `AuthorizationError` | `authorization` | `authorize()` returns false |
| `AutomaticPersistedQueriesError` | `apq` | APQ hash mismatch |

### Developer Errors (Exceptions)

Extend `RuntimeException`. Configuration/developer mistakes. NOT client-safe.

| Class | When Thrown |
|-------|-------------|
| `SchemaNotFound` | Invalid schema name |
| `TypeNotFound` | Unregistered type referenced |

## Error Handling in Mutations

Always wrap service calls in try/catch:

```php
public function resolve($root, array $args, $context, PostService $postService)
{
    try {
        $post = $postService->create(auth()->user(), $args);
        return $post;
    } catch (\Exception $e) {
        throw new \GraphQL\Error\Error($e->getMessage());
    }
}
```

### With a ResponseType Pattern

If your project uses a generic response type:

```php
public function resolve($root, array $args, $context, PostService $postService)
{
    try {
        $postService->create(auth()->user(), $args);
        return ['status' => true, 'message' => 'Post created successfully'];
    } catch (\Exception $e) {
        return ['status' => false, 'message' => $e->getMessage()];
    }
}
```

Tip: Create helper methods for consistent responses:

```php
// In a Helper class
public static function successResponse(string $message): array
{
    return ['status' => true, 'message' => $message];
}

public static function errorResponse(string $message): array
{
    return ['status' => false, 'message' => $message];
}
```

## Authorization Errors

Handled automatically by `authorize()`. Return `false` to deny:

```php
public function authorize($root, array $args, $ctx, $resolveInfo = null): bool
{
    return auth()->check();
}

public function getAuthorizationMessage(): string
{
    return 'You are not authorized to perform this action';
}
```

Response:
```json
{
    "errors": [{
        "message": "You are not authorized to perform this action",
        "extensions": { "category": "authorization" }
    }]
}
```

## Validation Errors

Handled automatically when rules are defined in `args()`.

Response:
```json
{
    "errors": [{
        "message": "validation",
        "extensions": {
            "category": "validation",
            "validation": {
                "email": ["The email field is required."]
            }
        }
    }]
}
```

Custom messages via `validationErrorMessages()`:

```php
public function validationErrorMessages(array $args = []): array
{
    return [
        'email.required' => 'Please provide your email',
    ];
}
```

## Custom Error Formatting

Override in config:

```php
'error_formatter' => [MyErrorFormatter::class, 'format'],
'errors_handler' => [MyErrorHandler::class, 'handle'],
```

Debug mode (`APP_DEBUG=true`) automatically includes `debugMessage` and `trace` in error responses.

## Best Practices

1. Return structured error responses from mutations — don't let exceptions bubble
2. Auth errors are handled by `authorize()` — don't duplicate auth checks in resolve
3. Validation errors are handled by the framework when rules are in args
4. Use `getAuthorizationMessage()` for custom auth error messages
5. Log unexpected exceptions before returning error responses
