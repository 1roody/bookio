# Bookio Backend

Projeto educacional em Node.js/Express para demonstrar uma pipeline de DevSecOps sobre uma API simples de livros.

O repositório expõe um CRUD com SQLite, mantém endpoints propositalmente vulneráveis para testes de segurança e inclui uma pipeline modular em GitHub Actions com build, scans, deploy e DAST.

## Visão geral

### Stack da aplicação

- Node.js + Express
- SQLite com arquivo local `livros.db`
- Docker e Docker Compose
- GitHub Actions para CI/CD e segurança

### Ferramentas de segurança da esteira

- `SAST`: `Semgrep`
- `Secret Scanning`: `Gitleaks`
- `SCA / Filesystem Scanning`: `Trivy` em modo `fs`
- `Container Scanning`: `Trivy` contra a imagem Docker
- `IaC Scanning`: `Trivy` em modo `config`
- `DAST`: `OWASP ZAP`

## O que a aplicação faz

Ao subir a aplicação, o arquivo [`src/index.js`](/home/roody/Documents/hakai/aulas-hc/bookio-backend/src/index.js) cria a tabela `livros` no SQLite local, se ela ainda não existir.

### Endpoints funcionais

- `GET /health`
- `GET /livros`
- `GET /livros/:id`
- `POST /livros`
- `PUT /livros/:id`
- `DELETE /livros/:id`

Exemplo de payload para `POST /livros` e `PUT /livros/:id`:

```json
{
  "titulo": "Livro Teste",
  "autor": "Autor Teste",
  "ano": 2024,
  "segredo": "segredo123"
}
```

### Endpoints usados em testes de segurança

- `GET /segredo`
- `GET /xss?nome=...`
- `GET /sql?q=...`

Esses endpoints existem para laboratório e demonstração de scanners. Eles não representam boas práticas de segurança.

## Como rodar localmente

### Opção 1: Node.js

```bash
npm install
npm start
```

Aplicação disponível em `http://localhost:3000`.

### Opção 2: Docker Compose

```bash
docker compose up --build
```

Ou:

```bash
docker compose -f docker-compose-ex.yml up --build
```

### Teste rápido manual

```bash
curl http://localhost:3000/health
curl http://localhost:3000/livros
```

Para importar uma coleção pronta no Insomnia, use [`config/insomnia.json`](/home/roody/Documents/hakai/aulas-hc/bookio-backend/config/insomnia.json).

## Estrutura do repositório

```text
bookio-backend/
├── .github/workflows/
│   ├── pipeline.yml
│   ├── build.yml
│   ├── secret-gitleaks.yml
│   ├── sast-semgrep.yml
│   ├── sca-trivy-fs.yml
│   ├── container-trivy.yml
│   ├── iac-trivy.yml
│   ├── deploy.yml
│   └── dast.yml
├── config/
│   └── insomnia.json
├── iac-demo/
│   └── k8s/insecure-pod.yaml
├── src/
│   └── index.js
├── Dockerfile
├── docker-compose.yml
├── docker-compose-ex.yml
├── docker-compose.dast.yml
├── livros.db
├── package.json
└── README.md
```

## Pipeline no GitHub

### Arquitetura da pipeline

A pipeline foi organizada em um workflow orquestrador e vários workflows reutilizáveis:

- Orquestrador: [`pipeline.yml`](/home/roody/Documents/hakai/aulas-hc/bookio-backend/.github/workflows/pipeline.yml)
- Reutilizáveis:
  - [`build.yml`](/home/roody/Documents/hakai/aulas-hc/bookio-backend/.github/workflows/build.yml)
  - [`secret-gitleaks.yml`](/home/roody/Documents/hakai/aulas-hc/bookio-backend/.github/workflows/secret-gitleaks.yml)
  - [`sast-semgrep.yml`](/home/roody/Documents/hakai/aulas-hc/bookio-backend/.github/workflows/sast-semgrep.yml)
  - [`sca-trivy-fs.yml`](/home/roody/Documents/hakai/aulas-hc/bookio-backend/.github/workflows/sca-trivy-fs.yml)
  - [`container-trivy.yml`](/home/roody/Documents/hakai/aulas-hc/bookio-backend/.github/workflows/container-trivy.yml)
  - [`iac-trivy.yml`](/home/roody/Documents/hakai/aulas-hc/bookio-backend/.github/workflows/iac-trivy.yml)
  - [`deploy.yml`](/home/roody/Documents/hakai/aulas-hc/bookio-backend/.github/workflows/deploy.yml)
  - [`dast.yml`](/home/roody/Documents/hakai/aulas-hc/bookio-backend/.github/workflows/dast.yml)

