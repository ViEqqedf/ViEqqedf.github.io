---
title: Github将停止使用RSA的SSH秘钥
date: 2022-08-31 00:12:41
tags:
- Github
categories:
- misc
---

如题，Github将停止使用RSA的SSH秘钥

改用Ed25519

ssh-keygen -t ed25519 -C "my@email.com"
