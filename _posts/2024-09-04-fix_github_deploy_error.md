---
layout: post
title: Fix Github deploy error with exit code 16
date: 2024-09-04 00:05:00
description:  The process '/opt/hostedtoolcache/Ruby/3.2.2/x64/bin/bundle' failed with exit code 16
tags: bundle, platform,
categories: fix bug
---

When you use M-chip Mac local docker compose up for local debugging to build a jekyll website and then push it to Github, the problem "The process '/opt/hostedtoolcache/Ruby/3.2.2/x64/bin/bundle' failed with exit code 16" may occur in Github deployment.

The main reason for the problem is that in your code bundle only supports platforms aarch64-linux-gnu but Github's deployment platform is x86_64-linux, in this case, we have to add x86_64-linux to bundle lock.

### Fix method

When running `docker compose up`, you cannot attach to a exec shell of this container, thus you can use `docker compose up -d` to start a container in a detached mode. But don't worry if you don't want to kill your current shell, you can open a new terminal shell and use `docker ps` to get the container id of your current jekyll container. Then, run `docker exec -it <container_id> /bin/bash` to build a new exec shell in a interactive mode of this container, finally use following shell code to add x86_64-linux to this bundle lock (then you can push your code to Github).

```shell
bundle lock --add-platform x86_64-linux
```