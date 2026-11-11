# Tutorial Completo: Stack de Monitoramento e Observabilidade Unificada no Kubernetes (Zabbix + Prometheus + Grafana)

Este guia prático fornece o passo a passo completo para implementar uma solução de observabilidade enterprise de alta disponibilidade no Kubernetes, integrando **Zabbix 7.0**, **kube-prometheus-stack** e **Grafana** automatizados via **Ansible** e **Ansible Semaphore**.

---

## 1. Arquitetura da Stack de Observabilidade

A solução foi projetada para combinar o melhor das duas abordagens de monitoramento:
1. **Prometheus + Kube-State-Metrics + Node Exporter:** Monitoramento baseado em métricas de séries temporais (Pull) focado nos componentes nativos do Kubernetes, pods, nós e tráfego de rede via Traefik.
2. **Zabbix Server + Zabbix Agent 2:** Monitoramento proativo focado na camada de infraestrutura física/virtual (Proxmox VE, VMs Ubuntu, storage NFS, banco de dados e testes ICMP/SNMP).
3. **Grafana Unificado:** Painel central de visualização conectando o Prometheus e o Zabbix como fontes de dados (DataSources), oferecendo dashboards correlacionados.

### Arquitetura de Comunicação e Portas

```text
                                +-----------------------------------+
                                |      Navegador do Administrador   |
                                +-----------------+-----------------+
                                                  | (HTTP/HTTPS)
                                                  v
                                +-----------------------------------+
                                |  MetalLB LoadBalancer (10.0.39.200)|
                                +-----------------+-----------------+
                                                  |
                                                  v
                                +-----------------------------------+
                                |    Traefik Ingress Controller     |
                                +----+------------+------------+----+
                                     |            |            |
             +-----------------------+            |            +-----------------------+
             |                                    |                                    |
             v                                    v                                    v
+------------------------+             +--------------------+             +------------------------+
|  zabbix-k8s.nverse...  |             | grafana-k8s...     |             | prometheus-k8s...      |
|  (Zabbix Web UI - 80)  |             | (Grafana UI - 80)  |             | (Prometheus UI - 80)   |
+-----------+------------+             +---------+----------+             +-----------+------------+
            |                                    |                                    |
            v                                    | (DataSource API)                   v
+------------------------+                       +-----------------------> +------------------------+
|     Zabbix Server      |                                                 |    Prometheus Server   |
|   (Porta 10051/TCP)    |                                                 |  (TSDB / Alertmanager) |
+-----------+------------+                                                 +-----------+------------+
            |                                                                          |
            | (Scrape Agent 10050)                                                     | (Scrape Metrics /metrics)
            v                                                                          v
+---------------------------------------------------------------------------------------------------+
|               Infraestrutura de VMs (Proxmox, Control Planes, Workers, NFS Server)                 |
+---------------------------------------------------------------------------------------------------+
```

---

## 2. Estrutura de Diretórios do Repositório (`monitoring-environment`)

O repositório dedicado **`monitoring-environment`** segue uma organização em camadas numeradas (`00` a `04`), garantindo ordenação estrita na execução do Ansible Semaphore:

```text
monitoring-environment/
├── README.md
├── inventories/
│   └── production.ini
├── playbooks/
│   ├── 00-base/
│   │   └── 00-prepara-namespaces-storage.yml
│   ├── 01-zabbix/
│   │   ├── 01-configura-banco-zabbix.yml
│   │   ├── 02-deploy-zabbix-server.yml
│   │   └── 03-instala-zabbix-agents-vms.yml
│   ├── 02-prometheus/
│   │   ├── 04-instala-prometheus-stack.yml
│   │   └── 05-configura-rules-alertas.yml
│   ├── 03-grafana/
│   │   ├── 06-instala-grafana.yml
│   │   ├── 07-configura-datasources.yml
│   │   └── 08-importa-dashboards.yml
│   └── 04-testing/
│       └── 09-valida-conectividade-stack.yml
└── site.yml
```

---

## 3. Inventário de Hosts (`inventories/production.ini`)

O inventário segrega os alvos entre o cluster Kubernetes (onde os pods da stack sobem) e as VMs do ambiente (onde os agentes do Zabbix e Node Exporters são instalados).

