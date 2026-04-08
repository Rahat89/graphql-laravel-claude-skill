# GraphQL Mutations

## Base Class

If your project defines a base Mutation class, extend that. Otherwise extend Rebing's `Mutation` directly.

```php
// If project has a base class:
use App\GraphQL\Mutations\Mutation;

// Otherwise:
use Rebing\GraphQL\Support\Mutation;
```

## Standard Mutation Structure

```php
<?php

namespace App\GraphQL\Mutations;

use App\Services\PostService;
use GraphQL\Type\Definition\Type;
use Rebing\GraphQL\Support\Facades\GraphQL;
use Rebing\GraphQL\Support\Mutation;

class CreatePostMutation extends Mutation
{
    protected $attributes = [
        'name' => 'createPost',
        'description' => 'Create a new post',
    ];

    public function authorize($root, array $args, $ctx, $resolveInfo = null): bool
    {
        return !auth()->guest();
    }

    public function type(): Type
    {
        return GraphQL::type('Post');
    }

    public function args(): array
    {
        return [
            'title' => [
                'type' => Type::nonNull(Type::string()),
                'rules' => ['required', 'string', 'max:255'],
            ],
            'body' => [
                'type' => Type::nonNull(Type::string()),
                'rules' => ['required', 'string'],
            ],
            'category_id' => [
                'type' => Type::int(),
                'rules' => ['nullable', 'exists:categories,id'],
            ],
        ];
    }

    public function validationErrorMessages(array $args = []): array
    {
        return [
            'title.required' => 'Please provide a post title',
            'title.max' => 'Title must not exceed 255 characters',
        ];
    }

    public function resolve($root, array $args, $context, PostService $postService)
    {
        try {
            return $postService->create(auth()->user(), $args);
        } catch (\Exception $e) {
            throw new \GraphQL\Error\Error($e->getMessage());
        }
    }
}
```

**Artisan:** `php artisan make:graphql:mutation CreatePostMutation`

## Validation

### Inline Rules in Args

```php
public function args(): array
{
    return [
        'email' => [
            'type' => Type::nonNull(Type::string()),
            'rules' => ['required', 'email', 'unique:users'],
        ],
    ];
}
```

### Custom Validation Closures

```php
'email' => [
    'type' => Type::nonNull(Type::string()),
    'rules' => ['required', 'email', function ($field, $value, $fail) {
        if (User::whereEmail($value)->exists()) {
            $fail('This email is already registered.');
        }
    }],
],
```

### Override rules() Method

For conditional validation:

```php
protected function rules(array $args = []): array
{
    return [
        'password' => $args['id'] !== 1 ? ['required', 'min:8'] : [],
    ];
}
```

### Custom Error Messages

```php
public function validationErrorMessages(array $args = []): array
{
    return [
        'email.required' => 'Please provide your email',
        'email.email' => 'Please provide a valid email address',
        'password.min' => 'Password must be at least :min characters',
    ];
}
```

### Custom Attribute Names

```php
public function validationAttributes(array $args = []): array
{
    return ['email' => 'email address'];
}
```

## File Uploads

Register `UploadType` in config, then use in args:

```php
'file' => [
    'type' => GraphQL::type('Upload'),
    'rules' => ['required', 'image', 'max:2048'],
],
```

Send as `multipart/form-data` following the graphql-multipart-request-spec.

## Error Handling in Mutations

Wrap service calls and return structured errors:

```php
public function resolve($root, array $args, $context, PostService $postService)
{
    try {
        return $postService->create(auth()->user(), $args);
    } catch (\Exception $e) {
        // Option 1: Throw GraphQL error (appears in errors array)
        throw new \GraphQL\Error\Error($e->getMessage());

        // Option 2: Return error response (if using a ResponseType)
        return ['status' => false, 'message' => $e->getMessage()];
    }
}
```

## Best Practices

1. **Always** delegate business logic to Service classes — resolve is glue code
2. **Always** put validation rules inline in `args()` and custom messages in `validationErrorMessages()`
3. **Always** wrap service calls in try/catch for error handling
4. **Only** add `authorize()` when the endpoint explicitly requires authentication
5. When the same endpoint exists as both Query and Mutation, use a shared trait
6. Register all mutations in `config/graphql.php` under the appropriate schema
