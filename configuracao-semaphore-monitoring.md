# Guia de Configuração do Ansible Semaphore e Estrutura do Repositório (`monitoring-environment`)

Este documento serve como o manual definitivo para a estruturação do repositório Git **`monitoring-environment`** e a criação do projeto de automação no **Ansible Semaphore**.

---

## 📁 1. Arquitetura e Estrutura de Pastas do Repositório

### Árvore Completa do Repositório (`monitoring-environment`)

```text
monitoring-environment/
├── README.md                                      # Documentação geral do repositório
├── production.ini                                 # Inventário principal com VMs e IPs do cluster
├── site-monitoring.yml                            # Playbook mestre para deploy automatizado em 1 clique
└── playbooks/
    ├── 00-base/
    │   └── 00-prepara-namespaces-storage.yml     # Criação do Namespace 'monitoring' e PVCs NFS
    ├── 01-zabbix/
    │   ├── 01-configura-banco-zabbix.yml          # Implantação e preparação do PostgreSQL
    │   ├── 02-deploy-zabbix-server.yml            # Deploy do Zabbix Server e Zabbix Web UI
    │   └── 03-instala-zabbix-agents-vms.yml       # Instalação do zabbix-agent2 em todas as VMs
    ├── 02-prometheus/
    │   ├── 04-instala-prometheus-stack.yml        # Instalação do kube-prometheus-stack via Helm
    │   └── 05-configura-rules-alertas.yml         # Definição das PrometheusRules (Alertmanager)
    ├── 03-grafana/
    │   ├── 06-instala-grafana.yml                 # Deploy do Grafana + Plugin oficial do Zabbix
    │   ├── 07-configura-datasources.yml           # Registro automático de DataSources (Prometheus + Zabbix API)
    │   └── 08-importa-dashboards.yml              # Importação de Dashboards via API / ConfigMap
    └── 04-testing/
        └── 09-valida-conectividade-stack.yml      # Testes E2E de integração entre a stack
```

### Script para Recriação Automatizada da Estrutura (`setup-estrutura.sh`)

```bash
#!/usr/bin/env bash
set -e

echo "🚀 Criando estrutura de diretórios para o repositório monitoring-environment..."

mkdir -p playbooks/00-base \
         playbooks/01-zabbix \
         playbooks/02-prometheus \
         playbooks/03-grafana \
         playbooks/04-testing

touch production.ini \
      site-monitoring.yml \
      README.md \
      playbooks/00-base/00-prepara-namespaces-storage.yml \
      playbooks/01-zabbix/01-configura-banco-zabbix.yml \
      playbooks/01-zabbix/02-deploy-zabbix-server.yml \
      playbooks/01-zabbix/03-instala-zabbix-agents-vms.yml \
      playbooks/02-prometheus/04-instala-prometheus-stack.yml \
      playbooks/02-prometheus/05-configura-rules-alertas.yml \
      playbooks/03-grafana/06-instala-grafana.yml \
      playbooks/03-grafana/07-configura-datasources.yml \
      playbooks/03-grafana/08-importa-dashboards.yml \
      playbooks/04-testing/09-valida-conectividade-stack.yml

echo "✅ Estrutura criada com sucesso!"
```

---

## ⚙️ 2. Configuração do Projeto no Ansible Semaphore

Para manter o isolamento e o padrão de governança DevOps, crie um **Novo Projeto** no Semaphore dedicado à camada de observabilidade.

### Passo 1: Criar o Projeto
* **Nome do Projeto:** `Monitoring & Observability`
* **Descrição:** `Esteira de automação para deploy da stack Zabbix, Prometheus e Grafana`

---

### Passo 2: Cadastrar Chaves no Key Store (`Key Store`)
Acesse **Key Store** > **New Key**:

1. **Chave SSH de Acesso aos Nós:**
   * **Name:** `SSH Key - Infra Nodes`
   * **Type:** `SSH Key`
   * **User:** `connect`
   * **Private Key:** *(Cole a chave privada SSH para acesso às VMs)*

