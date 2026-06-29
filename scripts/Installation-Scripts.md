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

Installation Script

```bash
#!/bin/bash
# ============================================================
#  ArcGIS Enterprise 12.0 — ArcGIS Server Installation
#  Run as: sudo bash 01_install_arcgis_server.sh
# ============================================================
set -e # exits immediately if any command fails,

echo ""
echo "============================================================"
echo "  ArcGIS Server 12.0 Installation"
echo "============================================================"

# ── Variables ─────────────────────────────────────────────────
INSTALLER_DIR="/data/installers"
INSTALL_DIR="/home/arcgis/arcgis/server"
LICENSE_FILE="$INSTALLER_DIR/Server_Ent_Adv_AllExt.prvc"
SERVICE_NAME="arcgisserver"

# ── Step 1: Extract Installer ──────────────────────────────────
echo ""
echo "[1/4] Extracting ArcGIS Server installer..."
cd "$INSTALLER_DIR"
tar xvf ArcGIS_Server_Linux_120_*.tar.gz
echo "✔ Extraction complete."

# ── Step 2: Run Silent Installation ───────────────────────────
echo ""
echo "[2/4] Installing ArcGIS Server as arcgis user..."
su - arcgis -c "
  cd $INSTALLER_DIR/ArcGISServer &&
  ./Setup -m silent -l yes -a $LICENSE_FILE
"
echo "✔ Installation complete."
echo "  Access Server Manager at: https://$(hostname -f):6443/arcgis/manager"

# ── Step 3: Register systemd Service ──────────────────────────
echo ""
echo "[3/4] Registering arcgisserver systemd service..."
cp $INSTALL_DIR/framework/etc/scripts/arcgisserver.service /etc/systemd/system/
systemctl enable $SERVICE_NAME
echo "✔ Service registered and enabled."

# ── Step 4: Start Service ──────────────────────────────────────
echo ""
echo "[4/4] Starting ArcGIS Server..."
su - arcgis -c "$INSTALL_DIR/stopserver.sh" || true
pkill -u arcgis -f server || true
sleep 5
systemctl daemon-reload
systemctl restart $SERVICE_NAME
systemctl status $SERVICE_NAME | head

echo ""
echo "============================================================"
echo "  ArcGIS Server Installation Complete!"
echo "  URL: https://$(hostname -f):6443/arcgis/manager"
echo "============================================================"
```

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

Installation Script

```bash
#!/bin/bash
# ============================================================
#  ArcGIS Enterprise 12.0 — Portal for ArcGIS Installation
#  Run as: sudo bash 02_install_arcgis_portal.sh
# ============================================================
set -e

echo ""
echo "============================================================"
echo "  Portal for ArcGIS 12.0 Installation"
echo "============================================================"

# ── Variables ─────────────────────────────────────────────────
INSTALLER_DIR="/data/installers"
INSTALL_DIR="/home/arcgis/arcgis/portal"
SERVICE_NAME="arcgisportal"

# ── Step 1: Extract Installer ──────────────────────────────────
echo ""
echo "[1/4] Extracting Portal for ArcGIS installer..."
cd "$INSTALLER_DIR"
tar xvf Portal_for_ArcGIS_Linux_120_*.tar.gz
echo "✔ Extraction complete."

# ── Step 2: Run Silent Installation ───────────────────────────
echo ""
echo "[2/4] Installing Portal for ArcGIS as arcgis user..."
su - arcgis -c "
  cd $INSTALLER_DIR/PortalForArcGIS &&
  ./Setup -m silent -l yes
"
echo "✔ Installation complete."
echo "  Access Portal at: https://$(hostname -f):7443/arcgis/home"

# ── Step 3: Register systemd Service ──────────────────────────
echo ""
echo "[3/4] Registering arcgisportal systemd service..."
cp $INSTALL_DIR/framework/etc/arcgisportal.service /etc/systemd/system/
systemctl enable $SERVICE_NAME
echo "✔ Service registered and enabled."

# ── Step 4: Start Service ──────────────────────────────────────
echo ""
echo "[4/4] Starting Portal for ArcGIS..."
su - arcgis -c "$INSTALL_DIR/stopportal.sh" || true
pkill -u arcgis -f portal || true
sleep 5
systemctl daemon-reload
systemctl restart $SERVICE_NAME
systemctl status $SERVICE_NAME | head

echo ""
echo "============================================================"
echo "  Portal for ArcGIS Installation Complete!"
echo "  URL: https://$(hostname -f):7443/arcgis/home"
echo "============================================================"
```

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

