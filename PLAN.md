# ECDA Technical Assessment: Movie Catalogue Plan

## Context
`Technical Assessment.pdf` asks for:
- a **Svelte + TypeScript** frontend
- **at least 2 Kotlin Spring Boot services** that talk to each other over **gRPC**
- **GraphQL** between the frontend and the backend

**Features:** list, add, update and delete movies (with artwork upload); list, add, update and delete cast and creators; search across all three; plus any extras we choose.

**Quality bar:** responsive UI; clear progress feedback for upload, listing, updating and searching; unit tests that cover edge cases; a README covering setup, how to run it and how AI was used. You will have to walk the assessors through the code, so every piece should be simple enough to explain.

---

## Architecture

```
                 GraphQL (HTTP POST /graphql)
 ┌──────────────┐  queries / mutations                     ┌──────────────────┐   gRPC (HTTP/2, protobuf)  ┌──────────────────┐
 │  movie-web   │ ───────────────────────────────────────▶ │  movie-service   │ ─────────────────────────▶ │  people-service  │
 │  SvelteKit+TS│ ◀──▶ REST /movies/{id}/artwork ───────── │  Spring GraphQL  │                            │  Spring gRPC srv │
 └──────────────┘                                          │  gRPC client     │                            │  H2: people      │
                                                           │  H2: movies      │                            └──────────────────┘
                                                           │  disk: artwork   │
                                                           └──────────────────┘
                     both services compile proto (git submodule) → Kotlin stubs
```

- **movie-service** is the single GraphQL entry point (a "backend for frontend", or BFF). It owns movies, credits (which person played which role in which movie) and artwork files.
- **people-service** owns cast and creators (`Person`). It only speaks gRPC.
- **Database per service:** each service has its own H2 database (file mode, under `./data/`). They share no tables, only the gRPC contract.
- **Cross-service flows (movie-service is always the gRPC client):**
  - **Movie → credits → person names:** `BatchGetPeople(ids)`. Spring GraphQL `@BatchMapping` turns this into one call per query rather than one per movie. This avoids the N+1 problem.
  - **Add a credit:** check that the person exists with `GetPerson`.
  - **Delete a person:** call `DeletePerson` over gRPC, then remove that person's credits locally.
  - **Unified search:** search movies locally and call `SearchPeople` over gRPC in parallel (Kotlin coroutines), then merge the results into a GraphQL union.

> **Optional upgrade:** if we later want 3 services, split the GraphQL layer into its own gateway. The proto contract and people-service stay the same.

## Repos

| Repo | Contents |
|---|---|
| `protos` | `.proto` files only (`people/v1/people.proto`), optional `buf.yaml` for linting, README |
| `movie-service` | Kotlin Spring Boot app. `protos` is a submodule at `protos/` |
| `people-service` | Kotlin Spring Boot app. `protos` is a submodule at `protos/` |
| `movie-web` | SvelteKit + TypeScript app |
| `movie-assignment` (umbrella, local folder `ecda`) | The main README and `PLAN.md`, plus an optional `docker-compose.yml`. `people-service`, `movie-service` and `movie-web` are its submodules. `protos` isn't listed, because it comes in nested inside each service. This gives the assessors the single repository the PDF asks for. One `git clone --recurse-submodules` brings in everything, including the nested `protos`. |

**How the submodule works:**
- Each service adds `protos/` as an extra proto source dir for `com.google.protobuf` (protobuf-gradle-plugin):
  ```kotlin
  sourceSets { main { proto { srcDir("protos") } } }
  ```
- The plugins `grpc-java` and `grpc-kotlin` generate the stubs during `./gradlew build`. Generated code is never committed.
- A submodule pins an exact commit. Bump it with `git submodule update --remote protos`, then commit the new pointer in each service. This explicit contract versioning is a good talking point in the walkthrough.
- **Gotchas to document:**
  - Clone with `--recurse-submodules`.
  - Each Dockerfile builds from a checkout where the submodule is present.
  - CI needs `submodules: recursive`.

## Tech choices
Use the latest stable versions available when each project is set up.

**Backend**
- Kotlin, Spring Boot, Gradle Kotlin DSL, JDK 21
- **GraphQL:** Spring for GraphQL, schema-first (`src/main/resources/graphql/*.graphqls`) with `@QueryMapping`, `@MutationMapping`, `@SchemaMapping` and `@BatchMapping`
- **gRPC:** Spring gRPC (the official Spring project) with grpc-kotlin coroutine stubs. If Spring gRPC doesn't support our Boot version, fall back to `net.devh` grpc-spring-boot-starter.
- **Persistence:** Spring Data JPA + H2 (file mode, so data survives restarts; H2 console enabled in dev), Flyway migrations. Search uses case-insensitive `LIKE`, which is enough here.
- **Validation:** Jakarta Validation on inputs. Domain errors map to GraphQL errors (`DataFetcherExceptionResolverAdapter`) and to gRPC `Status` codes (`NOT_FOUND`, `INVALID_ARGUMENT`).

