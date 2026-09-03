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

**Formato obrigatório de commit**: `X.Y.Z - description in English (US)`. O bump do
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

<!-- COMMIT-RULE:repodocs -->

## Commits — you commit, and nothing is delivered until you have

> Marked echo. The single source is **[samirhvbr/repodocs](https://github.com/samirhvbr/repodocs/blob/master/docs/versioning.md#who-commits-and-when)**
> — change it there, not here. This block is regenerated.

**Committing is your job.** Not "leave the tree ready and something downstream
packages it" — you run `git commit`, and `git push`, as the last step of the work
you were asked to do. The COMMITTER skill that used to commit on an agent's
behalf is `enabled: false` in every repository of this fleet since 03/09/2026;
what is left of it is a kill-switch, not a scheduler. **If you do not commit,
nobody does.**

**Do not report a task as finished before the commit exists.** "Done",
"delivered", "concluded" mean the work is in `git log` — never that it is sitting
uncommitted where only this session can see it. The commit is the last step *of
the task*, not a follow-up for someone else. If you are about to write
"finished", commit first, then write it.

**Every commit obeys the versioning rules**, with no exception:

- Subject `X.Y.Z - short description in English (US)`, the version taken from
  `version.md` and **bumped in the same commit**.
- The `CHANGELOG.md` entry is written first — its `## X.Y.Z - description`
  heading *is* the subject.
- No Conventional Commits prefix (`feat:`, `fix:`, `chore:`) and no vague
  subject ("update", "ajuste", "wip", "changes", "several improvements").

**The bump is the one clause a repository may override — in writing.** If this
repository's own documentation says the version is stamped some other way, and says
why, follow that. Otherwise the line above applies to you. An override nobody wrote
down is not an exception. Nothing else in this block bends: the changelog entry, the
subject, the language, one subject per commit, and committing before you report done
all hold regardless.

**One subject per commit.** The subject has to describe the whole commit
honestly. The moment your description needs an "and" to be true, it is two
commits.

**Split a large delivery into blocks.** A complex task is committed as a series
of commits grouped by subject, each small enough to be described in one line and
read on its own. They may share a version — bump `version.md` in the first and
repeat the number in the rest; two commits carrying one version is expected, not
a mistake. **Splitting is the default** for anything non-trivial, because the
history is the documentation of *how* the work was done, and one commit touching
six unrelated subjects documents none of them.

**The standard you are keeping:** someone reading `git log` alone — a year from
now, without the conversation that produced the work — can say what happened,
when, why, and at which version. If your commit would fail that test, it is too
big or its subject is too vague, and both are fixed the same way.

<!-- /COMMIT-RULE -->

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

## Language — English (US), everywhere in the repository

**Everything that lives in this repository, or in GitHub's interface around it,
is written in English (US)**: documents, **commit messages**, pull request titles
and bodies, issues, code comments, changelog entries, release notes.

Commit format: `X.Y.Z - short description in English`. The version comes from
`version.md` and is bumped in the same commit. Conventional Commits prefixes
(`feat:`, `fix:`, `chore:`) and vague one-word messages are forbidden.

**Exactly one carve-out:** end-user-facing strings — UI text, transactional
email, product copy. That is product i18n for a Brazilian audience, not
repository content.

History is not rewritten: Portuguese messages already in the log stay as they
are.

<!-- /RELEASES-RULE -->
