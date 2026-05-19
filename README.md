# widesoftware-dev/github-workflows

Reusable workflows do GitHub Actions usados pelos repositórios de aplicação dos clientes da Widesoftware. Fonte única — sem mirror, sem sync, sem duplicação.

## 1. Visão geral

Este repositório é a **fonte única de verdade** para o pipeline de CI/CD dos clientes. Cada repo de aplicação referencia diretamente um workflow daqui, pinado em uma tag semver.

O pipeline cobre:

- Scans de segurança (segredos, SAST, vulnerabilidades, misconfig, container).
- Testes específicos da linguagem (PHP/Composer ou Node/Yarn).
- Build e push da imagem Docker para o registry do cliente.
- Notificações por Slack e/ou email.
- Comentário automatizado no PR em caso de falha.

**Deploy não está aqui.** Todos os clientes usam ArgoCD observando o registry — o push da imagem é o sinal pra rolar uma nova versão.

## 2. Arquitetura

```
┌─────────────────────────────────────────────────────────────────────┐
│  widesoftware-dev/github-workflows  (este repo, privado)            │
│                                                                     │
│    .github/workflows/                                               │
│      base.yml       scans + build + push                            │
│      php.yml        test PHP -> base.yml                            │
│      node.yml       test Node -> base.yml                           │
└─────────────────────────────────────────────────────────────────────┘
                              ▲
                              │  uses: ...@v1.2.3
                              │
┌─────────────────────────────────────────────────────────────────────┐
│  Repo do cliente (qualquer org)                                     │
│    .github/workflows/ci.yml  (cópia de examples/consumer-*.yml)     │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼  push da imagem
┌─────────────────────────────────────────────────────────────────────┐
│  Registry do cliente                                                │
└─────────────────────────────────────────────────────────────────────┘
                              │  observado por
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│  ArgoCD (cluster do cliente)  -> aplica nova versão                 │
└─────────────────────────────────────────────────────────────────────┘
```

Este repositório **não faz deploy**. Para o ArgoCD, o registry é a fonte da verdade.

## 3. Visibilidade e acesso cross-org

Este repositório é **privado** e vive na organização `widesoftware-dev`.

Implicações:

- **Repos de aplicação dentro da org `widesoftware-dev`**: consumo livre via `uses:`, sem token.
- **Repos de aplicação em outras orgs**: o GitHub exige autenticação cross-org. Opções:
  1. **Mover o repo cliente** pra org `widesoftware-dev` (recomendado quando possível — consumo sem token).
  2. **Configurar acesso explícito** em `Settings → Actions → General → Access` permitindo o repo consumidor (só funciona dentro do mesmo Enterprise GitHub).
  3. **GitHub App ou PAT** com escopo `repo` salvo como secret no repo consumidor, usado em `actions/checkout` antes do `uses:`.

Workflows não contêm segredos em si — todos os secrets reais (registry, Slack, SMTP) ficam armazenados em cada repo consumidor.

## 4. Estrutura do repositório

```
.
├── .github/workflows/
│   ├── base.yml             # reusable: scans + build + push da imagem
│   ├── php.yml              # reusable: testes PHP + chama base.yml
│   ├── node.yml             # reusable: testes Node + chama base.yml
│   ├── go.yml               # reusable: testes Go + chama base.yml
│   └── self-test.yml        # CI próprio: actionlint + valida pin nos examples
├── examples/
│   ├── consumer-php.yml     # template PHP/Laravel
│   ├── consumer-node.yml    # template Node genérico
│   ├── consumer-nextjs.yml  # template Next.js (reusa node.yml)
│   ├── consumer-nestjs.yml  # template NestJS (reusa node.yml)
│   └── consumer-go.yml      # template Go
├── .gitignore
└── README.md                # este arquivo
```

## 5. Workflows disponíveis

### `base.yml`

Núcleo do pipeline. Pode ser chamado direto (sem testes de linguagem) ou via os wrappers `php.yml`/`node.yml`.

