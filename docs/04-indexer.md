# Indexer Wazuh (Puerto 9200)

## 5) Probar Indexer (9200) con curl

### 5.1 Confirmar auth (security authinfo)

```bash
curl -sk -u "admin:TU_PASSWORD_ADMIN" \
  "https://100.109.242.43:9200/_opendistro/_security/authinfo?pretty"
```

![Auth info del Indexer](images/authinfo.png)

### 5.2 Listar índices

```bash
curl -sk -u "admin:TU_PASSWORD_ADMIN" \
  "https://100.109.242.43:9200/_cat/indices?v"
```

Filtrar alertas y vulnerabilidades:

```bash
curl -sk -u "admin:TU_PASSWORD_ADMIN" \
  "https://100.109.242.43:9200/_cat/indices?v" | grep -E "wazuh-alerts|vulnerabil"
```

![Índices de Wazuh](images/indices.png)

### 5.3 Buscar alertas recientes (últimas 24h)

```bash
curl -sk -u "admin:TU_PASSWORD_ADMIN" \
  -H "Content-Type: application/json" \
  -X POST "https://100.109.242.43:9200/wazuh-alerts-4.x-*/_search" \
  -d '{
    "size": 10,
    "sort": [{ "@timestamp": { "order": "desc" } }],
    "_source": ["@timestamp","agent.id","agent.name","rule.id","rule.level","rule.description","data","manager.name"],
    "query": { "range": { "@timestamp": { "gte": "now-24h", "lt": "now" } } }
  }' | jq .
```

![Alertas recientes](images/alertas-recientes.png)

### 5.4 Agregación: quién genera más alertas (24h)

```bash
curl -sk -u "admin:TU_PASSWORD_ADMIN" \
  -H "Content-Type: application/json" \
  -X POST "https://100.109.242.43:9200/wazuh-alerts-4.x-*/_search" \
  -d '{
    "size": 0,
    "query": { "range": { "@timestamp": { "gte": "now-24h", "lt": "now" } } },
    "aggs": {
      "by_agent": { "terms": { "field": "agent.name", "size": 50, "order": { "_count": "desc" } } }
    }
  }' | jq .
```

![Agregación por agente](images/agregacion-agentes.png)

---

## Endpoints Indexer (9200)

| Endpoint | Método | Descripción |
|----------|--------|-------------|
| `/_cat/indices?v` | GET | Listar índices |
| `/wazuh-alerts-4.x-*/_search` | POST | Buscar alertas |
| `/wazuh-alerts-4.x-*/_count` | POST | Contar alertas |

---

[← Volver al README principal](../README.md)
