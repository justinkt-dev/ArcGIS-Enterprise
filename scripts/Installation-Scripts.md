# ArcGIS Enterprise 12.0 Installation Scripts

> Bash scripts for silent installation of ArcGIS Enterprise components on Ubuntu Linux. Built and tested in a lab environment, you may review and adjust as per your environment. 

---

## ⚠️ Before You Run Anything

**Read this first.**

These scripts are not officially supported by Esri. They were developed and validated in a specific lab environment for internal testing only. Do not run them in production without review and approval from your infrastructure and GIS teams.

### Environment Assumptions

Update the variables at the top of each script if your environment differs.

| Variable | Default |
|---|---|
| Installer directory | `/data/installers/` |
| Tomcat webapps | `/data/tomcat/webapps/` |
| ArcGIS install base | `/home/arcgis/arcgis/` |
| Web Adaptor install | `/home/arcgis/webadaptor12.0/` |
| ArcGIS version | `12.0` |
| Server admin user | `siteadmin` |
| Portal admin user | `admin` |

### Verify Installer Files

```bash
ls /data/installers/ArcGIS_Server_Linux_120_*.tar.gz
ls /data/installers/Portal_for_ArcGIS_Linux_120_*.tar.gz
ls /data/installers/ArcGIS_DataStore_Linux_120_*.tar.gz
ls /data/installers/ArcGIS_Web_Adaptor_Java_Linux_120_*.tar.gz
ls /data/installers/Server_Ent_Adv_AllExt.prvc
```

### Run in Order

```
01_install_arcgis_server.sh      ← Run first
02_install_arcgis_portal.sh      ← Requires Server running
03_install_arcgis_datastore.sh   ← Requires Server site created
04_install_arcgis_webadaptor.sh  ← Requires Tomcat + Server + Portal running
```

> ⚠️ Script 03 requires an ArcGIS Server site to already exist before running. Create one at `https://HOSTNAME:6443/arcgis/manager` before executing.

### Update Credentials

```bash
# 03_install_arcgis_datastore.sh
SERVER_ADMIN_USER="siteadmin"
SERVER_ADMIN_PASS="siteadmin"

# 04_install_arcgis_webadaptor.sh
PORTAL_ADMIN_USER="admin"
PORTAL_ADMIN_PASS="admin123"
SERVER_ADMIN_USER="siteadmin"
SERVER_ADMIN_PASS="siteadmin"
```

### Known Risks

- `pkill -u arcgis` terminates **all** matching arcgis processes, do not run on live machines as it will lead to services disruption. 
- `set -e` stops the script on any error, check output before running the next script
- TAR extraction overwrites existing files in installer directories
- Script 04 overwrites existing `/portal` and `/server` Tomcat contexts without warning
- Self-signed cert will show "Not Secure" in browsers until replaced with a CA-signed cert

---

## Make Scripts Executable

```bash
chmod +x 0*.sh
```

Or individually:

```bash
chmod +x 01_install_arcgis_server.sh
chmod +x 02_install_arcgis_portal.sh
chmod +x 03_install_arcgis_datastore.sh
chmod +x 04_install_arcgis_webadaptor.sh
```

---

## 1.0 ArcGIS Server

Installs ArcGIS Server, registers and starts the systemd service.

```bash
sudo bash 01_install_arcgis_server.sh
```

**What it does:**

```
1. Extracts ArcGIS_Server_Linux_120_*.tar.gz
2. Runs Setup silently as arcgis user with license file
3. Copies arcgisserver.service to /etc/systemd/system
4. Stops lingering processes, starts service, shows status
```

**Verify:**

```
https://HOSTNAME:6443/arcgis/manager
```

---

## 2.0 Portal for ArcGIS

Installs Portal for ArcGIS, registers and starts the systemd service.

```bash
sudo bash 02_install_arcgis_portal.sh
```

**What it does:**

```
1. Extracts Portal_for_ArcGIS_Linux_120_*.tar.gz
2. Runs Setup silently as arcgis user
3. Copies arcgisportal.service to /etc/systemd/system
4. Stops lingering processes, starts service, shows status
```

**Verify:**

```
https://HOSTNAME:7443/arcgis/home
```

---

## 3.0 ArcGIS DataStore

Installs ArcGIS DataStore, registers the systemd service, and registers Relational and Tile Cache stores with ArcGIS Server.

```bash
sudo bash 03_install_arcgis_datastore.sh
```

**What it does:**

```
1. Extracts ArcGIS_DataStore_Linux_120_*.tar.gz
2. Runs Setup silently as arcgis user
3. Copies arcgisdatastore.service to /etc/systemd/system
4. Stops lingering processes, starts service, shows status
5. Runs configuredatastore.sh → registers relational + tileCache stores
```

**Verify:**

```
https://HOSTNAME:6443/arcgis/manager → Site → Data Stores
```

Both stores should show as `Started`.

---

## 4.0 ArcGIS Web Adaptor

Installs Web Adaptor, deploys `arcgis.war` to Tomcat as `/portal` and `/server`, then configures both adaptors.

```bash
sudo bash 04_install_arcgis_webadaptor.sh
```

**What it does:**

```
1. Extracts ArcGIS_Web_Adaptor_Java_Linux_120_*.tar.gz
2. Runs Setup silently as arcgis user
3. Copies arcgis.war → portal.war and server.war into Tomcat webapps
4. Waits 15s for Tomcat to auto-extract the WARs
5. Runs configurewebadaptor.sh for Portal
6. Runs configurewebadaptor.sh for Server
```

> ⚠️ Tomcat must be fully configured with a valid SSL certificate before running this script. See the [Tomcat Setup Guide](../docs/Tomcat-Setup.md).

**Verify:**

```
https://HOSTNAME/portal/home
https://HOSTNAME/server/manager
```

---

## Final Health Check

```bash
systemctl status arcgisserver | head
systemctl status arcgisportal | head
systemctl status arcgisdatastore | head
systemctl status tomcat | head
```

---

## Support

These scripts are not covered under Esri Technical Support. For installation issues refer to official documentation:

- [ArcGIS Server](https://enterprise.arcgis.com/en/server/latest/install/linux/)
- [Portal for ArcGIS](https://enterprise.arcgis.com/en/portal/latest/install/linux/)
- [ArcGIS DataStore](https://enterprise.arcgis.com/en/data-store/latest/install/linux/)
- [ArcGIS Web Adaptor](https://enterprise.arcgis.com/en/web-adaptor/latest/install/java-linux/)

---

## Related

- [ArcGIS Enterprise Installation Guide](../docs/ArcGIS-Enterprise-Installation.md)
- [Tomcat Setup Guide](../docs/Tomcat-Setup.md)

---

*Validated on: ArcGIS Enterprise 12.0 · Ubuntu 22.04 LTS · Tomcat 10.1.55*