| Input | Tipo | Default | Descrição |
|---|---|---|---|
| `image-name` | string | — (obrig.) | Nome curto da imagem |
| `registry` | string | — (obrig.) | Host do registry (ex: `registry.cliente.com.br`) |
| `image-tag` | string | `''` | Tag da imagem (vazio = SHA curto) |
| `dockerfile` | string | `Dockerfile` | Caminho do Dockerfile |
| `dockerfile-target` | string | `''` | Stage do Dockerfile multi-stage |
| `build-args` | string | `''` | Build args (KEY=value por linha) |
| `build-context` | string | `.` | Contexto do build |
| `platforms` | string | `linux/amd64` | Plataformas alvo do buildx |
| `do-push` | boolean | `true` | Faz push (false = só builda) |
| `block-on-failure` | boolean | `true` | Falha de scan bloqueia build/push |
| `run-secrets-scan` | boolean | `true` | gitleaks |
| `run-vuln-scan` | boolean | `true` | Trivy FS + Trivy Config |
| `run-sast` | boolean | `true` | Semgrep |
| `run-dependency-scan` | boolean | `true` | Composer/Yarn audit (efeito nos wrappers) |
| `run-container-scan` | boolean | `true` | Trivy na imagem buildada |
| `semgrep-configs` | string | `p/owasp-top-ten` | Configs Semgrep (separadas por espaço) |
| `pr-report` | boolean | `true` | Comenta no PR em failure |
| `notify-channel` | string | `none` | `none` \| `slack` \| `email` \| `both` |
| `notify-on` | string | `failure` | `failure` \| `always` |
| `slack-channel` | string | `''` | Nome do canal (opcional) |
| `email-to` | string | `''` | Destinatários separados por vírgula |
| `auth-method` | string | `docker-login` | `docker-login` (user/pass) ou `gcp-wif` (OIDC pra GCP Artifact Registry) |
| `gcp-workload-identity-provider` | string | `''` | WIF provider full path. Só `gcp-wif` |
| `gcp-service-account` | string | `''` | Email da SA GCP. Só `gcp-wif` |
| `cargo-audit-paths` | string | `''` | Diretórios com `Cargo.lock` separados por vírgula (ex: `blockchain`). Roda `cargo audit` em cada. Pula se vazio |

### GitOps bump — composite action

A partir de `v1.5.0`, o GitOps bump **não vive mais dentro do reusable workflow**. Foi extraído pra uma **composite action** chamada pelo caller em um job próprio:

```yaml
jobs:
  ci:
    uses: widesoftware-dev/github-workflows/.github/workflows/node.yml@v1.5.0
    ...
  gitops:
    needs: ci
    if: ${{ startsWith(github.ref, 'refs/tags/v') || (github.event_name == 'push' && github.ref == 'refs/heads/homolog') }}
    runs-on: ubuntu-latest
    steps:
      - uses: widesoftware-dev/github-workflows/.github/actions/gitops-bump@v1.5.0
        with:
          gitops-repo: 'mkclub69/mk-microservice-ops'
          app-id: ${{ vars.APP_ID }}
          app-private-key: ${{ secrets.APP_PRIVATE_KEY }}
          image-name: <expression>
          image-tag: <expression>
```

**Por quê separar**: reusable workflows em outra organização **não recebem organization secrets** do caller (mesmo com `secrets: inherit` ou mapping explícito). Composite actions rodam no mesmo job do caller, então a private key flui normal como input. Veja `examples/consumer-nestjs.yml` pro template completo.

Inputs da composite action:

