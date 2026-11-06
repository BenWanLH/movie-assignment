# Movie Assignment

A movie catalogue for listing, creating, updating, deleting and searching movies and their cast and creators, with artwork upload.

- **Frontend:** Svelte + TypeScript
- **Backend:** two Kotlin Spring Boot services
- **Frontend ↔ backend:** GraphQL, through an Apollo Router that federates both services into one API
- **Service ↔ service:** gRPC

## Architecture

```
┌──────────────┐  GraphQL   ┌───────────────┐
│  movie-web   │ ─────────▶ │ Apollo Router │
│  Svelte + TS │            │ Docker  :4000 │
│  :5173       │            └───┬───────┬───┘
└──────┬───────┘     GraphQL    │       │    GraphQL
       │          (subgraph)    ▼       ▼   (subgraph)
       │        ┌──────────────────┐  ┌──────────────────┐
       │        │  movie-service   │  │  people-service  │
       └──────▶ │  movies, credits │  │  cast & creators │
  REST artwork  │  artwork         │  │                  │
  upload        │  GraphQL :8080   │  │  GraphQL :8081   │
                │  gRPC    :9091   │  │  gRPC    :9090   │
                └────────┬─────────┘  └────────┬─────────┘
                         │  GetPerson ──────▶  │
                         │  ◀── DeleteCreditsByPerson
                         └───────── gRPC ──────┘
                 H2 (movies, credits)    H2 (people)
```

