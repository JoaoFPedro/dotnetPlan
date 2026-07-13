# Implementation Plan: Week 2 — Day 6 — GET /api/tasks/completed

## Summary

Adicionar a rota `GET /api/tasks/completed` ao TaskFlowApi. A rota retorna apenas as tasks com `IsCompleted == true`, ordenadas por `CreatedAt` decrescente. Esta é a tarefa prática do **Dia 6** do plano de onboarding .NET.

## Scope

- Backend only (TaskFlowApi)
- Files to modify: `Program.cs`, `Services/ITaskService.cs`, `Services/TaskService.cs`
- Files to create: none

## Onboarding Goal

Praticar:
1. Adicionar uma rota Minimal API (`app.MapGet`)
2. Chamar um método do serviço injetado por DI
3. Retornar uma lista de DTOs (não entidades diretas)

## JS Parallel

```javascript
// Express equivalent
app.get('/api/tasks/completed', async (req, res) => {
  const tasks = await taskService.getCompleted();
  res.json(tasks);
});
```

```csharp
// ASP.NET Core Minimal API equivalent
app.MapGet("/api/tasks/completed", async (ITaskService taskService) =>
{
    var tasks = await taskService.GetCompletedAsync();
    return Results.Ok(tasks);
});
```

## Estado Atual vs Estado Alvo

### Estado atual
- TaskFlowApi scaffolded com `dotnet new webapi --use-minimal-apis`
- `TaskItem.cs` e `AppDbContext.cs` criados
- `ITaskService` e `TaskService` com métodos básicos

### Estado alvo
- Novo método `GetCompletedAsync()` em `ITaskService` e `TaskService`
- Nova rota `GET /api/tasks/completed` em `Program.cs`
- Retorno: lista de `TaskResponseDto`

## Implementation Checklist

- [ ] Adicionar `GetCompletedAsync()` na interface `ITaskService`
- [ ] Implementar `GetCompletedAsync()` em `TaskService` usando LINQ `.Where(t => t.IsCompleted)`
- [ ] Registrar a rota `app.MapGet("/api/tasks/completed", ...)` em `Program.cs`
- [ ] Testar via Swagger UI: `GET /api/tasks/completed`
- [ ] Verificar que retorna apenas tasks com `IsCompleted == true`

## Code

### ITaskService.cs

```csharp
public interface ITaskService
{
    Task<List<TaskResponseDto>> GetAllAsync();
    Task<List<TaskResponseDto>> GetCompletedAsync(); // NEW
    Task<TaskResponseDto?> GetByIdAsync(int id);
    Task<TaskResponseDto> CreateAsync(TaskCreateDto dto);
    Task<bool> DeleteAsync(int id);
}
```

### TaskService.cs

```csharp
public async Task<List<TaskResponseDto>> GetCompletedAsync()
{
    return await _db.Tasks
        .Where(t => t.IsCompleted)
        .OrderByDescending(t => t.CreatedAt)
        .Select(t => new TaskResponseDto(t.Id, t.Title, t.IsCompleted, t.CreatedAt))
        .ToListAsync();
}
```

### Program.cs (new route)

```csharp
app.MapGet("/api/tasks/completed", async (ITaskService taskService) =>
{
    var tasks = await taskService.GetCompletedAsync();
    return Results.Ok(tasks);
})
.WithName("GetCompletedTasks")
.WithTags("Tasks");
```

## Business Rules Involved

| Rule ID    | Rule                                          |
| ---------- | --------------------------------------------- |
| TASK-003   | `IsCompleted` defaults to `false` on creation |
| FILTER-002 | This endpoint returns only `IsCompleted == true` |

## Verification

1. `dotnet run`
2. Open Swagger UI: `https://localhost:xxxx/swagger`
3. Execute `GET /api/tasks/completed`
4. Response should be `200 []` if no completed tasks exist
5. Create a task, complete it, then call this endpoint again — should return that task


## Scope

- Backend only
- New files: 26 files (18 implementation + 8 spec files)
- Modified files: 0

## Estado Atual vs Estado Alvo

Este documento descreve o estado alvo (target architecture), nao o estado atual do repositorio.

### Estado atual (hoje)

- Existe o use case de autenticacao e seu unit test.
- Existem contratos de protocolo (HashComparer, Encrypter, LoadAccountByEmailRepository).
- Nao existem ainda as implementacoes reais de Infra/Main citadas abaixo:
  - BCrypterAdapter
  - JwtAdapter
  - PrismaHelper
  - AccountPrismaRepository
  - Factories e server bootstrap da camada main

### Estado alvo (apos implementar este plano)

- Fluxo completo de login com bcrypt + Prisma + JWT.
- Repositorio real de conta via Prisma.
- Composicao da aplicacao via factories e bootstrap do server.
- Testes unitarios e de integracao cobrindo comportamento principal.

## Checklist Minima de Testabilidade

### O que ja da para testar agora

- Unit test de DbAuthentication com stubs (sem banco real).

### O que falta para testar fluxo real com Prisma