| Input | Default | Notes |
|---|---|---|
| `gitops-repo` | (req.) | `owner/name` |
| `app-id` | (req.) | GitHub App ID |
| `app-private-key` | (req.) | Private key PEM (passar via `${{ secrets.X }}`) |
| `image-name` | (req.) | Pode ter sufixo `-prod`/`-homolog`; o action tira pra derivar `{repo}` |
| `image-tag` | (req.) | Valor que vai no `newTag` |
| `path-template` | `{repo}-{env}/kustomization.yaml` | placeholders `{repo}` `{env}` |
| `commit-message-template` | `chore({repo}): update {env} image to {tag}` | placeholders `{repo}` `{env}` `{tag}` |
| `env-override` | `''` | Força `prod` ou `homolog`; vazio = deriva do `github.ref` |

Secrets necessários (definidos no repo consumidor):

| Secret | Quando |
|---|---|
| `REGISTRY_USERNAME`, `REGISTRY_PASSWORD` | Se `do-push: true` |
| `SLACK_WEBHOOK_URL` | Se `notify-channel` ∈ {slack, both} |
| `SMTP_HOST`, `SMTP_PORT`, `SMTP_USERNAME`, `SMTP_PASSWORD`, `SMTP_FROM` | Se `notify-channel` ∈ {email, both} |

### `php.yml`

Wrapper PHP/Laravel. Aceita todos os inputs do `base.yml` mais:

| Input | Tipo | Default | Descrição |
|---|---|---|---|
| `php-version` | string | `8.3` | Versão do PHP |
| `php-extensions` | string | `mbstring, xml, ...` | Extensões (csv) |
| `composer-args` | string | `--no-interaction ...` | Args do composer install |
| `test-command` | string | `vendor/bin/pest` | Comando de testes |
| `run-static-analysis` | boolean | `false` | Roda phpstan |
| `run-code-style` | boolean | `false` | Roda Pint |

Default de `semgrep-configs` é `p/owasp-top-ten p/php`.

### `node.yml`

Wrapper Node/TypeScript. Aceita todos os inputs do `base.yml` mais:

| Input | Tipo | Default | Descrição |
|---|---|---|---|
| `node-version` | string | `22` | Versão do Node |
| `package-manager` | string | `yarn` | `yarn`, `npm` ou `pnpm` |
| `pnpm-version` | string | `9` | Versão do pnpm (só usado se `package-manager=pnpm`) |
| `install-command` | string | `''` | Vazio = `yarn install --immutable`, `npm ci` ou `pnpm install --frozen-lockfile` |
| `pre-install-command` | string | `''` | Roda antes do setup-node (ex: `corepack enable` pra yarn 4). pnpm já é resolvido pelo `pnpm/action-setup` |
| `lint-command` | string | `''` | Vazio = pula |
| `typecheck-command` | string | `''` | Vazio = pula |
| `test-command` | string | `''` | Vazio = pula |

Default de `semgrep-configs` é `p/owasp-top-ten p/typescript p/react`.

### `go.yml`

Wrapper Go. Aceita todos os inputs do `base.yml` mais:

| Input | Tipo | Default | Descrição |
|---|---|---|---|
| `go-version` | string | `1.23` | Versão do Go |
| `go-version-file` | string | `''` | `go.mod` para extrair versão (prevalece sobre `go-version`) |
| `test-command` | string | `go test -race -count=1 ./...` | Comando de testes |
| `build-tags` | string | `''` | Build tags do Go (csv) |
| `run-go-vet` | boolean | `true` | Roda `go vet ./...` |
| `run-staticcheck` | boolean | `false` | Roda staticcheck |
| `run-golangci-lint` | boolean | `false` | Roda golangci-lint (precisa de `.golangci.yml`) |

`run-dependency-scan` aciona `govulncheck` (ferramenta oficial Go). Default de `semgrep-configs` é `p/owasp-top-ten p/golang`.

### Frameworks que reusam wrappers

Não há wrapper dedicado para Next.js, NestJS, Vue, etc. — eles rodam em cima do `node.yml`. Ver `examples/consumer-nextjs.yml` e `examples/consumer-nestjs.yml` para configurações específicas (lint/typecheck/test command, build-args, semgrep-configs).

