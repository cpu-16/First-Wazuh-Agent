# Conectar Agentes al Manager

## 3) Conectar agentes al Manager

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

![Agente conectado](images/agente-conectado.png)

---

[← Volver al README principal](../README.md)
