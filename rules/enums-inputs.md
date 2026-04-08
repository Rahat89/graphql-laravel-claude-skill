# Enums, Input Types, Interfaces & Unions

## Enums

Extend `Rebing\GraphQL\Support\EnumType`. Define values in `$attributes['values']`.

```php
<?php

namespace App\GraphQL\Enums;

use Rebing\GraphQL\Support\EnumType;

class StatusEnum extends EnumType
{
    protected $attributes = [
        'name' => 'StatusEnum',
        'description' => 'The status of a resource',
        'values' => [
            'ACTIVE'   => 'active',    // key = GraphQL value, value = PHP/DB value
            'INACTIVE' => 'inactive',
            'PENDING'  => 'pending',
        ],
    ];
}
```

**Artisan:** `php artisan make:graphql:enum StatusEnum`

**Usage:** `GraphQL::type('StatusEnum')`

## Input Types

For complex mutation arguments. Extend `Rebing\GraphQL\Support\InputType`.

```php
<?php

namespace App\GraphQL\Inputs;

use GraphQL\Type\Definition\Type;
use Rebing\GraphQL\Support\InputType;

class CreatePostInput extends InputType
{
    protected $attributes = [
        'name' => 'CreatePostInput',
        'description' => 'Input for creating a post',
    ];

    public function fields(): array
    {
        return [
            'title' => [
                'type' => Type::nonNull(Type::string()),
                'rules' => ['required', 'max:255'],
            ],
            'body' => [
                'type' => Type::nonNull(Type::string()),
                'rules' => ['required'],
            ],
            'tags' => [
                'type' => Type::listOf(Type::string()),
            ],
        ];
    }
}
```

**Artisan:** `php artisan make:graphql:input CreatePostInput`

**Usage in mutation args:** `'input' => ['type' => GraphQL::type('CreatePostInput')]`

### OneOf Input Objects

Require exactly one field to be set:

```php
class SearchInput extends InputType
{
    protected $attributes = [
        'name' => 'SearchInput',
        'isOneOf' => true,
    ];
    // ...
}
```

**Artisan:** `php artisan make:graphql:input SearchInput --oneof`

## Interfaces

Shared contracts that multiple types implement.

```php
<?php

namespace App\GraphQL\Interfaces;

use Rebing\GraphQL\Support\InterfaceType;
use Rebing\GraphQL\Support\Facades\GraphQL;
use GraphQL\Type\Definition\Type;

class ContentInterface extends InterfaceType
{
    protected $attributes = [
        'name' => 'Content',
        'description' => 'A content item',
    ];

    public function fields(): array
    {
        return [
            'id' => ['type' => Type::nonNull(Type::int())],
            'title' => ['type' => Type::string()],
            'created_at' => ['type' => Type::string()],
        ];
    }

    public function resolveType($root)
    {
        if ($root instanceof \App\Models\Post) {
            return GraphQL::type('Post');
        }
        if ($root instanceof \App\Models\Page) {
            return GraphQL::type('Page');
        }
    }
}
```

**Artisan:** `php artisan make:graphql:interface ContentInterface`

**Implementing type declares interface:**

```php
class PostType extends GraphQLType
{
    public function interfaces(): array
    {
        return [GraphQL::type('Content')];
    }

    public function fields(): array
    {
        // Reuse interface fields
        $interface = GraphQL::type('Content');
        return array_merge($interface->getFields(), [
            'body' => ['type' => Type::string()],
        ]);
    }
}
```

## Unions

Polymorphic return types.

```php
<?php

namespace App\GraphQL\Unions;

use Rebing\GraphQL\Support\UnionType;
use Rebing\GraphQL\Support\Facades\GraphQL;

class SearchResultUnion extends UnionType
{
    protected $attributes = ['name' => 'SearchResult'];

    public function types(): array
    {
        return [
            GraphQL::type('Post'),
            GraphQL::type('User'),
            GraphQL::type('Comment'),
        ];
    }

    public function resolveType($value)
    {
        return match (true) {
            $value instanceof \App\Models\Post => GraphQL::type('Post'),
            $value instanceof \App\Models\User => GraphQL::type('User'),
            $value instanceof \App\Models\Comment => GraphQL::type('Comment'),
        };
    }
}
```

**Artisan:** `php artisan make:graphql:union SearchResultUnion`

## Custom Scalars

For custom data types (dates, JSON, URLs, etc.):

```php
<?php

namespace App\GraphQL\Scalars;

use GraphQL\Language\AST\StringValueNode;
use GraphQL\Type\Definition\ScalarType;

class DateTimeScalar extends ScalarType
{
    public $name = 'DateTime';

    public function serialize($value)
    {
        return $value instanceof \Carbon\Carbon ? $value->toISOString() : $value;
    }

    public function parseValue($value)
    {
        return \Carbon\Carbon::parse($value);
    }

    public function parseLiteral($valueNode, ?array $variables = null)
    {
        if ($valueNode instanceof StringValueNode) {
            return \Carbon\Carbon::parse($valueNode->value);
        }
        return null;
    }
}
```

**Artisan:** `php artisan make:graphql:scalar DateTimeScalar`

## Registration

All types must be registered in `config/graphql.php`:

```php
'types' => [
    'StatusEnum'       => StatusEnum::class,
    'CreatePostInput'  => CreatePostInput::class,
    'Content'          => ContentInterface::class,
    'SearchResult'     => SearchResultUnion::class,
    'DateTime'         => DateTimeScalar::class,
],
```
