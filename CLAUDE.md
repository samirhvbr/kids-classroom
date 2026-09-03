# Marthina Learning — Guia para Agentes de IA

Plataforma educacional gamificada para crianças (Laravel + Livewire): vocabulário
de inglês, quizzes por matéria, pontos/XP, troféus, ranking, perfis e painel
administrativo. Este documento é a referência operacional para agentes de IA.

> **Fonte de verdade**: o código manda. Em divergência, use `composer.json` /
> `php artisan about` / `php artisan route:list`.

---

## 🔄 Antes de começar: `git pull`

**SEMPRE** verifique atualizações remotas antes de escrever ou alterar qualquer coisa neste repositório:

```bash
git pull          # já está pré-autorizado (allow)
```

Trabalhar sobre uma base desatualizada gera conflitos. Puxe primeiro, sempre. Para só inspecionar antes: `git fetch && git status`.

---

## Comunicação

- **Idioma:** Português (pt-BR) para mensagens ao operador, comentários e textos de UI.
- **Identificadores de código:** Inglês (classes, métodos, variáveis, rotas).
- **Commits:** Formato `X.Y.Z - descrição` (versão de [`version.md`](version.md)).

---

## Stack

| Camada | Tecnologia |
|---|---|
| Backend | Laravel 12 / PHP 8.2+ |
| Componentes reativos | Livewire 4 |
| Front-end / build | Vite 7 + Tailwind CSS 4 |
| UI auxiliar (CDN) | Bootstrap 5 · Font Awesome 6 |
| Banco | **MariaDB/MySQL** (prod) — PostgreSQL aceito · **nunca SQLite** |
| Web server | Nginx + PHP-FPM (produção) |

> **Banco: sempre MariaDB/MySQL ou PostgreSQL. SQLite não é usado em nenhum
> contexto** (nem dev). `DB_CONNECTION=mysql` em `.env`.

---

## Versão e Commits

Versão em [`version.md`](version.md) (raiz), lida via `config('app.version')`
(primeiro semver do arquivo). Padrão `X.Y.Z`:

- **Z** sobe a cada entrega: criar tela, criar tabela, mudar layout, renomear
  label/rota, alterar regra de negócio (pontos/XP/troféus/quiz) ou config de segurança.
- **Y / X** são manuais (mudança estrutural / release estável).

**Formato obrigatório de commit**: `X.Y.Z - Descrição em português`. O bump do
`version.md` vai em **um** commit por entrega; registre o changelog no próprio `version.md`.

---

## Arquitetura (estado atual × alvo)

> ⚠️ **Importante para agentes:** hoje a maior parte da lógica vive em **closures
> em [`routes/web.php`](routes/web.php)** (~2k linhas), apoiada por *closures
> helper* no topo do arquivo (sessão de convidado, sessão autenticada, montagem
> do dashboard admin, leaderboard, etc.). A validação é **inline** (checagens
> manuais), não via Form Request. `app/Http/Controllers/` contém só o `Controller`
> base.

**Convenção-alvo (para código novo / refatorações):**

- **Controllers finos** + **Form Requests** (`app/Http/Requests/`) para validação.
- **Services** para regra de negócio reaproveitável.
- **Rotas nomeadas** com `->name()` (hoje só `vocabulary` é nomeada — ao mexer
  numa rota, considere nomeá-la).

Ao **editar** fluxos existentes, mantenha o estilo do arquivo para não quebrar as
closures compartilhadas; ao **criar** algo novo, prefira a convenção-alvo.

---

## Modelos (`app/Models/`)

| Model | Papel | Notas |
|---|---|---|
| `User` | Conta (aluno ou admin) | `SoftDeletes`; flags `is_admin`/`is_active`/`blocked_at`; `password` cast `hashed`; `$fillable` explícito; `isAdmin()`, `displayName()`, `statusLabel()` |
| `Subject` | Matéria (inglês/português/matemática) | `slug` com prefixo `eng_`/`prt_`/`mat_`, `is_active`, `icon` |
| `Category` | Categoria dentro da matéria | `QUIZ_TYPE_VOCABULARY` / `QUIZ_TYPE_MULTIPLE_CHOICE`; `isVocabularyQuiz()` |
| `Word` | Vocabulário (tabela `eng_words`) | `english`, `portuguese`, `example`, imagem, `category_id` |
| `Question` | Pergunta de múltipla escolha | `DIFFICULTY_EASY/NORMAL/HARD`; `WRONG_OPTIONS_BY_DIFFICULTY` (3/4/5); `difficultyLabels()` |
| `QuestionOption` | Alternativa | `option_key` (A–H), `is_correct`, `sort_order` |
| `Score` | Tentativa registrada | `XP_PER_CORRECT_ANSWER = 10`; `xpForAnswer()`; liga `word_id` ou `question_id` |
| `QuizResult` | Resultado por categoria | `trophy` = gold/silver/bronze (por % de acerto) |
| `AdminUserAction` | Auditoria de ações do admin | `ACTION_RESTORE/BLOCK/UNBLOCK/DELETE` + `justification` |

