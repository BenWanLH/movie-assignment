# Movie Assignment

A movie catalogue for listing, creating, updating, deleting and searching movies and their cast and creators, with artwork upload.

- **Frontend:** Svelte + TypeScript
- **Backend:** two Kotlin Spring Boot services
- **Frontend ↔ backend:** GraphQL
- **Service ↔ service:** gRPC

> **Status:** work in progress. The commands below describe the target setup and will be confirmed as each part is built.

## Architecture

```
┌──────────────┐   GraphQL (HTTP)   ┌──────────────────┐    gRPC (HTTP/2)    ┌──────────────────┐
│  movie-web   │ ─────────────────▶ │  movie-service   │ ──────────────────▶ │  people-service  │
│  Svelte + TS │                    │  movies, credits │                     │  cast & creators │
│  :5173       │                    │  artwork  :8080  │                     │  :9090           │
└──────────────┘                    └──────────────────┘                     └──────────────────┘
                                     H2 database (movies)                     H2 database (people)

                     both services include protos (the shared .proto contract) as a submodule at proto/
```

- **movie-web** is the browser app. It only ever talks to `movie-service`, through GraphQL.
- **movie-service** serves the GraphQL API and owns movies, credits (who played or made what) and uploaded artwork. Whenever it needs cast or creator data, it calls `people-service` over gRPC.
- **people-service** owns cast and creators and exposes a gRPC API only.
- **[protos](https://github.com/BenWanLH/protos)** holds the `.proto` files that define the gRPC contract. It isn't part of this umbrella repo directly. Instead, both Spring Boot services include it as their own git submodule at `proto/` and generate their Kotlin code from it at build time.
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
├── people-service/
│   └── proto/      ← protos submodule
├── movie-service/
│   └── proto/      ← protos submodule
└── movie-web/
```

## Prerequisites

| Tool | Version | Needed for |
|---|---|---|
| Git | any recent | Cloning with submodules |
| Java (JDK) | 21 | `people-service`, `movie-service` |
| Node.js | 22 LTS | `movie-web` |

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

- [grpcurl](https://github.com/fullstorydev/grpcurl) (`brew install grpcurl`) lets you call the gRPC API of `people-service` directly from the terminal.

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

Start the three apps in this order, each in its own terminal, from the root of this repo.

**1. people-service** (gRPC on port 9090)

```bash
cd people-service
./gradlew bootRun
```

**2. movie-service** (GraphQL on port 8080)

```bash
cd movie-service
./gradlew bootRun
```

To explore the API in the browser, open GraphiQL at <http://localhost:8080/graphiql>.

**3. movie-web** (port 5173)

```bash
cd movie-web
npm install
npm run dev
```

Then open <http://localhost:5173>.

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

- draft the implementation plan in [`PLAN.md`](PLAN.md), including the repo layout, architecture and task breakdown


More detail will be added here as the project progresses.
