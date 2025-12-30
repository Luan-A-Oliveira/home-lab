# 📘 Documentação do Ambiente de LAB – Acesso Remoto Seguro

> **Objetivo:** Documentar todo o passo a passo realizado até o momento para permitir acesso remoto **seguro** ao ambiente Proxmox utilizando **Cloudflare Tunnel** e **Cloudflare Zero Trust**, sem exposição de IP público, portas abertas ou credenciais sensíveis.

---

## 🧱 Visão Geral da Arquitetura

```
Internet
   ↓
Cloudflare Zero Trust (MFA / Identity)
   ↓
Cloudflare Tunnel
   ↓
Proxmox (Web UI e SSH)
```

### Benefícios da abordagem

* Nenhum IP público exposto
* Nenhuma porta aberta no roteador
* Autenticação com MFA
* Controle de acesso por identidade
* Auditoria via Cloudflare

---

## 🖥️ Ambiente Base

| Item          | Descrição         |
| ------------- | ----------------- |
| Host          | Notebook de LAB   |
| Hypervisor    | Proxmox VE        |
| Acesso local  | Rede interna      |
| Acesso remoto | Cloudflare Tunnel |
| Domínio       | `seudominio.com`  |

---

## 1 Criação da VM para Cloudflare Tunnel

Foi criada uma VM dedicada para executar o **cloudflared**, seguindo boas práticas de isolamento.

### Especificações da VM

* Sistema: Debian 12
* CPU: 1 vCPU
* Memória: 512 MB – 1 GB
* Disco: 8–10 GB
* Função: **Apenas Cloudflare Tunnel**

---

## 2 Instalação do Cloudflared

### Atualização do sistema

```bash
apt update && apt upgrade -y
```

### Instalação do cloudflared

```bash
apt install -y cloudflared
```

---

## 3 Autenticação do Cloudflared na Conta Cloudflare

```bash
cloudflared tunnel login
```

* O comando abriu uma URL para autenticação
* Login realizado na conta Cloudflare
* Selecionado o domínio `seudominio.com`
* Certificado salvo automaticamente em:

```bash
/root/.cloudflared/cert.pem
```

> ⚠️ **Arquivo sensível – nunca versionar no GitHub**

---

## 4 Criação do Tunnel

```bash
cloudflared tunnel create lab-tunnel
```

Resultado:

* Tunnel criado com sucesso
* Arquivo de credenciais gerado automaticamente

```text
/root/.cloudflared/<TUNNEL-ID>.json
```

> ⚠️ **Arquivo sensível – nunca versionar no GitHub**

---

## 5 Configuração do Tunnel (`config.yml`)

Arquivo criado em:

```bash
/root/.cloudflared/config.yml
```

### Exemplo seguro de configuração (sem dados sensíveis)

```yaml
tunnel: <TUNNEL-ID>
credentials-file: /root/.cloudflared/<TUNNEL-ID>.json

ingress:
  - hostname: proxmox.seudominio.com
    service: https://IP_INTERNO_PROXMOX:8006
    originRequest:
      noTLSVerify: true

  - hostname: ssh.seudominio.com
    service: ssh://IP_INTERNO_PROXMOX:22

  - service: http_status:404
```

---

## 6 Criação dos DNS Records

No painel Cloudflare:

| Tipo  | Nome        | Destino                        |
| ----- | ----------- | ------------------------------ |
| CNAME | proxmox     | `<TUNNEL-ID>.cfargotunnel.com` |
| CNAME | ssh | `<TUNNEL-ID>.cfargotunnel.com` |

> Os registros apontam para o Tunnel, não para IPs públicos.

---

## 7 Execução do Tunnel

Teste manual:

```bash
cloudflared tunnel run lab-tunnel
```

Posteriormente, pode ser configurado como serviço:

```bash
cloudflared service install
systemctl enable cloudflared
systemctl start cloudflared
```

---

## 8 Acesso Web ao Proxmox

* URL: `https://proxmox.seudominio.com`
* Tráfego protegido pelo Tunnel
* Nenhuma exposição direta do Proxmox

---

## 9 Acesso SSH via Cloudflare Tunnel

### Comando de acesso

```bash
ssh usuario@ssh.seudominio.com
```

> O SSH passa pelo Tunnel e **não expõe a porta 22** publicamente.

---

## 🔐 10 Configuração do Cloudflare Zero Trust

### Ativação do Zero Trust

* Criado ambiente Zero Trust no painel Cloudflare
* Domínio gerado automaticamente:

```
seugrupo.cloudflareaccess.com
```

---

## 11 Application Policies (Zero Trust)

### Aplicações criadas

| Aplicação   | Tipo        | Domínio                 |
| ----------- | ----------- | ----------------------- |
| Proxmox Web | Self-hosted | proxmox.seudominio.com     |
| Proxmox SSH | SSH         | ssh.seudominio.com |

### Controles aplicados

* Autenticação obrigatória
* MFA habilitado
* Acesso restrito por identidade

---

## ✅ Resultado Final

## Próximos Passos

Os próximos passos deste LAB visam evoluir o ambiente para um **nível próximo de produção**, incorporando práticas modernas de **segurança, observabilidade e controle de acesso**.

### 1. VM Bastion (Jump Host)

* Criar uma VM dedicada para administração
* Acesso exclusivo via Cloudflare Zero Trust
* Instalação de ferramentas administrativas (kubectl, helm, terraform, ansible)
* Bloquear acesso SSH direto aos nodes

### 2. Cluster Kubernetes

* Criação do cluster (kubeadm ou k3s)
* Separação de nodes (control-plane / worker)
* Hardening básico

### 3. Istio (Service Mesh)

* Instalação do Istio no cluster
* Habilitar Ingress Gateway
* Mutual TLS (mTLS) entre serviços
* Traffic management (VirtualService, DestinationRule)

### 4. API Gateway (Istio Ingress)

* Uso do Istio Ingress Gateway como API Gateway
* Exposição controlada de serviços
* Rate limiting e headers de segurança
* Integração com Cloudflare Tunnel

### 5. Keycloak (Identity & Access Management)

* Deploy do Keycloak no Kubernetes
* Integração com banco de dados (PostgreSQL)
* Configuração de realms, clients e users
* Integração do Keycloak com:

  * Istio (JWT / AuthZ)
  * Aplicações internas
  * Cloudflare Access (OIDC)

### 6. CI/CD e Segurança

* GitHub Actions
* SonarQube rodando no cluster
* Scan de imagens (Trivy)
* GitOps (ArgoCD)

### 7. Observabilidade

* Prometheus
* Grafana
* Alertmanager

✔ Proxmox acessível remotamente
✔ Web UI protegida por Zero Trust
✔ SSH protegido por identidade + MFA
✔ Nenhuma porta exposta
✔ Nenhum IP público

---

## 🔒 Boas Práticas de Segurança Aplicadas

* Isolamento do Tunnel em VM dedicada
* Uso de identidade em vez de IP
* MFA obrigatório
* Credenciais fora de versionamento
* Domínio controlado via Cloudflare

---

## 📌 Próximos Passos Planejados

* Criação de VM Bastion
* Acesso interno centralizado
* Cluster Kubernetes
* CI/CD com GitHub Actions
* SonarQube no cluster
* Exposição via Cloudflare Tunnel

---

📅 **Documentação criada para fins educacionais e LAB pessoal**