## ⚠️ Caller permissions (obrigatório)

GitHub Actions impõe que o **caller** define o limite superior de permissões pra qualquer reusable workflow. Sem o bloco abaixo, o reusable falha com `startup_failure` ao tentar usar WIF, ou roda mas não consegue postar PR report:

```yaml
permissions:
  contents: read
  id-token: write       # obrigatório se auth-method: gcp-wif
  pull-requests: write  # obrigatório pro pr-report comentar no PR
  packages: write       # obrigatório se push pra ghcr.io
  actions: read         # obrigatório pro pr-report ler logs de jobs falhos
```

Esse bloco está em todos os `examples/consumer-*.yml`.

## ⚠️ Secrets cross-org: reusable workflows não recebem org secrets

Reusable workflows em outra organização **não recebem organization secrets** do caller, **nem via `secrets: inherit` nem via mapping explícito**. Variables (`vars.*`) e repo-level secrets fluem; org secrets não.

Casos onde isso bate:
- `REGISTRY_USERNAME`/`REGISTRY_PASSWORD` (auth-method `docker-login`): se forem org secrets, copiar pro repo do caller.
- `SLACK_WEBHOOK_URL`, `SMTP_*` (notifications): idem.
- **GitHub App private key (GitOps bump)**: a forma correta é **NÃO usar o reusable workflow pra esse step** e sim a composite action `gitops-bump`, que roda no job do caller e enxerga qualquer secret normalmente. Veja `examples/consumer-nestjs.yml`.

`secrets: inherit` continua útil pra repos same-org e pra repo-level secrets cross-org.

## Convenção de naming `-prod` / `-homolog`

Padrão recomendado pra clientes Widesoftware: aplicar o sufixo do environment **no `image-name` e no path do registry** simultaneamente. Resultado final por evento:

| Evento | image-name | registry | image-tag | Ref completo |
|---|---|---|---|---|
| `tag v*` | `<repo>-prod` | `<host>/<project>/<repo>-prod` | `<ref_name>` | `.../<repo>-prod/<repo>-prod:v1.0.0` |
| `push homolog` | `<repo>-homolog` | `<host>/<project>/<repo>-homolog` | `<run_id>` | `.../<repo>-homolog/<repo>-homolog:1234567890` |
| `pull_request` / `push main` | (n/a — sem push) |

O `gitops-bump` derive `{env}` (prod/homolog) automaticamente do `github.ref`/`event_name`. O `gitops-path-template` default `{repo}-{env}/kustomization.yaml` espera que o GitOps repo tenha **dois diretórios separados** por env, cada um com `newName` apontando pro path duplo. Veja `examples/consumer-nestjs.yml` pro template completo de uma adoção real.

## 6. Como adotar em um repo cliente

### Caminho rápido (Claude Code)

Existe um prompt pronto em [`docs/onboard-prompt.md`](docs/onboard-prompt.md). Abra o Claude Code **dentro do repositório do cliente**, copie o bloco indicado no arquivo, preencha as respostas (linguagem, registry, notificações, etc.) e cole. O Claude valida, gera o `.github/workflows/ci.yml` baseado nos `examples/` desta versão e devolve checklist de secrets a configurar.

### Caminho manual

1. **Copiar template**. Pegue `examples/consumer-php.yml` ou `examples/consumer-node.yml` deste repo e salve como `.github/workflows/ci.yml` no repo da aplicação.

2. **Configurar secrets** no repo (Settings → Secrets and variables → Actions). Tabela por cenário:

   | Cenário | Secrets necessários |
   |---|---|
   | Só push pra registry | `REGISTRY_USERNAME`, `REGISTRY_PASSWORD` |
   | + Slack | adiciona `SLACK_WEBHOOK_URL` |
   | + Email | adiciona `SMTP_HOST`, `SMTP_PORT`, `SMTP_USERNAME`, `SMTP_PASSWORD`, `SMTP_FROM` |
   | Ambos | tudo acima |

