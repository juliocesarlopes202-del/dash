# CENA FIT — Gestão de Alunos

Sistema web administrativo da academia **CENA FIT — Performance Studio**: cadastro de
membros, tabela de alunos com pesquisa/filtros, calendário de vencimentos, controle de
pagamentos (previsto × recebido) e dashboard com indicadores reais do banco.

100% em português (pt-BR), com autenticação no servidor, banco PostgreSQL persistente e
configuração pronta para publicação na Vercel.

---

## 1. O que foi implementado

### Telas

| Rota | Conteúdo |
| --- | --- |
| `/login` | Acesso restrito. Autenticação real no servidor, sem credenciais no frontend. |
| `/dashboard` | Indicadores reais: total de alunos, previsto no mês, recebido no mês, pendentes, em atraso, alunos por plano/horário, próximos vencimentos e vencimentos em atraso. Estado "zero" com orientação quando não há alunos. |
| `/alunos` | Tabela com pesquisa (nome/telefone), filtros (plano, horário, vencimento), ordenação, paginação, cadastro, edição, exclusão com confirmação e exibição de arquivados. |
| `/calendario` | Calendário mensal com navegação, destaque de vencimentos, clique no dia listando os alunos do dia e o total previsto/recebido, marcação de atraso. Correto para meses de 28/29/30/31 dias. |
| `/pagamentos` | Resumo do mês (previsto, recebido, pendente, atrasado), registro de pagamento, histórico, estorno e arquivamento. |

### Regras de negócio

- **Previsto × recebido** são sempre valores distintos: um vencimento só passa a "pago"
  quando existe um **registro de pagamento** — nada é marcado automaticamente.
- Plano semanal de 2 a 5 dias; turmas com os 10 horários exatos (05:00 às 19:00);
  valor em R$ e dia de vencimento de 1 a 31.
- Dia 29/30/31 em meses curtos usa o **último dia válido** (nunca data inválida).
- Histórico de pagamentos **não é apagado** ao editar dados do aluno.
- Aluno com pagamentos é **arquivado** (preservando o histórico); a exclusão definitiva
  só é permitida quando não há pagamentos vinculados.
- Duplicidade de aluno (mesmo nome + telefone) exige confirmação explícita.

### Segurança

- Senhas com **scrypt** (N=16384, R=8, P=1) + salt de 16 bytes — sem MD5/SHA puro.
- Sessão em **cookie HttpOnly / SameSite=Lax / Secure (produção)**, com expiração e
  logout; nenhuma credencial ou token no `localStorage`.
- **CSRF** por validação de origem em todas as mutações; **rate limiting** no login.
- Validação **Zod** compartilhada entre frontend e backend (mensagem em pt-BR).
- Queries parametrizadas via **Prisma** (sem SQL concatenado).
- Cabeçalhos de segurança + **CSP** aplicados em produção (`next.config.ts`).
- Erros sem stack trace e sem dados pessoais em logs/URLs.
- Sem cadastro público de administradores (criação só via script local) e sem
  integrações externas (compatível com LGPD).
- Todas as rotas e APIs do painel exigem sessão válida.

---

## 2. Como rodar localmente

Requisitos: **Node.js 20+** e um **PostgreSQL 14+**.

```bash
# 1) Dependências
npm install

# 2) Variáveis de ambiente
cp .env.example .env.local      # Windows: copie manualmente
# edite DATABASE_URL, SESSION_SECRET e APP_URL

# 3) Banco de dados (cria tabelas)
npm run db:migrate

# 4) Primeiro administrador (veja a seção 4)
npm run setup:admin

# 5) Desenvolvimento
npm run dev                     # http://localhost:3000

# 6) Produção local
npm run build
npm start
```

### Scripts disponíveis

| Script | Função |
| --- | --- |
| `npm run dev` | Servidor de desenvolvimento |
| `npm run build` | Gera o Prisma Client e o build de produção |
| `npm start` | Sobe o build de produção |
| `npm run lint` | ESLint |
| `npm run typecheck` | TypeScript (`tsc --noEmit`) |
| `npm run db:migrate` | Aplica migrações (`prisma migrate deploy`) |
| `npm run db:migrate:dev` | Cria/aplica migração em desenvolvimento |
| `npm run db:studio` | Prisma Studio (inspeção do banco) |
| `npm run setup:admin` | Cria/atualiza o primeiro administrador |
| `npm test` | Suíte de testes HTTP (exige servidor rodando e `ADMIN_SENHA_TESTE`) |
| `node scripts/limpar-dados-teste.mjs --apagar` | Remove só os dados de teste (nomes "Aluno Teste …") |

---

## 3. Banco de dados e variáveis de ambiente

### Banco

- **PostgreSQL** com Prisma ORM (`prisma/schema.prisma`).
- Tabelas: `admins`, `sessoes`, `alunos`, `pagamentos` (com índices e unicidade).
- Migração inicial: `prisma/migrations/20261017194801_init/`.
- Local: crie um banco (`createdb cenafit`) e aponte o `DATABASE_URL`.
- Na Vercel: use **Vercel Postgres / Neon / Supabase** (plano gratuito) e habilite TLS
  (`sslmode=require`).

### Variáveis (`.env.example`)