2. **Chave de Acesso ao GitHub:**
   * **Name:** `GitHub Key - Monitoring Repo`
   * **Type:** `SSH Key`
   * **User:** `git`
   * **Private Key:** *(Cole a chave privada com permissão de leitura no repositório monitoring-environment)*

---

### Passo 3: Configurar o Repositório Git (`Repositories`)
Acesse **Repositories** > **New Repository**:

* **Name:** `monitoring-environment`
* **Git URL:** `git@github.com:JoelFernandes01/monitoring-environment.git`
* **Branch:** `main`
* **SSH Key:** `GitHub Key - Monitoring Repo`

---

### Passo 4: Configurar o Inventário (`Inventories`)
Acesse **Inventories** > **New Inventory**:

* **Name:** `Production Monitoring`
* **Type:** `File`
* **Repository:** `monitoring-environment`
* **File Path:** `production.ini`
* **SSH Key:** `SSH Key - Infra Nodes`

---

### Passo 5: Cadastrar o Mapeamento dos Task Templates (`Task Templates`)

Crie os 11 Task Templates conforme a tabela abaixo para manter a numeração estrita e a ordem de execução:

| Nº | Nome do Task Template no Semaphore | Playbook Filename | Repository | Inventory | Environment |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **00** | `00 - Prepara Namespaces e Storage` | `playbooks/00-base/00-prepara-namespaces-storage.yml` | `monitoring-environment` | `Production Monitoring` | Default |
| **01** | `01 - Configura Banco de Dados Zabbix` | `playbooks/01-zabbix/01-configura-banco-zabbix.yml` | `monitoring-environment` | `Production Monitoring` | Default |
| **02** | `02 - Deploy Zabbix Server e Web` | `playbooks/01-zabbix/02-deploy-zabbix-server.yml` | `monitoring-environment` | `Production Monitoring` | Default |
| **03** | `03 - Instala Zabbix Agent nas VMs` | `playbooks/01-zabbix/03-instala-zabbix-agents-vms.yml` | `monitoring-environment` | `Production Monitoring` | Default |
| **04** | `04 - Instala Prometheus Stack via Helm` | `playbooks/02-prometheus/04-instala-prometheus-stack.yml` | `monitoring-environment` | `Production Monitoring` | Default |
| **05** | `05 - Configura Regras de Alertas` | `playbooks/02-prometheus/05-configura-rules-alertas.yml` | `monitoring-environment` | `Production Monitoring` | Default |
| **06** | `06 - Deploy e Configuração do Grafana` | `playbooks/03-grafana/06-instala-grafana.yml` | `monitoring-environment` | `Production Monitoring` | Default |
| **07** | `07 - Registra DataSources no Grafana` | `playbooks/03-grafana/07-configura-datasources.yml` | `monitoring-environment` | `Production Monitoring` | Default |
| **08** | `08 - Importa Dashboards Automatizados` | `playbooks/03-grafana/08-importa-dashboards.yml` | `monitoring-environment` | `Production Monitoring` | Default |
| **09** | `09 - Teste Integrado E2E da Stack` | `playbooks/04-testing/09-valida-conectividade-stack.yml` | `monitoring-environment` | `Production Monitoring` | Default |
| **SITE**| `DEPLOY COMPLETO - Stack Monitoramento` | `site-monitoring.yml` | `monitoring-environment` | `Production Monitoring` | Default |

---

## 🚀 3. Ordem de Execução Recomendada

1. Executar a sequência individualmente de **00** a **09** na primeira implementação para acompanhar os testes e logs de cada camada.
2. Para novos deploys em ambientes limpos, utilizar o Task Template **`DEPLOY COMPLETO - Stack Monitoramento`** (`site-monitoring.yml`) para provisionar toda a solução em 1 clique.
