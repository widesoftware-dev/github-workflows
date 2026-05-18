# Onboarding Prompt — Widesoftware CI/CD

Prompt pronto pra você colar no Claude Code rodando **dentro do repositório do cliente**. O Claude lê as respostas que você preencher, cria o `.github/workflows/ci.yml`, lista os secrets necessários e te entrega um checklist final.

## Como usar

1. Abra o **Claude Code no repositório do cliente** (não neste repo).
2. Copie tudo abaixo a partir do `── COPIE A PARTIR DAQUI ──`.
3. Preencha os campos `[...]` com as respostas do cliente.
4. Cole no Claude.
5. Aprove as ações que ele propor (criação do `ci.yml`, possível `Dockerfile` placeholder etc.).

---

── COPIE A PARTIR DAQUI ──

Você está rodando dentro do repositório de aplicação de um cliente Widesoftware. Sua tarefa é configurar o CI/CD usando os reusable workflows do repo `widesoftware-dev/github-workflows`. Não invente nada — use exatamente os exemplos do repo público em `https://github.com/widesoftware-dev/github-workflows/tree/main/examples`.

Antes de qualquer escrita:

1. Leia as respostas abaixo.
2. Detecte conflitos (linguagem não suportada, registry vazio, etc.) e me pergunte antes de seguir.
3. Liste o plano em uma linha por etapa antes de aplicar.

## Respostas do cliente

```yaml
# ── Identificação ─────────────────────────────────────────────────────────
cliente: "[nome curto, ex: cliente-a]"
projeto: "[nome do app, ex: meu-app-api]"

# ── Stack ─────────────────────────────────────────────────────────────────
# Opções: php | node | nextjs | nestjs | go
linguagem: "[php|node|nextjs|nestjs|go]"
# Versão da linguagem. Ex: 8.3 (PHP), 22 (Node), 1.23 (Go).
versao_linguagem: "[ex: 8.3]"

# ── Imagem & registry ─────────────────────────────────────────────────────
image_name: "[ex: meu-app-api]"
registry: "[ex: registry.cliente.com.br ou ghcr.io/cliente-a]"
dockerfile: "Dockerfile"           # ajuste se o cliente usa outro path
dockerfile_target: ""              # vazio = última stage; ex: 'production', 'release'
platforms: "linux/amd64"           # ou 'linux/amd64,linux/arm64'

# ── Branches ──────────────────────────────────────────────────────────────
# Habilita 'homolog' como branch não-bloqueante além da 'main'?
usa_homolog: "[sim|nao]"

# ── Comandos (preencha só os relevantes pra linguagem) ────────────────────
test_command: "[ex: vendor/bin/pest | yarn test | go test ./...]"
lint_command: "[opcional, ex: yarn lint]"
typecheck_command: "[opcional, ex: yarn tsc --noEmit]"
# Para PHP apenas:
php_run_static_analysis: "[sim|nao]"   # phpstan/larastan
php_run_code_style: "[sim|nao]"        # Pint
# Para Go apenas:
go_run_staticcheck: "[sim|nao]"
go_run_golangci_lint: "[sim|nao]"

# ── Toggles de segurança ──────────────────────────────────────────────────
# Default: todos true. Liste apenas os que o cliente precisa desabilitar
# por ser legado (separados por vírgula). Vazio = todos true.
# Valores válidos: secrets, vuln, sast, dependency, container
desabilitar_scans: "[ex: sast,dependency  |  vazio]"

# ── Notificações ──────────────────────────────────────────────────────────
# Opções: none | slack | email | both
notify_channel: "[none|slack|email|both]"
notify_on: "[failure|always]"
slack_channel: "[ex: #ci-cliente-a, ou vazio]"
email_to: "[ex: dev@cliente.com,ops@cliente.com, ou vazio]"

# ── Versão do reusable workflow ───────────────────────────────────────────
# Pinar em tag semver. Consulte https://github.com/widesoftware-dev/github-workflows/tags
versao_workflow: "[ex: v1.1.0]"

# ── Build args do Docker (opcional, multiline) ────────────────────────────
build_args: |
  # APP_ENV=production
  # NEXT_PUBLIC_API_URL=https://api.cliente.com
```

## O que você (Claude) deve fazer

1. **Validar**:
   - `linguagem` deve ser uma das suportadas.
   - `image_name`, `registry`, `versao_workflow` não podem estar vazios.
   - `versao_workflow` deve ser uma tag semver real do repo. Verificar com `gh api /repos/widesoftware-dev/github-workflows/tags --jq '.[].name'`.
   - Se `notify_channel ∈ {slack, both}` mas `slack_channel` vazio → confirmar comigo.
   - Se `notify_channel ∈ {email, both}` mas `email_to` vazio → confirmar comigo.

2. **Buscar template** correspondente à `linguagem` em `https://raw.githubusercontent.com/widesoftware-dev/github-workflows/<versao_workflow>/examples/consumer-<linguagem>.yml`.

3. **Gerar `.github/workflows/ci.yml`** baseado no template, aplicando:
   - Substituir `image-name`, `registry`, `php-version` / `node-version` / `go-version`, etc.
   - Substituir `@v1.0.0` (e similares) pela `versao_workflow` informada.
   - `do-push: ${{ github.event_name == 'push' }}` (mantém padrão dos examples).
   - `block-on-failure: ${{ github.ref == 'refs/heads/main' }}` se `usa_homolog == sim`; caso contrário, `block-on-failure: true` fixo.
   - Ajustar `branches:` no `on:` conforme `usa_homolog`.
   - Para cada item em `desabilitar_scans`, setar `run-<item>-scan: false` (mapear: `secrets→run-secrets-scan`, `vuln→run-vuln-scan`, `sast→run-sast`, `dependency→run-dependency-scan`, `container→run-container-scan`).
   - Configurar `notify-channel`, `notify-on`, `slack-channel`, `email-to`.
   - Aplicar `build_args` se preenchido.
   - Para PHP: ativar `run-static-analysis` / `run-code-style` conforme respostas.
   - Para Go: ativar `run-staticcheck` / `run-golangci-lint` conforme respostas.

4. **Pré-requisitos do repo**:
   - Verificar se `Dockerfile` existe no path indicado. Se não, me alertar (não criar placeholder sem aprovação).
   - Verificar se a stack já tem manifesto (`composer.json` / `package.json` / `go.mod`). Se ausente, me alertar.

5. **Listar secrets que o cliente precisa criar** no repo, com base nas respostas:
   - Sempre: `REGISTRY_USERNAME`, `REGISTRY_PASSWORD`.
   - Se Slack: `SLACK_WEBHOOK_URL`.
   - Se Email: `SMTP_HOST`, `SMTP_PORT`, `SMTP_USERNAME`, `SMTP_PASSWORD`, `SMTP_FROM`.

6. **Entregar um checklist final** com:
   - Path do arquivo criado.
   - Secrets a configurar (link do Settings → Actions → Secrets).
   - Como abrir PR de adoção.
   - Como configurar Renovate/Dependabot pra auto-bump da tag do reusable.

7. **Não commitar nem abrir PR sozinho**. Mostrar o `ci.yml` gerado pra eu revisar antes.

── COPIE ATÉ AQUI ──

---

## Manutenção

Quando a tag do reusable (`widesoftware-dev/github-workflows`) ganhar novos inputs ou novos wrappers, atualize:

- `versao_workflow` recomendada no bloco de respostas.
- Validações novas em "O que você (Claude) deve fazer".

Mantenha o prompt em sincronia com `examples/*.yml` deste repo.
