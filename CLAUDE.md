# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repositório

Repositório **público** que hospeda os **reusable workflows** consumidos por todos os repos de aplicação dos clientes Widesoftware. Não tem código de aplicação. Repos consumidores (em qualquer org) referenciam diretamente via `uses: widesoftware-dev/github-workflows/.github/workflows/<arquivo>.yml@vX.Y.Z`.

Pipeline termina no push da imagem para o registry do cliente. **Deploy é por ArgoCD** observando o registry — nada de `kubectl apply` aqui.

## Arquitetura

```
base.yml (núcleo)
  ├─ scans paralelos: gitleaks, semgrep, trivy fs, trivy config
  ├─ build-push (needs scans) — gate por block-on-failure
  ├─ container-scan (trivy image, após push)
  ├─ pr-report (comenta no PR em failure)
  └─ notify (slack/email com skip silencioso)

php.yml | node.yml | go.yml (wrappers)
  ├─ job `test` (linguagem-específico, com dependency-scan)
  └─ job `pipeline` (needs: test) → uses: ./.github/workflows/base.yml
```

Wrappers chamam `base.yml` por **path relativo** (`./.github/workflows/base.yml`). Resolução é relativa ao repo do reusable, então ao taggear `vX.Y.Z` os wrappers ficam consistentes com o `base.yml` da mesma tag — sem precisar bumpar refs internas.

Frameworks (Next.js, NestJS, Vue, etc.) **não têm wrapper próprio**: reusam `node.yml` via `examples/consumer-<framework>.yml`.

## Convenções obrigatórias

### Workflow injection
**Toda** input usada em `run:` precisa passar por `env:`. Direto `${{ inputs.X }}` em `run:` é proibido (workflow injection). Padrão:
```yaml
env:
  CMD: ${{ inputs.command }}
run: bash -c "$CMD"
```

### Word splitting intencional
Quando precisar de word splitting (ex: `composer install $ARGS`), marca explícito:
```yaml
run: |
  # shellcheck disable=SC2086
  composer install $ARGS
```
Alternativa preferida: array bash (ver `base.yml` job `sast`).

### Block-on-failure
Caller decide por branch: `block-on-failure: ${{ github.ref == 'refs/heads/main' }}`. `true` → falha de scan bloqueia build/push. `false` (homolog) → scans rodam, `continue-on-error` libera build.

### Skip silencioso de notificações
Slack sem `SLACK_WEBHOOK_URL` → log de warning, `exit 0`. SMTP sem `SMTP_HOST` → step pulado via `steps.smtp-check.outputs.enabled`. Nunca falhe o run por credencial de notificação faltando.

### Retenção de artifacts
`retention-days: 15` em **todos** uploads. Não usar 30/90.

### Examples sempre pinados
`examples/consumer-*.yml` referenciam `widesoftware-dev/github-workflows/...@vX.Y.Z`. Nunca `@main` ou SHA. Self-test bloqueia.

### Caller permissions + secrets cross-org
Todo example tem bloco `permissions:` no nível do workflow (caller é teto pro reusable; reusable só estreita). Reusable workflows cross-org **não recebem organization secrets** (nem via `inherit` nem mapping explícito). Tarefas que precisam de org secret (ex: GitOps bump) ficam **fora do reusable**: composite action chamada em job próprio do caller.

### Convenção de naming `-prod` / `-homolog`
Padrão pra clientes Widesoftware: `image-name` e path do registry recebem sufixo do env (`<repo>-prod`, `<repo>-homolog`). `gitops-bump` resolve `{env}` automaticamente. GitOps repo precisa ter dois diretórios separados (`<repo>-prod/`, `<repo>-homolog/`), com `images[].newName` apontando pro path duplo (`<host>/<project>/<repo>-{env}/<repo>-{env}`).

### Sem `dockerfile-target` por default
Stages parciais (`node-builder`, `deps`, etc.) frequentemente não compilam standalone fora da cadeia multi-stage. Default seguro: omitir o input e buildar a imagem inteira. Habilitar só se o cliente confirmar que a stage roda isolada.

### Shell padrão de step em container é `sh` POSIX
Steps que usam `container:` rodam em `sh` (Alpine), não bash. **Arrays bash** (`x=()`, `x+=(...)`, `"${x[@]}"`) quebram. Usar `set --` POSIX (ver job `sast` em `base.yml`).

