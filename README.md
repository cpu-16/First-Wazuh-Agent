# Wazuh + Agentes + n8n + APIs (55000 / 9200)

<div align="center">

![Wazuh](https://img.shields.io/badge/Wazuh-SIEM-005571?style=for-the-badge&logo=wazuh&logoColor=white)
![OpenSearch](https://img.shields.io/badge/OpenSearch-Indexer-005EB8?style=for-the-badge&logo=opensearch&logoColor=white)
![n8n](https://img.shields.io/badge/n8n-Automation-EA4B71?style=for-the-badge&logo=n8n&logoColor=white)
![Security](https://img.shields.io/badge/Security-Monitoring-00D9FF?style=for-the-badge&logo=security&logoColor=white)

**Laboratorio completo de Wazuh con automatización n8n y consultas API**

</div>

---

## Documentación

| Sección | Descripción |
|---------|-------------|
| [01 - Instalación](docs/01-instalacion.md) | Instalación Wazuh All-in-one + Servicios y puertos |
| [02 - Agentes](docs/02-agentes.md) | Conectar agentes al Manager |
| [03 - API Wazuh](docs/03-api-wazuh.md) | API 55000 + autenticación JWT |
| [04 - Indexer](docs/04-indexer.md) | OpenSearch en puerto 9200 |
| [05 - Vulnerabilidades](docs/05-vulnerabilidades.md) | Consultas de vulnerabilidades |
| [06 - n8n](docs/06-n8n.md) | Integración con n8n |
| [07 - Certificados SSL](docs/07-certificados.md) | Solución self-signed certificate |
| [08 - Troubleshooting](docs/08-troubleshooting.md) | Preguntas, errores comunes, endpoints |

---

## Demo en video

![Demo Wazuh + n8n Integration](docs/images/video.gif)

Este README documenta el laboratorio completo:

- Instalación de Wazuh (Manager + Indexer + Dashboard)
- Conexión de agentes (hosts) hacia el Manager
- Consultas al Indexer (OpenSearch) por puerto **9200**
- Uso de la API de Wazuh por puerto **55000** (JWT)
- Flujo en **n8n** para:
  - consultar alertas
  - consultar vulnerabilidades
  - listar agentes activos/desconectados
- Solución del error **self-signed certificate** en n8n montando el CA/cert

---

## Datos del Laboratorio

**Ejemplo de configuración real:**

| Componente | Endpoint | Puerto |
|------------|----------|--------|
| Indexer | `https://100.109.242.43:9200` | 9200 |
| API Wazuh | `https://100.109.242.43:55000` | 55000 |
| Dashboard | `https://100.109.242.43:5601` | 5601 |
| Agentes (UDP) | `100.109.242.43:1514` | 1514/udp |
| Agentes (TCP) | `100.109.242.43:1515` | 1515/tcp |

**Credenciales:**
- Usuario Indexer: `admin`
- Usuario API: `wazuh`
- Stack n8n: `~/n8n-ngrok/docker-compose.yml`

> **Nota:** Usando Tailscale para acceso seguro remoto

---

## Servicios y puertos

### Verificar servicios

```bash
sudo systemctl status wazuh-manager
sudo systemctl status wazuh-indexer
sudo systemctl status wazuh-dashboard
sudo systemctl status filebeat
```

![Estado de servicios Wazuh](docs/images/servicios-status.png)

### Verificar puertos

```bash
sudo ss -tlnp | egrep '(:55000|:9200|:5601|:1514|:1515)'
```

![Puertos escuchando](docs/images/puertos.png)

---

## Conectar agentes al Manager

### Puertos para agentes

- **Registro/autenticación:** `1515/tcp`
- **Recepción de eventos:** `1514/udp`

### Instalar agente (ejemplo Debian/Ubuntu)

```bash
sudo dpkg -i wazuh-agent_*.deb
```

### Configurar el Manager en el agente

Editar `/var/ossec/etc/ossec.conf` y apuntar al Manager en `<server>`.

Luego:

```bash
sudo systemctl enable wazuh-agent
sudo systemctl restart wazuh-agent
```

![Agente conectado](docs/images/agente-conectado.png)

---

## Probar API Wazuh (55000) con curl (JWT)

### Obtener JWT (token raw)

```bash
TOKEN=$(curl -sk -u "wazuh:TU_PASSWORD_API" \
  -X POST "https://100.109.242.43:55000/security/user/authenticate?raw=true")

echo "$TOKEN"
```

![Token JWT obtenido](docs/images/jwt-token.png)

### Listar agentes con estado real

```bash
curl -sk -H "Authorization: Bearer $TOKEN" \
 "https://100.109.242.43:55000/agents?limit=100&select=id,name,status,lastKeepAlive,version,ip" | jq .
```

**Estados comunes:**
- `active`
- `disconnected`

![Lista de agentes](docs/images/lista-agentes.png)

---

## Probar Indexer (9200) con curl

### Confirmar auth (security authinfo)

```bash
curl -sk -u "admin:TU_PASSWORD_ADMIN" \
  "https://100.109.242.43:9200/_opendistro/_security/authinfo?pretty"
```

![Auth info del Indexer](docs/images/authinfo.png)

### Listar índices

```bash
curl -sk -u "admin:TU_PASSWORD_ADMIN" \
  "https://100.109.242.43:9200/_cat/indices?v"
```

Filtrar alertas y vulnerabilidades:

```bash
curl -sk -u "admin:TU_PASSWORD_ADMIN" \
  "https://100.109.242.43:9200/_cat/indices?v" | grep -E "wazuh-alerts|vulnerabil"
```

![Índices de Wazuh](docs/images/indices.png)

### Buscar alertas recientes (últimas 24h)

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

![Alertas recientes](docs/images/alertas-recientes.png)

### Agregación: quién genera más alertas (24h)

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

![Agregación por agente](docs/images/agregacion-agentes.png)

---

## Vulnerabilidades: `wazuh-states-vulnerabilities-*`

### Ver mapping para ubicar campos

```bash
curl -sk -u "admin:TU_PASSWORD_ADMIN" \
  "https://100.109.242.43:9200/wazuh-states-vulnerabilities-*/_mapping?pretty" | head -n 120
```

![Mapping de vulnerabilidades](docs/images/vuln-mapping.png)

### Probar si existen docs

```bash
curl -sk -u "admin:TU_PASSWORD_ADMIN" \
  -H "Content-Type: application/json" \
  -X POST "https://100.109.242.43:9200/wazuh-states-vulnerabilities-*/_search" \
  -d '{"size":1,"query":{"match_all":{}}}' | jq .
```

### Resumen por agente y severidad

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

> Si filtrar por `@timestamp` te da 0 resultados, es porque vulnerabilities puede no usar `@timestamp` como campo de tiempo. Primero revisa mapping y usa el campo correcto.

![Vulnerabilidades por severidad](docs/images/vuln-severidad.png)

---

## Integración con n8n (actualizado)

**Recomendación práctica:**

- **2 nodos HTTP para Indexer (9200)**
  - 1 para alertas/búsquedas y agregaciones
  - 1 para vulnerabilidades (Vuln summary by agent)
- **1 Tool node (Code)** para la API de agentes (JWT + Bearer), a través del proxy con certificado válido

**No conviene mezclar todo en un solo nodo porque:**
- cambian endpoints (9200 vs 55001)
- cambia el tipo de auth (Basic vs JWT/Bearer)
- cambia la semántica (alertas/vulns vs agentes)

### Nodo HTTP (Indexer 9200) — Body ejemplo (últimas 2h)

**Configuración:**

- **Method:** POST
- **URL:** `https://100.109.242.43:9200/wazuh-alerts-4.x-*/_search`
- **Auth:** Basic (`admin` + pass)
- **Header:** `Content-Type: application/json`

**Body:**

```json
{
  "size": 10,
  "sort": [{ "@timestamp": { "order": "desc" } }],
  "_source": ["@timestamp","agent.id","agent.name","rule.id","rule.level","rule.description"],
  "query": { "range": { "@timestamp": { "gte": "now-2h", "lt": "now" } } }
}
```

![Nodo HTTP Indexer en n8n](docs/images/n8n-indexer.png)

### Nodo HTTP (Indexer 9200) — Vuln summary by agent

**Configuración (ejemplo genérico para "resumen por agente"):**

- **Method:** POST
- **URL:** `https://100.109.242.43:9200/wazuh-states-vulnerabilities-*/_search`
- **Auth:** Basic (`admin` + pass)
- **Header:** `Content-Type: application/json`

**Body (agregación por agente + severidad):**

```json
{
  "size": 0,
  "query": {
    "bool": {
      "filter": [
        { "range": { "@timestamp": { "gte": "now-24h", "lt": "now" } } }
      ]
    }
  },
  "aggs": {
    "by_agent": {
      "terms": { "field": "agent.name", "size": 200 },
      "aggs": {
        "by_severity": {
          "terms": { "field": "vulnerability.severity", "size": 10 }
        }
      }
    }
  }
}
```

> **Nota:** si tu índice usa otros campos (por ejemplo `agent.name.keyword` o severidad en otro path), ajusta los `field` según tu mapping.

![Nodo HTTP Vuln summary by agent en n8n](docs/images/n8n-vuln-summary.png)

### Tool node (API Agentes) — wazuh_list_agents (JWT + Bearer vía proxy)

En vez de usar dos nodos (Auth + List agents) apuntando a `:55000` con certificados "solo localhost", se usa:

**Reverse proxy (Caddy)** con TLS interno exponiendo:
```
https://wazuh.taild88ec5.ts.net:55001
```

Un **Tool node** que:
1. hace POST auth → token raw
2. hace GET /agents con Bearer
3. devuelve un string TSV (requerido por AI Tool en n8n)

**Código completo del Tool (wazuh_list_agents):**

```javascript
// Tool: wazuh_list_agents
// n8n AI Tool (Code): debe devolver STRING (TSV)
// Proxy HTTPS (Caddy) -> Wazuh API local

const BASE = "https://wazuh.taild88ec5.ts.net:55001";
const USER = "wazuh";
const PASS = "TU_PASSWORD";

// 1) Auth (token raw como texto)
const tokenRaw = await this.helpers.httpRequest({
  method: "POST",
  url: `${BASE}/security/user/authenticate?raw=true`,
  auth: { username: USER, password: PASS },
  json: false,
});

// Normaliza token
const token = String(tokenRaw).replace(/^"+|"+$/g, "").trim();
if (!token) throw new Error("Token vacío");

// 2) List agents
const agentsResp = await this.helpers.httpRequest({
  method: "GET",
  url: `${BASE}/agents?limit=200&select=id,name,status,lastKeepAlive,version,ip`,
  headers: { Authorization: `Bearer ${token}` },
  json: true,
});

const items = agentsResp?.data?.affected_items ?? [];

// TSV
const header = "name\tid\tstatus\tlastKeepAlive\tversion\tip";
const rows = items.map((a) => [
  a?.name ?? "N/D",
  a?.id ?? "N/D",
  a?.status ?? "N/D",
  a?.lastKeepAlive ?? "N/D",
  a?.version ?? "N/D",
  a?.ip ?? "N/D",
].join("\t"));

// IMPORTANTE: devolver string directo
return [header, ...rows].join("\n");
```

> **Nota:** este Tool debe estar conectado al AI Agent por `ai_tool`.

![Tool node wazuh_list_agents](docs/images/n8n-tool-agents.png)

---

## Solución: certificados para API Wazuh en n8n (actualizado)

### Diagnóstico del problema original

El certificado del API expuesto en `:55000` era autofirmado y tenía SAN solo:

```
DNS:localhost
```

Por eso fallaba al conectarse usando:
- IP Tailscale (`100.x.x.x`)
- o DNS de Tailscale (`wazuh.taild88ec5.ts.net`)

### Solución aplicada: Reverse Proxy con Caddy + TLS Internal

Se instaló **Caddy** en el servidor Wazuh (Ubuntu) y se expuso el API en:

```
https://wazuh.taild88ec5.ts.net:55001
```

certificado válido para ese DNS (emitido por Caddy CA)

y se hizo proxy hacia:

```
https://127.0.0.1:55000 (API real)
```

#### Instalar Caddy (Ubuntu)

```bash
sudo apt update
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https curl

curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' \
| sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg

curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' \
| sudo tee /etc/apt/sources.list.d/caddy-stable.list

sudo apt update
sudo apt install -y caddy
```

![Instalación de Caddy](docs/images/caddy-install.png)

#### Configurar Caddyfile

Editar:

```bash
sudo nano /etc/caddy/Caddyfile
```

**Contenido:**

```caddy
wazuh.taild88ec5.ts.net:55001 {

  tls internal

  reverse_proxy https://127.0.0.1:55000 {
    transport http {
      tls
      tls_insecure_skip_verify
    }
  }
}
```

Validar y recargar:

```bash
sudo caddy validate --config /etc/caddy/Caddyfile
sudo systemctl reload caddy
```

![Caddyfile configurado](docs/images/caddyfile.png)

#### Probar el proxy (esperado 401)

```bash
curl -vk https://wazuh.taild88ec5.ts.net:55001/ | head
```

Debe responder **401 Unauthorized** porque no hay token.

### Confiar en el certificado de Caddy desde n8n

Como Caddy usa una CA interna, se exporta el CA root y se monta en el contenedor n8n.

#### Exportar CA root (en server Wazuh)

```bash
sudo cp /var/lib/caddy/.local/share/caddy/pki/authorities/local/root.crt /tmp/caddy-root.crt
sudo chmod 644 /tmp/caddy-root.crt
```

Copiar al host n8n (ejemplo):

```bash
scp /tmp/caddy-root.crt gar16@<HOST_N8N>:~/n8n-ngrok/certs/caddy-root.crt
```

![Exportar certificado Caddy](docs/images/caddy-cert-export.png)

#### Montar CA en docker-compose de n8n

En `~/n8n-ngrok/docker-compose.yml`, en el servicio `n8n`:

**volumes:**

```yaml
- ./certs/caddy-root.crt:/certs/caddy-root.crt:ro
```

**environment:**

```yaml
- NODE_EXTRA_CA_CERTS=/certs/caddy-root.crt
```

Reiniciar:

```bash
cd ~/n8n-ngrok
docker compose up -d
```

#### Prueba dentro del contenedor n8n (sin curl)

```bash
docker compose exec n8n sh -lc 'node -e "require(\"https\").get(\"https://wazuh.taild88ec5.ts.net:55001/\",res=>{console.log(\"status\",res.statusCode);res.resume();}).on(\"error\",e=>{console.error(e.message);});"'
```

Debe devolver `status 401`.

![Certificado montado en n8n](docs/images/n8n-caddy-cert.png)

---

## Preguntas de prueba para tu AI Agent (actualizado)

### Alertas (9200)

- "Muéstrame las últimas 10 alertas de las últimas 2 horas."
- "Resume las alertas de las últimas 24h por severidad."
- "¿Qué agente generó más alertas en las últimas 24h?"

### Agentes (Tool wazuh_list_agents)

- "Lista mis agentes y dime cuáles están active y cuáles disconnected."
- "Dime la versión y última conexión de cada agente."
- "¿Cuál es la IP y el estado del agente proxmox?"

### Vulnerabilidades (9200)

- "Dame el resumen de vulnerabilidades por agente (Critical/High/Medium/Low)."
- "¿Qué agente tiene más vulnerabilidades High?"

---

## Troubleshooting rápido (actualizado)

### 401 Unauthorized (Indexer)

Verifica usuario/clave `admin` del tar `wazuh-install-files.tar`

**Prueba:**

```bash
curl -sk -u "admin:PASS" https://100.109.242.43:9200/_opendistro/_security/authinfo?pretty
```

### Proxy Caddy (55001) responde 401

Correcto si llamas `/` sin token.

Para endpoints protegidos usa Bearer token.

### Error de certificado en n8n contra 55001

- Revisa montaje de `caddy-root.crt`
- Revisa `NODE_EXTRA_CA_CERTS=/certs/caddy-root.crt`
- Reinicia contenedor n8n

---

## Endpoints clave (actualizado)

### API Wazuh (vía Caddy proxy)

| Endpoint | Método | Descripción |
|----------|--------|-------------|
| `https://wazuh.taild88ec5.ts.net:55001/security/user/authenticate?raw=true` | POST | Obtener JWT token (texto raw) |
| `https://wazuh.taild88ec5.ts.net:55001/agents?limit=...&select=...` | GET | Listar agentes |

### Indexer (9200)

| Endpoint | Método | Descripción |
|----------|--------|-------------|
| `/_cat/indices?v` | GET | Listar índices |
| `/wazuh-alerts-4.x-*/_search` | POST | Buscar alertas |
| `/wazuh-alerts-4.x-*/_count` | POST | Contar alertas |
| `/wazuh-states-vulnerabilities-*/_search` | POST | Buscar vulnerabilidades |
| `/wazuh-states-vulnerabilities-*/_mapping` | GET | Ver mapping de vulnerabilidades |

---

## Contribuir

1. Fork el proyecto
2. Crea tu rama: `git checkout -b feature/nueva-funcionalidad`
3. Commit: `git commit -m 'Añade nueva funcionalidad'`
4. Push: `git push origin feature/nueva-funcionalidad`
5. Abre un Pull Request

---

## Licencia

Este proyecto es libre de usar para propósitos educativos y de laboratorio.

---

## Agradecimientos

- [Wazuh Documentation](https://documentation.wazuh.com/)
- [OpenSearch Documentation](https://opensearch.org/docs/)
- [n8n Documentation](https://docs.n8n.io/)
- [Tailscale](https://tailscale.com/)

---

<div align="center">

**SIEM completo con Wazuh + automatización n8n!**

</div>