**Artwork upload**
- Upload over a REST endpoint, `POST /movies/{id}/artwork` (`multipart/form-data`, field `file`). Spring for GraphQL doesn't support the GraphQL multipart request spec.
- **Server side:** validate type (jpeg/png/webp) and size (≤ 5 MB), store on disk under a UUID filename, and serve over `GET /movies/{id}/artwork`.
- GraphQL exposes `Movie.artworkUrl` (`/movies/{id}/artwork?v=<filename>`, the filename busts the browser cache on re-upload).
- The frontend sends the upload with `XMLHttpRequest`. Unlike `fetch`, it fires `upload.onprogress` events, which drive a real progress bar.

**Frontend**
- SvelteKit + TypeScript, `@urql/svelte` as the GraphQL client, Tailwind for responsive layout
- `graphql-codegen` to generate typed operations from the schema

**Tests**
- **Backend:** JUnit 5 + MockK (unit), `GraphQlTester` (GraphQL slice), grpc in-process server/channel (gRPC service and client), `@DataJpaTest` on in-memory H2 (repositories)
- **Frontend:** Vitest + @testing-library/svelte. Optionally Playwright for one end-to-end happy path.

## Contract sketches

**`people/v1/people.proto`:** `PeopleService` with these RPCs:
- `CreatePerson`, `GetPerson`, `BatchGetPeople`, `UpdatePerson`, `DeletePerson`
- `ListPeople()`, `SearchPeople(name)`

`Person` fields: `id`, `name`, `roles` (ACTOR / DIRECTOR / WRITER / CREATOR), `birthDate`, `bio`, `photoUrl`.

**GraphQL (movie-service):**
- **Types:**
  - `Movie { id title synopsis releaseYear runtimeMinutes genres artworkUrl credits { person role characterName } }`
  - `Person`
  - `union SearchResult = Movie | Person`
- **Queries:** `movies(filter, sort)`, `movie(id)`, `people(roles)`, `person(id)`, `search(query)`
- **Mutations:**
  - `create/update/deleteMovie`
  - `create/update/deletePerson`, `add/removeCredit`
- **Pagination:** not in v1

## Suggestions, since gRPC and GraphQL are new to you
1. **Write the contracts first.** Do the `.proto` and the `.graphqls` before any code. Both are schema-first technologies, and the generated or validated code follows from them.
2. **Learn gRPC in the order you'll use it:**
   1. a message and service in `.proto`
   2. generated Kotlin stub
   3. server implements the `CoroutineImplBase`
   4. client calls through the `CoroutineStub`
   5. errors travel as `Status` codes, not exceptions
   - Test with `grpcurl` and server reflection on before writing any client.
3. **Learn GraphQL in the order you'll use it:**
   1. a query hits a `@QueryMapping`
   2. nested fields resolve via `@SchemaMapping`
   3. N+1 fixed via `@BatchMapping`
   - Use the GraphiQL UI (`spring.graphql.graphiql.enabled=true`) to experiment.
4. **Build each layer as a thin vertical slice:** Person CRUD end-to-end first, then Movies. You'll learn both technologies on the simplest feature.
5. **Keep a `NOTES.md` of "why" decisions as you go.** It feeds the README's AI section and prepares you for the walkthrough.

## Extra features (functionality item 7)
- Sorting, genre/year filters
- Linking cast and crew to movies with a role and character name
- Debounced search-as-you-type
- Delete confirmation dialog
- Optimistic UI updates with rollback
- Drag-and-drop upload with client-side image preview
- Health checks (Spring Boot Actuator) and `docker compose` one-command startup
- **Feedback UX:** skeleton loaders for lists, spinners on buttons during mutations, toast notifications for success and errors, inline field validation, an upload progress bar with cancel, and empty and error states

---

## Task breakdown (tackle in order; each ends green and committed)

### Phase 0: Setup
- [x] T0.1 Create the 4 repos plus the umbrella `movie-assignment` (3 app submodules). Add `PLAN.md`, the umbrella README and `.gitignore`, and an initial README, `.gitignore` and `.tool-versions` in each repo.
- [x] T0.2 Local tooling via asdf (project `.tool-versions`): Java `openjdk-21`, Node `22.23.4`. Plus `grpcurl` (brew). `buf` optional.

### Phase 1: Contract (`protos`)
- [x] T1.1 Write `people.proto` (optionally lint with `buf`).

