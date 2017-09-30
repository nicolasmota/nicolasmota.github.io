---
layout: post
title: Hello World
author: Nicolas Mota
---

# Django Channels

---

Neste tutorial vamos falar um pouco sobre [Django Channels](https://channels.readthedocs.io/en/stable/) e criar uma aplicação simples em tempo real de usuários logados.

## Requisitos

- Python 3
- Django v1.10
- Django Channels v1.0.3
- Redis v3.2.8

## Objetivo

No final deste artigo você será capaz de:

- Adicionar web socket em um projeto Django.
- Configurar conexão entre o Django e o redis.
- Implementar um sistema simples de autenticação, com cadastro, login e logout.
- Usar o Django signals para realizar ações quando um usuário se logar e deslogar.

## Iniciando

Primeiro criamos um ambiente virtual com `pyenv` para isolar nossas dependências do projeto:
