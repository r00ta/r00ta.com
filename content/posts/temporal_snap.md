+++
date = '2025-03-24T00:06:05+01:00'
draft = false 
title = 'Temporal server as a snap'
toc = false
+++

Hey everyone!

I've been using [temporal](https://temporal.io/) is some projects up to know, but since I'm not using `docker` nor `k8s` I always find a bit annoying to run the server and configure it manually. So I decided to wrap it with a snap and provide utilities to setup it. 

You can download the snap from [here](https://snapcraft.io/temporal-server) and [here](https://github.com/r00ta/temporal-server-snap) you can find the repository that is crafting the snap. 

## How to install 

Super simple: 
```bash
sudo snap install --channel=latest/edge temporal-server
```

## Configure

You can use `postgres` or `sqlite` (more utilities will be provided in the future).

For `postgres` you can execute

```bash
sudo temporal-server.init-postgres --host <host> --port <port> --user <user> --password <password>
```

and for `sqlite` 
```
sudo temporal-server.init-sqlite 
```

and finally `sudo snap restart temporal-server` to apply the changes :) You can then customize the temporal config by editing `/var/snap/temporal-server/common/config/production.yaml` as you wish. 

Cheers!
