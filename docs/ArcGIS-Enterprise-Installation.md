# ArcGIS Enterprise 12.0 Installation on Ubuntu Linux

> A practical, field-tested guide for installing ArcGIS Enterprise on Ubuntu Linux using Apache Tomcat as the Java web server. Built from real lab deployments, not just documentation.

---

## What This Covers

| Component | Version |
|---|---|
| OS | Ubuntu 20.04 / 22.04 LTS |
| ArcGIS Enterprise | 12.0 |
| Apache Tomcat | 10.1.55 |
| Java | OpenJDK 21 |

---

## Prerequisites

Before starting, confirm the following:

- Root or sudo access on the target machine
- All installer `.tar.gz` files downloaded to `/data/installers/`
- Valid license file (`Server_Ent_Adv_AllExt.prvc`)
- A `.pfx` SSL certificate (self-signed or CA-signed)
- Apache Tomcat installed and running (see [Tomcat Setup Guide](../docs/Tomcat-Setup.md))
- Ports `6443`, `7443`, `443`, `8080` open on the firewall
- `arcgis` user account created on the machine

### Verify installer files

```bash
ls /data/installers/ArcGIS_Server_Linux_120_*.tar.gz
ls /data/installers/Portal_for_ArcGIS_Linux_120_*.tar.gz
ls /data/installers/ArcGIS_DataStore_Linux_120_*.tar.gz
ls /data/installers/ArcGIS_Web_Adaptor_Java_Linux_120_*.tar.gz
ls /data/installers/Server_Ent_Adv_AllExt.prvc
```

### Verify arcgis user exists

```bash
id arcgis
```

---

## Installation Order

> ⚠️ Components must be installed in this exact order. Running them out of sequence will cause failures.

```
1. ArcGIS Server
2. Portal for ArcGIS
3. ArcGIS DataStore      ← requires Server site to exist first
4. ArcGIS Web Adaptor    ← requires Tomcat, Server, and Portal running
```

---

## 1.0 ArcGIS Server

ArcGIS Server is the core GIS engine that hosts map, feature, and geoprocessing services.

### Install

```bash
cd /data/installers
tar xvf ArcGIS_Server_Linux_120_*.tar.gz

su - arcgis -c "
  cd /data/installers/ArcGISServer &&
  ./Setup -m silent -l yes -a /data/installers/Server_Ent_Adv_AllExt.prvc
"
```

### Enable and start service

```bash
cp /home/arcgis/arcgis/server/framework/etc/scripts/arcgisserver.service /etc/systemd/system
systemctl enable arcgisserver
su - arcgis -c "/home/arcgis/arcgis/server/stopserver.sh" || true
pkill -u arcgis -f server || true
sleep 5
systemctl daemon-reload
systemctl restart arcgisserver
systemctl status arcgisserver | head
```

### Verify

Navigate to:

```
https://HOSTNAME:6443/arcgis/manager
```

> 💡 Create your ArcGIS Server site here before proceeding to DataStore installation.

---

## 2.0 Portal for ArcGIS

Portal for ArcGIS is the web-based entry point for your ArcGIS Enterprise deployment — used for sharing maps, apps, and data across your organization.

### Install

```bash
cd /data/installers
tar xvf Portal_for_ArcGIS_Linux_120_*.tar.gz

su - arcgis -c "
  cd /data/installers/PortalForArcGIS &&
  ./Setup -m silent -l yes
"
```

### Enable and start service

```bash
cp /home/arcgis/arcgis/portal/framework/etc/arcgisportal.service /etc/systemd/system
systemctl enable arcgisportal
su - arcgis -c "/home/arcgis/arcgis/portal/stopportal.sh" || true
pkill -u arcgis -f portal || true
sleep 5
systemctl daemon-reload
systemctl restart arcgisportal
systemctl status arcgisportal | head
```

### Verify

Navigate to:

```
https://HOSTNAME:7443/arcgis/home
```

---

## 3.0 ArcGIS DataStore

ArcGIS DataStore provides managed databases for hosted feature layers, scene layers, and spatiotemporal data. Supports Relational and Tile Cache store types.

> ⚠️ An ArcGIS Server site must be created before running `configuredatastore.sh`.

### Install

```bash
cd /data/installers
tar xvf ArcGIS_DataStore_Linux_120_*.tar.gz

su - arcgis -c "
  cd /data/installers/ArcGISDataStore_Linux &&
  ./Setup -m silent -l yes
"
```

### Enable and start service