---

## Fluxos principais

- **Autenticação**: login/registro/recuperação de senha. O registro tem
  **honeypot** (`company`), **time-trap** (mín. 3s) e **pergunta humana** (soma).
  Login bem-sucedido faz `session()->regenerate()` e popula `user_id`,
  `is_admin`, `user_name`, `user_email`.
- **Modo convidado** (`/guest-login`): joga sem conta; progresso fica na **sessão**
  (`guest_metrics`, `guest_quiz_records`) e expira ao fechar o navegador.
- **Quiz** (`/quiz/{category}`): sorteia item não respondido (palavra ou questão),
  monta alternativas, registra `Score` no `check`, e ao terminar grava `QuizResult`
  com troféu (≥90% gold, ≥70% silver, ≥50% bronze).
- **Admin** (`/admin`, gate por `is_admin`): CRUD de questões (formulário **ou**
  importação por **JSON** com chaves `MATERIA`/`CATEGORIA`/`PERGUNTA`/`RESPOSTA`/
  `RESPOSTA ERRADA N`) e gestão de usuários (criar/editar, bloquear/desbloquear,
  excluir logicamente e restaurar) — **toda ação destrutiva exige justificativa**
  e é registrada em `AdminUserAction`.

---

## Banco de Dados & Migrations

**Banco: MariaDB/MySQL (ou PostgreSQL). Nunca SQLite.**

- Migrations **idempotentes** quando fizer sentido (`Schema::hasTable()` /
  `hasColumn()`), sempre com `down()` funcional.
- Índices em colunas usadas em `WHERE`/`ORDER BY` (`category_id`, `user_id`, `email`).
- `$table->timestamps()` por padrão.
- **NUNCA** `migrate:fresh` em produção. Use `migrate:rollback --step=N`.
- Seeders: `SubjectSeeder`, `CategorySeeder`, `WordSeeder`, `QuestionSeeder`
  (orquestrados por `DatabaseSeeder`).
- Admin inicial: migration `seed_initial_admin_user` lê `ADMIN_EMAIL`/`ADMIN_PASSWORD`
  do `.env` (senha **hasheada** ao gravar). Defina uma senha forte **antes** de migrar.

---

## UI & Frontend

- Público infantil: paleta suave, fontes _Baloo 2_ / _Nunito_.
- **Assets via Vite**: entradas em `resources/css/app.css`, `resources/js/app.js`
  (+ `resources/js/bootstrap.js`); Tailwind 4 via `@tailwindcss/vite`. Carregar com
  `@vite([...])` no `layout.blade.php`. Build: `npm run build`; dev: `npm run dev`.
- Imagens do tema em `public/assets/marthina-theme/`. Referenciar com `asset(...)`.
- **Blade**: output sempre escapado com `{{ }}`. `{!! !!}` **proibido** com dado de
  usuário (nome, bio, e-mail). Dados do servidor para JS via `@json(...)`. `@csrf`
  em **todos** os formulários POST.

---

## Segurança (resumo)

Regras completas em [SECURITY_GUIDELINES.md](SECURITY_GUIDELINES.md). Pontos-chave:

- Senhas via cast `hashed` (bcrypt, `BCRYPT_ROUNDS=12`); comparação com `Hash::check`.
- Anti-bot no registro: honeypot + time-trap + pergunta humana; honeypot no login/recuperação.
- Sessão: `regenerate()` no login, `invalidate()`+`regenerateToken()` no logout;
  sessões do usuário são apagadas ao bloquear/excluir/redefinir senha.
- Soft delete de usuários + auditoria (`AdminUserAction`) com justificativa.
- Upload de avatar validado (mime JPG/PNG/WEBP, ≤ 2 MB).
- Mass assignment: `$fillable` explícito; nunca confiar em input para `is_admin`.
- **LGPD — dados de criança** (nome, e-mail, telefone, foto): minimizar e proteger.
- ⚠️ **Lacuna conhecida:** ainda **não há rate limiting** em `/login`, `/register`,
  `/forgot-password`, `/reset-password`. Adicionar `throttle` é prioridade — ver SECURITY.