A ideia é simples:

- `pipeline.yml` decide o que precisa rodar
- cada scanner ou etapa pesada fica isolado no seu próprio workflow
- deploy e DAST só acontecem quando o contexto realmente pede

## Gatilhos da pipeline principal

Arquivo: [`pipeline.yml`](/home/roody/Documents/hakai/aulas-hc/bookio-backend/.github/workflows/pipeline.yml)

### Eventos monitorados

1. `pull_request` para:
   - `main`
   - `develop`

   Tipos aceitos:
   - `opened`
   - `synchronize`
   - `reopened`
   - `ready_for_review`

2. `push` para:
   - `main`
   - `develop`

3. `workflow_dispatch`

4. `schedule`
   - cron configurado: `23 3 * * *`

### Controle de concorrência

A pipeline usa:

```yaml
concurrency:
  group: security-pipeline-${{ github.event.pull_request.number || github.ref }}
  cancel-in-progress: true
```

Isso significa:

- um novo commit no mesmo PR cancela a execução antiga
- um novo push na mesma branch monitora só a execução mais recente
- evita desperdício de runner

## Fluxo completo da pipeline, etapa por etapa

### Etapa 1: `changes`

Job: `Detect Changes`

Objetivo:

- entender o contexto da execução
- descobrir quais arquivos mudaram
- decidir quais jobs serão chamados depois

Esse job sempre roda primeiro.

### O que o job `changes` detecta

Ele calcula os seguintes sinais:

- `app_changed`
- `deps_changed`
- `container_changed`
- `iac_changed`
- `workflow_changed`
- `full_scan`

### Como ele decide isso

#### Em `schedule` ou `workflow_dispatch`

- define `full_scan=true`
- define `secret_scan_mode=full`
- considera o repositório inteiro como base da análise

#### Em `pull_request`

- faz `git fetch` da branch base
- compara `origin/<base_ref>...HEAD`
- define `secret_log_opts` com esse range

#### Em `push`

- usa `github.event.before`
- se for branch nova, considera tudo como alterado
- se não for branch nova, compara `before_sha` com `GITHUB_SHA`

### Regras de classificação de mudança

#### `app_changed=true` quando muda:

- `src/`
- `package.json`
- `package-lock.json`

#### `deps_changed=true` quando muda:

- `package.json`
- `package-lock.json`

#### `container_changed=true` quando muda:

- `Dockerfile`
- qualquer `docker-compose*.yml`
- `package.json`
- `package-lock.json`

#### `iac_changed=true` quando muda:

- qualquer arquivo dentro de `iac-demo/`

#### `workflow_changed=true` quando muda:

- qualquer arquivo dentro de `.github/workflows/`

### Flags finais calculadas pelo job `changes`

O job também decide estas flags:

- `run_build`
- `run_secret_scan`
- `run_sast`
- `run_sca_fs`
- `run_container_scan`
- `run_iac_scan`
- `run_deploy`
- `run_dast`

### Regras finais de execução

#### `run_secret_scan`

- sempre `true`

Motivo:

- segredo pode aparecer em qualquer arquivo, então é o scan mais constante da pipeline

#### `run_sast`

- `true` quando:
  - `full_scan=true`
  - ou `app_changed=true`
  - ou `workflow_changed=true`

Motivo:

- mudanças de aplicação e de workflow são contextos relevantes para análise estática

#### `run_sca_fs`

- `true` quando:
  - `full_scan=true`
  - ou `deps_changed=true`

Motivo:

- SCA filesystem é mais útil quando dependências mudam ou quando uma auditoria completa foi pedida

#### `run_container_scan`

- `true` quando:
  - `full_scan=true`
  - ou `container_changed=true`

Motivo:

- não faz sentido gastar runner com scan de imagem se o contexto de imagem não mudou

#### `run_iac_scan`

- `true` quando:
  - `full_scan=true`
  - ou `iac_changed=true`

Motivo:

- o repositório tem IaC apenas no diretório de laboratório `iac-demo/`

#### `run_build`

- `true` somente quando:
  - o evento é `pull_request`
  - e `container_changed=true`

Motivo:

- em PR, o build é usado como validação técnica do contexto de container
- ele não roda em todo cenário para não desperdiçar runner

#### `run_deploy`

