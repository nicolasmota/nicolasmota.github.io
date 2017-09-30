---
layout: post
title: Criando uma aplicação Real Time com Django Channels
author: Nicolas Mota
published: true
---
# Django Channels

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

{% highlight shell %}
  mkdir exemplo_channels
  cd exemplo_channels
  pyenv virtualenv 3.6 exemplo_channels
  pyenv activate exemplo_channels
{% endhighlight %}

Instale o `Django`, `Django Channels` e o `ASGI Redis`, crie um app chamado `exemplo` e execute a `migrate`.

{% highlight shell %}
  pip install django==1.10 & channels==1.0.3 & asgi_redis==1.0.0
  django-admin.py startproject exemplo_channels
  cd exemplo_channels
  python manage.py startapp exemplo
  python manage.py migrate
{% endhighlight %}

Feito isso, baixe e instale o Redis

Mac:

{% highlight shell %}
  brew install redis
{% endhighlight %}

Linux:

{% highlight shell %}
  apt-get install redis
{% endhighlight %}

Abra uma janela nova do terminal e deixe rodando o server do redis:

{% highlight shell %}
  redis-server start
{% endhighlight %}

Abra o arquivo `esettings.py e insira o novo app criado e o django channels em `INSTALLED_APPS`:


{% highlight python %}

INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'channels',
    'exemplo',
]

{% endhighlight %}

Insira o `CHANNEL_LAYERS` na `settings.py`:

{% highlight python %}
CHANNEL_LAYERS = {
    'default': {
        'BACKEND': 'asgi_redis.RedisChannelLayer',
        'CONFIG': {
            'hosts': [('localhost', 6379)],
        },
        'ROUTING': 'exemplo_channels.routing.channel_routing',
    }
}
{% endhighlight %}

## WebSockets

Normalmente, o Django usa HTTP para se comunicar entre o cliente e o servidor:

* O cliente envia uma solicitação HTTP para o servidor.
* O Django analisa o pedido, extrai uma URL e, em seguida, combina com uma `view`.
* A `view` processa a solicitação e retorna uma resposta HTTP ao cliente.

Ao contrário do HTTP, o protocolo WebSockets permite comunicação bidirecional, o que significa que o servidor pode enviar dados para o cliente sem ser solicitado pelo usuário. Com HTTP, apenas o cliente que fez um pedido recebe uma resposta. Com o WebSockets, o servidor pode se comunicar com vários clientes simultaneamente. Como veremos mais adiante neste tutorial, enviamos mensagens do WebSockets usando o prefixo `ws://`, em oposição a `http://`.

Para um melhor entendimento, veja a documentação dos conceitos do Django Channels: https://channels.readthedocs.io/en/stable/concepts.html

## Consumers e Groups

