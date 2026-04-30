# Context
Test suite was initially testing: HTTP requests, route logic, database transactions, middleware, permissions AND 
business logic in the same test.

This made spotting a failure or bug in the code impossible to initially pinpoint leading to longer debugging times and 
slowing production when building on top of existing codebase.

## *Roadmap Alignment:  [Enforcing Single Responsibility], [Logic Abstraction]*

### Legacy code:
```public function test_new_mix_uploaded_success(): void
{
    $this->actingAsAdmin();

    $resident = Resident::factory()->create();
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
        'genres' => 'techno, house',
        'provider' => 'test',
        'provider_id' => null,
    ];

    $response = $this->postJson('/api/mixes', $testData);

    $response->assertStatus(201)
        ->assertJsonPath('message', 'Mix uploaded successfully');

    $this->assertDatabaseHas('mixes', [
        'slug' => 'test',
        'resident_id' => $resident->id,
    ]);

    $mix = Mix::where('slug', 'test')->firstOrFail();

    $this->assertDatabaseHas('mix_resident', [
        'mix_id' => $mix->id,
        'resident_id' => $resident->id,
    ]);

    $this->assertCount(2, $mix->genres);
    $this->assertTrue($mix->genres->contains('name', 'techno'));
    $this->assertTrue($mix->genres->contains('name', 'house'));

    Storage::disk('public')->assertExists('assets/'.$mix->cover_image_url);
    $this->assertNotEquals('test_cover.png', $mix->cover_image_url);
}
```

### Updated:
**Feature test**
```
    public function test_new_mix_endpoint_returns_correct_response(): void
    {
        $this->actingAsAdmin();

        $resident = Resident::factory()->create();
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
            'genres' => 'techno, house',
            'provider' => 'test',
            'provider_id' => null,
        ];

        $response = $this->postJson('/api/mixes', $testData);

        $response->assertStatus(201)
            ->assertJsonPath('message', 'Mix uploaded successfully')
            ->assertJsonStructure([
                'data' => [
                    'id',
                    'title',
                    'slug',
                    'description',
                    'release_date',
                    'cover_image',
                    'residents',
                    'genres',
                ],
            ]);
    }
```
**Unit Test**
```    public function test_mix_service_stores_data_and_files_correctly(): void
    {
        $resident = Resident::factory()->create();
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
            'genres' => 'techno, house',
            'provider' => 'test',
            'provider_id' => null,
        ];

        $service = new MixService;

        $mix = $service->storeMix($testData, $fakeImage);

        $this->assertDatabaseHas('mixes', [
            'slug' => 'test',
            'resident_id' => $resident->id,
        ]);

        $this->assertDatabaseHas('mix_resident', [
            'mix_id' => $mix->id,
            'resident_id' => $resident->id,
        ]);

        $this->assertCount(2, $mix->genres);
        $this->assertTrue($mix->genres->contains('name', 'techno'));
        $this->assertTrue($mix->genres->contains('name', 'house'));

        Storage::disk('public')->assertExists('assets/'.$mix->cover_image_url);
        $this->assertNotEquals('test_cover.png', $mix->cover_image_url);
    }
```

### Learnings: 
Despite being more boilerplate code written upfront initially, separating concerns not only across the 'working' code 
but also the test suites allows for much more efficient testing and making it much easier to easily identify where bugs 
are increasing debugging efficiency. 