```ini
[k8s_control_plane]
k8s-control-plane-01 ansible_host=10.0.39.51 ansible_user=joelfernandes
k8s-control-plane-02 ansible_host=10.0.39.52 ansible_user=joelfernandes
k8s-control-plane-03 ansible_host=10.0.39.53 ansible_user=joelfernandes

[k8s_workers]
k8s-worker-01 ansible_host=10.0.39.61 ansible_user=joelfernandes
k8s-worker-02 ansible_host=10.0.39.62 ansible_user=joelfernandes
k8s-worker-03 ansible_host=10.0.39.63 ansible_user=joelfernandes

[loadbalancers]
k8s-haproxy-01 ansible_host=10.0.39.41 ansible_user=joelfernandes
k8s-haproxy-02 ansible_host=10.0.39.42 ansible_user=joelfernandes

[storage]
nfs-server ansible_host=10.0.39.30 ansible_user=joelfernandes

[all_vms:children]
k8s_control_plane
k8s_workers
loadbalancers
storage

[all:vars]
ansible_python_interpreter=/usr/bin/python3
ansible_ssh_common_args='-o StrictHostKeyChecking=no'
```

---

## 4. Playbooks Ansible da Esteira de Monitoramento

### Playbook Mestre: `site.yml`

```yaml
---
# =========================================================================
# Playbook Mestre — Esteira Completa de Monitoramento e Observabilidade
# =========================================================================

- import_playbook: playbooks/00-base/00-prepara-namespaces-storage.yml
- import_playbook: playbooks/01-zabbix/01-configura-banco-zabbix.yml
- import_playbook: playbooks/01-zabbix/02-deploy-zabbix-server.yml
- import_playbook: playbooks/01-zabbix/03-instala-zabbix-agents-vms.yml
- import_playbook: playbooks/02-prometheus/04-instala-prometheus-stack.yml
- import_playbook: playbooks/02-prometheus/05-configura-rules-alertas.yml
- import_playbook: playbooks/03-grafana/06-instala-grafana.yml
- import_playbook: playbooks/03-grafana/07-configura-datasources.yml
- import_playbook: playbooks/03-grafana/08-importa-dashboards.yml
- import_playbook: playbooks/04-testing/09-valida-conectividade-stack.yml
```

---

### Step 00: `00-prepara-namespaces-storage.yml`

Cria os namespaces `monitoring` e `zabbix` no Kubernetes e garante a existência dos PVCs persistentes via NFS.

```yaml
---
- name: 00 - Prepara Namespaces e PVCs no Kubernetes
  hosts: k8s-control-plane-01
  become: yes
  vars:
    kubeconfig_path: "/etc/kubernetes/admin.conf"

  tasks:
    - name: Criar Namespace zabbix
      kubernetes.core.k8s:
        kubeconfig: "{{ kubeconfig_path }}"
        name: zabbix
        api_version: v1
        kind: Namespace
        state: present

    - name: Criar Namespace monitoring
      kubernetes.core.k8s:
        kubeconfig: "{{ kubeconfig_path }}"
        name: monitoring
        api_version: v1
        kind: Namespace
        state: present

    - name: Criar PVC para o Banco de Dados do Zabbix (NFS)
      kubernetes.core.k8s:
        kubeconfig: "{{ kubeconfig_path }}"
        state: present
        definition:
          apiVersion: v1
          kind: PersistentVolumeClaim
          metadata:
            name: zabbix-db-pvc
            namespace: zabbix
          spec:
            accessModes:
              - ReadWriteOnce
            storageClassName: nfs-client
            resources:
              requests:
                storage: 20Gi
```

---

### Step 01 e 02: Deployment do Zabbix Server & PostgreSQL (`02-deploy-zabbix-server.yml`)

Implanta o PostgreSQL 16 e o Zabbix Server 7.0 com Web UI exposta no Traefik Ingress.

