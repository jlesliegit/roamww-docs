# Context 
Controller was initially handling business logic, database transaction and file upload alongisde HTTP requests.
This also has a few redundancies which need to be addressed: 
- Manual masking (`makeHidden()`) is brittle and should utilise API Resources to explicitly define what is sent vs 
trusting user input
- Moving `DB::Transaction`into the Service Layer ensures that if the genre syncing fails, the database doesn't end up
with an orphaned Mix record without genres or residents. This preserves Data Integrity.
- Moving file handling to a dedicated service prepares codebase for further S3 Integration ahead of migration from local 
env to cloud based hosting

## *Roadmap Alignment: [Service Layer], [API Resources], [Request Standardisation]*

### Legacy code: 
```class MixApiController extends Controller
{
    public function all(Request $request): JsonResponse
    {
        $resident_id = $request->query('resident_id');
        $exclude_mix_id = $request->query('exclude_mix_id');

        $mixQuery = Mix::with(['genres' => function ($query) {
            $query->select('genres.name');
        }]);

        if ($resident_id) {
            $mixQuery->where('resident_id', $resident_id);
        }

        if ($exclude_mix_id) {
            $mixQuery->where('id', '!=', $exclude_mix_id);
        }

        $mixes = $mixQuery->get()->each(function ($mix) {
            $mix->genres->makeHidden(['pivot', 'created_at', 'updated_at']);
        })->makeHidden(['updated_at', 'created_at']);

        return response()->json([
            'message' => 'Mixes fetched successfully',
            'data' => $mixes,
        ]);
    }

    public function find(Mix $mix): JsonResponse
    {
        $mix->load(['residents', 'resident', 'genres'])
            ->makeHidden(['updated_at', 'created_at']);

        $mix->resident?->makeHidden(['slug', 'updated_at', 'created_at']);
        $mix->residents->makeHidden(['slug', 'updated_at', 'created_at', 'pivot']);
        $mix->genres->makeHidden(['id', 'updated_at', 'created_at', 'pivot']);

        return response()->json([
            'message' => 'Mix fetched successfully',
            'data' => $mix,
        ]);
    }

    public function upload(MixUploadRequest $request): JsonResponse
    {
        return DB::transaction(function () use ($request) {
            try {
                $mix = new Mix;
                $mix->fill($request->only($mix->getFillable()));

                $genreNames = explode(',', $request->input('genres', ''));

                $genreIds = collect($genreNames)
                    ->map(fn ($name) => trim($name))
                    ->filter()
                    ->map(function ($name) {
                        $formattedName = strtolower($name);

                        return Genre::firstOrCreate(
                            ['name' => $formattedName]
                        )->id;
                    });

                if ($request->hasFile('cover_image_url')) {
                    $file = $request->file('cover_image_url');
                    $path = $file->store('assets', 'public');
                    $mix->cover_image_url = basename($path);
                }

                $mix->save();

                $mix->residents()->sync([$request->resident_id]);
                $mix->genres()->sync($genreIds);

                return response()->json([
                    'message' => 'Mix uploaded successfully',
                    'url' => asset('storage/'.$mix->cover_image_url),
                ], 201);

            } catch (\Exception $e) {
                return response()->json([
                    'message' => 'Error uploading mix',
                    'error' => $e->getMessage(),
                ], 500);
            }
        });
    }

    public function update(MixUpdateRequest $request, Mix $mix): JsonResponse
    {
        try {
            $mix->fill($request->only($mix->getFillable()));

            if ($request->has('genres')) {
                $names = explode(',', $request->input('genres'));

                $genreIds = collect($names)
                    ->map(fn ($name) => trim($name))
                    ->filter()
                    ->map(function ($name) {
                        return Genre::firstOrCreate(['name' => strtolower($name)])->id;
                    });

                $mix->genres()->sync($genreIds);
            }

            if ($request->hasFile('cover_image_url')) {
                if ($mix->cover_image_url) {
                    Storage::disk('public')->delete('assets/'.$mix->cover_image_url);
                }

                $file = $request->file('cover_image_url');
                $path = $file->store('assets', 'public');
                $mix->cover_image_url = basename($path);
            }

            $mix->save();

            return response()->json([
                'message' => 'Mix edited',
            ], 200);
        } catch (\Exception $e) {
            return response()->json([
                'message' => 'Error updating mix content',
                'error' => $e->getMessage(),
            ], 500);
        }
    }

    public function delete(Mix $mix): JsonResponse
    {
        try {
            if ($mix->cover_image_url) {
                Storage::disk('public')->delete('assets/'.$mix->cover_image_url);
            }
            $mix->delete();

            return response()->json([
                'message' => 'Mix deleted',
            ]);
        } catch (\Exception $e) {
            return response()->json([
                'message' => 'Error deleting mix',
                'error' => $e->getMessage(),
            ], 500);
        }
    }
}
```

### Updated:

### Learnings: 
