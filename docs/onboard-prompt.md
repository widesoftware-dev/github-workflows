# Onboarding Prompt — Widesoftware CI/CD

Prompt pra colar no Claude Code rodando **dentro do repositório do cliente**. O Claude descobre quase tudo sozinho (linguagem, versões, comandos, registry, WIF, etc.), te pergunta só o que não dá pra inferir, gera o `ci.yml` e te entrega checklist de secrets.

## Como usar

1. Abra o Claude Code **no repositório do cliente** (não neste repo).
2. Copie o bloco abaixo a partir do `── COPIE ──`.
3. Cole no Claude.
4. Responda só as perguntas que ele te fizer.
5. Revise o YAML proposto e aprove a criação do arquivo.

---

── COPIE ──

Configure CI/CD neste repositório usando os reusable workflows do `widesoftware-dev/github-workflows` (https://github.com/widesoftware-dev/github-workflows). Trabalhe em duas fases: **descoberta automática** primeiro, **perguntas** só do que faltou.

### Fase 1 — descoberta automática

Inspecione o repo e detecte:

1. **Stack** (em ordem de prioridade):
   - `php` se existe `composer.json`.
   - `go` se existe `go.mod`.
   - `node` base; depois identifique framework:
     - `nestjs` se existe `nest-cli.json`.
     - `nextjs` se existe `next.config.{js,mjs,ts}`.
     - `node` genérico caso contrário.

2. **Package manager** (Node):
   - `pnpm` se `pnpm-lock.yaml`.
   - `yarn` se `yarn.lock`.
   - `npm` se `package-lock.json`.

3. **Versão da runtime** (na ordem; primeiro hit vence):
   - Node: `.nvmrc` → `package.json.engines.node` → env do CI legado (`NODE_VERSION`) → `22`.
   - PHP: `composer.json.config.platform.php` → `.php-version` → CI legado → `8.3`.
   - Go: linha `go` do `go.mod` → CI legado → `1.23`.
   - pnpm: `package.json.packageManager` → env do CI legado (`PNPM_VERSION`) → `9`.

4. **Comandos**:
   - Node: `scripts.test`, `scripts.lint`, `scripts.typecheck` do `package.json`.
   - PHP: `scripts.test` do `composer.json`; default `vendor/bin/pest`.
   - Go: default `go test -race -count=1 ./...`.
   - Use o prefixo do package manager detectado (ex: `pnpm test`).

5. **CI legado** — se `.github/workflows/*.yml` existir, leia todos e extraia:
   - **Registry**: procure por `docker/login-action`, `docker/build-push-action.tags`, e o host (ex: `southamerica-east1-docker.pkg.dev/<projeto>/<repo>`).
   - **WIF GCP**: presença de `google-github-actions/auth@v2` → `auth-method: gcp-wif`. Pegue `workload_identity_provider` e `service_account`.
   - **Branches**: `on.push.branches` e `on.pull_request.branches`.
   - **Semgrep configs**: linhas `--config p/...`.
   - **Cargo audit**: presença de step `cargo audit` ou `Cargo.lock` → setar `cargo-audit-paths`.

6. **Image name**: nome do diretório do repo (`basename "$(pwd)"`).

7. **Tag do reusable**: consulte as tags ativas:
   ```sh
   gh api /repos/widesoftware-dev/github-workflows/tags --jq '.[0:5] | .[].name'
   ```
   Proponha a mais recente (que case com semver `vX.Y.Z`).

### Fase 2 — resumo + perguntas mínimas

**Mostre** ao usuário um resumo em bullets do que detectou. Marque com `?` qualquer campo que ficou indeterminado.

**Pergunte APENAS o que faltou ou é ambíguo**. Casos comuns:
- Tag do reusable se múltiplas opções viáveis (ofereça a mais recente como recomendada).
- Notificação: `none` | `slack` | `email` | `both`. Se `slack`, peça canal. Se `email`, peça destinatários.
- Quais scans desligar (default: nenhum). Apenas se o usuário pedir.
- Se branch `homolog` deve ser incluída quando não detectada no legado.

Não pergunte nada que já tenha sido detectado com confiança.

### Fase 3 — geração

Use o exemplo de `examples/consumer-<stack>.yml` da tag escolhida (`https://raw.githubusercontent.com/widesoftware-dev/github-workflows/<tag>/examples/consumer-<stack>.yml`) como base.

**Saída** — onde criar:
- Se NÃO existe `.github/workflows/ci.yml`: crie como `.github/workflows/ci.yml`.
- Se EXISTE: crie como `.github/workflows/ci-v2.yml` lado a lado, preservando o legado. Comente no topo do arquivo que é piloto e que o legado será removido depois da validação.

**Regras obrigatórias do YAML**:
- `uses:` sempre em `@vX.Y.Z` (semver), nunca `@main`.
- `do-push: ${{ github.event_name == 'push' }}` (não faz push em PR).
- `block-on-failure: ${{ github.ref == 'refs/heads/main' }}` se houver `homolog` ou outra branch não-bloqueante. Caso só `main`, fixar `true`.
- Passar `package-manager`, `pnpm-version` (se pnpm), `node-version`/`php-version`/`go-version`, e os comandos detectados.
- Repassar inputs WIF (`auth-method`, `gcp-workload-identity-provider`, `gcp-service-account`) se detectados.
- Repassar `cargo-audit-paths` se Cargo.lock(s) encontrado(s).
- Notificações conforme respostas.

### Fase 4 — entrega

1. Mostre o YAML completo proposto para revisão. Não commite.
2. Liste secrets a configurar (Settings → Secrets and variables → Actions):
   - Sempre que `auth-method: docker-login`: `REGISTRY_USERNAME`, `REGISTRY_PASSWORD`.
   - Quando WIF: nenhum secret de registry (a WIF já resolve), mas confirme que a service account tem `Artifact Registry Writer`.
   - Se Slack: `SLACK_WEBHOOK_URL`.
   - Se Email: `SMTP_HOST`, `SMTP_PORT`, `SMTP_USERNAME`, `SMTP_PASSWORD`, `SMTP_FROM`.
3. Sugira o próximo passo (criar branch, abrir PR, etc.), mas **não execute git/gh sem aprovação**.

── FIM ──

---

## Manutenção do prompt

Ao bumpar a tag do reusable ou adicionar novos inputs:
- Atualize `examples/*.yml` deste repo (são a fonte da geração).
- Se uma nova capacidade exigir input que não dá pra inferir, adicione na Fase 2 com pergunta clara.
- Se uma capacidade nova for inferível, adicione na Fase 1.

Princípio: **toda nova feature começa como auto-detecção. Pergunta é último recurso.**
