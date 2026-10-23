# Instruções do Copilot — Autenticacao-Web

## Sobre o projeto
Produto de autenticação Web em canais. SPA em React 19 + TypeScript, build com Vite + SWC,
servida por Nginx (porta 8080) em container Docker, com deploy em GCP via Terraform (GitLab CI).
Os textos da interface são em português.

## Comandos
- Gerenciador de pacotes: **pnpm**.
- `pnpm install`: instala as dependências.
- `pnpm local`: sobe o Vite e o mock de API (json-server na porta 3001, dados em `mocks_services/api.json`).
- `pnpm dev`: sobe só o Vite.
- `pnpm lint`: roda o ESLint.
- `pnpm test`: roda o Vitest.
- `pnpm test-ui`: abre a interface visual do Vitest.
- `pnpm coverage`: gera o relatório de cobertura (v8).
- `pnpm build`: roda `tsc -b` e gera o build de produção em `dist/`.

## Stack
- **UI:** MUI v7 (`@mui/material`, `@mui/icons-material`) + Emotion.
- **Estado assíncrono:** TanStack React Query v5 (`QueryClient` configurado em `main.tsx` com `retry: 3`).
- **HTTP:** axios.
- **Validação:** Zod.
- **Rotas:** react-router-dom v7 (`createBrowserRouter` + `RouterProvider`).
- **Autenticação:** keycloak-js.
- **Erros:** react-error-boundary + `QueryErrorResetBoundary` do React Query.
- **Testes:** Vitest + React Testing Library + jest-dom.

## Estrutura de `src/`
```
src/
├── assets/
├── components/
│   ├── ErrorBoundary/   index.tsx, style.ts
│   └── Loader/          index.tsx
├── config/
│   └── env.ts           variáveis de ambiente validadas com Zod
├── hooks/
│   └── auth/            keycloak.ts, types.tsx, useAuth.tsx
├── pages/
│   └── Home/            Home.tsx
├── routes/
│   ├── index.tsx        definição das rotas (router)
│   └── protectedRoute.tsx
├── styles/
│   ├── Colors.ts        AppColors (paleta)
│   ├── Parameters.ts    AppParameters (espaçamentos, fontes, sombras)
│   └── Theme.ts         tema do MUI (createTheme)
├── App.tsx, App.css, App.test.tsx
├── index.css
├── main.tsx
├── setupTests.ts
└── vite-env.d.ts
```

Path aliases em uso: `@src/*`, `@components/*`, `@hooks/*`, `@pages/*`, `@assets/*`, `@routes/*`.

## Como o projeto está organizado

### Inicialização (`main.tsx`)
Monta, nesta ordem: `StrictMode` → `CssBaseline` (MUI, com `enableColorScheme`) →
`QueryErrorResetBoundary` → `ErrorBoundary` (com `ErrorBoundaryFallback`) → `QueryClientProvider`,
`AuthProvider` e `RouterProvider`. Importa o `index.css` global.

### Variáveis de ambiente (`src/config/env.ts`)
Em runtime, o `entrypoint.sh` injeta as variáveis em `window.__ENV__`.
O arquivo `env.ts` valida esse objeto com Zod e exporta `env`:
- `KEYCLOAK_URL` (URL válida)
- `KEYCLOAK_CLIENT_ID`
- `KEYCLOAK_REALM`
- `ENVIRONMENT`: `development` | `hml` | `prd` (padrão `development`)

O código lê configuração importando `env` de `src/config/env.ts`.

### Autenticação (`src/hooks/auth/`)
- `keycloak.ts`: cria a instância do Keycloak a partir de `env`.
- `types.tsx`: tipos `AuthContextType` e `UserInfos`.
- `useAuth.tsx`: exporta `AuthContext`, `AuthProvider` e o hook `useAuth`, que expõe
  `isAuthenticated`, `hasRole` e `logout`, entre outros.
- Roles e grupos de acesso são configurados no Terraform (`terraform/keyclock-client.tf`)
  e na variável `TF_VAR_access_group` do `.gitlab-ci.yml`.

### Rotas (`src/routes/`)
- `index.tsx` define as rotas como `RouteObject[]` e exporta `router`.
- Páginas são carregadas com `lazy(() => import(...))`.
- A rota raiz usa `ProtectedRoute` como elemento e `ErrorBoundaryFallback` como `errorElement`.
  `App` é o layout e as páginas são rotas filhas.
- `ProtectedRoute` aceita `requiredRole` opcional: sem autenticação, mostra mensagem de
  redirecionamento; sem a role exigida, redireciona para `/403`; caso contrário, renderiza `<Outlet />`.

