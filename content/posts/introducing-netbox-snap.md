+++
date = '2026-04-02T00:06:05+01:00'
draft = false
title = 'Introducing NetBox as a Snap'
toc = false
+++

## Introducing NetBox as a Snap

Hello everyone,

[NetBox](https://netboxlabs.com/products/netbox/) is the leading open-source solution for modeling and documenting modern networks, combining IPAM (IP Address Management) and DCIM (Data Center Infrastructure Management) into a single powerful tool. However, deploying it manually involves setting up Python virtual environments, configuring Gunicorn, managing static files, and more. To simplify this process, I have packaged NetBox as a Snap.

You can download the Snap from [Snapcraft](https://snapcraft.io/community-netbox), and the source code is available on [GitHub](https://github.com/r00ta/netbox-snap).

## Prerequisites

The Snap requires PostgreSQL and Redis to be available. These can be installed as system packages, run via Docker, or provided by managed services.

```bash
sudo apt install postgresql redis-server
sudo -u postgres createuser --superuser netbox
sudo -u postgres createdb -O netbox netbox
```

## Installation

Installing the Snap is straightforward:

```bash
sudo snap install community-netbox
```

## Configuration

Configure the database and Redis connections using `snap set`:

```bash
sudo snap set community-netbox \
  db.host=localhost db.port=5432 \
  db.name=netbox db.user=netbox db.password=""

sudo snap set community-netbox \
  redis.host=localhost redis.port=6379
```

After configuring, restart the services to apply the changes:

```bash
sudo snap restart community-netbox
```

Then create your admin user:

```bash
sudo community-netbox.manage createsuperuser
```

The NetBox UI will be available at `http://localhost:8080`. You can change the port with:

```bash
sudo snap set community-netbox http.port=9090
```

## What's Included

The Snap bundles NetBox with Gunicorn and WhiteNoise for serving static files, so no external web server (Nginx/Apache) is needed. It runs two services:

- **`community-netbox.netbox-web`** – the Gunicorn web server
- **`community-netbox.netbox-rqworker`** – a background task worker for Redis Queue

Database migrations, static file collection, and search indexing are handled automatically on startup.

## Management Commands

All Django management commands are available through `community-netbox.manage`:

```bash
sudo community-netbox.manage migrate
sudo community-netbox.manage nbshell
sudo community-netbox.manage reindex --lazy
```

## Feedback and Contributions

If you have any questions, suggestions, or contributions, feel free to reach out. Enjoy using NetBox with Snap!

---

For updates and further details, visit the [Snapcraft page](https://snapcraft.io/community-netbox) or check out the [GitHub repository](https://github.com/r00ta/netbox-snap).
