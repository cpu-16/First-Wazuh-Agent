# Instalación de Wazuh

## 1) Instalación de Wazuh (All-in-one)

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

![Instalación de Wazuh](images/instalacion-wazuh.png)

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

![Credenciales del instalador](images/credenciales.png)

---

## 2) Servicios y puertos

### 2.1 Verificar servicios

```bash
sudo systemctl status wazuh-manager
sudo systemctl status wazuh-indexer
sudo systemctl status wazuh-dashboard
sudo systemctl status filebeat
```

![Estado de servicios Wazuh](images/servicios-status.png)

### 2.2 Verificar puertos

```bash
sudo ss -tlnp | egrep '(:55000|:9200|:5601|:1514|:1515)'
```

![Puertos escuchando](images/puertos.png)

---

[← Volver al README principal](../README.md)