- **movie-web** is the browser app. All data goes through the router as GraphQL. Artwork uploads go straight to `movie-service` over REST.
- **Apollo Router** composes the two services' GraphQL schemas (subgraphs) into one supergraph and splits each query between them. For `movie { credits { person { name } } }`, it gets the movie and credits from `movie-service`, then fetches all the people from `people-service` in one batched call.
- **movie-service** owns movies, credits (who played or made what) and uploaded artwork. A credit only stores the person's id. It calls `people-service` over gRPC (`GetPerson`) to check the person exists and has the role before adding a credit.
- **people-service** owns cast and creators. When a person is deleted, it calls `movie-service` over gRPC (`DeleteCreditsByPerson`) to remove their credits, inside the same transaction, so a failed call rolls the delete back.
- **[protos](https://github.com/BenWanLH/protos)** holds the `.proto` files for the gRPC contracts (`people/v1`, `movies/v1`). Both services include it as a git submodule at `protos/` and generate their Kotlin code from it at build time.
- Each service has its own embedded H2 database, so no database server needs to be installed.

## Repositories

This repo ties the three app repos together as git submodules:

| Folder | Repo | Description |
|---|---|---|
| `people-service/` | [BenWanLH/people-service](https://github.com/BenWanLH/people-service) | Cast and creators service (gRPC) |
| `movie-service/` | [BenWanLH/movie-service](https://github.com/BenWanLH/movie-service) | Movies service (GraphQL API + gRPC client) |
| `movie-web/` | [BenWanLH/movie-web](https://github.com/BenWanLH/movie-web) | Frontend |

The shared gRPC contract, [BenWanLH/protos](https://github.com/BenWanLH/protos), is a nested submodule:

```
movie-assignment/
├── apollo-router/  ← router config and composed supergraph (part of this repo)
├── people-service/
│   └── protos/     ← protos submodule
├── movie-service/
│   └── protos/     ← protos submodule
└── movie-web/
```

## Prerequisites

| Tool | Version | Needed for |
|---|---|---|
| Git | any recent | Cloning with submodules |
| Java (JDK) | 21 | `people-service`, `movie-service` |
| Node.js | 22 LTS | `movie-web` |
| Docker + Compose | any recent | Apollo Router |

Any Docker runtime works. A free option on macOS is [Colima](https://github.com/abiosoft/colima): `brew install colima docker docker-compose && colima start`. Colima only shares your home folder with containers, so keep this repo somewhere under `~`.

You don't need to install Gradle. Each service ships with the Gradle wrapper (`./gradlew`), which downloads the right Gradle version on first use.

### Install with asdf (recommended)

Each repo pins its versions in a `.tool-versions` file. If you use [asdf](https://asdf-vm.com):

```bash
asdf plugin add java
asdf plugin add nodejs

cd people-service && asdf install && cd ..
cd movie-service  && asdf install && cd ..
cd movie-web      && asdf install && cd ..
```

asdf then switches to the pinned versions automatically whenever you're inside these folders.

### Install without asdf

Install JDK 21 and Node.js 22 any way you like. For example, on macOS with Homebrew:

```bash
brew install openjdk@21 node@22
```

Check the versions:

```bash
java -version   # should report 21
node -v         # should report v22.x
```

### Optional tools

- [grpcurl](https://github.com/fullstorydev/grpcurl) (`brew install grpcurl`) lets you call the services' gRPC APIs directly from the terminal.
- [Rover](https://www.apollographql.com/docs/rover/) is only needed to recompose the supergraph after a schema change (see [Apollo Router](#apollo-router)). It runs through `npx`, so there's nothing to install.

## Getting the code

Clone with `--recurse-submodules` so that the three app repos are pulled in, along with the `protos` submodule nested inside each service:

```bash
git clone --recurse-submodules https://github.com/BenWanLH/movie-assignment.git
cd movie-assignment
```

If you've already cloned without that flag, the submodule folders will be empty. Fill them with:

```bash
git submodule update --init --recursive
```

> GitHub's **Download ZIP** button does not include submodules. Please use `git clone` instead.

## Running the project

Start each part in its own terminal, from the root of this repo.

**1. people-service** (GraphQL on 8081, gRPC on 9090)

```bash
cd people-service
./gradlew bootRun
```

**2. movie-service** (GraphQL on 8080, gRPC on 9091)

```bash
cd movie-service
./gradlew bootRun
```

The two services call each other over gRPC, but connections are made lazily, so they can start in either order.

**3. Apollo Router** (port 4000)

```bash
cd apollo-router
docker compose up -d
curl localhost:8088/health   # {"status":"UP"}
```

To explore the combined API in the browser, open <http://localhost:4000/graphql>. Each service also has its own GraphiQL (<http://localhost:8080/graphiql>, <http://localhost:8081/graphiql>), showing only that service's part of the schema.

**4. movie-web** (port 5173)

```bash
cd movie-web
npm install
npm run dev
```

Then open <http://localhost:5173>.

## Apollo Router

`apollo-router/` holds everything the router needs:

| File | Used by | Purpose |
|---|---|---|
| `movies.graphql`, `people.graphql` | Rover | Each service's subgraph schema, exported from the running service |
| `supergraph.yaml` | Rover | Lists the subgraphs, where Rover reads their schemas and where the router sends queries |
| `supergraph.graphql` | router | The composed supergraph: both schemas plus routing information |
| `router.yaml` | router | Router settings: listen on `0.0.0.0:4000` at `/graphql`, health check on `8088` |
| `docker-compose.yaml` | Docker | Runs `ghcr.io/apollographql/router`, mounting the two router files |

Inside the container, `localhost` is the container itself, so `supergraph.yaml` reaches the services at `host.docker.internal`. `docker-compose.yaml` maps that name to your machine with `host-gateway`, which works with Colima, Docker Desktop and Linux.

`supergraph.graphql` is committed, so running the router doesn't need Rover.

### After changing a GraphQL schema

The router only knows the schema in `supergraph.graphql`, so it rejects new fields until you recompose. With both services running:

```bash
cd apollo-router
npx @apollo/rover subgraph introspect http://localhost:8080/graphql > movies.graphql
npx @apollo/rover subgraph introspect http://localhost:8081/graphql > people.graphql
APOLLO_ELV2_LICENSE=accept npx @apollo/rover supergraph compose --config supergraph.yaml > supergraph.graphql
docker compose restart router
```

Then run `npm run codegen` in `movie-web` to refresh the frontend types. `subgraph introspect` reads each service's `_service { sdl }` field, which keeps federation directives such as `@key`. The `APOLLO_ELV2_LICENSE` variable accepts the Elastic License of Apollo's composition plugin without an interactive prompt.

## Running the tests

```bash
cd people-service && ./gradlew test && cd ..
cd movie-service  && ./gradlew test && cd ..
cd movie-web      && npm install && npm run test && cd ..
```

## Updating the submodules

Each submodule is pinned to a specific commit. To pull the latest commit of every repo:

```bash
git submodule update --remote --recursive
```

## How AI tools were used

This project was built with help from [Claude Code](https://claude.com/claude-code). I used it to:

- draft the implementation plan in [`PLAN.md`](PLAN.md) and task breakdown.
- queried it on syntaxes 
- queried it on the technologies, GRPC/Graphql/Svelte
- relied on it on some of the unit tests


More detail will be added here as the project progresses.
