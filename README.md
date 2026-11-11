# Monitoring & Observability Environment (Zabbix + Prometheus + Grafana)

Este repositorio contem a automacao declarativa completa em **Ansible** para implantar uma stack integrada de Observabilidade no cluster Kubernetes Proxmox VE.

## 🚀 Estrutura do Repositorio

```text
monitoring-environment/
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

## ⚙️ IPs dos Servicos Expostos (MetalLB)

- **Zabbix Web UI:** `10.0.39.201`
- **Grafana UI:** `10.0.39.202`
- **Prometheus Server:** `kube-prometheus-stack-prometheus.monitoring.svc:9090` (Internal)
