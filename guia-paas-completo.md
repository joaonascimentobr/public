# Guia Completo: PaaS com Nomad + Consul + Traefik + Docker

> **Objetivo deste guia:** Entender o que cada ferramenta faz, como elas se comunicam, e como automatizar o ciclo completo de deploy — desde o `git push` do cliente até o site no ar — com foco no MVP de containers já prontos.

---

## Sumário

1. [O Mapa Geral — O que você construiu](#1-o-mapa-geral)
2. [Docker Engine — O motor de tudo](#2-docker-engine)
3. [Docker Registry — Seu armazém privado de imagens](#3-docker-registry)
4. [HashiCorp Consul — O catálogo de serviços](#4-hashicorp-consul)
5. [HashiCorp Nomad — O orquestrador](#5-hashicorp-nomad)
6. [Traefik — O porteiro inteligente](#6-traefik)
7. [Como os componentes se comunicam](#7-como-os-componentes-se-comunicam)
8. [Colocando um site no ar (passo a passo)](#8-colocando-um-site-no-ar)
9. [Gerenciando clientes — Adicionar, controlar e remover](#9-gerenciando-clientes)
10. [Controle de acesso — Quem vê o quê](#10-controle-de-acesso)
11. [Automação completa do deploy](#11-automação-completa-do-deploy)
12. [MVP: Deploy de containers já prontos](#12-mvp-deploy-de-containers-já-prontos)
13. [Monitoramento e diagnóstico](#13-monitoramento-e-diagnóstico)
14. [Referência rápida de comandos](#14-referência-rápida-de-comandos)

---

## 1. O Mapa Geral

Antes de entrar em cada ferramenta, veja o sistema como um todo. Você construiu uma plataforma onde:

- **Você** é o dono da infraestrutura (o servidor, os serviços)
- **Seus clientes** fazem push de código no GitHub e o site deles entra no ar automaticamente
- **Cada cliente** tem seu próprio ambiente isolado (container + domínio + SSL)
- **Você** controla tudo via sua API, sem precisar mexer no servidor manualmente

### O fluxo de ponta a ponta

```
Cliente faz git push no GitHub
         │
         ▼
GitHub envia Webhook para sua API
         │
         ▼
Sua API aciona o Nomad (Builder)
         │
         ▼
Nomad clona o repo, faz docker build, salva no Registry local
         │
         ▼
Sua API aciona o Nomad (Aplicação)
         │
         ▼
Nomad sobe o container e registra no Consul
         │
         ▼
Traefik detecta o novo serviço no Consul
         │
         ▼
Traefik cria a rota e gera o certificado SSL
         │
         ▼
Site do cliente no ar ✓
```

### A divisão de responsabilidades

| Ferramenta | Pergunta que ela responde | Analogia simples |
|---|---|---|
| **Docker Engine** | Como rodar o código em um ambiente isolado? | O motor que liga os carros |
| **Docker Registry** | Onde guardar as imagens construídas? | O depósito de carros prontos |
| **Consul** | Quem está rodando e em qual endereço? | O livro de telefones dos serviços |
| **Nomad** | Quem decide o que roda, onde e quando? | O despachante de frotas |
| **Traefik** | Como o tráfego da internet chega ao container certo? | O porteiro + recepcionista |

---

## 2. Docker Engine

### O que é

O Docker é o software que cria e executa **containers**. Um container é um ambiente isolado que contém tudo que uma aplicação precisa para rodar: código, dependências, configurações. É como uma caixa selada: o que está dentro não interfere no que está fora, e vice-versa.

### Para que serve no seu PaaS

Cada aplicação de cliente roda dentro de seu próprio container. Isso garante:

- **Isolamento**: o cliente A não consegue acessar os arquivos ou processos do cliente B
- **Portabilidade**: a mesma imagem roda igual em qualquer máquina
- **Reprodutibilidade**: se funcionou no build, vai funcionar no deploy

### Comandos essenciais do dia a dia

```bash
# Ver todos os containers rodando agora
docker ps

# Ver todos os containers (incluindo os parados)
docker ps -a

# Ver todas as imagens armazenadas localmente
docker images

# Parar um container pelo ID ou nome
docker stop <ID_OU_NOME>

# Remover um container parado
docker rm <ID_OU_NOME>

# Remover uma imagem
docker rmi <NOME_DA_IMAGEM>:<TAG>

# Ver os logs de um container em tempo real
docker logs -f <ID_OU_NOME>

# Entrar dentro de um container em execução (para debug)
docker exec -it <ID_OU_NOME> /bin/sh

# Liberar espaço: remover imagens não usadas
docker image prune -f

# Liberar espaço: remover tudo não usado (containers, imagens, volumes)
docker system prune -f
```

### Como o Docker se relaciona com o Nomad

Você **não vai rodar `docker run` manualmente** para as aplicações dos clientes. Quem faz isso é o Nomad. O Nomad lê o arquivo de job (`.nomad`), vê que o driver é `"docker"`, e chama o Docker Engine para subir o container com os parâmetros corretos.

---

## 3. Docker Registry

### O que é

O Registry é um **servidor de armazenamento de imagens Docker**. Funciona como um repositório privado (similar ao Docker Hub, mas rodando no seu próprio servidor).

### Para que serve no seu PaaS

Quando o Builder constrói uma imagem, ela precisa ser armazenada em algum lugar para que o Nomad consiga buscá-la depois e subir o container da aplicação. O Registry local resolve isso sem depender de serviços externos.

**Fluxo:**
```
Builder faz docker build → docker push → Registry (localhost:5000)
                                              │
Nomad pede para subir a app → docker pull ←──┘
```

### Estrutura das imagens

Cada imagem no Registry segue o padrão:

```
localhost:5000/<NOME_DO_APP>:<BUILD_ID>

Exemplo:
localhost:5000/site-cliente-joao:20240315-abc123
localhost:5000/site-cliente-maria:v2
```

O `BUILD_ID` é gerado pela sua API (pode ser um timestamp, um hash do commit Git, ou um número sequencial). Ele permite manter histórico de versões e fazer rollback.

### Comandos para gerenciar o Registry

```bash
# Ver todos os repositórios (aplicações) armazenadas
curl http://localhost:5000/v2/_catalog

# Ver todas as versões (tags) de uma aplicação específica
curl http://localhost:5000/v2/site-cliente-joao/tags/list

# Testar se o Registry está respondendo
curl http://localhost:5000/v2/
# Resposta esperada: {}
```

### Removendo imagens do Registry (ao remover um cliente)

O Registry não tem uma API de deleção habilitada por padrão. Para ativar:

```bash
# Acesse o container do Registry
docker exec -it <ID_DO_CONTAINER_REGISTRY> /bin/sh

# Delete uma tag específica
# Primeiro, obtenha o digest da imagem
curl -H "Accept: application/vnd.docker.distribution.manifest.v2+json" \
  -I http://localhost:5000/v2/<APP_NAME>/manifests/<TAG>
# Copie o valor do cabeçalho Docker-Content-Digest

# Delete usando o digest
curl -X DELETE \
  http://localhost:5000/v2/<APP_NAME>/manifests/<DIGEST>
```

**Solução mais simples para o MVP:** Ao remover um cliente, pare o job no Nomad e delete os arquivos diretamente da pasta `/opt/registry-data/docker/registry/v2/repositories/<APP_NAME>/`.

```bash
# Remover todos os dados de um cliente do Registry
sudo rm -rf /opt/registry-data/docker/registry/v2/repositories/<APP_NAME>
```

---

## 4. HashiCorp Consul

### O que é

O Consul é um sistema de **Service Discovery** (descoberta de serviços) e **Health Checking** (verificação de saúde). Em linguagem simples: é o lugar onde todos os serviços se registram dizendo "ei, estou aqui, nessa porta, e estou funcionando".

### Para que serve no seu PaaS

Sem o Consul, o Traefik não saberia que existe um novo container rodando, nem em qual porta ele está. O fluxo é:

1. Nomad sobe o container da aplicação
2. Nomad registra o serviço no Consul com nome, porta e tags
3. Traefik lê o Consul continuamente e detecta o novo serviço
4. Traefik usa as tags do serviço para montar as regras de roteamento

O Consul também remove automaticamente serviços que pararam de responder (health check falhou), e o Traefik para de rotear tráfego para eles.

### Interface web

Acesse `http://SEU_IP:8500` para ver todos os serviços registrados, seus status e health checks.

### Comandos essenciais

```bash
# Listar todos os serviços registrados
consul catalog services

# Ver detalhes e saúde de um serviço específico
consul health service webapp-site-joao

# Verificar se o Consul está saudável
consul members

# Ver os logs do Consul
journalctl -u consul.service -f
```

### O que você vai encontrar no Consul

Quando tudo estiver rodando, você verá serviços como:
- `nomad` — o próprio Nomad
- `nomad-client` — o agente Nomad
- `registry` — o Docker Registry
- `traefik` — o proxy reverso
- `webapp-site-joao` — a aplicação do cliente João
- `webapp-site-maria` — a aplicação da cliente Maria

Cada serviço aparece com status `passing` (verde), `warning` ou `critical` (vermelho).

---

## 5. HashiCorp Nomad

### O que é

O Nomad é o **orquestrador de workloads**. É ele que decide o que roda, quando roda, com quais recursos, e garante que continue rodando. Pense nele como um gerente de operações que nunca dorme.

### Para que serve no seu PaaS

O Nomad é o centro de controle. Toda vez que:
- Um cliente precisa ter seu site no ar → você submete um job ao Nomad
- Um deploy novo chega → o Nomad executa o rolling update
- Um container trava → o Nomad detecta e reinicia automaticamente
- O servidor reinicia → o Nomad restaura todos os jobs do estado salvo

### Conceitos fundamentais do Nomad

**Job:** É o arquivo de definição (`.nomad`) que descreve o que deve rodar. É como uma receita: "rode 1 instância desta imagem Docker, com 256 MB de RAM, na porta 80, e registre no Consul com essas tags".

**Allocation (Alloc):** É a instância em execução de um job. Quando o Nomad executa um job, ele cria uma allocation. Cada allocation tem um ID único e logs próprios.

**Task:** É a unidade mínima de trabalho dentro de um job. Geralmente corresponde a um container Docker.

**Tipo de job:**
- `service` → roda continuamente (ex: aplicação web do cliente)
- `batch` → roda uma vez e termina (ex: o builder)
- `system` → roda em todos os nós do cluster (ex: Traefik)

### Interface web

Acesse `http://SEU_IP:4646` para ver todos os jobs, suas allocations, logs e status.

### Comandos essenciais

```bash
# Ver status do cluster
nomad server members
nomad node status

# Listar todos os jobs
nomad job list

# Ver detalhes de um job específico
nomad job status webapp-site-joao

# Submeter ou atualizar um job a partir de um arquivo .nomad
nomad job run ~/nomad-jobs/webapp-site-joao.nomad

# Parar um job (mantém o histórico, não apaga o estado)
nomad job stop webapp-site-joao

# Parar e apagar completamente um job
nomad job stop -purge webapp-site-joao

# Forçar um redeploy sem alterar o arquivo
nomad job restart webapp-site-joao

# Ver o ID da allocation mais recente de um job
nomad job status webapp-site-joao | grep -A5 "Allocations"

# Ver logs de stdout de uma allocation
nomad alloc logs -f <ALLOC_ID>

# Ver logs de stderr (erros) de uma allocation
nomad alloc logs -stderr -f <ALLOC_ID>

# Ver detalhes completos de uma allocation
nomad alloc status <ALLOC_ID>

# Disparar um job parametrizado (o builder)
nomad job dispatch \
  -meta REPO_URL="https://github.com/cliente/repo" \
  -meta APP_NAME="site-joao" \
  -meta BUILD_ID="v3" \
  builder-dispatcher
```

### A API REST do Nomad

O Nomad expõe uma API REST completa na porta 4646. Sua aplicação backend vai usar essa API para automatizar os deploys. Os endpoints mais importantes:

```
GET  /v1/jobs                     → listar todos os jobs
POST /v1/jobs                     → criar um novo job
PUT  /v1/jobs/<JOB_ID>            → atualizar um job existente
GET  /v1/job/<JOB_ID>             → detalhes de um job
DELETE /v1/job/<JOB_ID>           → parar e remover um job
POST /v1/job/<JOB_ID>/dispatch    → disparar uma instância de job parametrizado
GET  /v1/job/<JOB_ID>/allocations → listar allocations de um job
```

---

## 6. Traefik

### O que é

O Traefik é um **proxy reverso e load balancer** que se configura automaticamente. Ele fica na frente de tudo, recebendo as requisições da internet e redirecionando para o container correto, além de gerenciar os certificados SSL.

### Para que serve no seu PaaS

Imagine que você tem 50 clientes, cada um com seu domínio (`joao.com`, `maria.com.br`, etc.), todos apontando para o IP do seu servidor. O Traefik é quem olha o domínio da requisição que chegou e decide qual container deve receber aquela requisição.

**Sem Traefik:** Você teria que configurar manualmente um Nginx ou Apache para cada novo cliente, gerenciar certificados SSL manualmente, e reiniciar o servidor a cada mudança.

**Com Traefik:** Você só precisa adicionar tags ao registro do serviço no Consul. O Traefik detecta em segundos e já roteia, já com HTTPS.

### As tags que controlam o Traefik

Quando você define um job no Nomad, as tags do serviço ensinam o Traefik o que fazer:

```hcl
tags = [
  # Diz ao Traefik para gerenciar este serviço
  "traefik.enable=true",

  # Define para qual domínio este serviço deve responder
  "traefik.http.routers.site-joao.rule=Host(`joao.com`)",

  # Define que usa o entrypoint HTTPS (porta 443)
  "traefik.http.routers.site-joao.entrypoints=websecure",

  # Solicita geração automática de SSL via Let's Encrypt
  "traefik.http.routers.site-joao.tls.certresolver=le"
]
```

### Interface web

Acesse `http://SEU_IP:8080` para ver o dashboard do Traefik: todas as rotas configuradas, os backends (containers), e o status dos certificados SSL.

### Como o SSL automático funciona

1. O domínio do cliente (`joao.com`) aponta para o IP do seu servidor (configurado pelo próprio cliente no painel do registrador de domínio)
2. O Traefik detecta a tag `tls.certresolver=le`
3. Na primeira requisição, o Traefik contata o Let's Encrypt
4. O Let's Encrypt verifica que o domínio realmente aponta para o servidor (via HTTP challenge na porta 80)
5. O certificado é emitido e armazenado automaticamente
6. O Traefik renova o certificado antes de expirar (a cada ~60 dias), sem intervenção manual

---

## 7. Como os componentes se comunicam

Entender as conexões entre os componentes é essencial para diagnosticar problemas.

```
┌─────────────────────────────────────────────────────────┐
│                      SERVIDOR UBUNTU                     │
│                                                         │
│  ┌──────────┐     registra     ┌──────────┐             │
│  │  NOMAD   │ ───────────────► │  CONSUL  │             │
│  │ :4646    │                  │  :8500   │             │
│  └────┬─────┘                  └─────┬────┘             │
│       │                             │                   │
│       │ controla                    │ lê serviços       │
│       │                             │                   │
│  ┌────▼─────┐   push/pull    ┌──────▼────┐             │
│  │  DOCKER  │ ◄────────────► │ REGISTRY  │             │
│  │  ENGINE  │                │  :5000    │             │
│  └────┬─────┘                └───────────┘             │
│       │                                                 │
│       │ roda                                            │
│       │                                                 │
│  ┌────▼─────────────────────────────────────┐           │
│  │           CONTAINERS DOS CLIENTES         │           │
│  │  [site-joao:porta-dinamica]               │           │
│  │  [site-maria:porta-dinamica]              │           │
│  │  [api-pedro:porta-dinamica]               │           │
│  └────────────────────────────┬─────────────┘           │
│                               │                         │
│                          rotas via tags                  │
│                               │                         │
│  ┌────────────────────────────▼─────────────┐           │
│  │              TRAEFIK                      │           │
│  │     :80 (http) | :443 (https)            │           │
│  └────────────────────────────┬─────────────┘           │
│                               │                         │
└───────────────────────────────┼─────────────────────────┘
                                │
                        INTERNET
                   joao.com → IP:443
                   maria.com → IP:443
```

### Fluxo de uma requisição de usuário final

1. Usuário acessa `https://joao.com`
2. DNS resolve para o IP do seu servidor
3. Requisição chega na porta 443 do Traefik
4. Traefik verifica qual container tem a regra `Host('joao.com')`
5. Traefik encontra `webapp-site-joao` registrado no Consul
6. Traefik encaminha a requisição para o container, na porta dinâmica alocada pelo Nomad
7. Container responde → Traefik devolve a resposta ao usuário

---

## 8. Colocando um site no ar

### Para o MVP (imagem já pronta no Registry)

Se você já tem a imagem no Registry (`localhost:5000/site-joao:v1`), o processo é direto:

#### Passo 1: Crie o arquivo de job

```bash
nano ~/nomad-jobs/webapp-site-joao.nomad
```

```hcl
job "webapp-site-joao" {
  datacenters = ["dc1"]
  type        = "service"

  group "web" {
    count = 1

    restart {
      attempts = 10
      interval = "5m"
      delay    = "25s"
      mode     = "delay"
    }

    network {
      port "http" {
        to = 80   # porta que o container escuta internamente
      }
    }

    service {
      name = "webapp-site-joao"
      port = "http"

      tags = [
        "traefik.enable=true",
        "traefik.http.routers.site-joao.rule=Host(`joao.com`)",
        "traefik.http.routers.site-joao.entrypoints=websecure",
        "traefik.http.routers.site-joao.tls.certresolver=le"
      ]

      check {
        type     = "http"
        path     = "/"
        interval = "10s"
        timeout  = "2s"
      }
    }

    task "server" {
      driver = "docker"

      config {
        image = "localhost:5000/site-joao:v1"
        ports = ["http"]
      }

      resources {
        cpu    = 250   # MHz de CPU
        memory = 256   # MB de RAM
      }
    }
  }
}
```

#### Passo 2: Suba o job

```bash
nomad job run ~/nomad-jobs/webapp-site-joao.nomad
```

#### Passo 3: Verifique o status

```bash
nomad job status webapp-site-joao
```

Aguarde o status da allocation mudar para `running`. Depois de alguns segundos, o Traefik já terá detectado o serviço no Consul e o site estará acessível.

#### Passo 4: Verifique no Consul e Traefik

- Consul: `http://SEU_IP:8500` → procure `webapp-site-joao` com status `passing`
- Traefik: `http://SEU_IP:8080` → procure a rota `site-joao` em HTTP Routers

### Para testes sem domínio real (usando nip.io)

Use o serviço `nip.io` que transforma qualquer IP em DNS sem configuração:

```hcl
tags = [
  "traefik.enable=true",
  "traefik.http.routers.site-joao.rule=Host(`site-joao.191.252.194.7.nip.io`)",
  "traefik.http.routers.site-joao.entrypoints=web"
  # Sem tls para testes em HTTP
]
```

Substitua `191.252.194.7` pelo IP real do seu servidor.

### Fazendo update (novo deploy sem downtime)

Edite o arquivo `.nomad` alterando apenas a tag da imagem:

```hcl
image = "localhost:5000/site-joao:v2"  # ← nova versão
```

Então rode novamente:

```bash
nomad job run ~/nomad-jobs/webapp-site-joao.nomad
```

O Nomad detecta a mudança e executa um **rolling update**:
1. Sobe o novo container (v2)
2. Aguarda o health check passar
3. Derruba o container antigo (v1)

Zero downtime para o usuário final.

---

## 9. Gerenciando clientes

### Estrutura de nomenclatura recomendada

Adote uma convenção consistente para nomear recursos. Sugestão:

```
Job Nomad:    webapp-<ID_DO_CLIENTE>
Imagem:       localhost:5000/<ID_DO_CLIENTE>:<BUILD_ID>
Serviço Consul: webapp-<ID_DO_CLIENTE>
Arquivo:      ~/nomad-jobs/webapp-<ID_DO_CLIENTE>.nomad
```

Exemplo com cliente de ID `cli-joao-123`:
- Job: `webapp-cli-joao-123`
- Imagem: `localhost:5000/cli-joao-123:v1`
- Arquivo: `~/nomad-jobs/webapp-cli-joao-123.nomad`

### Adicionando um novo cliente

Sua API deve:

1. Gerar o arquivo `.nomad` com os valores do cliente (nome, domínio, imagem, recursos)
2. Submeter via API REST do Nomad

```python
import requests
import json

def criar_job_cliente(app_name, dominio, image_tag, cpu=250, memoria=256):
    job = {
        "Job": {
            "ID": f"webapp-{app_name}",
            "Name": f"webapp-{app_name}",
            "Type": "service",
            "Datacenters": ["dc1"],
            "TaskGroups": [{
                "Name": "web",
                "Count": 1,
                "RestartPolicy": {
                    "Attempts": 10,
                    "Interval": 300000000000,
                    "Delay": 25000000000,
                    "Mode": "delay"
                },
                "Networks": [{
                    "DynamicPorts": [{"Label": "http", "To": 80}]
                }],
                "Services": [{
                    "Name": f"webapp-{app_name}",
                    "PortLabel": "http",
                    "Tags": [
                        "traefik.enable=true",
                        f"traefik.http.routers.{app_name}.rule=Host(`{dominio}`)",
                        f"traefik.http.routers.{app_name}.entrypoints=websecure",
                        f"traefik.http.routers.{app_name}.tls.certresolver=le"
                    ],
                    "Checks": [{
                        "Type": "http",
                        "Path": "/",
                        "Interval": 10000000000,
                        "Timeout": 2000000000
                    }]
                }],
                "Tasks": [{
                    "Name": "server",
                    "Driver": "docker",
                    "Config": {
                        "image": f"localhost:5000/{app_name}:{image_tag}",
                        "ports": ["http"]
                    },
                    "Resources": {
                        "CPU": cpu,
                        "MemoryMB": memoria
                    }
                }]
            }]
        }
    }

    response = requests.post(
        "http://localhost:4646/v1/jobs",
        json=job
    )
    return response.json()
```

### Atualizando um cliente (novo deploy)

```python
def atualizar_deploy(app_name, novo_build_id):
    # Gerar job com nova tag de imagem
    job = criar_job_cliente(
        app_name=app_name,
        dominio=f"{app_name}.seudominio.com",
        image_tag=novo_build_id
    )

    # PUT para o job existente
    response = requests.put(
        f"http://localhost:4646/v1/job/webapp-{app_name}",
        json={"Job": job["Job"]}
    )
    return response.json()
```

### Pausando um cliente temporariamente

```bash
# Via CLI
nomad job stop webapp-cli-joao-123

# Via API REST
curl -X DELETE http://localhost:4646/v1/job/webapp-cli-joao-123
```

O job é parado, o container é removido, mas o arquivo `.nomad` e as imagens no Registry continuam existindo. Para restaurar:

```bash
nomad job run ~/nomad-jobs/webapp-cli-joao-123.nomad
```

### Removendo um cliente completamente

Execute esta sequência para remover tudo relacionado ao cliente:

```bash
APP_NAME="cli-joao-123"

# 1. Parar e remover o job do Nomad (purge apaga o histórico também)
nomad job stop -purge webapp-${APP_NAME}

# 2. Remover o arquivo de job
rm ~/nomad-jobs/webapp-${APP_NAME}.nomad

# 3. Remover as imagens do Registry local
sudo rm -rf /opt/registry-data/docker/registry/v2/repositories/${APP_NAME}

# 4. Limpar imagens Docker que possam estar no cache local
docker rmi $(docker images "localhost:5000/${APP_NAME}*" -q) 2>/dev/null || true

# 5. (Opcional) Verificar se o serviço sumiu do Consul
consul catalog services | grep ${APP_NAME}
# Não deve retornar nada
```

**Script de remoção automático (para sua API chamar):**

```bash
#!/bin/bash
# remove-cliente.sh
APP_NAME="$1"

if [ -z "$APP_NAME" ]; then
  echo "Uso: $0 <APP_NAME>"
  exit 1
fi

echo "[1/4] Parando job no Nomad..."
nomad job stop -purge "webapp-${APP_NAME}" 2>/dev/null

echo "[2/4] Removendo arquivo de job..."
rm -f ~/nomad-jobs/webapp-${APP_NAME}.nomad

echo "[3/4] Removendo imagens do Registry..."
sudo rm -rf /opt/registry-data/docker/registry/v2/repositories/${APP_NAME}

echo "[4/4] Limpando cache Docker..."
docker rmi $(docker images "localhost:5000/${APP_NAME}*" -q) 2>/dev/null || true

echo "✓ Cliente ${APP_NAME} removido com sucesso."
```

---

## 10. Controle de acesso

### Separação entre clientes — Como funciona

Cada cliente roda em um **container isolado**. O isolamento é garantido pelo Docker:

- **Sistema de arquivos:** cada container tem seu próprio sistema de arquivos. O cliente A não consegue ler arquivos do cliente B
- **Processos:** cada container tem seu próprio espaço de processos. Um cliente não consegue ver ou matar processos de outro
- **Rede:** cada container tem sua própria interface de rede virtual. A comunicação entre containers só ocorre se você configurar explicitamente

### As interfaces administrativas (Nomad, Consul, Traefik)

**Problema:** As interfaces web do Nomad (4646), Consul (8500) e Traefik (8080) não têm autenticação por padrão e mostram tudo de todos os clientes. Elas devem ser acessíveis **somente para você**, nunca para os clientes.

**Solução para o MVP — Firewall:** Bloqueie essas portas para o mundo externo, permitindo apenas o seu IP:

```bash
# Instalar UFW se necessário
sudo apt install ufw -y

# Política padrão: bloquear entrada
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Permitir SSH (NUNCA esqueça disso antes de ativar o firewall!)
sudo ufw allow 22/tcp

# Permitir tráfego público (HTTP e HTTPS para os sites dos clientes)
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Permitir interfaces admin SOMENTE para o seu IP
sudo ufw allow from SEU_IP_PESSOAL to any port 4646   # Nomad
sudo ufw allow from SEU_IP_PESSOAL to any port 8500   # Consul
sudo ufw allow from SEU_IP_PESSOAL to any port 8080   # Traefik
sudo ufw allow from SEU_IP_PESSOAL to any port 5000   # Registry

# Ativar o firewall
sudo ufw enable

# Verificar as regras
sudo ufw status numbered
```

### Controle de acesso por cliente — O que cada um vê

Os clientes **não têm acesso direto** a nenhuma das ferramentas da infraestrutura. Eles interagem apenas com:

1. **GitHub/GitLab** — onde fazem push do código deles
2. **Sua API** — para ver status do deploy, logs, configurações

Sua API é a única que se comunica com o Nomad, Consul e Registry. Você controla o que cada cliente pode ver e fazer.

### Autenticando webhooks do GitHub

Quando o GitHub chama seu endpoint de webhook, valide a assinatura para garantir que a requisição é legítima:

```python
import hmac
import hashlib
from flask import request, abort

WEBHOOK_SECRET = "seu-segredo-aqui"  # mesmo valor configurado no GitHub

def verificar_webhook_github():
    signature = request.headers.get('X-Hub-Signature-256', '')
    payload = request.get_data()

    expected = 'sha256=' + hmac.new(
        WEBHOOK_SECRET.encode(),
        payload,
        hashlib.sha256
    ).hexdigest()

    if not hmac.compare_digest(signature, expected):
        abort(403, "Assinatura inválida")
```

### Limites de recurso por plano

O Nomad permite definir limites de CPU e memória por job. Use isso para criar planos:

```hcl
# Plano Básico
resources {
  cpu    = 250   # 250 MHz
  memory = 256   # 256 MB RAM
}

# Plano Pro
resources {
  cpu    = 500
  memory = 512
}

# Plano Business
resources {
  cpu    = 1000
  memory = 1024
}
```

---

## 11. Automação completa do deploy

### O fluxo completo que sua API precisa implementar

```
GitHub Webhook → Sua API → Nomad Builder → Sua API → Nomad App → Site no ar
```

### Estrutura recomendada da sua API

```
/webhook/github          POST  → recebe push do GitHub
/deploy/<app_name>       POST  → inicia deploy manual
/status/<app_name>       GET   → retorna status do deploy
/logs/<app_name>         GET   → retorna logs da aplicação
/clientes                GET   → lista todos os clientes
/clientes                POST  → cria novo cliente
/clientes/<id>           DELETE → remove cliente
```

### Exemplo de implementação do webhook (Python/Flask)

```python
from flask import Flask, request, jsonify
import requests
import time
import hmac, hashlib

app = Flask(__name__)

NOMAD_URL = "http://localhost:4646"
WEBHOOK_SECRET = "seu-segredo-aqui"

def verificar_assinatura(payload, signature):
    expected = 'sha256=' + hmac.new(
        WEBHOOK_SECRET.encode(), payload, hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(signature, expected)

def disparar_build(repo_url, app_name, build_id):
    """Dispara o job de build no Nomad"""
    payload = {
        "Job": {
            "ID": "builder-dispatcher",
            "Meta": {
                "REPO_URL": repo_url,
                "APP_NAME": app_name,
                "BUILD_ID": build_id
            }
        }
    }
    r = requests.post(f"{NOMAD_URL}/v1/job/builder-dispatcher/dispatch", json=payload)
    return r.json().get("DispatchedJobID")

def aguardar_build(dispatch_job_id, timeout=300):
    """Aguarda o build finalizar, com timeout em segundos"""
    inicio = time.time()
    while time.time() - inicio < timeout:
        r = requests.get(f"{NOMAD_URL}/v1/job/{dispatch_job_id}")
        status = r.json().get("Status")
        if status == "dead":
            # Verifica se foi sucesso ou falha
            allocs = requests.get(
                f"{NOMAD_URL}/v1/job/{dispatch_job_id}/allocations"
            ).json()
            if allocs and allocs[0].get("ClientStatus") == "complete":
                return True
            return False
        time.sleep(5)
    return False  # timeout

def subir_aplicacao(app_name, dominio, build_id):
    """Submete o job da aplicação ao Nomad"""
    job = {
        "Job": {
            "ID": f"webapp-{app_name}",
            "Name": f"webapp-{app_name}",
            "Type": "service",
            "Datacenters": ["dc1"],
            "TaskGroups": [{
                "Name": "web",
                "Count": 1,
                "RestartPolicy": {
                    "Attempts": 10, "Interval": 300000000000,
                    "Delay": 25000000000, "Mode": "delay"
                },
                "Networks": [{"DynamicPorts": [{"Label": "http", "To": 80}]}],
                "Services": [{
                    "Name": f"webapp-{app_name}",
                    "PortLabel": "http",
                    "Tags": [
                        "traefik.enable=true",
                        f"traefik.http.routers.{app_name}.rule=Host(`{dominio}`)",
                        f"traefik.http.routers.{app_name}.entrypoints=websecure",
                        f"traefik.http.routers.{app_name}.tls.certresolver=le"
                    ],
                    "Checks": [{
                        "Type": "http", "Path": "/",
                        "Interval": 10000000000, "Timeout": 2000000000
                    }]
                }],
                "Tasks": [{
                    "Name": "server",
                    "Driver": "docker",
                    "Config": {
                        "image": f"localhost:5000/{app_name}:{build_id}",
                        "ports": ["http"]
                    },
                    "Resources": {"CPU": 250, "MemoryMB": 256}
                }]
            }]
        }
    }
    r = requests.put(f"{NOMAD_URL}/v1/job/webapp-{app_name}", json={"Job": job["Job"]})
    return r.json()

@app.route('/webhook/github', methods=['POST'])
def webhook_github():
    # Validar assinatura
    sig = request.headers.get('X-Hub-Signature-256', '')
    if not verificar_assinatura(request.get_data(), sig):
        return jsonify({"erro": "Assinatura inválida"}), 403

    dados = request.json
    # Extrair informações do push
    repo_url = dados['repository']['clone_url']
    app_name = dados['repository']['name']  # ou lookup no seu banco de dados
    build_id = dados['after'][:8]  # primeiros 8 chars do hash do commit

    # 1. Disparar build
    dispatch_id = disparar_build(repo_url, app_name, build_id)

    # 2. Aguardar build (em produção, use um background task)
    sucesso = aguardar_build(dispatch_id)

    if not sucesso:
        return jsonify({"status": "erro", "mensagem": "Build falhou"}), 500

    # 3. Subir aplicação com a nova versão
    dominio = f"{app_name}.seudominio.com"  # ou lookup no banco de dados
    subir_aplicacao(app_name, dominio, build_id)

    return jsonify({
        "status": "ok",
        "app": app_name,
        "build_id": build_id,
        "dominio": dominio
    })

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=3000)
```

> **Nota para produção:** O `aguardar_build` com `time.sleep` bloqueia o processo. Em produção, use uma fila de tarefas (Celery + Redis, RQ, ou similar) para processar os deploys em background e retornar o status via polling ou websocket.

---

## 12. MVP: Deploy de containers já prontos

Para o MVP, onde a imagem já está pronta (o cliente já tem uma imagem Docker publicada no Docker Hub ou você vai carregá-la manualmente no Registry), o fluxo é ainda mais simples.

### Opção A: Imagem no Docker Hub (pública)

Se o cliente tem uma imagem pública no Docker Hub (`dockerhub-user/meu-site:latest`), use diretamente no job:

```hcl
task "server" {
  driver = "docker"

  config {
    image = "dockerhub-user/meu-site:latest"
    ports = ["http"]
  }
}
```

### Opção B: Carregar imagem manualmente no Registry local

```bash
# Puxar a imagem de qualquer fonte
docker pull dockerhub-user/meu-site:v1

# Renomear para o padrão do seu Registry
docker tag dockerhub-user/meu-site:v1 localhost:5000/site-joao:v1

# Enviar para o Registry local
docker push localhost:5000/site-joao:v1

# Verificar
curl http://localhost:5000/v2/_catalog
```

### Opção C: Script de onboarding de cliente

```bash
#!/bin/bash
# onboarding-cliente.sh
# Uso: ./onboarding-cliente.sh site-joao joao.com dockerhub-user/meu-site:v1

APP_NAME="$1"
DOMINIO="$2"
IMAGEM_FONTE="$3"

echo "=== Onboarding: ${APP_NAME} ==="

# Carregar imagem no Registry
echo "[1/3] Carregando imagem..."
docker pull "${IMAGEM_FONTE}"
docker tag "${IMAGEM_FONTE}" "localhost:5000/${APP_NAME}:v1"
docker push "localhost:5000/${APP_NAME}:v1"

# Gerar arquivo de job
echo "[2/3] Criando job Nomad..."
cat > ~/nomad-jobs/webapp-${APP_NAME}.nomad << EOF
job "webapp-${APP_NAME}" {
  datacenters = ["dc1"]
  type        = "service"

  group "web" {
    count = 1

    restart {
      attempts = 10
      interval = "5m"
      delay    = "25s"
      mode     = "delay"
    }

    network {
      port "http" { to = 80 }
    }

    service {
      name = "webapp-${APP_NAME}"
      port = "http"

      tags = [
        "traefik.enable=true",
        "traefik.http.routers.${APP_NAME}.rule=Host(\`${DOMINIO}\`)",
        "traefik.http.routers.${APP_NAME}.entrypoints=websecure",
        "traefik.http.routers.${APP_NAME}.tls.certresolver=le"
      ]

      check {
        type     = "http"
        path     = "/"
        interval = "10s"
        timeout  = "2s"
      }
    }

    task "server" {
      driver = "docker"

      config {
        image = "localhost:5000/${APP_NAME}:v1"
        ports = ["http"]
      }

      resources {
        cpu    = 250
        memory = 256
      }
    }
  }
}
EOF

# Subir o job
echo "[3/3] Subindo job no Nomad..."
nomad job run ~/nomad-jobs/webapp-${APP_NAME}.nomad

echo "✓ ${APP_NAME} está subindo!"
echo "→ Acompanhe: nomad job status webapp-${APP_NAME}"
echo "→ Acesse em: https://${DOMINIO} (após o DNS propagar)"
```

---

## 13. Monitoramento e diagnóstico

### Checklist de saúde do sistema

```bash
# 1. Docker funcionando?
sudo systemctl status docker

# 2. Consul funcionando?
sudo systemctl status consul
consul members   # deve mostrar o servidor com status "alive"

# 3. Nomad funcionando?
sudo systemctl status nomad
nomad server members   # deve mostrar o servidor como líder

# 4. Jobs essenciais rodando?
nomad job list
# Deve mostrar: registry (running), traefik (running), builder-dispatcher (running)

# 5. Registry respondendo?
curl http://localhost:5000/v2/

# 6. Traefik respondendo?
curl http://localhost:8080/api/rawdata
```

### Diagnóstico de um site fora do ar

Siga esta sequência:

```bash
APP_NAME="site-joao"

# Passo 1: O job está rodando no Nomad?
nomad job status webapp-${APP_NAME}
# Procure por "Status = running" e allocation com "ClientStatus = running"

# Passo 2: Qual é o ID da allocation atual?
ALLOC_ID=$(nomad job status webapp-${APP_NAME} | grep running | head -1 | awk '{print $1}')

# Passo 3: Ver logs da aplicação
nomad alloc logs -f ${ALLOC_ID}
nomad alloc logs -stderr -f ${ALLOC_ID}

# Passo 4: O serviço está no Consul?
consul health service webapp-${APP_NAME}
# Procure por "Status: passing"

# Passo 5: O Traefik está vendo a rota?
curl http://localhost:8080/api/http/routers
# Procure pelo router do cliente

# Passo 6: O container está respondendo diretamente?
# Descobrir a porta dinâmica alocada
nomad alloc status ${ALLOC_ID} | grep -A5 "Ports"
curl http://localhost:<PORTA_DINAMICA>/
```

### Logs centralizados dos serviços de infraestrutura

```bash
# Logs do Consul
journalctl -u consul.service -f

# Logs do Nomad
journalctl -u nomad.service -f

# Logs do Docker
journalctl -u docker.service -f

# Logs de um container específico pelo nome
docker logs -f <CONTAINER_NAME>
```

---

## 14. Referência rápida de comandos

### Nomad — operações do dia a dia

```bash
# Cluster
nomad server members                          # status do cluster
nomad node status                             # status dos nós

# Jobs
nomad job list                                # listar todos os jobs
nomad job status <JOB_ID>                     # detalhes de um job
nomad job run <ARQUIVO.nomad>                 # submeter/atualizar job
nomad job stop <JOB_ID>                       # parar job
nomad job stop -purge <JOB_ID>                # parar e apagar histórico
nomad job restart <JOB_ID>                    # forçar redeploy

# Allocations
nomad alloc logs -f <ALLOC_ID>                # logs stdout em tempo real
nomad alloc logs -stderr -f <ALLOC_ID>        # logs stderr em tempo real
nomad alloc status <ALLOC_ID>                 # detalhes completos

# Builder
nomad job dispatch \
  -meta REPO_URL="https://github.com/user/repo" \
  -meta APP_NAME="site-joao" \
  -meta BUILD_ID="v2" \
  builder-dispatcher
```

### Consul — operações do dia a dia

```bash
consul members                                # status dos membros
consul catalog services                       # listar todos os serviços
consul health service <NOME>                  # saúde de um serviço
consul catalog nodes                          # listar nós
```

### Docker Registry — operações do dia a dia

```bash
# Listar repositórios
curl http://localhost:5000/v2/_catalog

# Listar tags de uma imagem
curl http://localhost:5000/v2/<APP>/tags/list

# Enviar imagem manualmente
docker tag <IMAGEM_LOCAL> localhost:5000/<APP>:<TAG>
docker push localhost:5000/<APP>:<TAG>

# Limpar imagens não referenciadas
docker image prune -f
```

### Docker — operações do dia a dia

```bash
docker ps                                     # containers rodando
docker ps -a                                  # todos os containers
docker images                                 # todas as imagens
docker logs -f <CONTAINER>                    # logs em tempo real
docker exec -it <CONTAINER> /bin/sh           # entrar no container
docker stats                                  # uso de CPU/RAM em tempo real
docker system prune -f                        # limpar tudo não usado
```

### Systemd — gerenciar os serviços base

```bash
# Status
sudo systemctl status docker
sudo systemctl status consul
sudo systemctl status nomad

# Reiniciar
sudo systemctl restart docker
sudo systemctl restart consul
sudo systemctl restart nomad

# Logs
journalctl -u consul.service -f
journalctl -u nomad.service -f
```

---

## Resumo executivo

| Quero... | Comando / Ação |
|---|---|
| Colocar um site no ar | `nomad job run webapp-cliente.nomad` |
| Fazer novo deploy (update) | Editar a tag da imagem no `.nomad` → `nomad job run ...` |
| Ver se o site está rodando | `nomad job status webapp-cliente` |
| Ver logs do site | `nomad alloc logs -f <ALLOC_ID>` |
| Pausar um cliente | `nomad job stop webapp-cliente` |
| Remover um cliente | `nomad job stop -purge webapp-cliente` + `rm` do arquivo + `rm` das imagens |
| Ver todos os clientes | `nomad job list` |
| Ver o que está no Registry | `curl http://localhost:5000/v2/_catalog` |
| Ver serviços no Consul | `consul catalog services` |
| Ver rotas no Traefik | `http://SEU_IP:8080` |
| Ver painel do Nomad | `http://SEU_IP:4646` |
| Ver painel do Consul | `http://SEU_IP:8500` |

---

*Guia gerado com base na stack: Docker Engine + HashiCorp Consul + HashiCorp Nomad + Docker Registry + Traefik v2*
