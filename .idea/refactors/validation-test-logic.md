# Context
Initial test suite had repetition of boilerplate logic alongside tessting HTTP response and asserting that database also
saved the payload to the database. These validation rules were repeated across all functions meaning a large amount of
repetition of boilerplate code across test suite which slows down testing speed.

Splitting validation logic into a separate test suite makes the validation rules much easier to add or change any 
additional validation rules in the future as the application evolves

## *Roadmap Alignment:  [Enforcing Single Responsibility], [Logic Abstraction]*

### Legacy code

```
 public function test_new_mix_uploaded_success_without_provider_name_or_id(): void
    {
        $this->actingAsAdmin();

        $resident = Resident::factory()->create();
        Mix::factory()->create(['resident_id' => 1]);
        Storage::fake('public');

        $fakeImage = UploadedFile::fake()->create('test_cover.png', 100);

        $testData = [
            'title' => 'test',
            'slug' => 'test',
            'description' => 'test',
            'release_date' => '2025-01-01',
            'external_media_provider' => 'test',
            'external_media_url' => 'https://google.com',
            'cover_image_url' => $fakeImage,
            'resident_ids' => [$resident->id],
            'genres' => 'techno,house',
        ];

        $response = $this->postJson('/api/mixes', $testData);

        $response->assertStatus(201)
            ->assertJson(function (AssertableJson $json) {
                $json->has('message')
                    ->has('data');
            });

        $this->assertDatabaseHas('mixes', [
            'title' => 'test',
            'slug' => 'test',
            'description' => 'test',
            'release_date' => '2025-01-01',
            'external_media_provider' => 'test',
            'external_media_url' => 'https://google.com',
        ]);

        $mix = Mix::where('slug', 'test')->first();

        Storage::disk('public')->assertExists('assets/'.$mix->cover_image_url);

        $this->assertNotEquals('test_cover.png', $mix->cover_image_url);
    }

    public function test_new_mix_uploaded_success_without_provider_name(): void
    {
        $this->actingAsAdmin();

        Resident::factory()->create();
        Mix::factory()->create(['resident_id' => 1]);
        Storage::fake('public');

        $fakeImage = UploadedFile::fake()->create('test_cover.png', 100);

        $testData = [
            'title' => 'test',
            'slug' => 'test',
            'description' => 'test',
            'release_date' => '2025-01-01',
            'external_media_provider' => 'test',
            'external_media_url' => 'https://google.com',
            'cover_image_url' => $fakeImage,
            'resident_ids' => [1],
            'genres' => 'test,test1',
            'provider_id' => 'test',
        ];

        $response = $this->postJson('/api/mixes', $testData);

        $response->assertStatus(201)
            ->assertJson(function (AssertableJson $json) {
                $json->has('message')
                    ->has('data');
            });

        $this->assertDatabaseHas('mixes', [
            'title' => 'test',
            'slug' => 'test',
            'description' => 'test',
            'release_date' => '2025-01-01',
            'external_media_provider' => 'test',
            'external_media_url' => 'https://google.com',
            'provider_id' => 'test',
        ]);

        $mix = Mix::where('slug', 'test')->first();

        Storage::disk('public')->assertExists('assets/'.$mix->cover_image_url);

        $this->assertNotEquals('test_cover.png', $mix->cover_image_url);
    }

    public function test_new_mix_uploaded_success_without_provider_id(): void
    {
        $this->actingAsAdmin();

        Resident::factory()->create();
        Mix::factory()->create(['resident_id' => 1]);

        $fakeImage = UploadedFile::fake()->create('test_cover.png', 100);

        $testData = [
            'title' => 'test',
            'slug' => 'test',
            'description' => 'test',
            'release_date' => '2025-01-01',
            'external_media_provider' => 'test',
            'external_media_url' => 'https://google.com',
            'cover_image_url' => $fakeImage,
            'resident_ids' => [1],
            'provider' => 'test',
            'genres' => 'test',
        ];

        $response = $this->postJson('/api/mixes', $testData);

        $response->assertStatus(201)
            ->assertJson(function (AssertableJson $json) {
                $json->hasAll('message')
                    ->has('data');
            });

        $this->assertDatabaseHas('mixes', [
            'title' => 'test',
            'slug' => 'test',
            'description' => 'test',
            'release_date' => '2025-01-01',
            'external_media_provider' => 'test',
            'external_media_url' => 'https://google.com',
            'provider' => 'test',
        ]);

        $mix = Mix::where('slug', 'test')->first();
        Storage::disk('public')->assertExists('assets/'.$mix->cover_image_url);
    }

    public function test_new_mix_invalid_external_media_provider_wrong_datatype(): void
    {
        $this->actingAsAdmin();

        Resident::factory()->create();
        Mix::factory()->create(['resident_id' => 1]);

        $testData = [
            'title' => 'test',
            'slug' => 'test',
            'description' => 'test',
            'release_date' => '2025-01-01',
            'external_media_provider' => 1,
            'external_media_url' => 'https://google.com',
            'cover_image_url' => 'https://google.com',
            'resident_id' => 1,
            'provider' => 'test',
        ];

        $response = $this->postJson('/api/mixes', $testData);
        $response->assertInvalid('external_media_provider');
    }

    public function test_new_mix_invalid_external_media_provider_url(): void
    {
        $this->actingAsAdmin();

        Resident::factory()->create();
        Mix::factory()->create(['resident_id' => 1]);

        $testData = [
            'title' => 'test',
            'slug' => 'test',
            'description' => 'test',
            'release_date' => '2025-01-01',
            'external_media_provider' => 'test',
            'external_media_url' => 'www.google.com',
            'cover_image_url' => 'https://google.com',
            'resident_id' => 1,
            'provider' => 'test',
        ];

        $response = $this->postJson('/api/mixes', $testData);
        $response->assertInvalid('external_media_url');
    }

    public function test_new_mix_invalid_title_empty_title(): void
    {
        $this->actingAsAdmin();

        Resident::factory()->create();
        Mix::factory()->create(['resident_id' => 1]);

        $testData = [
            'title' => '',
            'slug' => 'test',
            'description' => 'test',
            'release_date' => '2025-01-01',
            'external_media_url' => 'https://google.com',
            'cover_image_url' => 'https://google.com',
            'resident_id' => 1,
        ];

        $response = $this->postJson('/api/mixes', $testData);
        $response->assertInvalid('title');
    }

    public function test_new_mix_invalid_title_too_long(): void
    {
        $this->actingAsAdmin();

        Resident::factory()->create();
        Mix::factory()->create(['resident_id' => 1]);

        $testData = [
            'title' => str_repeat('a', 256),
            'slug' => 'test',
            'description' => 'test',
            'release_date' => '2025-01-01',
            'media_url' => 'https://google.com',
            'cover_image_url' => 'https://google.com',
            'resident_id' => 1,
        ];

        $response = $this->postJson('/api/mixes', $testData);
        $response->assertInvalid('title');
    }

    public function test_new_mix_invalid_title_datatype(): void
    {
        $this->actingAsAdmin();

        Resident::factory()->create();
        Mix::factory()->create(['resident_id' => 1]);

        $testData = [
            'title' => 1,
            'slug' => 'test',
            'description' => 'test',
            'release_date' => '2025-01-01',
            'media_url' => 'https://google.com',
            'cover_image_url' => 'https://google.com',
            'resident_id' => 1,
        ];

        $response = $this->postJson('/api/mixes', $testData);
        $response->assertInvalid('title');
    }

    public function test_new_mix_invalid_slug_empty(): void
    {
        $this->actingAsAdmin();

        Resident::factory()->create();
        Mix::factory()->create(['resident_id' => 1]);

        $testData = [
            'title' => 'test',
            'slug' => '',
            'description' => 'test',
            'release_date' => '2025-01-01',
            'media_url' => 'https://google.com',
            'cover_image_url' => 'https://google.com',
            'resident_id' => 1,
        ];

        $response = $this->postJson('/api/mixes', $testData);
        $response->assertInvalid('slug');
    }

    public function test_new_mix_invalid_slug_too_long(): void
    {
        $this->actingAsAdmin();

        Resident::factory()->create();
        Mix::factory()->create(['resident_id' => 1]);

        $testData = [
            'title' => 'test',
            'slug' => str_repeat('a', 256),
            'description' => 'test',
            'release_date' => '2025-01-01',
            'media_url' => 'https://google.com',
            'cover_image_url' => 'https://google.com',
            'resident_id' => 1,
        ];

        $response = $this->postJson('/api/mixes', $testData);
        $response->assertInvalid('slug');
    }

    public function test_new_mix_invalid_slug_datatype(): void
    {
        $this->actingAsAdmin();

        Resident::factory()->create();
        Mix::factory()->create(['resident_id' => 1]);

        $testData = [
            'title' => 'test',
            'slug' => 1,
            'description' => 'test',
            'release_date' => '2025-01-01',
            'media_url' => 'https://google.com',
            'cover_image_url' => 'https://google.com',
            'resident_id' => 1,
        ];

        $response = $this->postJson('/api/mixes', $testData);
        $response->assertInvalid('slug');
    }

    public function test_new_mix_invalid_description(): void
    {
        $this->actingAsAdmin();

        Resident::factory()->create();
        Mix::factory()->create(['resident_id' => 1]);

        $testData = [
            'title' => 'test',
            'slug' => 'test',
            'description' => 1,
            'release_date' => '2025-01-01',
            'media_url' => 'https://google.com',
            'cover_image_url' => 'https://google.com',
            'resident_id' => 1,
        ];

        $response = $this->postJson('/api/mixes', $testData);
        $response->assertInvalid('description');
    }

    public function test_new_mix_invalid_description_too_long(): void
    {
        $this->actingAsAdmin();

        Resident::factory()->create();
        Mix::factory()->create(['resident_id' => 1]);

        $testData = [
            'title' => 'test',
            'slug' => 'test',
            'description' => str_repeat('a', 1025),
            'release_date' => '2025-01-01',
            'media_url' => 'https://google.com',
            'cover_image_url' => 'https://google.com',
            'resident_id' => 1,
        ];

        $response = $this->postJson('/api/mixes', $testData);
        $response->assertInvalid('description');
    }

    public function test_new_mix_invalid_media_url_not_url(): void
    {
        $this->actingAsAdmin();

        Resident::factory()->create();
        Mix::factory()->create(['resident_id' => 1]);

        $testData = [
            'title' => 'test',
            'slug' => 'test',
            'description' => 'test',
            'release_date' => '2025-01-01',
            'external_media_provider' => 'test',
            'external_media_url' => 'test',
            'cover_image_url' => 'https://google.com',
            'resident_id' => 1,
        ];

        $response = $this->postJson('/api/mixes', $testData);
        $response->assertInvalid('external_media_url');
    }

    public function test_new_mix_invalid_media_url(): void
    {
        $this->actingAsAdmin();

        Resident::factory()->create();
        Mix::factory()->create(['resident_id' => 1]);

        $testData = [
            'title' => 'test',
            'slug' => 'test',
            'description' => 'test',
            'release_date' => '2025-01-01',
            'external_media_provider' => 'test',
            'external_media_url' => 'www.test.com',
            'cover_image_url' => 'https://google.com',
            'resident_id' => 1,
        ];

        $response = $this->postJson('/api/mixes', $testData);
        $response->assertInvalid('external_media_url');
    }

    public function test_new_mix_invalid_cover_image_url_not_url(): void
    {
        $this->actingAsAdmin();

        Resident::factory()->create();
        Mix::factory()->create(['resident_id' => 1]);

        $testData = [
            'title' => 'test',
            'slug' => 'test',
            'description' => 'test',
            'release_date' => '2025-01-01',
            'media_url' => 'https://google.com',
            'cover_image_url' => 'test',
            'resident_id' => 1,
        ];

        $response = $this->postJson('/api/mixes', $testData);
        $response->assertInvalid('cover_image_url');
    }

    public function test_new_mix_invalid_cover_image_url(): void
    {
        $this->actingAsAdmin();

        Resident::factory()->create();
        Mix::factory()->create(['resident_id' => 1]);

        $testData = [
            'title' => 'test',
            'slug' => 'test',
            'description' => 'test',
            'release_date' => '2025-01-01',
            'media_url' => 'https://google.com',
            'cover_image_url' => 'www.test.com',
            'resident_id' => 1,
        ];

        $response = $this->postJson('/api/mixes', $testData);
        $response->assertInvalid('cover_image_url');
    }

    public function test_new_mix_invalid_resident_id(): void
    {
        $this->actingAsAdmin();

        Resident::factory()->create();
        Storage::fake('public');

        $fakeImage = UploadedFile::fake()->create('test_cover.jpg', 100);

        $testData = [
            'title' => 'test',
            'slug' => 'test',
            'description' => 'test',
            'release_date' => '2025-01-01',
            'external_media_provider' => 'test',
            'external_media_url' => 'https://google.com',
            'cover_image_url' => $fakeImage,
            'genres' => 'techno, house',
            'resident_ids' => [999],
        ];

        $response = $this->postJson('/api/mixes', $testData);

        $response->assertInvalid(['resident_ids.0']);
    }
```

