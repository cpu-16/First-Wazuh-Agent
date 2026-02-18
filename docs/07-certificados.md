# Certificados SSL

## 8) Solución: self-signed certificate en n8n (Docker)

### 8.1 Exportar el cert/CA desde Wazuh

En el host donde corre n8n:

```bash
mkdir -p ~/n8n/certs

echo | openssl s_client -connect 100.109.242.43:55000 -servername 100.109.242.43 -showcerts 2>/dev/null \
| awk '/BEGIN CERTIFICATE/{flag=1} flag{print} /END CERTIFICATE/{exit}' \
> ~/n8n/certs/wazuh-ca.crt

openssl x509 -in ~/n8n/certs/wazuh-ca.crt -noout -subject -issuer -dates
```

![Exportar certificado](images/exportar-cert.png)

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

![Certificado montado en n8n](images/n8n-cert-mounted.png)

---

[← Volver al README principal](../README.md)
