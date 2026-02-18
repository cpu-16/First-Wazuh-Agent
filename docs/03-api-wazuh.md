# API Wazuh (Puerto 55000)

## 4) Probar API Wazuh (55000) con curl (JWT)

### 4.1 Obtener JWT (token raw)

```bash
TOKEN=$(curl -sk -u "wazuh:TU_PASSWORD_API" \
  -X POST "https://100.109.242.43:55000/security/user/authenticate?raw=true")

echo "$TOKEN"
```

![Token JWT obtenido](images/jwt-token.png)

### 4.2 Listar agentes con estado real

```bash
curl -sk -H "Authorization: Bearer $TOKEN" \
 "https://100.109.242.43:55000/agents?limit=100&select=id,name,status,lastKeepAlive,version,ip" | jq .
```

**Estados comunes:**
- `active`
- `disconnected`

![Lista de agentes](images/lista-agentes.png)

---

## Endpoints API Wazuh (55000)

| Endpoint | Método | Descripción |
|----------|--------|-------------|
| `/security/user/authenticate?raw=true` | POST | Obtener JWT token |
| `/agents?limit=...&select=...` | GET | Listar agentes |

---

[← Volver al README principal](../README.md)