```bash
cp /home/arcgis/arcgis/datastore/framework/etc/scripts/arcgisdatastore.service /etc/systemd/system
systemctl enable arcgisdatastore
su - arcgis -c "/home/arcgis/arcgis/datastore/stopdatastore.sh" || true
pkill -u arcgis -f datastore || true
sleep 5
systemctl daemon-reload
systemctl restart arcgisdatastore
systemctl status arcgisdatastore | head
```

### Register with ArcGIS Server

```bash
su - arcgis -c "
  cd /home/arcgis/arcgis/datastore/tools &&
  ./configuredatastore.sh \
    https://HOSTNAME:6443/arcgis/admin \
    siteadmin \
    siteadmin \
    /home/arcgis/arcgis/datastore \
    --stores relational,objectstore
"
```

### Verify

Go to ArcGIS Server Manager → **Site** → **Data Stores**, both Relational and Object Store should show as `Started`.

---

## 4.0 ArcGIS Web Adaptor

ArcGIS Web Adaptor routes requests from port 443 through Tomcat to ArcGIS Server and Portal, enabling access via standard HTTPS URLs.

### Install

```bash
cd /data/installers
tar xvf ArcGIS_Web_Adaptor_Java_Linux_120_*.tar.gz

su - arcgis -c "
  cd /data/installers/WebAdaptor &&
  ./Setup -m silent -l yes
"
```

### Deploy to Tomcat

```bash
cp /home/arcgis/webadaptor12.0/java/arcgis.war /data/tomcat/webapps/portal.war
cp /home/arcgis/webadaptor12.0/java/arcgis.war /data/tomcat/webapps/server.war
```

Wait ~15 seconds for Tomcat to auto-extract the WARs, then verify:

```bash
ls /data/tomcat/webapps/
```

You should see `portal/` and `server/` directories.

### Configure for Portal

```bash
su - arcgis -c "
  cd /home/arcgis/webadaptor12.0/java/tools &&
  ./configurewebadaptor.sh \
    -m portal \
    -w https://HOSTNAME/portal/webadaptor \
    -g HOSTNAME \
    -u admin \
    -p YOUR_PORTAL_PASSWORD \
    -a true
"
```

### Configure for Server

```bash
su - arcgis -c "
  cd /home/arcgis/webadaptor12.0/java/tools &&
  ./configurewebadaptor.sh \
    -m server \
    -w https://HOSTNAME/server/webadaptor \
    -g HOSTNAME \
    -u siteadmin \
    -p YOUR_SERVER_PASSWORD \
    -a true
"
```

### Verify

```
https://HOSTNAME/portal/home      ← Portal via Web Adaptor
https://HOSTNAME/server/manager   ← Server Manager via Web Adaptor
```

---

## Final Health Check

### Service status

```bash
systemctl status arcgisserver | head
systemctl status arcgisportal | head
systemctl status arcgisdatastore | head
systemctl status tomcat | head
```

### All access URLs

| Component | Direct URL | Via Web Adaptor |
|---|---|---|
| Portal | `https://HOSTNAME:7443/arcgis/home` | `https://HOSTNAME/portal/home` |
| Server Manager | `https://HOSTNAME:6443/arcgis/manager` | `https://HOSTNAME/server/manager` |
| Tomcat Manager | `http://HOSTNAME:8080/manager/html` | `https://HOSTNAME/manager/html` |

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---|---|---|
| Service `inactive (dead)` after install | Lingering processes from previous version | `pkill -u arcgis -f <component>` then `systemctl restart` |
| `configurewebadaptor.sh` fails | HTTPS not working on Tomcat | Test with `curl -k https://HOSTNAME` — check SSL connector in `server.xml` |
| `Unable to connect to WebAdaptor URL` | Wrong URL format | URL must be HTTPS and include `/webadaptor` path |
| DataStore registration fails | Server site not created | Create site at `https://HOSTNAME:6443/arcgis/manager` first |
| Portal/Server not accessible via Web Adaptor | Web Adaptor not configured | Re-run `configurewebadaptor.sh` for both portal and server |
| Browser shows "Not Secure" | Self-signed certificate | Replace `cert.pfx` with CA-signed cert and restart Tomcat |
| Hostname not resolving from client | DNS not configured | Add to `C:\Windows\System32\drivers\etc\hosts` on Windows |

---

## Related

- [Tomcat Setup Guide](../docs/Tomcat-Setup.md)
- [SSL Certificate Guide](../docs/SSL-Certificates.md)
- [ArcGIS Enterprise Cheat Sheet](../cheatsheets/ArcGIS-Enterprise.md)
- [Installation Scripts](../scripts/)
- [Esri Enterprise Documentation](https://enterprise.arcgis.com/en/documentation/)

---

*Validated on: ArcGIS Enterprise 12.0 · Ubuntu 22.04 LTS · Tomcat 10.1.55 · Java 21*
