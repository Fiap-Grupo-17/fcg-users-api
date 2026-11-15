# FCG Users API — Microsserviço de Usuários

Microsserviço responsável por **cadastro, autenticação (JWT) e autorização** de usuários da plataforma FIAP Cloud Games (FCG).

## Responsabilidades

- Registro de usuários comuns e administradores
- Autenticação via JWT (access token 15min + refresh token rotativo 7 dias)
- Listagem e promoção de usuários (admin)
- Publicação de `UserCreatedEvent` via RabbitMQ (consumido pelo NotificationsAPI)

## Stack

| Componente | Tecnologia |
|---|---|
| Framework | .NET 8 Minimal API |
| ORM | EF Core 8 + Npgsql (PostgreSQL) |
| Mensageria | MassTransit + RabbitMQ |
| Auth | JWT (HMAC SHA256) + Refresh Token |
| Hash | BCrypt (WorkFactor 12) |
| Docs | Swagger / OpenAPI |

## Endpoints

| Método | Rota | Auth | Descrição |
|---|---|---|---|
| POST | `/api/auth/login` | público | Autentica e retorna tokens |
| POST | `/api/auth/refresh` | público | Renova access token |
| POST | `/api/auth/logout` | autenticado | Invalida refresh token |
| GET | `/api/usuarios` | admin | Lista usuários |
| POST | `/api/usuarios/registrar` | público | Registra usuário comum |
| POST | `/api/usuarios/administradores` | admin | Cria administrador |
| PATCH | `/api/usuarios/{id}/promover` | admin | Promove para admin |
| GET | `/health` | público | Health check |

## Variáveis de Ambiente

| Variável | Descrição | Padrão |
|---|---|---|
| `ConnectionStrings__Postgres` | Connection string do PostgreSQL | — (via Secret/.env) |
| `Jwt__Secret` | Secret HMAC SHA256 (mín. 32 chars) | — (via Secret/.env) |
| `Jwt__Issuer` | Issuer do token | `fcg-users-api` |
| `Jwt__Audience` | Audience do token | `fcg-platform` |
| `RabbitMQ__Host` | Host do RabbitMQ | `localhost` |
| `RabbitMQ__Username` | Usuário RabbitMQ | `guest` |
| `RabbitMQ__Password` | Senha RabbitMQ | — (via Secret/.env) |
| `SeedAdmin__Email` | E-mail do admin inicial | `admin@fcg.com` |
| `SeedAdmin__Senha` | Senha do admin inicial | — (via Secret/.env) |

## Segredos

O repositório é público, então **nenhuma credencial é versionada**. Os arquivos
`.env` (Docker Compose) e `k8s/secret.yaml` (Kubernetes) estão no `.gitignore`.
O repositório traz apenas os modelos `.env.example` e `k8s/secret.example.yaml`,
com placeholders. Em produção (AKS), os valores são aleatórios, criados fora do Git.

## CI/CD (GitHub Actions)

O workflow `.github/workflows/ci.yml` tem dois jobs:

- **build-and-test:** restaura, compila e roda os testes em todo push e pull request para a `main`. Os testes de integração usam um RabbitMQ como serviço do job.
- **deploy:** roda só em push na `main`. Faz login no Azure por OIDC (sem senha guardada), constrói a imagem, roda o scan de vulnerabilidades (Trivy), publica no ACR com a tag igual ao hash do commit e atualiza o Deployment no AKS com `kubectl set image` (Rolling Update, sem downtime). Se o rollout falhar, o pipeline desfaz com `rollout undo`.

**Configuração no GitHub** (Settings → Secrets and variables → Actions):

| Tipo | Nome | Observação |
|---|---|---|
| Secret | `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_SUBSCRIPTION_ID` | identificadores da identidade do pipeline, não são senhas |
| Variable | `ACR_NAME`, `ACR_LOGIN_SERVER`, `AKS_RESOURCE_GROUP`, `AKS_CLUSTER_NAME`, `K8S_NAMESPACE` | nomes dos recursos no Azure |

**Login por OIDC:** o GitHub prova ao Azure que o job vem deste repositório e da branch `main`, e recebe um acesso temporário. A organização usa o formato de subject com IDs (`repo:<org>@<id>/<repo>@<id>:ref:refs/heads/main`), então a credencial federada da identidade `id-fcg-github` foi cadastrada com esse texto exato.

**Regras do pipeline:** deploy só na `main`; sem `environment:` no job de deploy; tag da imagem = hash do commit (nunca `latest`); o pipeline não aplica manifestos, só troca a imagem (os manifestos ficam no `fcg-orchestration`).

## Executar com Docker Compose

Copie o modelo de variáveis e preencha os valores (o `.env` não é versionado):

```bash
cp .env.example .env        # no PowerShell: Copy-Item .env.example .env
docker compose up -d --build
```

API disponível em: `http://localhost:8081`
Swagger: `http://localhost:8081/swagger`

## Executar localmente

```bash
# Pré-requisitos: .NET 8 SDK + PostgreSQL + RabbitMQ

dotnet restore
dotnet ef database update -p src/FCG.UsersAPI.Infrastructure -s src/FCG.UsersAPI.API
dotnet run --project src/FCG.UsersAPI.API
```

## Gerar Migrations

```bash
dotnet ef migrations add Initial \
  -p src/FCG.UsersAPI.Infrastructure \
  -s src/FCG.UsersAPI.API \
  -o Persistencia/Migrations
```

## Deploy Kubernetes

Crie o Secret a partir do modelo (o `secret.yaml` é ignorado pelo Git), troque cada
`<placeholder>` por um valor seu e aplique os arquivos **um a um**.

> Não use `kubectl apply -f k8s/`: isso aplicaria também o `secret.example.yaml`,
> com placeholders no lugar dos valores reais.

```bash
cp k8s/secret.example.yaml k8s/secret.yaml   # PowerShell: Copy-Item
# edite k8s/secret.yaml

kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/secret.yaml
kubectl apply -f k8s/configmap.yaml -f k8s/deployment.yaml -f k8s/service.yaml
kubectl get pods -n fcg
```

## Evento publicado

```json
// UserCreatedEvent (publicado após cada registro bem-sucedido)
{
  "userId": "guid",
  "nome": "João Silva",
  "email": "joao@email.com",
  "dataCadastro": "2026-01-01T00:00:00Z"
}
```

## Grupo 17 — Pos-Tech FIAP
- Letícia Lopes Ribeiro Vasconcelos
- Marcelo Henrique Cornelis Rei
- Washington Santana dos Santos
- Raul Hentz Rodrigues