---

## Comandos Rápidos

```bash
composer setup          # instala deps, .env, key, migrate, build (config o banco antes!)
composer dev            # serve + queue + logs (pail) + vite, em paralelo
composer test           # php artisan test
php artisan serve       # http://localhost:8000
npm run dev / build     # Vite
php artisan migrate --seed
php artisan route:list
php artisan optimize:clear
php artisan pint        # formatação (se configurado)
php -l caminho/arquivo.php
```

---

## DEV Files (não vão para produção)

`.env`, `.env.*`, `storage/`, `bootstrap/cache/`, `.git/`, `vendor/`,
`node_modules/`, `public/build/`, `.vscode/`, `CLAUDE.md`, `SECURITY_GUIDELINES.md`,
`README.md`, `version.md`.

---

## Checklist Pré-Commit

- [ ] `php -l` nos arquivos PHP alterados
- [ ] `php artisan route:list` sem erros
- [ ] `php artisan view:cache && php artisan view:clear` — valida Blade
- [ ] Banco MariaDB/MySQL (ou PostgreSQL) — **nunca** SQLite
- [ ] Migrations com `down()` funcional; idempotentes quando aplicável
- [ ] `$fillable` explícito; `is_admin` nunca vem de input direto
- [ ] `@csrf` em todos os formulários; output escapado com `{{ }}`
- [ ] `.env.example` atualizado se adicionou variável
- [ ] `version.md` com bump + changelog se aplicável
- [ ] `APP_DEBUG=false` em produção

---

## PS — Commits: a skill COMMITTER cuida disso

**Existe `.committer.yml` na raiz deste repositório** — é o opt-in da skill
**COMMITTER**, que roda em ciclo (cron, via `~/x/GIT/run.sh`). Enquanto esse arquivo
existir com `enabled: true`, **commitar e pushar não é trabalho seu**.

**O que muda para você:**

- **Não commite nem pushe por padrão.** Conclua a entrega bumpando o `version.md`
  **com a entrada de changelog** e deixe a árvore pronta. É dali que a mensagem do
  commit sai — o changelog virou o artefato de handoff entre você e a skill.
- A skill monta `X.Y.Z - descrição`, commita e pusha a branch atual sozinha. Ela
  **nunca bumpa versão** (isso continua sendo julgamento seu) e nunca inventa
  mensagem: sem entrada de changelog ela cai num fallback Sonnet, e sem conseguir
  descrever com honestidade ela aborta e espera.

**Você ainda commita quando:**

- o Samir pedir explicitamente;
- a tarefa exigir o SHA na hora (deploy, abrir PR, referência cruzada);
- o `.committer.yml` sumir ou estiver `enabled: false` — aí vale o fluxo antigo,
  você bumpa, commita e pusha.

**Por que isso existe:** tirar de um modelo caro (Opus/Fable) o trabalho mecânico de
empacotar commit, que um Sonnet — ou, na maioria das vezes, nenhum modelo — resolve.
Economiza token e devolve tempo de desenvolvimento.

---

<!-- RELEASES-RULE:repodocs -->

## Releases — the `version.md` on GitHub is what the Releases show

> Marked echo. The single source is **[samirhvbr/repodocs](https://github.com/samirhvbr/repodocs/blob/master/docs/versioning.md)**
> — change it there, not here. This block is regenerated.

**The `version.md` of the default branch, on GitHub, is what the GitHub Releases
must show.** The local checkout does not enter the calculation: it can be behind,
ahead or mid-work, and none of that is published — GitHub cannot tag a commit it
does not have.

**The bump and the Release are one act.** A commit that bumps `version.md` is not
finished until that version has a tag, a published Release, and the **`Latest`
badge on it** — the same push, not "later". A badge sitting on an older release
tells whoever looks that the project is at a version it is not.

- `.github/workflows/release.yml` does it on any push that touches `version.md`.
- `./tools/release.sh` does it by hand. It is **idempotent and self-healing**:
  it publishes whatever is missing and moves a drifted badge back. Running it is
  always safe, so it is both the check and the fix.

A PR publishes nothing while it is a PR. The moment it merges, the push moves
`version.md` on the default branch and the Release becomes that version.

Tag and Release title are the **bare version — no `v` prefix**.

<!-- /RELEASES-RULE -->