- `true` somente quando:
  - o evento é `push`
  - a branch é `main`
  - e houve mudança em aplicação, dependências ou contexto de container

Motivo:

- deploy só existe para branch crítica
- deploy só acontece quando a imagem realmente precisa ser republicada

#### `run_dast`

- `true` quando:
  - `run_deploy=true`
  - ou `full_scan=true`

Motivo:

- DAST é um scan caro e mais profundo
- só faz sentido com ambiente real de teste ou em varredura completa

### Etapa 2: `build`

Workflow chamado: [`build.yml`](/home/roody/Documents/hakai/aulas-hc/bookio-backend/.github/workflows/build.yml)

Esse job só roda quando `run_build == true`.

Passos:

1. faz checkout do código
2. sobe o Docker Buildx
3. faz build da imagem com `docker/build-push-action@v5`
4. não publica a imagem
5. usa cache `gha`

Tag usada:

- `bookio-backend:build-validation`

Objetivo:

- validar que o contexto Docker continua construindo corretamente em PRs relevantes

### Etapa 3: `secret-detection`

Workflow chamado: [`secret-gitleaks.yml`](/home/roody/Documents/hakai/aulas-hc/bookio-backend/.github/workflows/secret-gitleaks.yml)

Esse job roda sempre.

Entradas aceitas pelo reusable workflow:

- `mode`
  - `diff`
  - `full`
- `log-opts`
- `version`

Passos:

1. checkout com `fetch-depth: 0`
2. download do binário do Gitleaks
3. execução do comando `gitleaks git .`
4. se o modo for `diff` e houver `log-opts`, a varredura fica restrita ao range calculado
5. upload do relatório SARIF para GitHub Security
6. upload do artefato `gitleaks-results.sarif`

Comportamento:

- `schedule` e `workflow_dispatch` usam modo `full`
- PRs e pushes normais usam modo `diff` sempre que possível

### Etapa 4: `sast`

Workflow chamado: [`sast-semgrep.yml`](/home/roody/Documents/hakai/aulas-hc/bookio-backend/.github/workflows/sast-semgrep.yml)

Esse job só roda quando `run_sast == true`.

Passos:

1. checkout
2. instalação do Semgrep com `pip`
3. execução de:

```bash
semgrep scan \
  --config auto \
  --sarif \
  --output semgrep-results.sarif \
  --error \
  || true
```

4. upload do SARIF
5. upload do artefato

Observações:

- `continue-on-error: true`
- job não roda para `dependabot[bot]`

### Etapa 5: `sca-fs`

Workflow chamado: [`sca-trivy-fs.yml`](/home/roody/Documents/hakai/aulas-hc/bookio-backend/.github/workflows/sca-trivy-fs.yml)

Esse job só roda quando `run_sca_fs == true`.

Passos:

1. checkout
2. execução do Trivy em modo `fs`
3. severidades observadas:
   - `CRITICAL`
   - `HIGH`
   - `MEDIUM`
4. geração de `trivy-fs-results.sarif`
5. upload SARIF
6. upload do artefato

Objetivo:

- identificar vulnerabilidades e problemas conhecidos em dependências e conteúdo do projeto

### Etapa 6: `container-scan`

Workflow chamado: [`container-trivy.yml`](/home/roody/Documents/hakai/aulas-hc/bookio-backend/.github/workflows/container-trivy.yml)

Esse job só roda quando `run_container_scan == true`.

Passos:

1. faz login no `ghcr.io` com `docker/login-action`
2. usa como alvo uma imagem já publicada no registry
3. executa o Trivy contra a imagem remota indicada por `image-ref`
4. gera `trivy-results.sarif`
5. faz upload SARIF
6. faz upload do artefato

Objetivo:

- avaliar a imagem publicada no registry, sem rebuild local redundante

Imagem usada:

- em `push` para `main`: `ghcr.io/<repo>:<github.sha>`
- em `schedule` e `workflow_dispatch`: `ghcr.io/<repo>:latest`

### Etapa 7: `iac-scan`

Workflow chamado: [`iac-trivy.yml`](/home/roody/Documents/hakai/aulas-hc/bookio-backend/.github/workflows/iac-trivy.yml)

Esse job só roda quando `run_iac_scan == true`.

Passos:

1. checkout
2. execução do Trivy em modo `config`
3. alvo:
   - [`iac-demo/`](/home/roody/Documents/hakai/aulas-hc/bookio-backend/iac-demo)
4. severidades observadas:
   - `CRITICAL`
   - `HIGH`
   - `MEDIUM`
   - `LOW`
