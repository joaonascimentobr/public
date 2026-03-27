# Guia de Configuração: PaaS com Nomad + Consul + Traefik

> Ubuntu Server 22.04 LTS — instalação do zero

---

## Sumário

1. [Visão Geral da Arquitetura](#1-visão-geral-da-arquitetura)
2. [Pré-Requisitos](#2-pré-requisitos)
3. [Preparação do Sistema](#3-preparação-do-sistema)
4. [Instalação do Docker](#4-instalação-do-docker)
5. [Instalação do Nomad e Consul](#5-instalação-do-nomad-e-consul)
6. [Configuração do Consul](#6-configuração-do-consul)
7. [Configuração do Nomad](#7-configuração-do-nomad)
8. [Garantindo a Ordem de Inicialização no Boot](#8-garantindo-a-ordem-de-inicialização-no-boot)
9. [Organização dos Arquivos de Jobs](#9-organização-dos-arquivos-de-jobs)
10. [Job: Docker Registry Privado](#10-job-docker-registry-privado)
11. [Job: Traefik](#11-job-traefik)
12. [Builder: Script de Build e Dockerfile](#12-builder-script-de-build-e-dockerfile)
13. [Job: Builder Parametrizado](#13-job-builder-parametrizado)
14. [Job: Aplicação Web (Template)](#14-job-aplicação-web-template)
15. [Fluxo Completo de Deploy](#15-fluxo-completo-de-deploy)
16. [Referência de Comandos Úteis](#16-referência-de-comandos-úteis)

---

## 1. Visão Geral da Arquitetura

Esta stack substitui painéis tradicionais de hospedagem por uma plataforma PaaS própria. O fluxo completo funciona assim:

```
git push
  -> webhook chega na sua API
  -> Nomad executa o job de build (clone + docker build + push para registry local)
  -> Nomad sobe o container da aplicação
  -> Traefik detecta via Consul, cria o roteamento e gera o SSL automaticamente
```

| Componente       | Função                                          | Porta padrão |
|------------------|-------------------------------------------------|--------------|
| Docker Engine    | Runtime dos containers                          | —            |
| HashiCorp Consul | Service Discovery e DNS interno                 | 8500         |
| HashiCorp Nomad  | Orquestrador (scheduler + executor)             | 4646         |
| Docker Registry  | Armazenamento local de imagens                  | 5000         |
| Traefik v2       | Proxy reverso, roteamento e SSL via Let's Encrypt | 80, 443, 8080 |

---

## 2. Pré-Requisitos

- Servidor com Ubuntu Server 22.04 LTS **zerado** (sem Apache, Nginx, ou qualquer painel instalado)
- Mínimo de 2 GB de RAM e 20 GB de disco
- Acesso root via SSH
- IP público fixo
- As seguintes portas abertas no firewall: `80`, `443`, `4646`, `8500`, `5000`, `8080`

---

## 3. Preparação do Sistema

Atualize o sistema e instale utilitários básicos:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget git nano
```

Crie os diretórios de dados persistentes:

```bash
sudo mkdir -p /opt/nomad
sudo mkdir -p /opt/consul
sudo mkdir -p /opt/registry-data
sudo chmod 777 /opt/registry-data
```

Crie a pasta de organização dos jobs Nomad:

```bash
mkdir -p ~/nomad-jobs
```

---

## 4. Instalação do Docker

Use o script oficial para garantir a versão mais recente do Docker Engine:

```bash
curl -fsSL https://get.docker.com | sh
```

Habilite o Docker para iniciar automaticamente com o sistema:

```bash
sudo systemctl enable docker --now
```

Verifique se está rodando:

```bash
sudo systemctl status docker
```

---

## 5. Instalação do Nomad e Consul

Adicione o repositório oficial da HashiCorp:

```bash
wget -O- https://apt.releases.hashicorp.com/gpg | \
  sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
  https://apt.releases.hashicorp.com $(lsb_release -cs) main" | \
  sudo tee /etc/apt/sources.list.d/hashicorp.list

sudo apt update && sudo apt install -y nomad consul
```

---

## 6. Configuração do Consul

O Consul é responsável pelo Service Discovery. Quando um container sobe, ele se registra no Consul. O Traefik lê esse registro para rotear o tráfego automaticamente.

Antes de criar o arquivo de configuração, descubra o IP da interface de rede principal do servidor:

```bash
ip addr show eth0 | grep "inet " | awk '{print $2}' | cut -d/ -f1
```

Guarde esse IP, pois ele será usado na configuração abaixo.

> **Atencao:** Nao use `{{ GetPrivateIP }}` no `bind_addr`. Em servidores com Docker instalado, esse template resolve para o IP da interface `docker0` (ex: `172.17.0.1`) em vez do IP real do servidor, fazendo o Consul escutar na interface errada. Sempre defina o IP explicitamente.

Crie o arquivo de configuração:

```bash
sudo nano /etc/consul.d/consul.hcl
```

Conteúdo (substitua `SEU_IP_REAL` pelo IP obtido no comando acima):

```hcl
datacenter = "dc1"
data_dir   = "/opt/consul"
server     = true

# Servidor único — espera apenas 1 nó
bootstrap_expect = 1

ui_config {
  enabled = true
}

client_addr = "0.0.0.0"
bind_addr   = "SEU_IP_REAL"
```

Habilite e inicie o serviço:

```bash
sudo systemctl enable consul --now
```

Verifique se o `Cluster Addr` mostra o IP correto (e não `172.17.x.x`):

```bash
journalctl -u consul.service --no-pager | grep "Cluster Addr"
```

A interface web do Consul fica disponível em `http://SEU_IP:8500`.

---

## 7. Configuração do Nomad

O Nomad atuará como Server (agendamento) e Client (execução) na mesma máquina.

Crie o arquivo de configuração:

```bash
sudo nano /etc/nomad.d/nomad.hcl
```

Conteúdo:

```hcl
datacenter = "dc1"
data_dir   = "/opt/nomad"

# Conexão com o Consul — essencial para o Traefik funcionar
consul {
  address = "127.0.0.1:8500"
}

server {
  enabled          = true
  bootstrap_expect = 1
}

client {
  enabled = true
  options = {
    "driver.docker.enable"       = "1"
    "docker.privileged.enabled"  = "true"
  }
}

plugin "docker" {
  config {
    allow_privileged = true
    volumes {
      enabled = true
    }
  }
}
```

Habilite e inicie o serviço:

```bash
sudo systemctl enable nomad --now
```

A interface web do Nomad fica disponível em `http://SEU_IP:4646`.

---

## 8. Garantindo a Ordem de Inicialização no Boot

O Nomad depende do Consul para funcionar corretamente. É necessário configurar o systemd para respeitar essa dependência ao inicializar o servidor.

Localize o arquivo de serviço do Nomad:

```bash
sudo find /etc/systemd /lib/systemd -name "nomad.service" 2>/dev/null
```

Edite o arquivo encontrado:

```bash
sudo nano /lib/systemd/system/nomad.service
```

Na seção `[Unit]`, remova o `#` das duas linhas referentes ao Consul:

```ini
[Unit]
Description=Nomad
Documentation=https://developer.hashicorp.com/nomad/docs
Wants=network-online.target
After=network-online.target

# Remova o # das duas linhas abaixo:
Wants=consul.service
After=consul.service
```

Recarregue o systemd para aplicar a mudança:

```bash
sudo systemctl daemon-reload
sudo systemctl restart nomad
```

Com isso, ao reiniciar o servidor, o Linux garante a ordem: Docker -> Consul -> Nomad. Todos os jobs são restaurados automaticamente pelo Nomad ao subir.

---

## 9. Organização dos Arquivos de Jobs

Mantenha todos os arquivos `.nomad` organizados em uma única pasta:

```
~/nomad-jobs/
  registry.nomad
  traefik.nomad
  builder.nomad
  webapp-exemplo.nomad
```

---

## 10. Job: Docker Registry Privado

O Registry armazena as imagens construídas localmente, evitando dependência do Docker Hub e acelerando os deploys.

Crie o arquivo:

```bash
nano ~/nomad-jobs/registry.nomad
```

Conteúdo:

```hcl
job "registry" {
  datacenters = ["dc1"]
  type        = "service"

  group "registry-config" {
    count = 1

    network {
      port "http" {
        static = 5000
      }
    }

    task "registry-server" {
      driver = "docker"

      config {
        image        = "registry:2"
        network_mode = "host"

        mounts = [
          {
            type   = "bind"
            source = "/opt/registry-data"
            target = "/var/lib/registry"
          }
        ]
      }

      resources {
        cpu    = 500
        memory = 512
      }

      service {
        name = "registry"
        port = "http"
        check {
          type     = "http"
          path     = "/"
          interval = "10s"
          timeout  = "2s"
        }
      }
    }
  }
}
```

Registre e suba o job:

```bash
nomad job run ~/nomad-jobs/registry.nomad
```

Teste se está respondendo:

```bash
curl http://localhost:5000/v2/
# Resposta esperada: {}
```

> **Nota sobre rede:** O `network_mode = "host"` é necessário para que o Registry escute em `0.0.0.0:5000` e seja acessível via `localhost` pelos outros jobs.

---

## 11. Job: Traefik

O Traefik substitui o servidor web tradicional. Ele detecta automaticamente novos serviços no Consul e configura o roteamento com SSL.

Crie o arquivo:

```bash
nano ~/nomad-jobs/traefik.nomad
```

Conteúdo (substitua `seu-email@dominio.com` pelo seu e-mail real para o Let's Encrypt):

```hcl
job "traefik" {
  datacenters = ["dc1"]
  type        = "system"

  group "traefik" {
    network {
      port "http"  { static = 80   }
      port "https" { static = 443  }
      port "ui"    { static = 8080 }
    }

    task "traefik" {
      driver = "docker"

      config {
        image        = "traefik:v2.10"
        network_mode = "host"

        args = [
          "--api.insecure=true",
          "--providers.consulcatalog=true",
          "--providers.consulcatalog.exposedByDefault=false",
          "--entrypoints.web.address=:80",
          "--entrypoints.websecure.address=:443",
          "--certificatesresolvers.le.acme.email=seu-email@dominio.com",
          "--certificatesresolvers.le.acme.storage=/acme.json",
          "--certificatesresolvers.le.acme.httpchallenge.entrypoint=web",
        ]
      }

      resources {
        cpu    = 200
        memory = 256
      }
    }
  }
}
```

Suba o job:

```bash
nomad job run ~/nomad-jobs/traefik.nomad
```

O dashboard do Traefik fica disponível em `http://SEU_IP:8080`.

> **Sobre SSL em produção:** O certificado Let's Encrypt só funciona com um domínio real apontando para o IP do servidor. Para testes locais, remova as linhas de `certificatesresolvers` e use apenas o entrypoint `web` (porta 80).

---

## 12. Builder: Script de Build e Dockerfile

O Builder é o "robô de construção" que clona um repositório Git, executa o `docker build` e envia a imagem para o Registry local.

### 12.1 Criar a pasta do projeto

```bash
mkdir -p ~/paas-builder
cd ~/paas-builder
```

### 12.2 Script Python (`builder.py`)

```bash
nano ~/paas-builder/builder.py
```

Conteúdo:

```python
import os
import subprocess
import sys

REPO_URL = os.getenv('NOMAD_META_REPO_URL')
APP_NAME = os.getenv('NOMAD_META_APP_NAME')
BUILD_ID = os.getenv('NOMAD_META_BUILD_ID')
REGISTRY = "localhost:5000"

def run_command(command):
    print(f"--> Executando: {command}", flush=True)
    try:
        subprocess.check_call(command, shell=True)
    except subprocess.CalledProcessError as e:
        print(f"Erro ao executar comando: {e}")
        sys.exit(1)

def main():
    if not all([REPO_URL, APP_NAME, BUILD_ID]):
        print("Erro: faltam variaveis de ambiente (REPO_URL, APP_NAME, BUILD_ID)")
        sys.exit(1)

    work_dir  = f"/builds/{APP_NAME}-{BUILD_ID}"
    image_tag = f"{REGISTRY}/{APP_NAME}:{BUILD_ID}"

    print(f"=== Iniciando Build para {APP_NAME} (ID: {BUILD_ID}) ===", flush=True)

    run_command(f"rm -rf {work_dir}")
    run_command(f"git clone --depth 1 {REPO_URL} {work_dir}")

    print("=== Construindo Imagem ===", flush=True)
    run_command(f"docker build -t {image_tag} {work_dir}")

    print("=== Enviando para Registry Local ===", flush=True)
    run_command(f"docker push {image_tag}")

    run_command(f"rm -rf {work_dir}")
    run_command(f"docker rmi {image_tag}")

    print("=== SUCESSO! Imagem pronta. ===", flush=True)

if __name__ == "__main__":
    main()
```

### 12.3 Dockerfile do Builder (`Dockerfile.builder`)

```bash
nano ~/paas-builder/Dockerfile.builder
```

Conteúdo:

```dockerfile
FROM python:3.9-slim

RUN apt-get update && apt-get install -y \
    git \
    docker.io \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app
COPY builder.py .

RUN mkdir /builds

CMD ["python", "builder.py"]
```

### 12.4 Construir a imagem do Builder

Este comando deve ser executado no servidor. A imagem fica armazenada localmente no Docker do host:

```bash
cd ~/paas-builder
docker build -t meu-paas-builder:v1 -f Dockerfile.builder .
```

Verifique se a imagem foi criada:

```bash
docker images | grep builder
```

> **Importante:** Se o servidor for recriado ou a imagem for removida, este passo precisa ser executado novamente antes de disparar qualquer build.

---

## 13. Job: Builder Parametrizado

Este job fica "registrado" no Nomad aguardando chamadas. Cada vez que sua API acionar um deploy, ela dispara uma instância deste job passando os parâmetros do repositório.

Crie o arquivo:

```bash
nano ~/nomad-jobs/builder.nomad
```

Conteúdo:

```hcl
job "builder-dispatcher" {
  datacenters = ["dc1"]
  type        = "batch"

  parameterized {
    payload       = "forbidden"
    meta_required = ["REPO_URL", "APP_NAME", "BUILD_ID"]
  }

  group "builder" {
    task "build-script" {
      driver = "docker"

      config {
        image = "meu-paas-builder:v1"

        # Monta o socket do Docker do host para que o container
        # possa executar comandos docker sem precisar de Docker-in-Docker
        mounts = [
          {
            type   = "bind"
            source = "/var/run/docker.sock"
            target = "/var/run/docker.sock"
          }
        ]

        # Necessário para acessar localhost:5000 (o Registry)
        network_mode = "host"
      }

      env {
        NOMAD_META_REPO_URL = "${NOMAD_META_REPO_URL}"
        NOMAD_META_APP_NAME = "${NOMAD_META_APP_NAME}"
        NOMAD_META_BUILD_ID = "${NOMAD_META_BUILD_ID}"
        PYTHONUNBUFFERED    = "1"
      }

      resources {
        cpu    = 500
        memory = 512
      }
    }
  }
}
```

Registre o job no Nomad:

```bash
nomad job run ~/nomad-jobs/builder.nomad
```

### 13.1 Como disparar um build (teste manual)

```bash
nomad job dispatch \
  -meta REPO_URL="https://github.com/dockersamples/linux_tweet_app" \
  -meta APP_NAME="meu-site" \
  -meta BUILD_ID="v1" \
  builder-dispatcher
```

Acompanhe os logs com o Allocation ID retornado pelo comando acima:

```bash
nomad alloc logs -stderr -f <ALLOC_ID>
```

Se quiser verificar o status detalhado da execução:

```bash
nomad alloc status <ALLOC_ID>
```

---

## 14. Job: Aplicação Web (Template)

Este é o job que coloca a aplicação de um cliente no ar. Ele deve ser gerado pela sua API após um build bem-sucedido, substituindo as variáveis `APP_NAME`, `IMAGE_TAG` e `DOMAIN`.

Exemplo de arquivo (`~/nomad-jobs/webapp-exemplo.nomad`):

```hcl
job "webapp-meu-site" {
  datacenters = ["dc1"]
  type        = "service"

  group "web" {
    count = 1

    # Politica de restart em caso de falha
    restart {
      attempts = 10
      interval = "5m"
      delay    = "25s"
      mode     = "delay"
    }

    network {
      # O Nomad aloca uma porta dinamica no host e mapeia para a porta interna
      port "http" {
        to = 80
      }
    }

    service {
      name = "webapp-meu-site"
      port = "http"

      # As tags abaixo instruem o Traefik a criar o roteamento automaticamente
      tags = [
        "traefik.enable=true",
        "traefik.http.routers.meu-site.rule=Host(`meu-site.seudominio.com`)",
        "traefik.http.routers.meu-site.entrypoints=websecure",
        "traefik.http.routers.meu-site.tls.certresolver=le"
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
        # Imagem construida pelo Builder e armazenada no Registry local
        image = "localhost:5000/meu-site:v1"
        ports = ["http"]
      }

      env {
        NODE_ENV = "production"
        PORT     = "80"
      }

      # Limites de recurso — ajuste conforme o plano do cliente
      resources {
        cpu    = 250
        memory = 256
      }
    }
  }
}
```

Suba o job:

```bash
nomad job run ~/nomad-jobs/webapp-exemplo.nomad
```

### 14.1 Sobre o DNS

**Para testes sem domínio real**, use o serviço `nip.io`. Ele resolve qualquer subdomínio que contenha um IP diretamente para aquele IP, sem necessidade de configurar DNS:

```
meu-site.191.252.194.7.nip.io  ->  resolves para  191.252.194.7
```

Substitua a tag do Traefik por:

```hcl
"traefik.http.routers.meu-site.rule=Host(`meu-site.SEU_IP.nip.io`)"
"traefik.http.routers.meu-site.entrypoints=web"
# Remova a linha de tls.certresolver para testes sem HTTPS
```

**Para produção com domínio real**, o cliente aponta o DNS dele (tipo A) para o IP do servidor. Nenhuma configuração adicional é necessária no Traefik — ele detecta o domínio e gera o certificado SSL automaticamente.

---

## 15. Fluxo Completo de Deploy

```
1. Cliente faz  git push  no repositório

2. GitHub/GitLab envia webhook para  api.sua-plataforma.com/deploy

3. Sua API:
   - Autentica a requisicao
   - Gera um BUILD_ID (ex: timestamp ou hash do commit)
   - Chama a API do Nomad:
       POST /v1/jobs  (dispatch do builder-dispatcher)
     com os metadados: REPO_URL, APP_NAME, BUILD_ID

4. Nomad (Builder):
   - Clona o repositorio
   - Executa  docker build
   - Envia a imagem para  localhost:5000/APP_NAME:BUILD_ID
   - Encerra o container de build

5. Sua API (apos o build):
   - Gera o JSON do job da aplicacao com a nova BUILD_ID
   - Chama:  PUT /v1/jobs/webapp-APP_NAME

6. Nomad (Aplicacao):
   - Baixa a nova imagem do Registry local
   - Executa Rolling Update (sobe o novo, verifica health check, derruba o antigo)
   - Registra o servico no Consul

7. Traefik:
   - Detecta o novo servico via Consul
   - Atualiza o roteamento
   - Gera/renova o certificado SSL se necessario

8. Site no ar sem downtime.
```

### 15.1 Sobrevivência a reinicializações

Todos os componentes estão configurados para sobreviver a um reboot do servidor:

- `systemctl enable docker` — Docker inicia automaticamente
- `systemctl enable consul` — Consul inicia automaticamente
- `systemctl enable nomad` — Nomad inicia após o Consul (configurado na seção 8)
- Nomad lê o estado salvo em `/opt/nomad` e restaura todos os jobs automaticamente
- O Registry restaura as imagens salvas em `/opt/registry-data`

---

## 16. Referência de Comandos Úteis

### Nomad

```bash
# Status geral do cluster
nomad server members
nomad node status

# Listar todos os jobs
nomad job list

# Ver status de um job
nomad job status <JOB_ID>

# Ver logs de uma alocacao
nomad alloc logs -stderr -f <ALLOC_ID>
nomad alloc logs -f <ALLOC_ID>

# Status detalhado de uma alocacao
nomad alloc status <ALLOC_ID>

# Parar um job
nomad job stop <JOB_ID>

# Forcar novo deploy (sem alterar o arquivo)
nomad job restart <JOB_ID>
```

### Consul

```bash
# Listar servicos registrados
consul catalog services

# Verificar saude dos servicos
consul health service <NOME_DO_SERVICO>
```

### Docker Registry

```bash
# Listar repositorios no registry local
curl http://localhost:5000/v2/_catalog

# Listar tags de uma imagem
curl http://localhost:5000/v2/<NOME_DA_IMAGEM>/tags/list

# Testar push manual
docker pull alpine:latest
docker tag alpine:latest localhost:5000/teste:v1
docker push localhost:5000/teste:v1
```

### Docker geral

```bash
# Ver containers rodando
docker ps

# Ver todas as imagens
docker images

# Remover imagens nao utilizadas
docker image prune -f
```

---

## Estrutura final de arquivos no servidor

```
/opt/
  nomad/           <- estado persistente do Nomad
  consul/          <- estado persistente do Consul
  registry-data/   <- imagens do Docker Registry

~/nomad-jobs/
  registry.nomad
  traefik.nomad
  builder.nomad
  webapp-*.nomad

~/paas-builder/
  builder.py
  Dockerfile.builder
```