### Estilos (`src/styles/`)
- Cores ficam em `AppColors` (`Colors.ts`), divididas em `base` e `decorative`.
- Espaçamentos (em `vw`/`vh`), tamanhos de fonte (em `rem`) e sombras ficam em `AppParameters` (`Parameters.ts`).
- O tema do MUI (`Theme.ts`) é montado com `createTheme` a partir de `AppColors` e `AppParameters`,
  e aplicado com `ThemeProvider` em `App.tsx`.
- Existe CSS global em `index.css` (variáveis em `:root`) e o arquivo `App.css`.

### Componentes (`src/components/`)
- Cada componente fica em uma pasta com o nome dele em PascalCase, com `index.tsx`
  e, quando tem estilos próprios, `style.ts` (ex.: `ErrorBoundary/style.ts` exporta `Wrapper` e `Msg`).
- Componentes usam named export e arrow function (ex.: `export const Loader = () => ...`).
- Usam componentes do MUI (`Box`, `Button`, `Typography`, `CircularProgress`, `IconButton`)
  e ícones de `@mui/icons-material`.

### Páginas (`src/pages/`)
Cada página fica em uma pasta com o nome dela e um arquivo de mesmo nome (ex.: `pages/Home/Home.tsx`).

## Testes
- O projeto usa apenas testes unitários, com Vitest + React Testing Library em ambiente jsdom.
- Arquivo de teste fica ao lado do arquivo testado, com sufixo `.test.tsx` (ex.: `App.test.tsx`).
- `setupTests.ts` importa `@testing-library/jest-dom`.
- Os testes importam `describe`, `it`, `expect` e `vi` de `vitest`, e `render` e `screen`
  de `@testing-library/react`.
- Dependências como `useAuth` são mockadas com `vi.mock('@hooks/auth/useAuth', ...)`.
- Elementos são buscados por acessibilidade (ex.: `screen.getByLabelText('sair da aplicação')`).

## Infraestrutura
- `terraform/`: `cloudrun.tf`, `keyclock-client.tf`, `loadbalancer.tf`, `providers.tf`,
  `remote-state.tf`, `sa.tf`, `variables.tf`.
- `.gitlab-ci.yml`: usa os templates `terraform/tf-react.yml` e `backstage/build-techdocs.yml`
  do projeto `devops-templates`, com jobs de deploy para `hml` e `prd`.

## Formatação
ESLint + Prettier. O lint-staged roda `eslint --fix` e `prettier --write` em arquivos `.ts`/`.tsx`
e `prettier --write` em `.json`, `.css`, `.scss` e `.md`.

# Regras do time

Siga estas regras em todo código novo ou alterado. Quando o código existente não seguir uma regra,
não o reescreva por conta própria: aplique a regra apenas ao que estiver criando ou modificando.
Se uma tarefa exigir quebrar uma regra, pare e explique o motivo antes de continuar.

## 1. Imports
- Para importar algo de outra pasta, use sempre path alias (`@components/...`, `@hooks/...`,
  `@config/...`, `@routes/...`, `@pages/...`, `@assets/...`).
- Caminho relativo só para arquivos da mesma pasta (ex.: `./style`).
- Componentes do MUI: `import { Box, Button } from '@mui/material'`.
- Ordem dos imports: bibliotecas externas, depois aliases internos, depois arquivos da mesma pasta.

## 2. Estilização
- Use apenas MUI + Emotion. Não use Tailwind nem crie arquivos `.css` novos.
- Estilo pontual: prop `sx`. Estilo reutilizável: `styled` no arquivo `style.ts` do componente.
- Cores, espaçamentos e fontes vêm sempre do `theme` (baseado em `AppColors` e `AppParameters`).
  Não use valores fixos como `#1e5bc6` ou `16px` direto no componente.

## 3. Componentes
- Cada componente fica em uma pasta em PascalCase com `index.tsx`, `style.ts` (se tiver estilos)
  e o teste `NomeDoComponente.test.tsx`.
- Use named export (`export const Loader = () => ...`).
- Exceção: páginas carregadas com `lazy()` em `src/routes/index.tsx` usam `export default`.
- Props tipadas com `interface NomeDoComponenteProps`.
- Elementos interativos precisam ser acessíveis: use componentes semânticos do MUI e
  `aria-label` em botões sem texto (como já é feito em `App.tsx`).

