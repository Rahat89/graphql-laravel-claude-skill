# GraphQL Types

## Class Hierarchy

```
Type (abstract)        # Base for all GraphQL types
├── InputType          # → InputObjectType (mutation arguments)
├── EnumType           # → EnumType (fixed value sets)
├── InterfaceType      # → InterfaceType (shared contracts)
├── UnionType          # → UnionType (polymorphic returns)
└── UploadType         # → ScalarType (file uploads)
```

## Standard Type Structure

```php
<?php

namespace App\GraphQL\Types;

use App\Models\Example;
use GraphQL\Type\Definition\Type;
use Rebing\GraphQL\Support\Facades\GraphQL;
use Rebing\GraphQL\Support\Type as GraphQLType;

class ExampleType extends GraphQLType
{
    protected $attributes = [
        'name'        => 'Example',
        'description' => 'An example resource',
        'model'       => Example::class, // Required for SelectFields optimization
    ];

    public function fields(): array
    {
        return [
            'id' => [
                'type' => Type::nonNull(Type::int()),
                'description' => 'The id of the example',
            ],
            'name' => [
                'type' => Type::string(),
                'description' => 'The name',
            ],
            // DB column alias — maps GraphQL field name to model attribute
            'created' => [
                'type' => Type::string(),
                'alias' => 'created_at',
            ],
            // Computed field — excluded from SQL SELECT
            'display_name' => [
                'type' => Type::string(),
                'selectable' => false,
            ],
            // Single relation
            'author' => [
                'type' => GraphQL::type('User'),
                'alias' => 'user',
            ],
            // List relation
            'comments' => [
                'type' => Type::listOf(GraphQL::type('Comment')),
            ],
        ];
    }
}
```

**Artisan:** `php artisan make:graphql:type ExampleType`

## Field Attributes

| Attribute | Purpose |
|-----------|---------|
| `type` | Required. The GraphQL type |
| `description` | Human-readable description for introspection |
| `alias` | Maps field name to DB column (for SelectFields) |
| `selectable` | Set `false` for computed/virtual fields not in DB |
| `always` | Array of columns to always include in SELECT |
| `is_relation` | Set `false` for JSON columns that aren't relations |
| `deprecationReason` | Marks field as deprecated |
| `privacy` | Closure or Privacy class for field-level access control |
| `query` | Closure to customize the eager-load query (for SelectFields) |
| `complexity` | Callback for query complexity analysis |
| `args` | Field-level arguments |

## Custom Field Resolvers

Use `resolve{FieldName}Field()` methods for custom field logic. The method name is StudlyCase of the field name.

```php
// For a field named 'thumbnail'
protected function resolveThumbnailField($root, array $args)
{
    return asset($root->thumbnail);
}

// For a field named 'full_name'
protected function resolveFullNameField($root, array $args)
{
    return "{$root->first_name} {$root->last_name}";
}
```

## Resolver Traits for Shared Logic

When multiple types need the same field resolution, extract into a trait:

```php
trait ThumbnailResolver
{
    protected function resolveThumbnailField($root, array $args)
    {
        return asset($root->thumbnail);
    }
}

// Usage
class ItemType extends GraphQLType
{
    use ThumbnailResolver;
    // ...
}
```

## Relationship Fields with Custom Queries

For SelectFields optimization, use the `query` attribute:

```php
'posts' => [
    'type' => Type::listOf(GraphQL::type('Post')),
    'args' => [
        'date_from' => ['type' => Type::string()],
    ],
    'query' => function (array $args, $query, $ctx): void {
        $query->where('posts.created_at', '>', $args['date_from']);
    },
],
```

## Type Modifiers (Shorthand)

```php
GraphQL::type('MyType!')      // Type::nonNull(GraphQL::type('MyType'))
GraphQL::type('[MyType]')     // Type::listOf(GraphQL::type('MyType'))
GraphQL::type('[MyType]!')    // Type::nonNull(Type::listOf(...))
GraphQL::type('[MyType!]!')   // Type::nonNull(Type::listOf(Type::nonNull(...)))
```

## Registration

All types must be registered in `config/graphql.php`:

```php
// Global types (shared across schemas)
'types' => [
    'Example' => ExampleType::class,
],

// Or per-schema
'schemas' => [
    'default' => [
        'types' => [ExampleType::class],
    ],
],
```