3. **Ajustar inputs** do `with:` no `ci.yml`:
   - `image-name`, `registry` — obrigatórios.
   - `notify-channel`, `slack-channel`, `email-to` — conforme escolha.
   - Toggles de segurança em `false` se for projeto legado que ainda não passa.

4. **Pinar versão**. Use `@v1.0.0` (ou versão atual). Nunca `@main`.

5. **Garantir branches** `main` e `homolog` existindo. O exemplo usa o ref pra decidir `block-on-failure` e `do-push`.

## 7. Toggles de segurança

Cada scan tem um toggle. Cliente novo herda tudo ligado. Cliente legado pode desligar o que ainda não passa e ir reativando aos poucos.

| Toggle | Desliga quando | Como religar |
|---|---|---|
| `run-secrets-scan` | gitleaks acusou falsos positivos antigos | Adicionar `.gitleaks.toml` com `allowlist`, depois ativar |
| `run-vuln-scan` | Trivy quebra em CVE conhecidos sem patch | Rodar `composer/yarn update`, depois ativar |
| `run-sast` | Semgrep quebra em padrões legados | Refatorar pontos críticos, ou adicionar `.semgrepignore`, depois ativar |
| `run-dependency-scan` | `composer/yarn audit` quebra em deps desatualizadas | Bump das deps, depois ativar |
| `run-container-scan` | Imagem base tem CVEs antigos | Atualizar base do Dockerfile, depois ativar |

**Recomendação**: ativar 1 toggle por sprint. Não desligue tudo e nunca religue.

Modo `block-on-failure`: em `homolog`, as falhas viram warning e o build segue. Em `main`, qualquer falha bloqueia o push. O exemplo usa `${{ github.ref == 'refs/heads/main' }}` pra decidir automaticamente.

## 8. Notificações

### Slack

1. Crie um Slack App em https://api.slack.com/apps → "Incoming Webhooks" → ativar → "Add New Webhook to Workspace" → escolher canal.
2. Copie a URL gerada.
3. Salve no repo consumidor como `SLACK_WEBHOOK_URL`.
4. No `ci.yml`:
   ```yaml
   notify-channel: slack
   notify-on: failure
   slack-channel: '#meu-canal'   # opcional, depende do webhook
   ```

> Webhooks classic publicam num canal fixo. Webhooks Bot Token aceitam override de canal via payload. O input `slack-channel` é repassado mas só funciona com Bot Token.

### Email

Providers comuns e como obter SMTP:

| Provider | Host | Porta | Auth |
|---|---|---|---|
| Gmail | `smtp.gmail.com` | `587` | App Password (não a senha normal) |
| AWS SES | `email-smtp.<region>.amazonaws.com` | `587` | SMTP credentials (IAM) |
| SendGrid | `smtp.sendgrid.net` | `587` | API key como password, `apikey` como username |
| Mailgun | `smtp.mailgun.org` | `587` | SMTP credentials do dashboard |

Salve `SMTP_HOST`, `SMTP_PORT`, `SMTP_USERNAME`, `SMTP_PASSWORD`, `SMTP_FROM` como secrets. No `ci.yml`:

```yaml
notify-channel: email
notify-on: failure
email-to: 'dev@cliente.com,ops@cliente.com'
```

### Ambos

```yaml
notify-channel: both
slack-channel: '#ci'
email-to: 'ops@cliente.com'
```

## 9. Versionamento

Versionamento segue **semver**:

- `MAJOR` — breaking changes (input removido, comportamento incompatível).
- `MINOR` — features novas, retrocompatíveis (input opcional novo).
- `PATCH` — bugfix.

Como criar uma release:

```sh
git tag v1.2.0
git push --tags
```

**Consumidores devem pinar em tag fixa** (`@v1.2.0`), nunca em `@main`. Pinagem garante que uma mudança aqui não quebra silenciosamente os clientes. O job `validate-examples-pin` do self-test bloqueia PRs que violem essa regra.