Vamos criar o nosso primeiro [consumer](https://channels.readthedocs.io/en/stable/generics.html), que lida com as conexões básicas entre o cliente e o servidor. Crie um novo arquivo chamado `exemplo_channels/exemplo/consumers.py`:

 
{% highlight python %}
import json
from channels import Group
from channels.auth import channel_session_user, channel_session_user_from_http


@channel_session_user_from_http
def ws_connect(message):
    Group('users').add(message.reply_channel)
    Group('users').send({
        'text': json.dumps({
            'username': message.user.username,
            'is_logged_in': True
        })
    })


@channel_session_user
def ws_disconnect(message):
    Group('users').send({
        'text': json.dumps({
            'username': message.user.username,
            'is_logged_in': False
        })
    })
    Group('users').discard(message.reply_channel)
 
{% endhighlight %}


Os consumers são a contrapartida das Views do Django. Qualquer usuário que se conecte ao nosso app será adicionado ao grupo "usuários" e receberá mensagens enviadas pelo servidor. Quando o cliente se desconecta do nosso app, o canal é removido do grupo e o usuário irá parar de receber mensagens.

Em seguida, vamos configurar as `routes`, que funcionam quase da mesma maneira que a configuração do Django URL, adicionando o seguinte código a um novo arquivo chamado `exemplo_channels/routing.py`:

{% highlight python %}

from channels.routing import route
from exemplo.consumers import ws_connect, ws_disconnect


channel_routing = [
    route('websocket.connect', ws_connect),
    route('websocket.disconnect', ws_disconnect),
]
 
{% endhighlight %}

Perceba que definimos `channel_routing` em vez de `urlpatterns` e `route()` em vez de `url()`. Observe que ligamos nossos consumers ao WebSockets.


## Templates

Vamos escrever `HTML` simples que possa se comunicar com o nosso servidor via WebSocket. Crie uma pasta chamada `templates` dentro do nosso app `exemplo` e, em seguida, adicione uma pasta chamada `exemplo` dentro da pasta `templates` fincando assim: `exemplo_channels/exemplo/templates/exemplo".

Crie um arquivo chamado `base.html` e adicione o seguinte código:

{% highlight jinja %}

<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, user-scalable=no, initial-scale=1.0, maximum-scale=1.0, minimum-scale=1.0">
  <meta http-equiv="X-UA-Compatible" content="ie=edge">
  <link href="//maxcdn.bootstrapcdn.com/bootstrap/3.3.7/css/bootstrap.min.css" rel="stylesheet">
  <title>Exemplo com Django Channels</title>
</head>
<body>
  <div class="container">
    <br>
    {% block content %}{% endblock content %}
  </div>
  <script src="//code.jquery.com/jquery-3.1.1.min.js"></script>
  {% block script %}{% endblock script %}
</body>
</html>
 
{% endhighlight %}

Crie um outro arquivo chamado `user_list.html` na mesma pasta que o `base.html`:

{% highlight jinja %}

{% extends 'exemplo/base.html' %}

{% block content %}{% endblock content %}

{% block script %}
  <script>
    var socket = new WebSocket('ws://' + window.location.host + '/users/');

    socket.onopen = function open() {
      console.log('WebSockets connection created.');
    };

    if (socket.readyState == WebSocket.OPEN) {
      socket.onopen();
    }
  </script>
{% endblock script %}

{% endhighlight %}


## Views

Vamos configurar agora a `views` para que possa renderizar nossos htmls no seguinte diretório: `exemplo_channels/exemplo/views.py`:

{% highlight python %}

from django.shortcuts import render


def user_list(request):
    return render(request, 'exemplo/user_list.html')
 
{% endhighlight %}

Feito isso, crie um arquivo de urls no app do exemplo `exemplo_channels/exemplo/urls.py`:

{% highlight python %}

from django.conf.urls import url
from exemplo.views import user_list


urlpatterns = [
    url(r'^$', user_list, name='user_list'),
]
 
{% endhighlight %}


Altere o arquivo de `url` raiz do projeto para que possa apontar para nosso app de exemplo em: `exemplo_channels/exemplo_channels/urls.py`:

{% highlight python %}

from django.conf.urls import include, url
from django.contrib import admin

urlpatterns = [
    url(r'^admin/', admin.site.urls),
    url(r'^', include('exemplo.urls', namespace='exemplo')),
]

{% endhighlight %}


## Testando a primeira execução

Volte para a pasta raiz do projeto e roda a sua aplicação django:

{% highlight shell %}

(exemplo_channels)$ python manage.py runserver

{% endhighlight %}

Quando você acessar `http://localhost:8000/`, você verá a seguinte mensagem printada no terminal: 

{% highlight shell %}
 
[2017/09/30 13:51:37] HTTP GET / 200 [0.02, 127.0.0.1:52757]
[2017/09/30 13:51:39] WebSocket HANDSHAKING /users/ [127.0.0.1:52789]
[2017/09/30 13:51:43] WebSocket DISCONNECT /users/ [127.0.0.1:52789]

{% endhighlight %}

## Autenticação de usuário

Agora que mostramos que é possível abrir uma conexão, nosso próximo passo é lidar com a autenticação do usuário, lembrando que queremos que um usuário ao se logar, consiga ver uma lista de usuários e seu status atual(offline/online). 

Primeiramente, precisamos criar uma página para que o usuário consiga se cadastrar e se logar no sistema com um usuário e senha, o Django já nos facilitou bastante esse trabalho com uma lib de autenticação, que nos prove um `form` e com duas contribs, sendo uma de `login` e outra de `logout`


Crie um novo arquivo `html` chamado `login.html` no diretório do app `exemplo_channels/exemplo/templates/exemplo`:


{% highlight jinja %}


{% extends 'exemplo/base.html' %}

{% block content %}
  <form action="{% url 'exemplo:login' %}" method="post">
    {% csrf_token %}
    {% for field in form %}
      <div>
        {{ field.label_tag }}
        {{ field }}
      </div>
    {% endfor %}
    <button type="submit">Log in</button>
  </form>
  <p>Don't have an account? <a href="{% url 'exemplo:sign_up' %}">Sign up!</a></p>
{% endblock content %}

{% endhighlight %}


Depois, atualize sua `exemplo_channels/exemplo/views.py` para que contemple as funções de `login` e `logout`, ficando assim:

{% highlight python %}

from django.contrib.auth import login, logout
from django.contrib.auth.forms import AuthenticationForm
from django.core.urlresolvers import reverse
from django.shortcuts import render, redirect


def user_list(request):
    return render(request, 'exemplo/user_list.html')


def login(request):
    form = AuthenticationForm()
    if request.method == 'POST':
        form = AuthenticationForm(data=request.POST)
        if form.is_valid():
            login(request, form.get_user())
            return redirect(reverse('exemplo:user_list'))
        else:
            print(form.errors)
    return render(request, 'exemplo/login.html', {'form': form})


def logout(request):
    logout(request)
    return redirect(reverse('exemplo:login'))


{% endhighlight %}

Como dito anteriormente, o Django vem com forms que oferecem suporte à autenticações simples. Podemos usar o `AuthenticationForm` para gerenciar o login do usuário. Este formulário verifica o nome de usuário e a senha fornecidos e, em seguida, retorna um objeto do tipo `User` se um usuário validado for encontrado. Feito isso, é realizado um redirect para a url de listagem de usuários.

Um usuário também deve ter a capacidade de realizar logout no sistema, a `views` que criamos de logout fornece essa funcionalidade e, em seguida, faz um redirect para a tela de login.


Agora precisamos criar uma forma do usuário poder se cadastrar. Da mesma forma que fizemos para o login, faremos para a criação de usuário, criando um html e uma função na views com a funcionalidade:

Primeiro criamos o `template` de cadastro chamado `sign_up.html` no diretório: `exemplo_channels/exemplo/templates/exemplo`


{% highlight jinja %}


{% extends 'exemplo/base.html' %}

{% block content %}
  <form action="{% url 'exemplo:sign_up' %}" method="post">
    {% csrf_token %}
    {% for field in form %}
      <div>
        {{ field.label_tag }}
        {{ field }}
      </div>
    {% endfor %}
    <button type="submit">Sign up</button>
    <p>Already have an account? <a href="{% url 'exemplo:login' %}">Log in!</a></p>
  </form>
{% endblock content %}


{% endhighlight %}

Agora criamos uma função com a funcionalidade de cadastrar o usuário em nosso app, adicione a seguinte função na views:

{% highlight python %}

def sign_up(request):
    form = UserCreationForm()
    if request.method == 'POST':
        form = UserCreationForm(data=request.POST)
        if form.is_valid():
            form.save()
            return redirect(reverse('exemplo:login'))
        else:
            print(form.errors)
    return render(request, 'exemplo/sign_up.html', {'form': form})
 
{% endhighlight %}

Note que usamos mais um form nativo do Django, chamado `UserCreationForm`, como dito anteriormente, o Django disponibiliza algumas facilidades para que seja feita de maneira rápida coisas simples. Lembre-se de importar o `UserCreationForm`na `views`

{% highlight python %}

from django.contrib.auth.forms import AuthenticationForm, UserCreationForm
 
{% endhighlight %}

Feito isso, precisamos atualizar novamente o arquivo de urls do app para incluir nossas funções da `views`, `exemplo_channels/exemplo/urls.py`:

{% highlight python %}

from django.conf.urls import url
from exemplo.views import login, logout, sign_up, user_list


urlpatterns = [
    url(r'^login/$', login, name='login'),
    url(r'^logout/$', logout, name='logout'),
    url(r'^sign_up/$', sign_up, name='sign_up'),
    url(r'^$', user_list, name='user_list')
]
 
{% endhighlight %}


Até aqui, conseguimos criar um usuário, realizar login no sistema. Acesse http://localhost:8000/sign_up/ no seu navegador e crie alguns usuários para que possamos testar a tela de listagem de usuários logados.

Perceba que após criado o usuário, ele será redirecionado para a tela de login.

## Alertas de Login

Nós temos um sistema de criação e autenticação de usuário funcionando, mas ainda precisamos exibir uma lista de usuários e precisamos que o servidor diga ao grupo quando um usuário efetua login. Precisamos editar nossas funções de consumer para que elas enviem uma mensagem logo após a conexão de um `client` e logo antes de um `client` se desconectar. Os dados da mensagem incluirão o nome de usuário e o status da conexão do usuário (online/offline).

Primeiro, vamos atualizar nosso consumer para que seja feito os envios das mensagens, vamos alterar o arquivo `exemplo_channels/exemplo/consumers.py`, ficando da seguinte maneira:

{% highlight python %}

import json
from channels import Group
from channels.auth import channel_session_user, channel_session_user_from_http


@channel_session_user_from_http
def ws_connect(message):
    Group('users').add(message.reply_channel)
    Group('users').send({
        'text': json.dumps({
            'username': message.user.username,
            'is_logged_in': True
        })
    })


@channel_session_user
def ws_disconnect(message):
    Group('users').send({
        'text': json.dumps({
            'username': message.user.username,
            'is_logged_in': False
        })
    })
    Group('users').discard(message.reply_channel)

{% endhighlight %}

Observe que nós adicionamos `decorators` às funções para obter o usuário da sessão do Django. Além disso, todas as mensagens devem ser serializadas em JSON.

Agora precisamos atualizar nosso template `exemplo_channels/exemplo/templates/exemplo/user_list.html` para adicionar a listagem dos usuários e também para que ele possa realizar o logout do nosso app, ficando da seguinte maneira:

{% highlight jinja %}

{% extends 'exemplo/base.html' %}

{% block content %}
  <a href="{% url 'exemplo:logout' %}">Log out</a>
  <br>
  <ul>
    {% for user in users %}
      <!-- NOTA: Perceba o scape no username, isso evita ataques XSS. -->
      <li data-username="{{ user.username|escape }}">
        {{ user.username|escape }}: {{ user.status|default:'Offline' }}
      </li>
    {% endfor %}
  </ul>
{% endblock content %}

{% block script %}
  <script>
    var socket = new WebSocket('ws://' + window.location.host + '/users/');

    socket.onopen = function open() {
      console.log('WebSockets connection created.');
    };

    socket.onmessage = function message(event) {
      var data = JSON.parse(event.data);
      // NOTA: Perceba o scape no username, isso evita ataques XSS.
      var username = encodeURI(data['username']);
      var user = $('li').filter(function () {
        return $(this).data('username') == username;
      });

      if (data['is_logged_in']) {
        user.html(username + ': Online');
      }
      else {
        user.html(username + ': Offline');
      }
    };

    if (socket.readyState == WebSocket.OPEN) {
      socket.onopen();
    }
  </script>
{% endblock script %}


{% endhighlight %}


Observe que adicionamos um `event listerner` ao nosso WebSocket que pode lidar com mensagens do servidor. Quando recebemos uma mensagem, analisamos os dados JSON, procuramos pelo elemento `<li>` para o usuário fornecido e atualizamos o status desse usuário.

O Django não rastreia se um usuário está logado, então precisamos criar uma `model` simples para fazer isso por nós. Crie uma `model` chamada `LoggedInUser` onde ela faz uma relação de 1 pra 1 com a `model`de `User` nativo do Django. Crie as models em nosso app em `exemplo_channels/exemplo/models.py`:

{% highlight python %}

from django.conf import settings
from django.db import models


class LoggedInUser(models.Model):
    user = models.OneToOneField(
        settings.AUTH_USER_MODEL, related_name='logged_in_user')

{% endhighlight %}

Nosso app criará uma instância de `LoggedInUser` quando um usuário efetuar login e o app excluirá a instância quando o usuário efetuar o logout.

Agora precisamos atualizar nossa base de dados para que nossa model seja realmente criada, pra isso rodamos o comando do django `makemigrations` para criar o `schema` da tabela e o comando `migrate` para que seja criada a tabela em si.

{% highlight shell %}

(exemplo_channels)$ python manage.py makemigrations
(exemplo_channels)$ python manage.py migrate

{% endhighlight %}

Em seguinda, precisamos atualizar nossa `views.py` novamente para que seja renderizado uma lista de usuários para que seja exibido em nosso template. Alteramos o seguinte arquivo `exemplo_channels/exemplo/views.py` ficando da seguinte maneira:

{% highlight python %}

from django.contrib.auth import get_user_model, login, logout
from django.contrib.auth.decorators import login_required
from django.contrib.auth.forms import AuthenticationForm, UserCreationForm
from django.core.urlresolvers import reverse
from django.shortcuts import render, redirect


User = get_user_model()


@login_required(login_url='/login/')
def user_list(request):
    """
    NOTA: Essa não é a melhor maneira de se fazer, Se o volume de usuário for muito grande,
    teremos serios problemas de performance nesse trecho.
    """
    users = User.objects.select_related('logged_in_user')
    for user in users:
        user.status = 'Online' if hasattr(user, 'logged_in_user') else 'Offline'
    return render(request, 'exemplo/user_list.html', {'users': users})


def login(request):
    form = AuthenticationForm()
    if request.method == 'POST':
        form = AuthenticationForm(data=request.POST)
        if form.is_valid():
            login(request, form.get_user())
            return redirect(reverse('exemplo:user_list'))
        else:
            print(form.errors)
    return render(request, 'exemplo/login.html', {'form': form})


@login_required(login_url='/login/')
def logout(request):
    logout(request)
    return redirect(reverse('exemplo:login'))


def sign_up(request):
    form = UserCreationForm()
    if request.method == 'POST':
        form = UserCreationForm(data=request.POST)
        if form.is_valid():
            form.save()
            return redirect(reverse('exemplo:login'))
        else:
            print(form.errors)
    return render(request, 'exemplo/sign_up.html', {'form': form})

 
{% endhighlight %}

Se um usuário tiver um registro de `LoggedInUser` associado ao seu `User` então mostramos o status `Online`, caso contrario, exibimos `Offline`. Perceba também a adição do `decorator` `login_required` para que as funções sejam restritas somente a usuários logados. Caso o usuário esteja deslogado, ele é redirecionado para a url de login informada no parametro.

Não se esqueça de importar as novas funções:

{% highlight python %}


from django.contrib.auth import get_user_model, login, logout
from django.contrib.auth.decorators import login_required

{% endhighlight %}

## Signals

Até aqui, usuários podem se cadastrar, efetuar login e logout e visualizar a lista de usuários e seus status, porém ainda não temos como saber quais usuários iniciaram sessão quando o usuário efetua o primeiro logon. O usuário só ve os status quando é feito mudanças durante seu periodo logado. Aqui é o lugar onde o LoggedInUser entra em ação, mas antes precisamos de uma maneira de criar instancias de ` LoggedInUser` quando um usuário efetuar o login e, em seguida, excluí-lo quando esse usuário efetuar o logout.

Novamente o Django nos da um presentinho. Existe um recurso chamado `signals`, que transmitem notificações quando ocorrem certas ações. Os apps podem "ouvir" essas notificações e depois atuar sobre elas. Podemos explorar dois `signals` úteis nativos no Django (user_logged_in e user_logged_out) para lidar com o nosso comportamento do `LoggedInUser`.

Dentro da pasta `exemplo_channels/exemplo`, criamos um novo arquivo chamado `signals.py`:

{% highlight python %}

from django.contrib.auth import user_logged_in, user_logged_out
from django.dispatch import receiver
from exemplo.models import LoggedInUser


@receiver(user_logged_in)
def on_user_login(sender, **kwargs):
    LoggedInUser.objects.get_or_create(user=kwargs.get('user'))


@receiver(user_logged_out)
def on_user_logout(sender, **kwargs):
    LoggedInUser.objects.filter(user=kwargs.get('user')).delete()

{% endhighlight %}


Precisamos também fazer com que os `signals` fiquem disponiveis para nosso app, pra isso, precisamos alterar o arquivo `apps.py` de nosso app e importar nosso arquivo `signals`:


from django.apps import AppConfig


class exemploConfig(AppConfig):
    name = 'exemplo'

    def ready(self):
        import exemplo.signals


{% endhighlight %}

e por ultimo, atualizar o arquivo `exemplo_channels/exemplo/__init__.py` para que interprete nossa classe:

{% highlight python %}
 
default_app_config = 'exemplo.apps.exemploConfig'

{% endhighlight %}


## Testando

Finalmente finalizamos o código, agora é a parte mais aguardada, estamos prontos para se conectar ao nosso servidor com varios usuários para testar nossa aplicação \o/\o/\o/

Execute o servidor Django rodando o comando `runserver`:


(exemplo_channels)$ python manage.py runserver


Abra em `porn mode`(ou modo anonimo) janelas em seu navegador e acesse o link de login em uma e a listagem em outra, visualize lado a lado uma janela com a listagem dos usuários e realize logins na outra janela e veja os status mudando. Teste o login com multiplos usuários.

Observe também como fica o console com as mensagens do WebSocket.

## Considerações finais

Nesse tutorial, conseguimos mostrar pouco de bastante coisas, mostramos como o Django nos facilita com rotinas comuns em sistemas web, como a criação e autenticação de usuários, vimos também `signals`, Websockets, a maravilha e o quão poderoso é o Django Channels e um front-end bem fuleiro!

O número de aplicações em `real time` que você pode criar com o Django Channels é infinita, conseguimos criar dashboards, sistemas de comunicação, bate papos online, atualizações de status de pedidos, jogos multiplayers e etc., basta colocar a criatividade em prática.  aqui mostramos o básico do básico do Django Channels, acesse a [documentação](https://channels.readthedocs.io/en/stable/) e se divirta.

o código desse tutorial se encontra em: https://github.com/nicolasmota/exemplo_channels


Este é um tutorial traduzido e baseado no divulgado pela [Real Python](https://realpython.com/blog/python/getting-started-with-django-channels/)