### Phase 2: people-service (learn gRPC)
- [x] T2.1 Spring Boot skeleton. Add the `protos` repo as a submodule at `protos/` and get stubs generating. H2 and Flyway `person` table.
- [x] T2.2 Domain + repository + service layer, with validation
- [x] T2.3 gRPC server implementing all RPCs, with error → `Status` mapping and reflection on. Verify with `grpcurl`.
- [x] T2.4 Tests:
  - service unit tests (MockK)
  - gRPC service tests (`@SpringBootTest`, in-memory H2): not found, invalid input, unknown and duplicate batch ids
  - Not covered:
    - search: case-insensitive and partial match, blank input over gRPC, `%` / `_` treated as text
    - full field round-trip: roles, birthDate, bio, photoUrl
    - delete, then get → `NOT_FOUND`; deleting a person with roles (cascade on `person_role`)
    - update: `""` clears bio/photoUrl/birthDate; invalid input on update
    - createPerson blank bio → `null` (current unit test only passes the default `null`)
    - `getPersonById` found, `deletePerson` on an existing id (service unit tests)
    - over-length fields (no `@Size`, a too-long value surfaces as `INTERNAL`)
    - empty `BatchGetPeople` request, `ListPeople`
    - over the wire via a real gRPC client (status codes and descriptions as a caller sees them)

### Phase 3: movie-service (learn GraphQL + gRPC client)
- [x] T3.1 Skeleton, `protos` repo as a submodule at `protos/`, H2 and Flyway (`movie`, `credit` tables)
- [x] T3.2 GraphQL schema + Person queries and mutations proxied to people-service over gRPC. Try them in GraphiQL.
- [x] T3.3 Movie CRUD + credits + `@BatchMapping` for credit → person
- [x] T3.4 Artwork: REST `POST` / `GET /movies/{id}/artwork`, validation, disk storage, `Movie.artworkUrl`
- [ ] T3.5 Unified search (coroutines, parallel local search + gRPC call)
- [ ] T3.6 Error handling: map gRPC `Status` codes and domain errors to GraphQL errors. Handle people-service being unavailable gracefully.
- [ ] T3.7 Tests:
  - unit tests (MockK the gRPC stub)
  - `GraphQlTester` tests
  - upload edge cases: wrong type, too large, empty file, unknown movie
  - search edge cases: empty, special characters, no results, one side failing

### Phase 4: movie-web
- [ ] T4.1 SvelteKit + TS + Tailwind + urql + codegen. Layout and navigation, responsive.
- [ ] T4.2 Movies: grid with artwork, skeletons, empty and error states
- [ ] T4.3 Movie create/edit form with validation, delete confirmation, toasts
- [ ] T4.4 Artwork upload: XHR with a progress bar, preview, cancel, error display
- [ ] T4.5 People list plus CRUD, and credit management on the movie page
- [ ] T4.6 Global debounced search with a loading indicator and grouped results
- [ ] T4.7 Tests (Vitest + Testing Library):
  - form validation
  - upload progress and error states
  - search debounce
  - list loading, empty and error states

### Phase 5: Integration and submission
- [ ] T5.1 README run steps (and optional scripts) to start all 3 apps locally. Optional: Dockerfiles + umbrella `docker-compose.yml` (artwork volume, health checks).
- [ ] T5.2 Seed data (a few movies and people) via Flyway or a seed script
- [ ] T5.3 READMEs:
  - umbrella: finalise the existing draft (architecture, setup, run, test) and confirm every command works
  - an "How AI tools were used" section
  - each repo's own README
- [ ] T5.4 Final pass: run every test suite, test on mobile width, dry-run the walkthrough script

### Enhancements (beyond v1)
- [ ] E1 Orphaned credits: calling `DeletePerson` directly on people-service (not via movie-service) leaves that person's credits in movie-service. Options:
  - people-service publishes a `PersonDeleted` event that movie-service consumes to remove credits
  - movie-service treats people missing from `BatchGetPeople` as unknown and skips or cleans up those credits
- [ ] E2 Person photo upload: REST `POST /people/{id}/photo` in movie-service, reusing the artwork upload and storage, then `UpdatePerson` with the served URL as `photoUrl`; frontend progress bar like the artwork upload

## Verification
- **people-service:** `./gradlew test` passes. `grpcurl -plaintext localhost:9090 list` shows `PeopleService`. CRUD works through `grpcurl`.
- **movie-service:** `./gradlew test` passes. In GraphiQL at `/graphiql`, nested movie → credits → person queries work, using a single `BatchGetPeople` call (confirm in the logs).
- **movie-web:** `npm run test` and `npm run check` pass.
- **Full stack:** start people-service, movie-service and movie-web (or the optional `docker compose up`), then in the browser:
  - list, create, update and delete movies and people
  - upload artwork and watch the progress bar
  - search across both
  - check the layout at phone and desktop widths
