# Versão — Marthina Learning

**Versão atual:** `0.1.4`

> Esta versão é a fonte da verdade do projeto e é lida em runtime via
> `config('app.version')` — a aplicação extrai o **primeiro número semver
> (`X.Y.Z`)** encontrado neste arquivo. Mantenha a linha **"Versão atual"**
> sempre como a primeira ocorrência de um número de versão.

---

## 1. Convenção de Versionamento (`X.Y.Z`)

Padrão semântico simplificado, herdado do guia de projetos do Samir e adaptado
para esta plataforma educacional.

| Componente | Significado | Como sobe |
|---|---|---|
| **X** | Versão estável final / release público | Manual |
| **Y** | Mudança estrutural (nova área, refatoração grande, nova integração) | Manual |
| **Z** | Incremento a cada entrega (ver gatilhos) | A cada entrega |

### Gatilhos de bump do `Z`

Incremente o `Z` (e registre no changelog) sempre que:

- Criar uma **tela / view** nova (quiz, vocabulário, perfil, admin, etc.)
- Criar uma **tabela / migration** nova
- **Modificar layout** (estrutura visual de uma tela)
- **Renomear** botão, label, rota nomeada ou coluna
- Alterar **regra de negócio** (pontuação, XP, troféus, fluxo de convidado, quiz)
- Alterar **configuração de segurança** (rate limit, CSRF, headers, auth, admin)

> Correções de texto, comentários e formatação (`pint`) **não** exigem bump.

---

## 2. Formato de Commit Obrigatório

```
X.Y.Z - Descrição curta em português
```

Exemplo:

```bash
git commit -m "0.1.1 - Adiciona rate limit no login e na recuperacao de senha"
```

O bump do `version.md` entra em **um único commit** por entrega (o primeiro da
entrega). Commits adicionais da mesma entrega repetem a versão sem novo bump.

---

## 3. Changelog

> Ordem decrescente (mais recente no topo). Cada entrada lista as mudanças e os
> gatilhos que justificaram o bump.

### `0.1.4` — 2026-07-29 — Adota o COMMITTER: marcador de opt-in para o ciclo automático de commit

- `.committer.yml` na raiz — opt-in explícito (sem marcador o repo não existe para
  a skill). `branch_only: master` limita o ciclo à branch de trabalho.
- A varredura é do `~/x/GIT/run.sh` (cron de 30 min), que descobre os
  participantes pelo marcador em vez de lista fixa de caminhos.

### `0.1.3` — 2026-07-06 — Rate limiting nas rotas de autenticação

Fecha a lacuna prioritária da seção 8 do `SECURITY_GUIDELINES.md`.

**Segurança**
- `AppServiceProvider::boot()` — define 4 limiters via `RateLimiter::for(...)`
  aplicados com `->middleware('throttle:nome')` nas rotas POST:
  - `throttle:login` — 5/min por **IP + e-mail** (brute-force de conta).
  - `throttle:register` — 5/min por **IP** (cadastro em massa).
  - `throttle:forgot-password` — 3/min por **IP + e-mail** (spam de recuperação).
  - `throttle:reset-password` — 5/min por **IP** (brute-force de token).
- `resources/views/errors/429.blade.php` — página amigável no tema infantil para
  a resposta HTTP 429 (em vez da tela de erro padrão do framework).

**Correção**
- `routes/web.php` — adiciona o `use App\Models\QuizResult;` que faltava; sem ele
  o `GET /quiz/{cat}/reset` do lote 0.1.2 lançava erro ao tentar limpar o troféu.

_Gatilhos:_ configuração de segurança (rate limit) e nova view.

### `0.1.2` — 2026-07-06 — Correções de revisão (UI, LGPD e antiduplicação)

Lote de correções levantadas em revisão visual/funcional da plataforma.

**UI**
- `layout.blade.php` — navbar corrigido: texto de pontuação/usuário e botões
  "Sair"/"Entrar" ficavam brancos sobre fundo claro (invisíveis); agora usam
  tom escuro do tema e contorno legível.
- `categories.blade.php` — o selo de contagem exibia o código PHP cru
  (`$categories->count() . ' trilhas'`) por faltar o bind `:label`; agora mostra
  a contagem real (ex.: "9 trilhas").

**Segurança / LGPD**
- `ranking.blade.php` + `buildLeaderboard()` — o e-mail de todos os usuários
  (incluindo crianças) aparecia no ranking para qualquer pessoa logada. E-mail
  removido da exibição e da própria consulta (minimização de dados).
- Quiz de múltipla escolha — a alternativa correta era sempre gravada com a letra
  "A" e o quiz reexibia essa chave após embaralhar, entregando a resposta. As
  letras passam a ser atribuídas pela posição exibida (A–H), sem vazamento.

**Regra de negócio (antiduplicação de pontos/troféus)**
- `POST /quiz/{cat}/check` — `Score` agora é idempotente por usuário/questão:
  reenviar a mesma resposta (voltar + reenviar) não soma pontos/XP de novo.
- Conclusão de quiz — `QuizResult` (troféu) só é criado se ainda não existir para
  o usuário/categoria, evitando novo troféu a cada relogin ou revisita da tela de
  conclusão. `GET /quiz/{cat}/reset` passa a limpar também os `QuizResult`, para
  que um replay legítimo volte a premiar.

_Gatilhos:_ mudança de layout, configuração de segurança (LGPD/vazamento) e regra
de negócio (pontuação/troféus).

### `0.1.0` — 2026-06-18 — Padronização de documentação e versionamento

Primeira entrega da plataforma sob o **padrão de projetos do Samir**. O código
da aplicação já existia (auth, modo convidado, quizzes, vocabulário, ranking e
painel administrativo); esta entrega formaliza a documentação e o versionamento.

**Documentação**
- `version.md` (este arquivo) — convenção de versionamento + changelog, lido em
  runtime via `config('app.version')`.
- `CLAUDE.md` — guia operacional para agentes de IA.
- `SECURITY_GUIDELINES.md` — diretrizes de segurança adaptadas (auth de usuário
  final + painel admin + dados de criança / LGPD).
- `README.md` — atualizado para o padrão (links, seção de versão, banco).

**Banco de dados**
- Padrão de banco alinhado: **MariaDB/MySQL (ou PostgreSQL) — nunca SQLite**.
  `.env.example` e `config/database.php` passam a usar `mysql` por padrão.

_Gatilhos:_ baseline de documentação/versionamento e mudança de configuração de
banco (estrutural).
