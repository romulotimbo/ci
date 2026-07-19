# romulotimbo/ci

Workflows reutilizáveis de CI/CD compartilhados entre os projetos da VPS `romulohub`.

## `build-and-deploy`

Builda imagens Docker no runner do GitHub, publica no GHCR e faz deploy na VPS por SSH
(a VPS executa `docker compose pull` + `up -d`). Opcionalmente sincroniza código host
via rsync (útil quando o app também roda Python/scripts no host fora do container).

### Como usar num projeto

Crie `.github/workflows/deploy.yml` no repositório do projeto:

```yaml
name: Deploy
on:
  push:
    branches: [main]
jobs:
  deploy:
    uses: romulotimbo/ci/.github/workflows/deploy.yml@main
    with:
      app: meu-app
      deploy_dir: /srv/data/meu-app
      compose_file: docker-compose.prod.yml
      migrate_service: migrate            # opcional; vazio = pular
      sync_host_code: true               # opcional; rsync do checkout → VPS
      images: '[{"suffix":"","dockerfile":"Dockerfile","context":"."}]'
    secrets: inherit
```

### Inputs

| Input | Obrigatório | Descrição |
|-------|-------------|-----------|
| `app` | sim | Base da imagem: `ghcr.io/romulotimbo/<app><suffix>` |
| `images` | sim | JSON: lista de `{suffix,dockerfile,context,build_args}` |
| `deploy_dir` | sim | Caminho do projeto na VPS |
| `compose_file` | não | Padrão `docker-compose.prod.yml` |
| `migrate_service` | não | Service de migração a rodar antes do `up` |
| `services` | não | Limita `pull`/`up` a serviços específicos (evita tocar em compartilhados, ex.: `postgres`) |
| `sync_host_code` | não | Default `false`. Se `true`, rsync do checkout do caller para `deploy_dir` (sem `--delete`; preserva `.env`, `data*`, `.venv`, `logs`) e grava `DEPLOYED_SHA` |
| `pip_install` | não | Default `true`. Com `sync_host_code`, roda `.venv/bin/pip install -r requirements.txt` se existirem |

### Secrets (por repositório)

Conta pessoal não tem org-secrets; defina em cada repo (ou via `gh secret set`):

| Secret | Valor |
|--------|-------|
| `VPS_HOST` | IP da VPS |
| `VPS_PORT` | Porta SSH |
| `VPS_USER` | Usuário SSH |
| `VPS_SSH_KEY` | Chave privada de deploy |
| `GHCR_TOKEN` | PAT com `read:packages` (login na VPS antes do pull) |

### Pré-requisitos na VPS

- `docker login ghcr.io` feito uma vez (PAT com `read:packages`) para puxar imagens privadas.
- Compose do projeto referencia `image: ghcr.io/romulotimbo/<app>...` (sem bloco `build:`).
- Com `sync_host_code`: diretório `deploy_dir` já existe; `.env` / `data*` / `.venv` ficam fora do rsync.