```yaml
---
- name: 02 - Deploy do Zabbix Server e Web UI no Kubernetes
  hosts: k8s-control-plane-01
  become: yes
  vars:
    kubeconfig_path: "/etc/kubernetes/admin.conf"
    zabbix_domain: "zabbix-k8s.nverse.local"

  tasks:
    - name: Criar Deployment do Zabbix Server e Web Interface
      kubernetes.core.k8s:
        kubeconfig: "{{ kubeconfig_path }}"
        state: present
        definition:
          apiVersion: apps/v1
          kind: Deployment
          metadata:
            name: zabbix-server
            namespace: zabbix
          spec:
            replicas: 1
            selector:
              matchLabels:
                app: zabbix-server
            template:
              metadata:
                labels:
                  app: zabbix-server
              spec:
                containers:
                  - name: zabbix-server
                    image: zabbix/zabbix-server-pgsql:ubuntu-7.0-latest
                    env:
                      - name: DB_SERVER_HOST
                        value: "zabbix-db-service"
                      - name: POSTGRES_DB
                        value: "zabbix"
                      - name: POSTGRES_USER
                        value: "zabbix"
                      - name: POSTGRES_PASSWORD
                        value: "<DB_PASSWORD_PLACEHOLDER>"
                    ports:
                      - containerPort: 10051

    - name: Criar IngressRoute no Traefik para o Zabbix Web UI
      kubernetes.core.k8s:
        kubeconfig: "{{ kubeconfig_path }}"
        state: present
        definition:
          apiVersion: traefik.io/v1alpha1
          kind: IngressRoute
          metadata:
            name: zabbix-web-ingress
            namespace: zabbix
          spec:
            entryPoints:
              - web
            routes:
              - match: Host(`{{ zabbix_domain }}`)
                kind: Rule
                services:
                  - name: zabbix-web-service
                    port: 8080
```

---

### Step 03: Instalação dos Zabbix Agents nas VMs (`03-instala-zabbix-agents-vms.yml`)

Instala o Zabbix Agent 2 em todas as VMs da infraestrutura apontando para o IP do Zabbix Server.

```yaml
---
- name: 03 - Instala e Configura Zabbix Agent 2 em todas as VMs
  hosts: all_vms
  become: yes
  vars:
    zabbix_server_ip: "10.0.39.200"

  tasks:
    - name: Instalar Zabbix Agent 2
      ansible.builtin.apt:
        name: zabbix-agent2
        update_cache: yes
        state: present

    - name: Configurar IP do Zabbix Server no agent
      ansible.builtin.lineinfile:
        path: /etc/zabbix/zabbix_agent2.conf
        regexp: '^Server='
        line: "Server={{ zabbix_server_ip }}"

    - name: Reiniciar e habilitar o serviço Zabbix Agent 2
      ansible.builtin.systemd:
        name: zabbix-agent2
        state: restarted
        enabled: yes
```

---

### Step 04: Kube-Prometheus-Stack via Helm (`04-instala-prometheus-stack.yml`)

```yaml
---
- name: 04 - Instala Kube-Prometheus-Stack via Helm
  hosts: k8s-control-plane-01
  become: yes
  vars:
    kubeconfig_path: "/etc/kubernetes/admin.conf"
    prometheus_domain: "prometheus-k8s.nverse.local"

  tasks:
    - name: Adicionar repositorio prometheus-community
      kubernetes.core.helm_repository:
        name: prometheus-community
        repo_url: "prometheus-community-repo"

    - name: Instalar Chart kube-prometheus-stack
      kubernetes.core.helm:
        name: kube-prometheus-stack
        chart_ref: prometheus-community/kube-prometheus-stack
        release_namespace: monitoring
        create_namespace: false
        kubeconfig: "{{ kubeconfig_path }}"
        values:
          prometheus:
            prometheusSpec:
              retention: 15d
              storageSpec:
                volumeClaimTemplate:
                  spec:
                    storageClassName: nfs-client
                    resources:
                      requests:
                        storage: 20Gi
```

---

### Step 06 e 07: Instalação do Grafana e DataSources (`07-configura-datasources.yml`)

Instala o Grafana e adiciona automaticamente as fontes de dados do Prometheus interno e do plugin do Zabbix.