### Updated
```
class MixValidationTest extends TestCase
{
    use RefreshDatabase;

    protected function setUp(): void
    {
        parent::setUp();
        Storage::fake('public');
    }

    #[DataProvider('invalidMixData')]
    public function test_mix_store_validation_rules($invalidData, $error): void
    {
        $this->actingAsAdmin();

        $resident = Resident::factory()->create();
        $fakeImage = UploadedFile::fake()->create('cover.jpg', 100, 'image/jpeg');

        $basePayload = [
            'title' => 'test',
            'slug' => 'test',
            'description' => 'test',
            'release_date' => '2025-01-01',
            'external_media_provider' => 'test',
            'external_media_url' => 'https://google.com',
            'cover_image_url' => $fakeImage,
            'resident_ids' => [$resident->id],
            'genres' => 'techno, house',
            'provider' => 'test',
            'provider_id' => null,
        ];

        $response = $this->postJson('/api/mixes', array_merge($basePayload, $invalidData));
        $response->assertStatus(422)
            ->assertJsonValidationErrors([$error]);
    }

    public static function invalidMixData(): array
    {
        return [
            'title is empty' => [['title' => ''], 'title'],
            'title is too long' => [['title' => str_repeat('a', 256)], 'title'],
            'title wrong datatype' => [['title' => 1], 'title'],

            'slug is empty' => [['slug' => ''], 'slug'],
            'slug is too long' => [['slug' => str_repeat('a', 256)], 'slug'],
            'slug wrong datatype' => [['slug' => 1], 'slug'],

            'description wrong datatype' => [['description' => 1], 'description'],
            'description is too long' => [['description' => str_repeat('a', 1025)], 'description'],

            'external media provider wrong datatype' => [['external_media_provider' => 1], 'external_media_provider'],

            'external media url missing http' => [['external_media_url' => 'www.google.com'], 'external_media_url'],
            'external media url just string' => [['external_media_url' => 'test'], 'external_media_url'],

            'cover image url missing http' => [['cover_image_url' => 'www.test.com'], 'cover_image_url'],
            'cover image url just string' => [['cover_image_url' => 'test'], 'cover_image_url'],

            'resident id does not exist in db' => [['resident_ids' => [999]], 'resident_ids.0'],
        ];
    }
}
```

### Learnings
Able to cut down on the legacy boilerplate and define all validation rules in one place. This makes it much easier to 
update, alter or remove validation rules as the application evolves. This also places all validation logic in one 
centralised place to further enforce single responsibility principle