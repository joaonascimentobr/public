# PaaS Próprio com Nomad + Consul + Traefik

Uma plataforma PaaS self-hosted, leve e sem dependências externas. Substitui painéis tradicionais de hospedagem com uma stack moderna baseada em containers, service discovery e proxy reverso automático.

---

## Funcionalidades

- **Deploy via Git** — faça push e a plataforma cuida do resto
- **Build automatizado** — clone, build e push de imagens Docker sem intervenção manual
- **Roteamento automático** — Traefik detecta novos serviços e configura as rotas dinamicamente
- **SSL automático** — certificados Let's Encrypt gerados e renovados sem configuração extra
- **Registry privado** — armazenamento local de imagens, sem dependência do Docker Hub
- **Zero downtime** — rolling updates com health checks antes de derrubar a versão anterior
- **Alta resiliência** — todos os serviços sobrevivem a reboots e são restaurados automaticamente

---

## Arquitetura

```
git push
  → webhook chega na sua API
  → Nomad executa o job de build (clone + docker build + push para registry local)
  → Nomad sobe o container da aplicação
  → Traefik detecta via Consul, cria o roteamento e gera o SSL automaticamente
```

| Componente          | Função                                            | Porta padrão   |
|---------------------|---------------------------------------------------|----------------|
| **Docker Engine**   | Runtime dos containers                            | —              |
| **HashiCorp Consul**| Service Discovery e DNS interno                   | 8500           |
| **HashiCorp Nomad** | Orquestrador (scheduler + executor)               | 4646           |
| **Docker Registry** | Armazenamento local de imagens                    | 5000           |
| **Traefik v2**      | Proxy reverso, roteamento e SSL via Let's Encrypt | 80, 443, 8080  |

---

## Pré-Requisitos

- Ubuntu Server 22.04 LTS (sem Apache, Nginx ou painéis instalados)
- Mínimo de **2 GB de RAM** e **20 GB de disco**
- Acesso root via SSH
- IP público fixo
- Portas abertas no firewall: `80`, `443`, `4646`, `8500`, `5000`, `8080`

---

## Fluxo de Deploy

```
1. git push  no repositório do cliente

2. Webhook chega em  api.sua-plataforma.com/deploy

3. Sua API:
   → Autentica a requisição
   → Gera um BUILD_ID (timestamp ou hash do commit)
   → Dispara o job de build no Nomad com: REPO_URL, APP_NAME, BUILD_ID

4. Nomad (Builder):
   → Clona o repositório
   → Executa docker build
   → Envia a imagem para localhost:5000/APP_NAME:BUILD_ID
   → Encerra o container de build

5. Sua API (após o build):
   → Gera o job da aplicação com a nova BUILD_ID
   → Submete via PUT /v1/jobs/webapp-APP_NAME

6. Nomad (Aplicação):
   → Baixa a nova imagem do Registry local
   → Executa Rolling Update com health check
   → Registra o serviço no Consul

7. Traefik:
   → Detecta o novo serviço via Consul
   → Atualiza o roteamento automaticamente
   → Gera/renova o certificado SSL

8.  Site no ar, sem downtime.
```

---

## Estrutura do Repositório

```
/opt/
  nomad/            ← estado persistente do Nomad
  consul/           ← estado persistente do Consul
  registry-data/    ← imagens do Docker Registry

~/nomad-jobs/
  registry.nomad    ← Docker Registry privado
  traefik.nomad     ← Proxy reverso + SSL
  builder.nomad     ← Job de build parametrizado
  webapp-*.nomad    ← Jobs das aplicações

~/paas-builder/
  builder.py        ← Script de build
  Dockerfile.builder
```

---

## Documentação

| Tópico | Descrição |
|--------|-----------|
| [Guia de Instalação](./docs/setup.md) | Instalação completa do zero no Ubuntu 22.04 |
| [Jobs Nomad](./docs/nomad-jobs.md) | Referência dos jobs: Registry, Traefik, Builder, Webapp |
| [Fluxo de Deploy](./docs/deploy-flow.md) | Como a API deve integrar com o Nomad |
| [Comandos Úteis](./docs/commands.md) | Referência rápida de CLI para Nomad, Consul e Docker |

> A documentação de instalação cobre: preparação do sistema, instalação do Docker, Nomad e Consul, configuração de cada componente e ordem de boot garantida via systemd.

---

## Tecnologias

- [HashiCorp Nomad](https://www.nomadproject.io/)
- [HashiCorp Consul](https://www.consul.io/)
- [Traefik v2](https://traefik.io/)
- [Docker Engine](https://docs.docker.com/engine/)
- [Let's Encrypt](https://letsencrypt.org/)
