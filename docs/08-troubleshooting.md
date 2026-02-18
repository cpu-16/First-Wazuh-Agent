# Troubleshooting y Referencias

## 9) Preguntas de prueba para tu AI Agent

### Alertas (9200)

- "Muéstrame las últimas 10 alertas de las últimas 2 horas."
- "Resume las alertas de las últimas 24h por severidad."
- "¿Qué agente generó más alertas en las últimas 24h?"

### Agentes (55000)

- "Lista mis agentes y dime cuáles están active y cuáles disconnected."
- "Dime la versión y última conexión de cada agente."

### Vulnerabilidades (9200)

- "Dame el resumen de vulnerabilidades por agente (Critical/High/Medium/Low)."
- "¿Qué agente tiene más vulnerabilidades High?"

---

## 10) Troubleshooting rápido

### 10.1 401 Unauthorized (Indexer)

Verifica usuario/clave `admin` del tar `wazuh-install-files.tar`

**Prueba:**

```bash
curl -sk -u "admin:PASS" https://100.109.242.43:9200/_opendistro/_security/authinfo?pretty
```

### 10.2 Indexer solo responde en localhost

**Confirma:**

```bash
sudo ss -tlnp | grep 9200
```

Debe escuchar en `0.0.0.0:9200` o `:::9200` si quieres acceso remoto.

### 10.3 n8n self-signed certificate

**Solución segura:** `NODE_EXTRA_CA_CERTS` + volumen con CA/cert.

---

## 11) Endpoints clave

### API Wazuh (55000)

| Endpoint | Método | Descripción |
|----------|--------|-------------|
| `/security/user/authenticate?raw=true` | POST | Obtener JWT token |
| `/agents?limit=...&select=...` | GET | Listar agentes |

### Indexer (9200)

| Endpoint | Método | Descripción |
|----------|--------|-------------|
| `/_cat/indices?v` | GET | Listar índices |
| `/wazuh-alerts-4.x-*/_search` | POST | Buscar alertas |
| `/wazuh-alerts-4.x-*/_count` | POST | Contar alertas |
| `/wazuh-states-vulnerabilities-*/_search` | POST | Buscar vulnerabilidades |
| `/wazuh-states-vulnerabilities-*/_mapping` | GET | Ver mapping de vulnerabilidades |

---

[← Volver al README principal](../README.md)