Installation Script

```bash
#!/bin/bash
# ============================================================
#  ArcGIS Enterprise 12.0 — ArcGIS DataStore Installation
#  Run as: sudo bash 03_install_arcgis_datastore.sh
# ============================================================
set -e

echo ""
echo "============================================================"
echo "  ArcGIS DataStore 12.0 Installation"
echo "============================================================"

# ── Variables ─────────────────────────────────────────────────
INSTALLER_DIR="/data/installers"
INSTALL_DIR="/home/arcgis/arcgis/datastore"
SERVICE_NAME="arcgisdatastore"
HOSTNAME=$(hostname -f)

# ── Configuredatastore Parameters ─────────────────────────────
SERVER_ADMIN_URL="https://$HOSTNAME:6443/arcgis/admin"
SERVER_ADMIN_USER="siteadmin"
SERVER_ADMIN_PASS="siteadmin"
DATASTORE_DIR="$INSTALL_DIR"
DATASTORE_TYPES="relational,tileCache"

# ── Step 1: Extract Installer ──────────────────────────────────
echo ""
echo "[1/5] Extracting ArcGIS DataStore installer..."
cd "$INSTALLER_DIR"
tar xvf ArcGIS_DataStore_Linux_120_*.tar.gz
echo "✔ Extraction complete."

# ── Step 2: Run Silent Installation ───────────────────────────
echo ""
echo "[2/5] Installing ArcGIS DataStore as arcgis user..."
su - arcgis -c "
  cd $INSTALLER_DIR/ArcGISDataStore_Linux &&
  ./Setup -m silent -l yes
"
echo "✔ Installation complete."

# ── Step 3: Register systemd Service ──────────────────────────
echo ""
echo "[3/5] Registering arcgisdatastore systemd service..."
cp $INSTALL_DIR/framework/etc/scripts/arcgisdatastore.service /etc/systemd/system/
systemctl enable $SERVICE_NAME
echo "✔ Service registered and enabled."

# ── Step 4: Start Service ──────────────────────────────────────
echo ""
echo "[4/5] Starting ArcGIS DataStore..."
su - arcgis -c "$INSTALL_DIR/stopdatastore.sh" || true
pkill -u arcgis -f datastore || true
sleep 5
systemctl daemon-reload
systemctl restart $SERVICE_NAME
systemctl status $SERVICE_NAME | head

# ── Step 5: Register DataStore with ArcGIS Server ─────────────
echo ""
echo "[5/5] Registering Relational and Tile Cache datastores with ArcGIS Server..."
echo "  Server Admin URL : $SERVER_ADMIN_URL"
echo "  DataStore Path   : $DATASTORE_DIR"
echo "  Store Types      : $DATASTORE_TYPES"
echo ""

su - arcgis -c "
  cd $INSTALL_DIR/tools &&
  ./configuredatastore.sh $SERVER_ADMIN_URL $SERVER_ADMIN_USER $SERVER_ADMIN_PASS $DATASTORE_DIR --stores $DATASTORE_TYPES
"

echo ""
echo "============================================================"
echo "  ArcGIS DataStore Installation Complete!"
echo "  Verify at: https://$HOSTNAME:6443/arcgis/manager"
echo "  Go to Site > Data Stores to confirm registration."
echo "============================================================"
```

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

Installation Script