### Lint, container scan, gitleaks são obrigatórios
A partir de `v1.8.0`: `lint-command` é `required: true`; `run-container-scan` foi removido (sempre roda); gitleaks (`run-secrets-scan`) default `true` e **nunca desligar** — falsos positivos viram allowlist em `.gitleaks.toml`.

### Cargo audit e GitOps bump são composite actions (não vivem no reusable)
Reusable workflows cross-org não recebem org secrets; composite actions sim. Cargo audit também saiu (v1.8.0) pra não aparecer skipped em clientes sem Rust. Cliente declara job próprio em `ci.yml` chamando a composite quando precisa.

### Cache (v1.9.0+)
- Docker: input `docker-cache-mode` aceita `gha` (default), `registry` (Artifact Registry, sem cap) ou `both`. `docker-cache-ref` é o ref do cache no registry — vazio = derivado de `<registry>/<image-name>-cache`.
- `node_modules`: cacheado por hash do lockfile em todos os 4 jobs paralelos do `node.yml`. Automático, sem config no caller.

### Report mk-frontend style
Cada job emite `counts-<tool>.json` artifact + escreve no próprio Step Summary. Job `Report` consolida no PR comment + step summary do workflow. Sempre roda, qualquer outcome.

## Comandos comuns

### Validar workflows local antes de pushar

```sh
bash <(curl -fsSL https://raw.githubusercontent.com/rhysd/actionlint/main/scripts/download-actionlint.bash)
./actionlint
```

Replica o job `actionlint` do `self-test.yml`. Roda `shellcheck` em blocos `run:` automaticamente.

### Verificar pin nos examples

```sh
grep -nE 'widesoftware-dev/github-workflows/' examples/*.yml | grep -vE '@v[0-9]+'
```

Vazio = OK.

### Tag release

```sh
git checkout main && git pull
git tag vX.Y.Z
git push origin vX.Y.Z
```

Semver:
- **MAJOR**: remoção/breaking de input.
- **MINOR**: input opcional novo, novo wrapper, novo example.
- **PATCH**: bugfix em comportamento existente.

Tags `v*` protegidas — só admin cria/deleta (ruleset `semver-tags-protection`).

### Inspecionar run de CI

```sh
gh run list --branch main --limit 5
gh run view <run-id> --log-failed
```

### Branch protection / rulesets

Branch `main` protegida via ruleset `main-protection`: PR obrigatório, 1 approval, dismiss stale, checks `actionlint`+`Validate examples pin`, sem force-push, sem delete. Configurada via `/repos/.../rulesets` (não via UI clássica).

**Context de check exigido = `name:` do job, não o id do job.** O job `validate-examples-pin` do `self-test.yml` tem `name: Validate examples pin`, então o context reportado ao ruleset é `Validate examples pin`. Se o ruleset exigir o id (`validate-examples-pin`), o check nunca reporta e o PR trava em `BLOCKED` esperando status pra sempre — sintoma: "checks não terminam". Ao renomear job ou mudar `name:`, atualizar o context no ruleset junto.

Inspecionar contexts exigidos:

```sh
gh api /repos/widesoftware-dev/github-workflows/rulesets/16535392 \
  --jq '.rules[] | select(.type=="required_status_checks") | .parameters.required_status_checks'
```

Diagnosticar PR travado:

```sh
gh pr view <n> --repo widesoftware-dev/github-workflows --json mergeStateStatus,reviewDecision,statusCheckRollup
```

`mergeStateStatus: BLOCKED` com todos os checks `SUCCESS` e `reviewDecision: APPROVED` = context exigido que nunca reporta.

## O que NÃO fazer

- **Não adicionar deploy** (kubectl, helm, argocd cli) ao pipeline — ArgoCD trata.
- **Não inserir registry/credencial hardcoded** — tudo via secrets do consumidor.
- **Não criar arquivos `.env`** neste repo — `.gitignore` cobre, mas não tente.
- **Não bumpar refs internas manualmente** entre `php.yml`/`node.yml`/`go.yml` e `base.yml` — path relativo já resolve.
- **Não usar `@main` em examples** — self-test bloqueia.
- **Não rodar `gh repo create`/`gh repo edit --visibility`** sem confirmação explícita do usuário.
- **Não commitar/pushar** sem o usuário pedir.
- **Mensagens de commit em português, Conventional Commits**, sem `Co-Authored-By Claude`.

## Comunicação

Português brasileiro. Direto. Pergunte antes de assumir defaults não óbvios (versão de runtime, registry, ferramenta de SAST, política de notificação).
