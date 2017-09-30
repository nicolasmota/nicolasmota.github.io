# Django Channels

Neste artigo vamos falar um pouco sobre Django Channels e criar uma aplicação simples em tempo real de usuários logados.

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
Primeiro criamos um ambiente virtual com pyenv para isolar nossas dependências do projeto:

{% highlight %}
  mkdir exemplo_channels
  cd exemplo_channels
  pyenv virtualenv 3.6 exemplo_channels
  pyenv activate exemplo_channels
{% endhighlight %}

Instale o Django, Django Channels e o ASGI Redis.

{% highlight %}
  pip install django==1.10 & channels==1.0.3 & asgi_redis==1.0.0
  django-admin.py startproject exemplo_channels
  cd exemplo_channels
  python manage.py startapp exemplo
  python manage.py migrate
{% endhighlight %}

Feito isso, baixe e instale o Redis

Mac:

{% highlight %}
  brew install redis
{% endhighlight %}

Linux:

{% highlight %}
  apt-get install redis
{% endhighlight %}

Abra uma janela nova do terminal e deixe rodando o server do redis:

{% highlight %}
  redis-server start
{% endhighlight %}

Abra o settings.py e insira o novo app criado e o django channels em `INSTALLED_APPS`

Insira o CHANNEL_LAYERS na settings.py