> **Bump em wrappers internos**: `php.yml` e `node.yml` chamam `base.yml` via path relativo (`./.github/workflows/base.yml`). Isso garante que ao tagear `v1.2.0`, todos os wrappers usem o `base.yml` da mesma tag — sem necessidade de bump manual entre eles.

## 10. Como atualizar um workflow

1. Criar branch `feat/...` ou `fix/...` neste repo.
2. Editar workflow.
3. Abrir PR pra `main`. O `self-test.yml` roda actionlint e valida examples.
4. Merge depois de review.
5. Criar tag semver:
   ```sh
   git checkout main && git pull
   git tag v1.2.0
   git push --tags
   ```
6. **Comunicar consumidores** ou deixar o Renovate automatizar o bump.

### Auto-bump no consumidor (Renovate)

Adicione ao `renovate.json` do repo consumidor:

```json
{
  "github-actions": {
    "enabled": true
  },
  "packageRules": [
    {
      "matchPackageNames": ["widesoftware-dev/github-workflows"],
      "groupName": "widesoftware workflows"
    }
  ]
}
```

Renovate abre PR a cada nova tag publicada.

## 11. Troubleshooting

| Sintoma | Causa provável | Fix |
|---|---|---|
| `Error: secret X is required` | Caller não passou `secrets: inherit` | Adicionar `secrets: inherit` no job que chama |
| `unauthorized: authentication required` no push | `REGISTRY_USERNAME` ou `REGISTRY_PASSWORD` faltando/errado | Conferir secrets do repo. Para ghcr.io, username = `${{ github.actor }}`, password = `${{ secrets.GITHUB_TOKEN }}` ou PAT com `write:packages` |
| Slack não chega | `SLACK_WEBHOOK_URL` não setado (skip silencioso) ou canal não tem o app | Conferir warnings no log; reinstalar app no canal |
| Email não chega | `SMTP_HOST` não setado (skip silencioso) ou SMTP recusa auth | Conferir log; testar credenciais com `swaks` localmente |
| Build do PR falha em "Resource not accessible by integration" | Permissão de PR write não habilitada no repo | Settings → Actions → Workflow permissions → "Read and write" + "Allow GitHub Actions to create and approve PRs" |
| `Permission denied: composer audit` falha em vuln conhecida sem patch | Dep transitiva sem versão segura | Setar `run-dependency-scan: false` temporariamente e abrir issue pra bump |
| Workflow não encontra `base.yml` | Path errado ou versão ainda não taggeada | Pinar em tag que já existe (`v1.0.0`+) |
| `actionlint` reclama de expressão | Sintaxe inválida em `if:` ou interpolação | Rodar `actionlint` local: `brew install actionlint && actionlint` |
| `Error: Input required and not supplied: private-key` no `gitops-bump` | Reusable workflow cross-org não recebe org secrets — restrição confirmada do GitHub Actions | Não usar `gitops-bump` no reusable. Migrar pra composite action `gitops-bump` em job próprio do caller (v1.5.0+) |
| `startup_failure` em segundos, sem nenhum job rodar | Caller sem bloco `permissions:`, ou reusable pede permission que o caller não concede | Garantir o bloco `permissions:` (em "Caller permissions"); incluir `id-token: write` se WIF, `actions: read` pro pr-report |
| `syntax error: unexpected "("` dentro do Semgrep | Reusable < `v1.3.1` usando array bash em container `sh` | Bumpar pin pra `v1.3.1`+ |
| `gitops-bump` falha com "kustomization nao encontrado" | Path no GitOps repo não bate com `gitops-path-template` resolvido | Criar `<repo>-prod/kustomization.yaml` e `<repo>-homolog/kustomization.yaml` no GitOps repo, com `newName` apontando pro path duplo |

---

Dúvidas ou propostas de mudança: abrir issue ou PR neste repositório.
