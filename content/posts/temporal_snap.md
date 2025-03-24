+++
date = '2025-03-24T00:06:05+01:00'  
draft = false  
title = 'Introducing Temporal Server as a Snap  '  
toc = false  
+++  

Hello everyone,  

I've been using [Temporal](https://temporal.io/) in several projects, but since I don't use `docker` or `k8s` for such projects, setting up and configuring the server manually has always been a bit cumbersome. To simplify this process, I’ve packaged Temporal Server as a Snap, providing built-in utilities for easy setup and configuration.  

You can download the Snap from [Snapcraft](https://snapcraft.io/temporal-server), and the source repository is available on [GitHub](https://github.com/r00ta/temporal-server-snap).  

## Installation  

Installing the Snap is straightforward:  

```bash
sudo snap install --channel=latest/edge temporal-server
```  

## Configuration  

Currently, you can use either `PostgreSQL` or `SQLite` as the database backend (additional options may be added in the future).  

### PostgreSQL Configuration  

Run the following command, replacing the placeholders with your database details:  

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

You can further customize Temporal's configuration by modifying the file:  

```
/var/snap/temporal-server/common/config/production.yaml
```  

Let me know if you have any questions or suggestions. Enjoy!  