5. geração de `trivy-iac-results.sarif`
6. upload SARIF
7. upload do artefato

Objetivo:

- demonstrar scan de configuração insegura em Kubernetes/IaC

O arquivo de demonstração atual é:

- [`iac-demo/k8s/insecure-pod.yaml`](/home/roody/Documents/hakai/aulas-hc/bookio-backend/iac-demo/k8s/insecure-pod.yaml)

### Etapa 8: `predeploy-gate`

Job interno no [`pipeline.yml`](/home/roody/Documents/hakai/aulas-hc/bookio-backend/.github/workflows/pipeline.yml)

Esse job sempre avalia o resultado dos anteriores com `if: always()`.

Dependências observadas:

- `build`
- `secret-detection`
- `sast`
- `sca-fs`
- `iac-scan`

O job lê os resultados com:

- `needs.<job>.result`

Depois:

1. monta uma lista com os resultados
2. se encontrar `failure` ou `cancelled`, encerra com erro
3. se os jobs estiverem `success` ou `skipped`, o gate deixa a pipeline seguir

Importante:

- os scanners continuam `continue-on-error: true`
- isso significa que findings não derrubam o job automaticamente
- o gate serve para bloquear apenas falhas reais de execução ou cancelamentos

### Etapa 9: `deploy`

Workflow chamado: [`deploy.yml`](/home/roody/Documents/hakai/aulas-hc/bookio-backend/.github/workflows/deploy.yml)

Esse job só roda quando `run_deploy == true`.

Condições práticas para isso acontecer:

- evento `push`
- branch `main`
- mudança relevante de app, dependência ou container

Passos:

1. checkout
2. setup do Buildx
3. login no `ghcr.io` com `docker/login-action`
4. build e push da imagem com `docker/build-push-action`

Tags publicadas:

- `ghcr.io/<repo>:<github.sha>`
- `ghcr.io/<repo>:latest`

Permissões usadas:

- `contents: read`
- `packages: write`

### Etapa 10: `dast`

Workflow chamado: [`dast.yml`](/home/roody/Documents/hakai/aulas-hc/bookio-backend/.github/workflows/dast.yml)

Esse job só roda quando `run_dast == true`.

Condições práticas para isso acontecer:

- após deploy em `main`
- ou em `schedule`
- ou em `workflow_dispatch`

Passos:

1. checkout
2. sobe ambiente com:

```bash
docker compose -f docker-compose.dast.yml up -d
```

3. espera 10 segundos
4. faz health check em `http://localhost:8080/health`
5. baixa a imagem `ghcr.io/zaproxy/zaproxy:stable`
6. roda `zap-baseline.py` contra `http://localhost:8080`
7. gera:
   - `zap-report.html`
   - `zap-report.json`
8. publica os artefatos
9. desmonta o ambiente com `docker compose ... down -v`

Características:

- `continue-on-error: true`
- timeout de 30 minutos
- não publica SARIF, apenas artefatos do ZAP

## Matriz resumida de execução

### `pull_request` para `main` ou `develop`

- `changes`: sempre
- `secret-detection`: sempre
- `sast`: se mudou app ou workflow
- `sca-fs`: se mudaram dependências
- `container-scan`: não roda
- `iac-scan`: se mudou `iac-demo`
- `build`: se mudou contexto Docker/container
- `predeploy-gate`: sempre
- `deploy`: nunca
- `dast`: nunca

### `push` para `develop`

- `changes`: sempre
- `secret-detection`: sempre
- `sast`: se mudou app ou workflow
- `sca-fs`: se mudaram dependências
- `container-scan`: não roda
- `iac-scan`: se mudou `iac-demo`
- `build`: não roda
- `predeploy-gate`: sempre
- `deploy`: nunca
- `dast`: nunca

### `push` para `main`

- `changes`: sempre
- `secret-detection`: sempre
- `sast`: se mudou app ou workflow
- `sca-fs`: se mudaram dependências
- `container-scan`: roda depois do `deploy`, usando a imagem publicada no GHCR
- `iac-scan`: se mudou `iac-demo`
- `build`: não roda
- `predeploy-gate`: sempre
- `deploy`: se mudou app, dependência ou contexto Docker/container
- `dast`: quando houver deploy

### `workflow_dispatch`

- `changes`: sempre
- todos os scans: rodam
- `deploy`: não roda
- `container-scan`: usa `ghcr.io/<repo>:latest`
- `dast`: roda

### `schedule`

