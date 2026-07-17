# romulotimbo/ci

Workflows reutilizáveis de CI/CD compartilhados entre os projetos da VPS `romulohub`.

## `build-and-deploy`

Builda imagens Docker no runner do GitHub, publica no GHCR e faz deploy na VPS por SSH
(a VPS apenas executa `docker compose pull` + `up -d`, sem buildar).

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

### Secrets (por repositório)

Conta pessoal não tem org-secrets; defina em cada repo (ou via `gh secret set`):

| Secret | Valor |
|--------|-------|
| `VPS_HOST` | IP da VPS |
| `VPS_PORT` | Porta SSH |
| `VPS_USER` | Usuário SSH |
| `VPS_SSH_KEY` | Chave privada de deploy |

### Pré-requisitos na VPS

- `docker login ghcr.io` feito uma vez (PAT com `read:packages`) para puxar imagens privadas.
- Compose do projeto referencia `image: ghcr.io/romulotimbo/<app>...` (sem bloco `build:`).