```yaml
---
- name: 07 - Configura DataSources no Grafana (Prometheus + Zabbix)
  hosts: k8s-control-plane-01
  become: yes
  vars:
    kubeconfig_path: "/etc/kubernetes/admin.conf"
    grafana_domain: "grafana-k8s.nverse.local"

  tasks:
    - name: Aplicar ConfigMap de DataSources no Grafana
      kubernetes.core.k8s:
        kubeconfig: "{{ kubeconfig_path }}"
        state: present
        definition:
          apiVersion: v1
          kind: ConfigMap
          metadata:
            name: grafana-datasources
            namespace: monitoring
            labels:
              grafana_datasource: "1"
          data:
            datasources.yaml: |
              apiVersion: 1
              datasources:
                - name: Prometheus-K8s
                  type: prometheus
                  access: proxy
                  url: http://kube-prometheus-stack-prometheus.monitoring.svc.cluster.local:9090
                  isDefault: true
                - name: Zabbix-Server
                  type: alexanderzobnin-zabbix-datasource
                  access: proxy
                  url: http://zabbix-web-service.zabbix.svc.cluster.local:8080/api_jsonrpc.php
                  jsonData:
                    username: "Admin"
```

---

## 5. Mapeamento de Task Templates no Ansible Semaphore

| ID | Nome do Task Template | Playbook Filename | Inventário |
| :--- | :--- | :--- | :--- |
| **00** | `00 - Prepara Namespaces e Storage` | `playbooks/00-base/00-prepara-namespaces-storage.yml` | `production.ini` |
| **01** | `01 - Configura Banco do Zabbix` | `playbooks/01-zabbix/01-configura-banco-zabbix.yml` | `production.ini` |
| **02** | `02 - Deploy Zabbix Server e Web` | `playbooks/01-zabbix/02-deploy-zabbix-server.yml` | `production.ini` |
| **03** | `03 - Instala Zabbix Agents em VMs` | `playbooks/01-zabbix/03-instala-zabbix-agents-vms.yml` | `production.ini` |
| **04** | `04 - Instala Kube-Prometheus-Stack` | `playbooks/02-prometheus/04-instala-prometheus-stack.yml` | `production.ini` |
| **05** | `05 - Configura Regras e Alertas` | `playbooks/02-prometheus/05-configura-rules-alertas.yml` | `production.ini` |
| **06** | `06 - Instala Grafana` | `playbooks/03-grafana/06-instala-grafana.yml` | `production.ini` |
| **07** | `07 - Configura DataSources Grafana` | `playbooks/03-grafana/07-configura-datasources.yml` | `production.ini` |
| **08** | `08 - Importa Dashboards Grafana` | `playbooks/03-grafana/08-importa-dashboards.yml` | `production.ini` |
| **09** | `09 - Valida Conectividade da Stack` | `playbooks/04-testing/09-valida-conectividade-stack.yml` | `production.ini` |
| **MASTER** | `00 - Esteira Completa Monitoramento` | `site.yml` | `production.ini` |

---

## 6. Endpoints e Acesso aos Dashboards

Após a execução da esteira completa, adicione a entrada no `/etc/hosts` da sua máquina física apontando para o IP do MetalLB (`10.0.39.200`):

```bash
# Adicionar no /etc/hosts local:
10.0.39.200 zabbix-k8s.nverse.local grafana-k8s.nverse.local prometheus-k8s.nverse.local
```

### URLs de Acesso:
* **Grafana:** `http://grafana-k8s.nverse.local` (Usuário: `admin` | Senha inicial: `prom-operator`)
* **Zabbix Web UI:** `http://zabbix-k8s.nverse.local` (Usuário: `Admin` | Senha: `zabbix`)
* **Prometheus UI:** `http://prometheus-k8s.nverse.local`

---

## 7. Guia de Troubleshooting

### Cenário 1: Plugin do Zabbix não aparece no Grafana
**Causa:** O container do Grafana não baixou o plugin `alexanderzobnin-zabbix-datasource` na inicialização.  
**Solução:** Instale o plugin no container do Grafana e reinicie o deployment:
```bash
kubectl exec -it deployment/grafana -n monitoring -- grafana-cli plugins install alexanderzobnin-zabbix-datasource
kubectl rollout restart deployment/grafana -n monitoring
```

### Cenário 2: Zabbix Agent status "Refused" ou "Timeout"
**Causa:** Bloqueio na porta `10050` ou IP do Zabbix Server configurado incorretamente no `/etc/zabbix/zabbix_agent2.conf`.  
**Solução:**
```bash
# Testar serviço e logs na VM do agente:
sudo systemctl status zabbix-agent2
sudo journalctl -u zabbix-agent2 -n 50 --no-pager
```

---
*Documentação gerada para o repositório `monitoring-environment`.*