- `changes`: sempre
- todos os scans: rodam
- `deploy`: não roda
- `container-scan`: usa `ghcr.io/<repo>:latest`
- `dast`: roda

## Artefatos e saídas geradas

### Security / SARIF

Os workflows abaixo enviam achados para GitHub Security via `github/codeql-action/upload-sarif@v4`:

- [`secret-gitleaks.yml`](/home/roody/Documents/hakai/aulas-hc/bookio-backend/.github/workflows/secret-gitleaks.yml)
- [`sast-semgrep.yml`](/home/roody/Documents/hakai/aulas-hc/bookio-backend/.github/workflows/sast-semgrep.yml)
- [`sca-trivy-fs.yml`](/home/roody/Documents/hakai/aulas-hc/bookio-backend/.github/workflows/sca-trivy-fs.yml)
- [`container-trivy.yml`](/home/roody/Documents/hakai/aulas-hc/bookio-backend/.github/workflows/container-trivy.yml)
- [`iac-trivy.yml`](/home/roody/Documents/hakai/aulas-hc/bookio-backend/.github/workflows/iac-trivy.yml)

### Artefatos publicados

- `gitleaks-results.sarif`
- `semgrep-results.sarif`
- `trivy-fs-results.sarif`
- `trivy-results.sarif`
- `trivy-iac-results.sarif`
- `zap-report.html`
- `zap-report.json`

## Permissões por etapa

### Pipeline principal

- `contents: read`

### Deploy

- `contents: read`
- `packages: write`

### Scanners com SARIF

- `contents: read`
- `security-events: write`
- `pull-requests: read`
- `actions: read`

### DAST

- `contents: read`

## Timeouts configurados

- `build.yml`: 20 minutos
- `secret-gitleaks.yml`: 15 minutos
- `sast-semgrep.yml`: 20 minutos
- `sca-trivy-fs.yml`: 15 minutos
- `container-trivy.yml`: 20 minutos
- `iac-trivy.yml`: 15 minutos
- `deploy.yml`: 20 minutos
- `dast.yml`: 30 minutos

## Comportamento de falha

### Jobs não bloqueantes por findings

Estes workflows estão com `continue-on-error: true`:

- secret detection
- SAST
- SCA filesystem
- container scan
- IaC scan
- DAST

Na prática:

- achados de segurança não derrubam automaticamente a pipeline
- falhas operacionais do job ainda podem impactar o fluxo
- o `predeploy-gate` só bloqueia quando existe `failure` ou `cancelled`

## Arquivos de infraestrutura relacionados

### [`Dockerfile`](/home/roody/Documents/hakai/aulas-hc/bookio-backend/Dockerfile)

- usa `node:14`
- copia `package*.json`
- executa `npm install --legacy-peer-deps`
- copia o projeto inteiro
- expõe a porta `3000`
- inicia com `node src/index.js`

### [`docker-compose.dast.yml`](/home/roody/Documents/hakai/aulas-hc/bookio-backend/docker-compose.dast.yml)

- sobe a aplicação na porta `8080`
- define `NODE_ENV=production`
- faz health check em `http://localhost:3000/health`
- é usado exclusivamente pelo workflow de DAST

## Variáveis de ambiente

Variáveis observadas no código e nos arquivos de infraestrutura:

- `PORT`
- `SEGREDO_SUPERSECRETO`

Exemplo:

```env
PORT=3000
SEGREDO_SUPERSECRETO=valor-muito-secreto
```

## Observações importantes

- O repositório ainda carrega `mongoose` em [`package.json`](/home/roody/Documents/hakai/aulas-hc/bookio-backend/package.json), mas a aplicação atual usa SQLite.
- O diretório [`iac-demo/`](/home/roody/Documents/hakai/aulas-hc/bookio-backend/iac-demo) existe apenas para demonstração de scan de IaC.
- A pipeline está desenhada para laboratório e ensino, por isso vários scanners estão em modo não bloqueante.
- O DAST atual usa ambiente efêmero local no runner, e não um ambiente remoto persistente.

## Resumo operacional

Em linguagem simples, a pipeline funciona assim:

1. recebe um evento suportado
2. detecta o tipo de mudança
3. ativa apenas os scans necessários
4. valida se houve falha real de execução
5. se estiver em `main` e houver contexto relevante, publica imagem
6. se houver contexto de DAST, sobe ambiente temporário e executa ZAP

Esse desenho reduz custo, evita duplicidade desnecessária e mantém a pipeline legível para estudo e evolução.