## 4. Nomes
- Componentes, tipos e interfaces: PascalCase (`Loader`, `UserInfos`).
- Funções e variáveis: camelCase (`handleSignout`, `isLoading`).
- Hooks começam com `use` (`useAuth`). Handlers de evento começam com `handle` (`handleClick`).
- Booleanos começam com `is`, `has` ou `should` (`isAuthenticated`, `hasRole`).
- Variáveis de ambiente e constantes globais: UPPER_SNAKE_CASE (`KEYCLOAK_URL`).
- Arquivo sem JSX usa extensão `.ts`; arquivo com JSX usa `.tsx`.

## 5. Chamadas de API
- `src/services/api.ts`: única instância do axios, com `baseURL` vinda de `env` e interceptor
  que adiciona o token do Keycloak.
- `src/services/<dominio>.ts`: funções que chamam a API e validam a resposta com Zod.
- `src/hooks/<dominio>/use<Algo>.ts`: hooks do React Query que usam essas funções.
- Query keys centralizadas por domínio (ex.: `userKeys.all`, `userKeys.detail(id)`).
- Componentes nunca chamam o axios diretamente.

## 6. Testes
- Escreva apenas testes unitários com Vitest + React Testing Library.
  Não crie testes end-to-end (e2e) nem sugira ferramentas como Playwright ou Cypress.
- Todo hook, service e componente com lógica novo ou alterado precisa de teste.
- Mocke a API (`axios-mock-adapter`) e hooks externos (`vi.mock`); nunca chame APIs reais.
- Busque elementos por acessibilidade (`getByRole`, `getByLabelText`), não por classe CSS.
- Simule interações com `@testing-library/user-event`.
- Use `@faker-js/faker` para gerar dados de teste.

## 7. TypeScript e lint
- Não use `any`. Use `unknown` com verificação de tipo ou tipos inferidos do Zod (`z.infer`).
- Declare todas as dependências de `useEffect`, `useMemo` e `useCallback`.
- Não deixe variáveis, parâmetros ou imports sem uso.
- Não use `// @ts-ignore` nem `eslint-disable`. Se for inevitável, use `// @ts-expect-error`
  com um comentário explicando o motivo.

## 8. Formulários
- Use React Hook Form + Zod com `zodResolver`.
- O schema do formulário fica em `schema.ts`, na pasta do componente do formulário.
- O tipo dos dados vem do schema: `type FormData = z.infer<typeof schema>`.
- Mensagens de validação em português, definidas no schema.

## 9. Tratamento de erros
- Erros de renderização são capturados pelo `ErrorBoundary` (já configurado em `main.tsx` e nas rotas).
- Erros de API são tratados no componente pelo estado de erro do React Query (`isError`, `error`).
- Mensagens para o usuário são em português, claras e sem detalhes técnicos
  (sem stack trace, status HTTP ou mensagem crua do servidor).
- Para avisos e confirmações, use componentes do MUI (`Snackbar`, `Alert`).
- Resposta 401 dispara novo login pelo Keycloak; falta de permissão redireciona para `/403`.

## 10. Segurança
- Não salve tokens nem dados do usuário em `localStorage`, `sessionStorage` ou cookies no código
  da aplicação. O gerenciamento do token é responsabilidade do `keycloak-js`.
- Não registre tokens, senhas ou dados pessoais em `console.log` ou em mensagens de erro.
- Não use `dangerouslySetInnerHTML`.
- Proteja rotas com `ProtectedRoute`, usando `requiredRole` quando a rota exigir uma role.
- Esconder elementos na tela não substitui a validação de permissão, que é feita pelo backend.

## 11. Git
- Branches: `feat/<descricao>`, `fix/<descricao>`, `chore/<descricao>`, `docs/<descricao>`.
- Mensagens de commit no padrão Conventional Commits, em português: `feat: adiciona tela de login`.

## 12. Verificação antes de finalizar
Antes de dizer que uma tarefa está concluída, rode os comandos abaixo:
1. `pnpm lint`
2. `pnpm typecheck`
3. `pnpm test --run`

Se algum falhar, corrija e rode novamente. Nunca declare uma tarefa concluída com falhas.

## 13. O que não fazer sem pedir aprovação
- Instalar, remover ou atualizar dependências.
- Alterar `vite.config.ts`, `vitest.config.ts`, `tsconfig*.json`, `eslint.config.js`,
  `.gitlab-ci.yml`, `Dockerfile`, `nginx.conf`, `entrypoint.sh`, `.husky/` ou a pasta `terraform/`.
- Ler, copiar ou commitar arquivos `.env*`.
- Desligar regras de lint, pular testes ou reduzir cobertura para fazer algo passar.
