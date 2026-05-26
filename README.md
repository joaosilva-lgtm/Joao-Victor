# Interlis E2E Tests

Testes end-to-end automatizados com [Playwright](https://playwright.dev) + TypeScript para o Interlis LIMS.

## Setup

```bash
pnpm install
pnpm exec playwright install --with-deps
cp .env.example .env  # configure suas credenciais
```

## Comandos

```bash
pnpm test                            # Todos os testes
pnpm test:smoke                      # Smoke tests
pnpm test:login                      # Login tests (multi-tenant)
pnpm test nome-do-arquivo            # Arquivo específico
pnpm test --grep "@tag"              # Por tag
pnpm test:headed                     # Com navegador visível
pnpm test:debug                      # Modo debug (step-by-step)
pnpm test:ui                         # Interface interativa
pnpm report                          # Relatório da última execução
CURRENT_TENANT=reunidos pnpm test    # Trocar tenant
```

## Variáveis de Ambiente

```bash
CURRENT_TENANT=rr                    # Tenant padrão (rr, reunidos, rorainopolis)
E2E_USER_PREFIX=e2eTestUser          # Prefixo dos usuários RBAC
E2E_USER_PASSWORD=***                # Senha dos usuários RBAC
NODE_ENV=staging                     # Ambiente (dev, staging, production)
USE_LOCALHOST=false                  # true = http://localhost:3000
CF_ACCESS_CLIENT_ID=***              # Cloudflare Access (CI)
CF_ACCESS_CLIENT_SECRET=***          # Cloudflare Access (CI)
```

> Usuários seguem o padrão `{E2E_USER_PREFIX}{Role}` — ex: `e2eTestUserAdmin`, `e2eTestUserPhysician`.

---

## Organização dos Testes

### Tipos de teste

| Tipo      | Diretório      | Objetivo                                                                                                           |
| --------- | -------------- | ------------------------------------------------------------------------------------------------------------------ |
| **Smoke** | `tests/smoke/` | Testes individuais de uma página/feature. CRUD, validações, permissões. Rápidos e isolados. Usam fixtures de auth. |
| **Login** | `tests/login/` | Testes de autenticação multi-tenant. Rodam sem fixtures pois testam o próprio fluxo de login.                      |
| **Flow**  | `tests/flow/`  | Testes de fluxo completo cruzando várias páginas. Ex: Paciente → Solicitação → Amostra → Laudo.                    |

### Onde colocar cada coisa

| Artefato            | Local                            | Quando usar                                                             |
| ------------------- | -------------------------------- | ----------------------------------------------------------------------- |
| **Fixtures**        | `tests/support/fixtures/`        | Sempre. Setup reutilizável (autenticação, formulários pré-preenchidos). |
| **Setup (auth/CF)** | `tests/support/setup/`           | Configuração de projeto Playwright (auth e Cloudflare).                 |
| **Scripts**         | `tests/support/scripts/`         | Scripts de apoio (ex: run-postman.sh).                                  |
| **Postman/API**     | `tests/support/postman/`         | Coleções e environments Newman.                                         |
| **Helpers globais** | `utils/`                         | Funções usadas em múltiplos arquivos de teste (auth, inputs, filters).  |
| **Helpers locais**  | Mesmo arquivo ou no dir do teste | Funções específicas de um único spec file ou grupo de specs.            |
| **Config**          | `config/`                        | Tenants, rotas, constantes, timeouts.                                   |

### Exemplo de smoke test

```typescript
import { test, expect } from '@/tests/support/fixtures/auth.fixtures';
import { getTenantConfig } from '@/config/tenants';
import { ROUTES } from '@/config/constants';

const { url } = getTenantConfig(ROUTES.PATIENTS);

test.describe('Cadastro de Paciente', () => {
  test('Deve criar paciente com dados válidos', async ({
    authenticatedPageAdmin: page,
  }) => {
    await page.goto(url);
    await page.getByTitle('Novo Paciente').click();

    await page.getByLabel('Nome *').fill('E2E_PACIENTE_12345');
    await page.getByRole('button', { name: 'Salvar' }).click();

    await expect(
      page.getByRole('alert').filter({ hasText: /sucesso/i })
    ).toBeVisible();
  });
});
```

---

## Boas Práticas

### Seletores (em ordem de preferência)

```typescript
// 1. Role-based (melhor — acessível e estável)
page.getByRole('button', { name: 'Salvar' });
page.getByLabel('Nome *');
page.getByRole('heading', { name: 'Dashboard' });

// 2. Texto visível
page.getByText('Sucesso');

// 3. Test ID (quando não há alternativa semântica)
page.getByTestId('patient-table');

// 4. Locator com atributo (último recurso)
page.locator('[data-status="pending"]');

// ❌ Nunca: seletores frágeis por classe/estrutura DOM
page.locator('.btn-primary');
page.locator('div > span > button');
```

### Espera e estabilidade

```typescript
// ✅ Auto-waiting do Playwright (preferir sempre)
await expect(page.getByText('Dashboard')).toBeVisible();

// ✅ Esperar resposta de rede quando necessário
await page.waitForResponse((r) => r.url().includes('/api/patients'));

// ✅ Retry com loop quando a UI é instável (ex: re-render do Next.js)
for (let attempt = 1; attempt <= MAX_RETRIES; attempt++) {
  await element.click();
  try {
    await expect(modal).toBeVisible({ timeout: TIMEOUTS.SHORT });
    break;
  } catch {
    if (attempt === MAX_RETRIES) throw new Error('Modal não abriu');
  }
}

// ⚠️ waitForTimeout: usar apenas como estabilização, nunca como espera principal
await page.waitForTimeout(300); // OK: aguardar animação curta
await page.waitForTimeout(5000); // ❌: substituir por expect/waitFor
```

### Fixtures e autenticação

```typescript
// ✅ Sempre use fixtures para autenticação
import { test } from '@/tests/support/fixtures/auth.fixtures';

test('teste', async ({ authenticatedPageAdmin: page }) => {
  // Já autenticado, cache de cookies automático
});

// ✅ Fixture específica para setup complexo
import { test } from '@/tests/support/fixtures/solicitation.fixtures';

test('adicionar exame', async ({ solicitationFormPage: page }) => {
  // Formulário já preenchido com campos obrigatórios
});
```

### Dados de teste

```typescript
// ✅ Gerar dados únicos para evitar conflitos em paralelo
const suffix = randomInt(10000, 99999);
const name = `E2E_PATIENT_${suffix}`;

// ✅ Restaurar estado ao final — testes devem ser idempotentes
const original = await field.inputValue();
await field.fill(newValue);
await save();
// ... assert ...
await field.fill(original);
await save();
```

### Independência e isolamento

Cada teste deve funcionar sozinho, em qualquer ordem. Nunca dependa do resultado de outro teste. Fixtures garantem contexto isolado automaticamente.

---

## Multi-tenant

A URL é construída automaticamente a partir de `CURRENT_TENANT` e `NODE_ENV`. Use `getTenantConfig()` com as rotas de `config/constants.ts`:

```typescript
const { url } = getTenantConfig(ROUTES.PATIENTS);
await page.goto(url);
```

Para features exclusivas de um tenant, use `tenant-auth.fixtures.ts`:

```typescript
import { sigTest as test } from '@/tests/support/fixtures/tenant-auth.fixtures';

test('Feature SIG', async ({ sigPageAdmin: page }) => {
  // Sempre roda no tenant SIG
});
```

---

## CI/CD

- **Pull Requests**: testes obrigatórios (blocking)
- **Pré-deploy produção**: smoke tests (blocking)
- **Diário**: smoke + login tests às 6h (monitoramento)

Relatórios e traces de falha ficam como artefatos no GitHub Actions.

---

## Troubleshooting

| Problema                       | Solução                                                                                 |
| ------------------------------ | --------------------------------------------------------------------------------------- |
| Timeout nos testes             | Aumentar timeout em `playwright.config.ts` ou verificar se a aplicação está respondendo |
| `E2E_USER_PREFIX não definido` | Configurar `.env` (`cp .env.example .env`)                                              |
| Browsers não instalados        | `pnpm exec playwright install --with-deps`                                              |
| Passa local, falha no CI       | Verificar secrets no GitHub, rodar localmente com `CI=true`                             |
| `net::ERR_NAME_NOT_RESOLVED`   | Tenant inexistente — verificar `config/tenants.ts`                                      |