1. Implementar classes de Infra: BCrypterAdapter, JwtAdapter, PrismaHelper, AccountPrismaRepository.
2. Implementar composicao da camada main (factory + server).
3. Criar model Account no schema Prisma.
4. Gerar client Prisma.
5. Aplicar migration no banco de desenvolvimento/teste.
6. Configurar DATABASE_URL e JWT_SECRET.
7. Criar e rodar teste de integracao do AccountPrismaRepository.
8. Criar e rodar teste de rota/controller do login.

### Criterio objetivo para dizer "esta testavel"

- Unit tests de autenticacao passando.
- Integration test de repositorio Prisma passando com banco de teste.
- Teste de endpoint POST /api/login cobrindo:
  - 200 com accessToken em credenciais validas
  - 401 em credenciais invalidas
  - 400 em payload invalido

## Business Rules Involved

| Rule ID  | Rule                                                                        |
| -------- | --------------------------------------------------------------------------- |
| AUTH-001 | Senha nunca armazenada em texto plano, apenas hash bcrypt (saltRounds = 12) |
| AUTH-002 | Mesmo erro para email nao encontrado e senha incorreta (previne enumeracao) |
| AUTH-003 | O campo password (hash) nunca aparece em resposta de API                    |
| AUTH-004 | JWT contem apenas `{ id: accountId }`                                       |
| AUTH-005 | `JWT_SECRET` sempre via variavel de ambiente                                |
| AUTH-006 | `DATABASE_URL` obrigatoria para Prisma em runtime e testes                  |

## Architecture Diagram

```mermaid
flowchart TD
    App["Mobile App"]
    Route["Express Route\\nlogin-routes.ts"]
    Adapter["adaptRoute"]
    Controller["LoginController"]
    Validation["ValidationComposite"]
    UC["DbAuthentication"]
    RepoIface["LoadAccountByEmailRepository"]
    Repo["AccountPrismaRepository"]
    Prisma["Prisma Client"]
    DB[("PostgreSQL\\naccounts")]
    Bcrypt["BCrypterAdapter"]
    JWT["JwtAdapter"]

    App -->|POST /api/login| Route
    Route --> Adapter --> Controller
    Controller --> Validation
    Controller --> UC
    UC --> RepoIface -.->|implements| Repo
    Repo --> Prisma --> DB
    UC --> Bcrypt
    UC --> JWT
```

## Implementation Order

> Segue TDD (Red -> Green). Spec files antes das implementacoes.
> Camada main sem testes (codigo de composicao).

### Phase 0 - Domain Contracts

```text
0. AccountModel                      - domain/models/account/account.ts
1. Authentication interface          - domain/use-cases/authentication/authentication.ts
2. HashComparer interface            - data/protocols/criptography/hash-comparer.ts
3. Encrypter interface               - data/protocols/criptography/encrypter.ts
4. LoadAccountByEmailRepository      - data/protocols/db/account/load-account-by-email-repository.ts
```

### Phase 1 - Data Use Case (TDD)

```text
5.          db-authentication-protocols.ts  - barrel re-exports
6. [RED]    db-authentication.spec.ts       - testes falham
7. [GREEN]  db-authentication.ts            - implementar ate passar
```

### Phase 2 - Presentation (TDD)

```text
8.          presentation shared (protocols, helpers, errors)
9.          login-controller-protocols.ts
10. [RED]   login-controller.spec.ts
11. [GREEN] login-controller.ts
```

### Phase 3 - Validation (TDD)

```text
12. [RED] required-fields.spec.ts
13. [GREEN] required-fields.ts
14. [RED] email-validation.spec.ts
15. [GREEN] email-validation.ts
16. [RED] validation-composite.spec.ts
17. [GREEN] validation-composite.ts
```

### Phase 4 - Infra (TDD)

```text
18.         prisma-helper.ts                       - singleton PrismaClient
19. [RED]   account-prisma-repository.spec.ts      - integration test em Postgres de teste
20. [GREEN] account-prisma-repository.ts
21. [RED]   bcrypter-adapter.spec.ts
22. [GREEN] bcrypter-adapter.ts
23. [RED]   jwt-adapter.spec.ts
24. [GREEN] jwt-adapter.ts
25.         email-validator-adapter.ts
```

### Phase 5 - Main (composicao)

```text
26. env.ts
27. app.ts + middlewares.ts + routes.ts
28. express-route-adapter.ts
29. db-authentication-factory.ts
30. login-validation-factory.ts
31. login-controller-factory.ts
32. login-routes.ts
33. server.ts
```

---

## Prisma-Specific Changes

### 1) Prisma schema

Adicionar modelo em `prisma/schema.prisma`:

```prisma
model Account {
  id       String @id @default(uuid())
  name     String
  email    String @unique
  password String

  @@map("accounts")
}
```

### 2) Prisma helper

Arquivo: `src/infra/db/prisma/helpers/prisma-helper.ts`

```typescript
import { PrismaClient } from "@/generated/prisma";

export const PrismaHelper = {
  client: new PrismaClient(),

  async connect(): Promise<void> {
    await this.client.$connect();
  },

  async disconnect(): Promise<void> {
    await this.client.$disconnect();
  },
};
```

