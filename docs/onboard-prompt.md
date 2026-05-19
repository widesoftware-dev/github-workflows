# Onboarding Prompt — Widesoftware CI/CD

Prompt pra colar no Claude Code rodando **dentro do repositório do cliente**. O Claude descobre tudo que conseguir do repo (linguagem, versões, registry, WIF, GitOps repo, convenção de tags, etc.), te pergunta só o que sobrou, gera o `ci.yml` pinado na tag corrente do reusable e entrega checklist de pré-requisitos.

## Como usar

1. Abra o Claude Code **no repositório do cliente** (não neste repo).
2. Copie o bloco abaixo a partir do `── COPIE ──`.
3. Cole no Claude.
4. Responda só as perguntas que ele te fizer.
5. Revise o YAML proposto e aprove a criação do arquivo.

---

── COPIE ──

Configure CI/CD neste repositório usando os reusable workflows do `widesoftware-dev/github-workflows` (https://github.com/widesoftware-dev/github-workflows). Trabalhe em quatro fases: **descoberta automática** → **perguntas mínimas** → **geração do `ci.yml`** → **pré-flight + handoff**. Não pergunte nada que dê pra inferir.

## Fase 1 — descoberta automática

Inspecione o repo e detecte:

1. **Stack** (em ordem de prioridade):
   - `php` se existe `composer.json`.
   - `go` se existe `go.mod`.
   - `node` como base; identifique framework:
     - `nestjs` se existe `nest-cli.json`.
     - `nextjs` se existe `next.config.{js,mjs,ts}`.
     - `node` genérico caso contrário.

2. **Package manager** (Node):
   - `pnpm` se `pnpm-lock.yaml`.
   - `yarn` se `yarn.lock`.
   - `npm` se `package-lock.json`.

3. **Versão da runtime** (primeiro hit vence):
   - Node: `.nvmrc` → `package.json.engines.node` → env `NODE_VERSION` no CI legado → `22`.
   - PHP: `composer.json.config.platform.php` → `.php-version` → CI legado → `8.3`.
   - Go: linha `go` do `go.mod` → CI legado → `1.23`.
   - pnpm: `package.json.packageManager` → env `PNPM_VERSION` no CI legado → `9`.

4. **Comandos**:
   - Node: `scripts.test`, `scripts.lint`, `scripts.typecheck` do `package.json` (use o prefixo do package manager).
   - PHP: `scripts.test` do `composer.json`; default `vendor/bin/pest`.
   - Go: default `go test -race -count=1 ./...`.

5. **CI legado** — se `.github/workflows/*.yml` existir, leia todos. Identifique:
   - **Registry & auth**:
     - Procure `docker/login-action` → username/password (`docker-login`).
     - Procure `google-github-actions/auth@v2` → WIF (`gcp-wif`). Capture `workload_identity_provider` e `service_account`.
     - Procure o host do registry em `docker/build-push-action.tags` (ex: `southamerica-east1-docker.pkg.dev/<project>/<repo>/...`).
   - **Branches**: `on.push.branches` e `on.pull_request.branches`. Note se há `homolog`.
   - **Semgrep configs**: linhas `--config p/...`.
   - **Cargo audit**: `cargo audit` ou `Cargo.lock` em subdiretórios → setar `cargo-audit-paths`.
   - **GitOps bump**: se existe um `on_release.yml` (ou step) que faz `actions/create-github-app-token` + checkout de outro repo + `yq` em kustomization, capture:
     - `gitops-repo` (do `repository:` do checkout)
     - `gitops-app-id-var` (var referenciada em `app-id:`, default `APP_ID`)
     - Estrutura de path do kustomization atual (compare com `{repo}-{env}/kustomization.yaml`)

6. **Image name base**: `github.event.repository.name` (resolvido em runtime no ci.yml; não hardcode).

7. **Tag do reusable**: consulte tags ativas:
   ```sh
   gh api /repos/widesoftware-dev/github-workflows/tags --jq '.[].name' | head -10
   ```
   Use a mais recente compatível (>= `v1.4.1` recomendada).

## Fase 2 — perguntas mínimas

Mostre ao usuário um **resumo em bullets** do que detectou. Marque com `?` qualquer campo indeterminado.

**Pergunte APENAS o que faltou ou é ambíguo**. Casos comuns:
- Confirmar tag do reusable se múltiplas opções viáveis (sugira a mais recente).
- Confirmar `gitops-bump-enabled` se detectou `on_release.yml` com bump. Default sugerido: `true` (substitui o on_release legado).
- Notificação: `none` | `slack` | `email` | `both`. Se algo além de `none`, peça canal/destinatários.
- Quais scans desligar (default: nenhum). Pergunte só se o usuário pedir.
- Convenção de sufixo `-prod`/`-homolog` no `image-name` e no path do registry? Default: **sim** (convenção Widesoftware atual). Se o cliente usa paths sem sufixo, ajustar o ci.yml gerado.

Não pergunte nada que já tenha sido detectado.

## Fase 3 — geração do `ci.yml`

Use `examples/consumer-<stack>.yml` da tag escolhida como base. Pra Node/Nest/Next/Go/PHP padrão Widesoftware, gere com **as expressions context-aware**:

```yaml
name: CI

on:
  pull_request:
    branches: [main, homolog]   # ajustar se cliente só usa main
  push:
    branches: [main, homolog]   # ajustar idem
    tags: ['v*']

# OBRIGATÓRIO: caller define teto de permissões pro reusable.
permissions:
  contents: read
  id-token: write       # WIF
  pull-requests: write  # PR report
  packages: write       # ghcr.io (drop se não usar)
  actions: read         # pr-report do reusable lê run metadata

jobs:
  ci:
    uses: widesoftware-dev/github-workflows/.github/workflows/<wrapper>.yml@<TAG>
    # IMPORTANTE: 'secrets: inherit' NÃO propaga organization secrets
    # quando o reusable está em outra org. Passe explicitamente.
    secrets:
      GITOPS_APP_PRIVATE_KEY: ${{ secrets.GITOPS_APP_PRIVATE_KEY }}
      # Adicione SLACK_WEBHOOK_URL, SMTP_*, REGISTRY_USERNAME/PASSWORD
      # se o cliente usar essas features.
    with:
      # Convenção -prod/-homolog (omitir se cliente usa paths sem sufixo).
      image-name: >-
        ${{
          startsWith(github.ref, 'refs/tags/v')
            && format('{0}-prod', github.event.repository.name)
            || format('{0}-homolog', github.event.repository.name)
        }}
      registry: >-
        ${{
          startsWith(github.ref, 'refs/tags/v')
            && format('<host>/<project>/{0}-prod', github.event.repository.name)
            || format('<host>/<project>/{0}-homolog', github.event.repository.name)
        }}
      image-tag: ${{ startsWith(github.ref, 'refs/tags/v') && github.ref_name || github.run_id }}
      do-push: ${{ startsWith(github.ref, 'refs/tags/v') || (github.event_name == 'push' && github.ref == 'refs/heads/homolog') }}
      block-on-failure: ${{ startsWith(github.ref, 'refs/tags/v') || github.event_name == 'pull_request' }}

      # ... stack-specific (node-version, package-manager, lint-command, etc.)

      # auth-method: gcp-wif (se detectado)
      auth-method: 'gcp-wif'
      gcp-workload-identity-provider: '<resolved>'
      gcp-service-account: '<resolved>'

      # gitops-bump (se detectado on_release com bump)
      gitops-bump-enabled: true
      gitops-repo: '<resolved>'
      gitops-bump-on: 'both'
      gitops-path-template: '{repo}-{env}/kustomization.yaml'
      gitops-app-id-var: 'APP_ID'

      notify-channel: none
```

**Regras invariantes**:
- `uses:` sempre pinado em `@vX.Y.Z`. Nunca `@main`.
- Bloco `permissions:` sempre presente. Caller é teto pro reusable.
- `secrets:` explícito sempre que precisar de org secret cross-org.
- Quando `gitops-bump-enabled: true`, **OBRIGATÓRIO** passar `GITOPS_APP_PRIVATE_KEY` no `secrets:` map.
- **Não setar** `dockerfile-target` por default — stages parciais frequentemente não compilam standalone (especialmente Nest). Ative só se o cliente confirmar que a stage roda isolada.

**Saída — onde criar**:
- Se NÃO existe `.github/workflows/ci.yml`: crie como `.github/workflows/ci.yml`.
- Se EXISTE: crie como `.github/workflows/ci-v2.yml` lado a lado durante validação. Marque pra renomear quando o cliente confirmar paridade.
- Se cliente tinha `on_release.yml` com build/push duplicado: marque pra deletar (o reusable absorveu).

## Fase 4 — pré-flight + handoff

**Antes** de declarar pronto, verifique (via `gh api`):

1. **Variables/secrets no repo** (org-level OK):
   - `gh api /repos/<owner>/<repo>/actions/organization-variables` deve incluir a variable referenciada por `gitops-app-id-var` (default `APP_ID`).
   - `gh api /repos/<owner>/<repo>/actions/organization-secrets` deve incluir `GITOPS_APP_PRIVATE_KEY` (se gitops-bump habilitado).
   - Se faltar: pare e peça ao usuário criar antes de mergear.

2. **Kustomizations no GitOps repo** (se gitops-bump habilitado):
   - `gh api /repos/<gitops-owner>/<gitops-name>/contents/<repo>-prod/kustomization.yaml`
   - `gh api /repos/<gitops-owner>/<gitops-name>/contents/<repo>-homolog/kustomization.yaml`
   - Ambos precisam existir, com `images[].newName` apontando pra `<host>/<project>/<repo>-{env}/<repo>-{env}` (path duplo da convenção).
   - Se faltar: pare e peça ao usuário criar antes do primeiro push pra homolog ou tag v*.

3. **Mostre o YAML completo** pra revisão. **NÃO commite, NÃO abra PR** sem aprovação explícita.

4. **Liste secrets adicionais a configurar** (se algum input requerer):
   - `REGISTRY_USERNAME`/`REGISTRY_PASSWORD` se `auth-method: docker-login`.
   - `SLACK_WEBHOOK_URL` se notify slack.
   - `SMTP_HOST/PORT/USERNAME/PASSWORD/FROM` se notify email.

5. **Sugira próximo passo** (criar branch, abrir PR), mas não execute git/gh sem aprovação.

── FIM ──

---

## Manutenção do prompt

Ao bumpar a tag do reusable ou adicionar novos inputs:
- Atualize `examples/*.yml` deste repo (fonte da geração).
- Se uma nova capacidade for **inferível**, adicione na Fase 1.
- Se exigir input não-inferível, adicione na Fase 2 com pergunta clara.
- Se introduzir um secret novo, ajuste o exemplo do bloco `secrets:` na Fase 3.

**Princípio**: toda nova feature começa como auto-detecção. Pergunta é último recurso. Pré-flight (Fase 4) evita falhas que só apareceriam no primeiro deploy.

## Aprendizados do piloto `mintvrs-admin-backend`

Hard-won lessons codificados acima:

1. **Caller permissions block é obrigatório**. Reusable só pode estreitar; ausente, `startup_failure` em segundos.
2. **`secrets: inherit` não propaga organization secrets cross-org**. Use `secrets: { NAME: ${{ secrets.NAME }} }`.
3. **Bash arrays não rodam em `sh`** (container Semgrep usa Alpine sh). Reusable usa `set --` POSIX desde v1.3.1.
4. **Stage parcial do Dockerfile (`dockerfile-target`) pode não compilar standalone**. Default seguro: omitir.
5. **Jobs paralelos > sequenciais**. `node.yml@v1.3.0+` quebra `audit/lint/typecheck/test` em jobs separados pra falhas não mascararem umas às outras.
6. **GitOps bump usa convenção `{repo}-{env}/kustomization.yaml`**. Cliente precisa ter os diretórios e o `newName` da imagem alinhados com o path duplo.
