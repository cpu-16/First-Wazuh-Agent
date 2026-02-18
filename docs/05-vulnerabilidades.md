# Vulnerabilidades

## 6) Vulnerabilidades: `wazuh-states-vulnerabilities-*`

### 6.1 Ver mapping para ubicar campos

```bash
curl -sk -u "admin:TU_PASSWORD_ADMIN" \
  "https://100.109.242.43:9200/wazuh-states-vulnerabilities-*/_mapping?pretty" | head -n 120
```

![Mapping de vulnerabilidades](images/vuln-mapping.png)

### 6.2 Probar si existen docs

```bash
curl -sk -u "admin:TU_PASSWORD_ADMIN" \
  -H "Content-Type: application/json" \
  -X POST "https://100.109.242.43:9200/wazuh-states-vulnerabilities-*/_search" \
  -d '{"size":1,"query":{"match_all":{}}}' | jq .
```

### 6.3 Resumen por agente y severidad

Ajusta el campo según tu mapping. En este lab suele ser: `vulnerability.severity`.

```bash
curl -sk -u "admin:TU_PASSWORD_ADMIN" \
  -H "Content-Type: application/json" \
  -X POST "https://100.109.242.43:9200/wazuh-states-vulnerabilities-*/_search" \
  -d '{
    "size": 0,
    "aggs": {
      "by_agent": {
        "terms": { "field": "agent.name", "size": 50 },
        "aggs": {
          "by_severity": { "terms": { "field": "vulnerability.severity", "size": 10 } }
        }
      }
    }
  }' | jq .
```

> ⚠️ Si filtrar por `@timestamp` te da 0 resultados, es porque vulnerabilities puede no usar `@timestamp` como campo de tiempo. Primero revisa mapping y usa el campo correcto.

![Vulnerabilidades por severidad](images/vuln-severidad.png)

---

## Endpoints Vulnerabilidades (9200)

| Endpoint | Método | Descripción |
|----------|--------|-------------|
| `/wazuh-states-vulnerabilities-*/_search` | POST | Buscar vulnerabilidades |
| `/wazuh-states-vulnerabilities-*/_mapping` | GET | Ver mapping de vulnerabilidades |

---

[← Volver al README principal](../README.md)