### 3) Account repository (Prisma)

Arquivo: `src/infra/db/prisma/account/account-prisma-repository.ts`

```typescript
import { LoadAccountByEmailRepository } from "@/data/protocols/db/account/load-account-by-email-repository";
import { AccountModel } from "@/domain/models/account/account";
import { PrismaHelper } from "../helpers/prisma-helper";

export class AccountPrismaRepository implements LoadAccountByEmailRepository {
  async loadByEmail(email: string): Promise<AccountModel | null> {
    const account = await PrismaHelper.client.account.findUnique({
      where: { email },
    });

    return account;
  }
}
```

### 4) Integration test repository

Arquivo: `src/infra/db/prisma/account/account-prisma-repository.spec.ts`

```typescript
import { PrismaHelper } from "../helpers/prisma-helper";
import { AccountPrismaRepository } from "./account-prisma-repository";

describe("AccountPrismaRepository", () => {
  beforeAll(async () => {
    await PrismaHelper.connect();
  });

  afterAll(async () => {
    await PrismaHelper.disconnect();
  });

  afterEach(async () => {
    await PrismaHelper.client.account.deleteMany();
  });

  test("should return an account on success", async () => {
    await PrismaHelper.client.account.create({
      data: {
        name: "any_name",
        email: "any_email@mail.com",
        password: "any_password",
      },
    });

    const sut = new AccountPrismaRepository();
    const account = await sut.loadByEmail("any_email@mail.com");

    expect(account).toBeTruthy();
    expect(account?.id).toBeTruthy();
    expect(account?.email).toBe("any_email@mail.com");
  });

  test("should return null if loadByEmail fails", async () => {
    const sut = new AccountPrismaRepository();
    const account = await sut.loadByEmail("any_email@mail.com");

    expect(account).toBeNull();
  });
});
```

### 5) Factory update

Arquivo: `src/main/factories/usecases/authentication/db-authentication-factory.ts`

```typescript
import { DbAuthentication } from "@/data/use-cases/authentication/db-authentication";
import { BCrypterAdapter } from "@/infra/criptography/bcrypter-adapter";
import { JwtAdapter } from "@/infra/criptography/jwt-adapter";
import { AccountPrismaRepository } from "@/infra/db/prisma/account/account-prisma-repository";
import env from "@/main/config/env";

export const makeDbAuthentication = (): DbAuthentication => {
  const accountPrismaRepository = new AccountPrismaRepository();
  const bcrypterAdapter = new BCrypterAdapter(12);
  const jwtAdapter = new JwtAdapter(env.jwtSecret);

  return new DbAuthentication(
    accountPrismaRepository,
    bcrypterAdapter,
    jwtAdapter,
  );
};
```

### 6) Server bootstrap update

Arquivo: `src/main/server.ts`

```typescript
import { PrismaHelper } from "@/infra/db/prisma/helpers/prisma-helper";
import env from "@/main/config/env";

PrismaHelper.connect()
  .then(async () => {
    const { setupApp } = await import("@/main/config/app");
    const app = await setupApp();

    app.listen(env.port, () => {
      console.log(`Server running at http://localhost:${env.port}`);
    });
  })
  .catch(console.error);
```

---

## Env Variables

```env
DATABASE_URL=postgresql://user:pass@localhost:5432/runner_ai
JWT_SECRET=your_secret
PORT=5050
```

---

## Commands (Prisma)

```bash
npx prisma generate
npx prisma migrate dev --name create_accounts
npx prisma migrate deploy
```

---

## Dependencies to Install

```bash
npm install express bcrypt jsonwebtoken validator @prisma/client
npm install --save-dev prisma @types/express @types/bcrypt @types/jsonwebtoken @types/validator jest ts-jest
```

---

## Test Strategy Notes

- Unit tests de `DbAuthentication` nao mudam (continuam com stubs de protocolo).
- Integration tests de repositorio passam a usar Prisma + banco SQL de teste.
- Recomendada limpeza por `deleteMany()` entre testes.
- Em CI, subir PostgreSQL de teste e rodar `prisma migrate deploy` antes dos testes.

---

## Estimated Effort

| Task                                   | Effort |
| -------------------------------------- | ------ |
| Phase 0 - Domain contracts             | 0.5h   |
| Phase 1 - DbAuthentication (RED+GREEN) | 1.5h   |
| Phase 2 - LoginController (RED+GREEN)  | 1.5h   |
| Phase 3 - Validators (RED+GREEN)       | 1.5h   |
| Phase 4 - Infra Prisma + adapters      | 2h     |
| Phase 5 - Main composition             | 1h     |
| Total                                  | ~8h    |

## Next Steps After Implementation

1. Implementar `POST /api/signup` com `Hasher`.
2. Implementar middleware de autenticacao com `Decrypter`.
3. Aplicar middleware em rotas protegidas.
4. Configurar `expiresIn` no JWT e refresh token (V2).
5. Adicionar migrations Prisma no pipeline de deploy.
