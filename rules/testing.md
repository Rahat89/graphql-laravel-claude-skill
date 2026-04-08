# Testing GraphQL

## Integration Tests

Test GraphQL endpoints via HTTP using Laravel's test helpers:

```php
// Basic query
$response = $this->postJson('/graphql', [
    'query' => '{ posts { data { id title } total } }',
]);

$response->assertOk()
    ->assertJsonStructure([
        'data' => [
            'posts' => [
                'data' => [['id', 'title']],
                'total',
            ],
        ],
    ]);
```

## Query with Variables

```php
$response = $this->postJson('/graphql', [
    'query' => '
        query($id: Int!) {
            post(id: $id) {
                id
                title
                body
            }
        }
    ',
    'variables' => ['id' => 1],
]);

$response->assertOk()
    ->assertJsonPath('data.post.id', 1);
```

## Testing Mutations

```php
$response = $this->postJson('/graphql', [
    'query' => '
        mutation($title: String!, $body: String!) {
            createPost(title: $title, body: $body) {
                id
                title
            }
        }
    ',
    'variables' => [
        'title' => 'Test Post',
        'body' => 'Test body content',
    ],
]);

$response->assertOk()
    ->assertJsonPath('data.createPost.title', 'Test Post');
```

## Testing Non-Default Schemas

```php
// POST to /graphql/{schemaName}
$response = $this->postJson('/graphql/admin', [
    'query' => '{ adminPosts { id } }',
]);
```

## Testing Authorization

```php
// Unauthenticated — should fail
$response = $this->postJson('/graphql', [
    'query' => '{ myPosts { data { id } } }',
]);

$response->assertOk()
    ->assertJsonPath('errors.0.extensions.category', 'authorization');

// Authenticated — should succeed
$user = User::factory()->create();
$this->actingAs($user);

$response = $this->postJson('/graphql', [
    'query' => '{ myPosts { data { id } } }',
]);

$response->assertOk()
    ->assertJsonStructure(['data' => ['myPosts' => ['data']]]);
```

## Testing Validation

```php
$response = $this->postJson('/graphql', [
    'query' => '
        mutation {
            createPost(title: "", body: "") {
                id
            }
        }
    ',
]);

$response->assertOk()
    ->assertJsonPath('errors.0.extensions.category', 'validation')
    ->assertJsonPath('errors.0.extensions.validation.title.0', 'The title field is required.');
```

## Testing with Factories

```php
it('returns paginated posts', function () {
    $posts = Post::factory()->count(5)->create();

    $response = $this->postJson('/graphql', [
        'query' => '{ posts(per_page: 2, page: 1) { data { id } total per_page } }',
    ]);

    $response->assertOk()
        ->assertJsonPath('data.posts.total', 5)
        ->assertJsonPath('data.posts.per_page', 2)
        ->assertJsonCount(2, 'data.posts.data');
});
```

## Asserting Errors

```php
// Assert specific error message
$response->assertJsonPath('errors.0.message', 'Post not found');

// Assert error category
$response->assertJsonPath('errors.0.extensions.category', 'authorization');

// Assert no errors
$response->assertJsonMissing(['errors']);
```

## Best Practices

1. Use factories for test data — never create models manually
2. Test both success and error paths for every query/mutation
3. Test authorization by making requests with and without auth
4. Test validation by sending invalid data and checking error messages
5. Verify response structure with `assertJsonStructure()`, not just values
6. For authenticated endpoints, use `actingAs()` or pass token in variables
7. Test pagination by creating more items than `per_page` and verifying counts
