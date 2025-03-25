+++
date = '2025-03-24T00:06:05+01:00'  
draft = false  
title = 'Introducing Temporal Server and UI as a Snap'  
toc = false  
+++

## Introducing Temporal Server and UI as a Snap

Hello everyone,

Temporal is a powerful tool for building reliable and scalable applications. However, setting up and configuring the Temporal Server manually can be cumbersome, especially for those who prefer not to use Docker or Kubernetes. To simplify this process, I have packaged Temporal Server as a Snap, providing built-in utilities for easy installation and configuration.

You can download the Snap from [Snapcraft](https://snapcraft.io/temporal-server), and the source code is available on [GitHub](https://github.com/r00ta/temporal-server-snap).

## Installation

Installing the Snap is straightforward:

```bash
sudo snap install --channel=latest/edge temporal-server
```

## Configuration

Currently, the Snap has utilities to use PostgreSQL and SQLite as database backends, with potential for additional options in future updates.

### PostgreSQL Configuration

To configure Temporal Server with PostgreSQL, run the following command, replacing the placeholders with your database details:

```bash
sudo temporal-server.init-postgres --host <host> --port <port> --user <user> --password <password>
```

### SQLite Configuration

For SQLite, simply execute:

```bash
sudo temporal-server.init-sqlite
```

After configuring the database, restart the service to apply the changes:

```bash
sudo snap restart temporal-server
```

You can further customize Temporal's configuration by modifying the following file:

```
/var/snap/temporal-server/common/config/production.yaml
```

## Temporal UI

The Temporal UI is also available as a Snap. To install it, run:

```bash
sudo snap install --channel=latest/edge temporal-ui
```

Configuration can be customized in:

```
/var/snap/temporal-ui/common/production.yaml
```

By default, the UI listens at `http://localhost:8233` and connects to the Temporal Server at `127.0.0.1:7233`.

## Feedback and Contributions

If you have any questions, suggestions, or contributions, feel free to reach out. Enjoy using Temporal with Snap!

---

For updates and further details, visit the [Snapcraft page](https://snapcraft.io/temporal-server) or check out the [GitHub repository](https://github.com/r00ta/temporal-server-snap).