| Variável | Descrição |
| --- | --- |
| `DATABASE_URL` | Conexão PostgreSQL. Ex.: `postgresql://USER:SENHA@host:5432/cenafit?schema=public&sslmode=require` |
| `SESSION_SECRET` | Segredo usado como "pepper" no hash de sessões/IPs. Gere um forte: `node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"` |
| `APP_URL` | URL pública (validação de origem/CSRF). Produção: `https://seu-dominio.vercel.app` |
| `NODE_ENV` | `production` ativa cookie Secure, CSP e mensagens de erro genéricas. Na Vercel é definido automaticamente. |

> O arquivo `.env` está no `.gitignore`. **Nunca** versione valores reais.
> Em desenvolvimento local use `.env.local`.

---

## 4. Criando o primeiro administrador

Não existe cadastro público: o único caminho é o script local.

```bash
# Interativo (a senha é digitada oculta — recomendado)
npm run setup:admin

# Ou não interativo
ADMIN_EMAIL=admin@cenafit.com.br ADMIN_NOME=Administrador ADMIN_SENHA="SuaSenhaForte@123" npm run setup:admin
```

- O script **cria ou atualiza** o administrador com o e-mail informado.
- A senha é gravada apenas como hash scrypt; nunca aparece em arquivo ou log.
- Depois é só entrar em `/login` com o e-mail e a senha escolhidos.

---

## 5. Testes executados e resultados

Todos executados de fato nesta máquina (nada foi presumido):

| Verificação | Comando | Resultado |
| --- | --- | --- |
| Tipos | `npx tsc --noEmit` | **0 erros** |
| Lint | `npx eslint .` | **0 problemas** |
| Build de produção | `npm run build` | **concluído** (Next.js 16.4.2) |
| Suíte HTTP | `ADMIN_SENHA_TESTE=... npm test` (com `npm start` rodando) | **86/86 testes aprovados** |
| Auditoria (Lighthouse) | `/dashboard` | **Acessibilidade 100 · Best Practices 100 · SEO 100**, sem falhas |
| Inspeção visual | Navegador real em `/login`, `/dashboard`, `/alunos`, `/calendario`, `/pagamentos` | telas renderizadas com dados reais da logo e do banco; cadastro completo testado ponta a ponta |

A suíte (`scripts/testes.mjs`) cobre: proteção de rotas e APIs, CSRF, login/sessão,
CRUD de alunos, validações e mensagens em pt-BR, duplicidades, pagamentos (registrar,
histórico, estorno, arquivamento), dashboard, calendário (dias de 28 a 31) e rate
limiting.

> `npm test` precisa do servidor de produção rodando (`npm start` em outra janela) e da
> **senha do administrador via variável de ambiente** — nenhuma credencial está gravada
> no código:
>
> ```bash
> # Linux/macOS
> ADMIN_SENHA_TESTE="sua-senha" npm test
>
> # Windows (PowerShell)
> $env:ADMIN_SENHA_TESTE="sua-senha"; npm test
> ```
>
> Sem a variável a suíte recusa rodar e explica como proceder.

---

## 6. Publicando na Vercel (etapas faltantes)

O projeto já está preparado; faltam apenas passos que dependem das suas contas:

1. **Criar conta na Vercel** e no provedor de banco (Vercel Postgres, Neon ou Supabase).
2. **Criar o repositório Git** e enviar o código:
   ```bash
   git init
   git add .
   git commit -m "CENA FIT — gestão de alunos"
   git remote add origin https://github.com/SEU-USUARIO/cena-fit.git
   git push -u origin main
   ```
   (`.gitignore` já exclui `.env`, `.next`, `node_modules` e `.vercel`.)
3. **Importar o projeto na Vercel** (Framework: Next.js — detectado automaticamente).
4. **Configurar as variáveis de ambiente** no painel da Vercel (Production e Preview):
   `DATABASE_URL`, `SESSION_SECRET`, `APP_URL`.
5. **Aplicar as migrações** no banco de produção, uma única vez:
   ```bash
   DATABASE_URL="postgresql://...sslmode=require" npx prisma migrate deploy
   ```
6. **Criar o primeiro administrador** apontando para o banco de produção:
   ```bash
   DATABASE_URL="postgresql://..." ADMIN_EMAIL=... ADMIN_NOME=... ADMIN_SENHA=... npm run setup:admin
   ```
7. **Deploy**: `vercel deploy --prod` (ou `git push` com deploy automático).
8. **Ajustar `APP_URL`** para o domínio final (ex.: `https://cenafit.vercel.app`) —
   sem isso o CSRF bloqueia as mutações.
9. **Testar em produção**: login, cadastro, edição, exclusão, pagamento e calendário.

### Verificação pós-deploy

- Acessar `/login` com o administrador criado.
- Cadastrar um aluno e conferir dashboard/calendário.
- Confirmar que rotas sem sessão redirecionam para `/login`.

---

## 7. Estrutura do projeto

```
prisma/
  schema.prisma          Modelos Admin, Sessao, Aluno, Pagamento
  migrations/            Migrações aplicadas
scripts/
  create-admin.mjs       Cria/atualiza o administrador
  testes.mjs             Suíte de testes HTTP
src/
  app/
    login/               Tela de login
    (painel)/            Área autenticada (layout com guarda de sessão)
      dashboard/         Indicadores
      alunos/            Tabela + formulário
      calendario/        Calendário de vencimentos
      pagamentos/        Resumo + registro de pagamento
    api/                 Rotas REST (auth, alunos, pagamentos, dashboard, calendario)
  components/            Estrutura (menu/cabeçalho), Modal, Indicadores, Ícones...
  lib/                   db, session, senha, charges, validation, http, dates...
public/                  Logo CENA FIT
```