```bash
#!/bin/bash
# ============================================================
#  ArcGIS Enterprise 12.0 — ArcGIS Web Adaptor Installation
#  Run as: sudo bash 04_install_arcgis_webadaptor.sh
# ============================================================
set -e

echo ""
echo "============================================================"
echo "  ArcGIS Web Adaptor 12.0 Installation"
echo "============================================================"

# ── Variables ─────────────────────────────────────────────────
INSTALLER_DIR="/data/installers"
WEBADAPTOR_DIR="/home/arcgis/webadaptor12.0"
TOMCAT_WEBAPPS="/data/tomcat/webapps"
HOSTNAME=$(hostname -f)

# ── Web Adaptor Configuration Parameters ──────────────────────
PORTAL_ADMIN_USER="admin"
PORTAL_ADMIN_PASS="admin123"
SERVER_ADMIN_USER="siteadmin"
SERVER_ADMIN_PASS="siteadmin"

# ── Step 1: Extract Installer ──────────────────────────────────
echo ""
echo "[1/5] Extracting ArcGIS Web Adaptor installer..."
cd "$INSTALLER_DIR"
tar xvf ArcGIS_Web_Adaptor_Java_Linux_120_*.tar.gz
echo "✔ Extraction complete."

# ── Step 2: Run Silent Installation ───────────────────────────
echo ""
echo "[2/5] Installing ArcGIS Web Adaptor as arcgis user..."
su - arcgis -c "
  cd $INSTALLER_DIR/WebAdaptor &&
  ./Setup -m silent -l yes
"
echo "✔ Installation complete."
echo "  Web Adaptor installed at: $WEBADAPTOR_DIR"

# ── Step 3: Deploy arcgis.war to Tomcat ───────────────────────
echo ""
echo "[3/5] Deploying arcgis.war to Tomcat as /portal and /server..."
cp $WEBADAPTOR_DIR/java/arcgis.war $TOMCAT_WEBAPPS/portal.war
cp $WEBADAPTOR_DIR/java/arcgis.war $TOMCAT_WEBAPPS/server.war
echo "✔ WAR files deployed."
echo "  Waiting 15 seconds for Tomcat to extract the WARs..."
sleep 15

# Verify deployment
echo ""
echo "  Deployed webapps:"
ls $TOMCAT_WEBAPPS/

# ── Step 4: Configure Web Adaptor for Portal ──────────────────
echo ""
echo "[4/5] Configuring Web Adaptor for Portal for ArcGIS..."
echo "  Portal Web Adaptor URL : https://$HOSTNAME/portal/webadaptor"
echo "  Portal FQDN            : $HOSTNAME"
echo "  Admin User             : $PORTAL_ADMIN_USER"
echo ""

su - arcgis -c "
  cd $WEBADAPTOR_DIR/java/tools &&
  ./configurewebadaptor.sh \
    -m portal \
    -w https://$HOSTNAME/portal/webadaptor \
    -g $HOSTNAME \
    -u $PORTAL_ADMIN_USER \
    -p $PORTAL_ADMIN_PASS \
    -a true
"
echo "✔ Portal Web Adaptor configured."

# ── Step 5: Configure Web Adaptor for Server ──────────────────
echo ""
echo "[5/5] Configuring Web Adaptor for ArcGIS Server..."
echo "  Server Web Adaptor URL : https://$HOSTNAME/server/webadaptor"
echo "  Server FQDN            : $HOSTNAME"
echo "  Admin User             : $SERVER_ADMIN_USER"
echo ""

su - arcgis -c "
  cd $WEBADAPTOR_DIR/java/tools &&
  ./configurewebadaptor.sh \
    -m server \
    -w https://$HOSTNAME/server/webadaptor \
    -g $HOSTNAME \
    -u $SERVER_ADMIN_USER \
    -p $SERVER_ADMIN_PASS \
    -a true
"
echo "✔ Server Web Adaptor configured."

echo ""
echo "============================================================"
echo "  ArcGIS Web Adaptor Installation Complete!"
echo ""
echo "  Access URLs:"
echo "  Portal Home    : https://$HOSTNAME/portal/home"
echo "  Server Manager : https://$HOSTNAME/server/manager"
echo "============================================================"
```

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
