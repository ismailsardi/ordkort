# Ordkort

Spaced repetition app for learning Norwegian (Bokmål) vocabulary. Single user for now, no auth.
Later milestones add dictionary lookups (Ordbøkene API) and LLM-generated example sentences.

Specs live in `docs/specs/`. The spec for the current milestone is the source of truth. If the code and the spec disagree, stop and ask.

## Stack

- Backend: .NET 10, ASP.NET Core Web API, EF Core, PostgreSQL
- Frontend: React + TypeScript (Vite)
- Tests: xUnit (plain `Assert`, no FluentAssertions)
- Local DB: Postgres in Docker Compose

## Structure

```
src/
  Ordkort.Core/            Domain: entities, enums, scheduler. No EF, no ASP.NET references.
  Ordkort.Infrastructure/  EF Core DbContext, configurations, migrations
  Ordkort.Api/             Endpoints, DTOs, validation, DI setup
tests/
  Ordkort.Core.Tests/      Unit tests for domain logic (scheduler first)
  Ordkort.Api.Tests/       Integration tests against the API
web/                       React app
docs/specs/                One spec per milestone
```

## Commands

```bash
docker compose up -d db                                  # start Postgres
dotnet build
dotnet test
dotnet run --project src/Ordkort.Api
dotnet ef migrations add <Name> -p src/Ordkort.Infrastructure -s src/Ordkort.Api
cd web && npm install && npm run dev
```

## Conventions

C#
- Nullable reference types on, warnings as errors.
- DTOs are `record` types and live in the Api project. Never return EF entities from endpoints.
- Endpoints stay thin: validate, call a service, map the result. Logic goes in Core.
- Get the current time through `TimeProvider` (injected), never `DateTime.Now` / `DateTimeOffset.UtcNow`. Store times as UTC `DateTimeOffset`.
- Async all the way for I/O. No `.Result` or `.Wait()`.

TypeScript
- `strict` on. No `any`.
- All API calls go through `web/src/api/`. Components don't call `fetch` directly.
- Function components and hooks only.

Tests
- Test names describe behavior: `Good_on_new_card_sets_interval_to_1_day`.
- Domain logic is written test first.

## How to work in this repo

- Plan before editing. For anything bigger than a one-line fix, show a short plan (files to touch, approach) and wait for my OK.
- One slice at a time. Don't start the next task in the spec until I say so.
- For domain logic, write the failing tests first, show them to me, then implement.
- Don't add NuGet or npm packages without asking.
- Never commit or push. I review the diff and write the commit messages myself.
- Run `dotnet test` (and `npm run build` if the frontend changed) before saying a task is done.
- If something in the spec is unclear or seems wrong, ask instead of guessing.
- Keep diffs small. Don't refactor or reformat code outside the task.
