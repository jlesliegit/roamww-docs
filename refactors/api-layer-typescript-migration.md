#Context
Whilst migrating application to TypeScript, this involved revisiting the current api layers. When looking back at these
I noticed that there were a few issues with how these were currently implemented. The first, biggest issue was lack of 
typing and type safety, the reason for the migration in the first place. 

The `fetch` urls were also currently hardcoded, requiring significant manual updating when looking to move to a 
production environment. 

To combat this, the application is being migrated to TypeScript to introduce strict typing and also using axios to 
centralise the api calls.

## *Roadmap Alignment: [Frontend Rigour], [Architecture], [Infrastructure]*

### Legacy code
```
export async function fetchResidents() {
    const res = await fetch("http://localhost:8000/api/residents/");
    if (!res.ok) throw new Error("Failed to fetch residents");

    const json = await res.json();
    return json.data;
}

export async function fetchSingleResident(slug) {
    const res = await fetch(`http://localhost:8000/api/residents/${slug}`);
    if (!res.ok) throw new Error("Failed to fetch single resident");

    const json = await res.json();
    return json.data;
}
```

### Updated
```
import api from "../lib/axios";
import type {Resident} from "../types/resident";
import type {LaravelCollection} from "../types/shared";


interface ResidentResult {
    data: Resident;
    message: string;
}

export async function fetchResidents(): Promise<Resident[]> {
    const response = await api.get<LaravelCollection<Resident>>('/api/residents');

    return response.data.data;
}

export async function fetchSingleResident(slug: string): Promise<Resident> {
    const response = await api.get<ResidentResult>(`/api/residents/${slug}`);
    return response.data.data;
}
```

### Learnings
Implementation of strict typing ensures that the response is always predictable which is imperative when looking at reducing
errors at runtime vs compile time - the reason for introducing TypeScript. 

Switching the fetch logic from a hardcoded url `fetch` to utilise `axios api.get` allows me to easily switch the url this
points at from my local development to cloud environment when switching to AWS environment. Removes repetition of manually 
changing each API layer and centralises where this change happens. 

Introducing `LaravelCollection<T>` generic interface allows me to avoid repeating the same collection wrapper shape across
every API file. A separate `ResidentResult` interface handles single resource responses that have a different shape to 
collections.