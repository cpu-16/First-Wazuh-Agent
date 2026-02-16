# 🛡️ Wazuh + Agentes + n8n + APIs (55000 / 9200)

<div align="center">

![Wazuh](https://img.shields.io/badge/Wazuh-SIEM-005571?style=for-the-badge&logo=wazuh&logoColor=white)
![OpenSearch](https://img.shields.io/badge/OpenSearch-Indexer-005EB8?style=for-the-badge&logo=opensearch&logoColor=white)
![n8n](https://img.shields.io/badge/n8n-Automation-EA4B71?style=for-the-badge&logo=n8n&logoColor=white)
![Security](https://img.shields.io/badge/Security-Monitoring-00D9FF?style=for-the-badge&logo=security&logoColor=white)

**Laboratorio completo de Wazuh con automatización n8n y consultas API** 🚀

</div>

---

## 📋 Tabla de Contenidos

- [Demo](#-demo-en-video)
- [Datos del Laboratorio](#-datos-del-laboratorio)
- [Instalación de Wazuh](#-1-instalación-de-wazuh-all-in-one)
- [Servicios y Puertos](#-2-servicios-y-puertos)
- [Conectar Agentes](#-3-conectar-agentes-al-manager)
- [API Wazuh (55000)](#-4-probar-api-wazuh-55000-con-curl-jwt)
- [Indexer (9200)](#-5-probar-indexer-9200-con-curl)
- [Vulnerabilidades](#-6-vulnerabilidades-wazuh-states-vulnerabilities-)
- [Integración n8n](#-7-integración-con-n8n)
- [Certificados SSL](#-8-solución-self-signed-certificate-en-n8n-docker)
- [Preguntas de Prueba](#-9-preguntas-de-prueba-para-tu-ai-agent)
- [Troubleshooting](#-10-troubleshooting-rápido)
- [Endpoints Clave](#-11-endpoints-clave)

---

## 🎥 Demo en video

![Demo Wazuh + n8n Integration](docs/images/demo.gif)

Este README documenta el laboratorio completo:

- ✅ Instalación de Wazuh (Manager + Indexer + Dashboard)
- ✅ Conexión de agentes (hosts) hacia el Manager
- ✅ Consultas al Indexer (OpenSearch) por puerto **9200**
- ✅ Uso de la API de Wazuh por puerto **55000** (JWT)
- ✅ Flujo en **n8n** para:
  - consultar alertas
  - consultar vulnerabilidades
  - listar agentes activos/desconectados
- ✅ Solución del error **self-signed certificate** en n8n montando el CA/cert

---

## 📊 Datos del Laboratorio

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

> 💡 **Nota:** Usando Tailscale para acceso seguro remoto

---

## 🔧 1) Instalación de Wazuh (All-in-one)

### 1.1 Requisitos

- Linux con systemd (Ubuntu/Debian/RHEL)
- Acceso root/sudo
- Puertos necesarios:
  - **55000** (Wazuh API)
  - **9200** (Indexer)
  - **1514/udp** y **1515/tcp** (agentes)
  - **5601** (Dashboard)

### 1.2 Instalación con el asistente

En el servidor Wazuh:

```bash
sudo bash ./wazuh-install.sh -a
```

Si existe una instalación previa y deseas borrar config y datos:

```bash
sudo bash ./wazuh-install.sh -a -o
```

![Instalación de Wazuh](docs/images/instalacion-wazuh.png)

### 1.3 Ver contraseñas y certificados generados

El instalador genera un tar con:

- cluster key
- certificados
- contraseñas

**Ejemplo:**

```bash
ls -lah ~/wazuh-install-files.tar
```

Ver el archivo de contraseñas sin extraer todo:

```bash
sudo tar -O -xvf ~/wazuh-install-files.tar wazuh-install-files/wazuh-passwords.txt
```

Extraer credenciales del usuario `admin` (Indexer):

```bash
sudo tar -O -xvf ~/wazuh-install-files.tar wazuh-install-files/wazuh-passwords.txt | \
  grep -n "indexer_username: 'admin'" -A1
```

![Credenciales del instalador](docs/images/credenciales.png)

---

## ⚙️ 2) Servicios y puertos

### 2.1 Verificar servicios

```bash
sudo systemctl status wazuh-manager
sudo systemctl status wazuh-indexer
sudo systemctl status wazuh-dashboard
sudo systemctl status filebeat
```

![Estado de servicios Wazuh](docs/images/servicios-status.png)

### 2.2 Verificar puertos

```bash
sudo ss -tlnp | egrep '(:55000|:9200|:5601|:1514|:1515)'
```

![Puertos escuchando](docs/images/puertos.png)

---

## 🔗 3) Conectar agentes al Manager

### 3.1 Puertos para agentes

- **Registro/autenticación:** `1515/tcp`
- **Recepción de eventos:** `1514/udp`

### 3.2 Instalar agente (ejemplo Debian/Ubuntu)

```bash
sudo dpkg -i wazuh-agent_*.deb
```

### 3.3 Configurar el Manager en el agente

Editar `/var/ossec/etc/ossec.conf` y apuntar al Manager en `<server>`.

Luego:

```bash
sudo systemctl enable wazuh-agent
sudo systemctl restart wazuh-agent
```

![Agente conectado](docs/images/agente-conectado.png)

---

## 🔑 4) Probar API Wazuh (55000) con curl (JWT)

### 4.1 Obtener JWT (token raw)

```bash
TOKEN=$(curl -sk -u "wazuh:TU_PASSWORD_API" \
  -X POST "https://100.109.242.43:55000/security/user/authenticate?raw=true")

echo "$TOKEN"
```

![Token JWT obtenido](docs/images/jwt-token.png)

### 4.2 Listar agentes con estado real

```bash
curl -sk -H "Authorization: Bearer $TOKEN" \
 "https://100.109.242.43:55000/agents?limit=100&select=id,name,status,lastKeepAlive,version,ip" | jq .
```

**Estados comunes:**
- `active`
- `disconnected`

![Lista de agentes](docs/images/lista-agentes.png)

---

## 🔍 5) Probar Indexer (9200) con curl

### 5.1 Confirmar auth (security authinfo)

```bash
curl -sk -u "admin:TU_PASSWORD_ADMIN" \
  "https://100.109.242.43:9200/_opendistro/_security/authinfo?pretty"
```

![Auth info del Indexer](docs/images/authinfo.png)

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

![Índices de Wazuh](docs/images/indices.png)

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

![Alertas recientes](docs/images/alertas-recientes.png)

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

![Agregación por agente](docs/images/agregacion-agentes.png)

---

## 🐛 6) Vulnerabilidades: `wazuh-states-vulnerabilities-*`

### 6.1 Ver mapping para ubicar campos

```bash
curl -sk -u "admin:TU_PASSWORD_ADMIN" \
  "https://100.109.242.43:9200/wazuh-states-vulnerabilities-*/_mapping?pretty" | head -n 120
```

![Mapping de vulnerabilidades](docs/images/vuln-mapping.png)

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

![Vulnerabilidades por severidad](docs/images/vuln-severidad.png)

---

## 🤖 7) Integración con n8n

**Recomendación práctica:**

- 1 nodo HTTP para Indexer (9200) (Basic Auth)
- 1 nodo HTTP para API 55000 (JWT + Bearer)

**No conviene mezclar todo en un solo nodo porque:**
- cambian endpoints
- cambia el tipo de auth
- cambia la semántica (alertas vs agentes)

### 7.1 Nodo HTTP (Indexer 9200) — Body ejemplo (últimas 2h)

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

### 7.2 Nodo HTTP (API 55000) — flujo JWT

#### Nodo A: Authenticate (token)

- **Method:** POST
- **URL:** `https://100.109.242.43:55000/security/user/authenticate?raw=true`
- **Auth:** Basic Auth (`wazuh` + pass)
- **Respuesta:** texto (JWT raw)

#### Nodo B: List agents (Bearer)

- **Method:** GET
- **URL:** `https://100.109.242.43:55000/agents?limit=200&select=id,name,status,lastKeepAlive,version,ip`
- **Header:**

```
Authorization: Bearer {{$json}}
```

> Si el nodo anterior devuelve el token como texto directo. Si el token viene dentro de JSON, ajusta a: `Bearer {{$json.data}}` o el campo exacto que estés recibiendo.

![Flujo JWT en n8n](docs/images/n8n-jwt-flow.png)

---

## 🔐 8) Solución: self-signed certificate en n8n (Docker)

### 8.1 Exportar el cert/CA desde Wazuh

En el host donde corre n8n:

```bash
mkdir -p ~/n8n/certs

echo | openssl s_client -connect 100.109.242.43:55000 -servername 100.109.242.43 -showcerts 2>/dev/null \
| awk '/BEGIN CERTIFICATE/{flag=1} flag{print} /END CERTIFICATE/{exit}' \
> ~/n8n/certs/wazuh-ca.crt

openssl x509 -in ~/n8n/certs/wazuh-ca.crt -noout -subject -issuer -dates
```

![Exportar certificado](docs/images/exportar-cert.png)

### 8.2 Montar el cert en el stack n8n-ngrok

**Copiar el cert al folder del compose:**

```bash
mkdir -p ~/n8n-ngrok/certs
cp ~/n8n/certs/wazuh-ca.crt ~/n8n-ngrok/certs/wazuh-ca.crt
```

**Editar `~/n8n-ngrok/docker-compose.yml`** y en el servicio `n8n` agregar:

**En `volumes`:**

```yaml
      - ./certs/wazuh-ca.crt:/certs/wazuh-ca.crt:ro
```

**En `environment`:**

```yaml
      - NODE_EXTRA_CA_CERTS=/certs/wazuh-ca.crt
```

**Reiniciar:**

```bash
cd ~/n8n-ngrok
sudo docker compose down
sudo docker compose up -d
sudo docker compose logs -f n8n
```

![Certificado montado en n8n](docs/images/n8n-cert-mounted.png)

---

## 💬 9) Preguntas de prueba para tu AI Agent

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

## 🔧 10) Troubleshooting rápido

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

## 📍 11) Endpoints clave

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

## 🤝 Contribuir

¿Mejoras o sugerencias? ¡Pull requests bienvenidos!

1. Fork el proyecto
2. Crea tu rama: `git checkout -b feature/nueva-funcionalidad`
3. Commit: `git commit -m 'Añade nueva funcionalidad'`
4. Push: `git push origin feature/nueva-funcionalidad`
5. Abre un Pull Request

---

## 📄 Licencia

Este proyecto es libre de usar para propósitos educativos y de laboratorio.

---

## 🙏 Agradecimientos

- [Wazuh Documentation](https://documentation.wazuh.com/)
- [OpenSearch Documentation](https://opensearch.org/docs/)
- [n8n Documentation](https://docs.n8n.io/)
- [Tailscale](https://tailscale.com/)

---

<div align="center">

**⭐ SIEM completo con Wazuh + automatización n8n! ⭐**

Hecho con ❤️ para profesionales de ciberseguridad

</div>
