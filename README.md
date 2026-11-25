# Template geral para projetos Django

## 1. Estrutura de diretórios

```
django-webapp-template/
├── .env.example              # Modelo de variáveis de ambiente
├── .gitignore                # Ignora .env, db.sqlite3, media/, staticfiles/, __pycache__
├── manage.py
├── requirements.txt
├── core/                     # Configurações centrais do projeto
│   ├── __init__.py
│   ├── asgi.py
│   ├── settings.py
│   ├── urls.py
│   └── wsgi.py
├── apps/                     # Diretório dedicado exclusivamente aos apps
│   ├── __init__.py
│   └── home/                 # App inicial e de autenticação
│       ├── __init__.py
│       ├── admin.py
│       ├── apps.py
│       ├── forms.py
│       ├── models.py
│       ├── urls.py
│       └── views.py
├── templates/                # Templates globais (base.html, partials)
│   ├── base.html
│   └── home/
│       ├── index.html
│       └── login.html
├── static/                   # Arquivos estáticos em desenvolvimento (CSS, JS, imagens do tema)
│   ├── css/
│   ├── js/
│   └── img/
├── staticfiles/              # Gerado automaticamente pelo collectstatic (ignorado no Git)
└── media/                    # Uploads feitos pelos usuários (ignorado no Git)

```

## 2. Etapas da criação do projeto


```bash
# 1. Crie a pasta raiz e o ambiente virtual
mkdir django-webapp-template && cd django-webapp-template
python -m venv .venv
source .venv/bin/activate  # No Windows: .venv\Scripts\activate

# 2. Instale o Django e o python-decouple (boa prática para variáveis de ambiente)
pip install django python-decouple
pip freeze > requirements.txt

# 3. Crie o projeto 'core' na raiz atual
django-admin startproject core .

# 4. Crie a estrutura de apps, templates, static e media
mkdir -p apps/home templates/home static/css static/js static/img media
touch apps/__init__.py

# 5. Crie o app 'home' dentro da pasta 'apps'
python manage.py startapp home apps/home

```

## 3. Configurações Essenciais

### A. Registrar o App em `apps/home/apps.py`

Como o app está dentro da pasta `apps/`, o atributo name precisa refletir esse caminho completo:


```python
from django.apps import AppConfig


class HomeConfig(AppConfig):
    default_auto_field = 'django.db.models.BigAutoField'
    name = 'apps.home'
```

### B. Configurar `core/settings.py`

Ajutar `INSTALLED_APPS`, os diretórios de templates, asquivos estáticos (`static` vs `staticfiles`), mídia (`midea`) e as rotas padrão de autenticação:


```python
from pathlib import Path
from decouple import config

BASE_DIR = Path(__file__).resolve().parent.parent

SECRET_KEY = config('SECRET_KEY', default='django-insecure-chave-temporaria-para-dev')
DEBUG = config('DEBUG', default=True, cast=bool)
ALLOWED_HOSTS = config('ALLOWED_HOSTS', default='127.0.0.1,localhost').split(',')

INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    # Apps do projeto
    'apps.home',
]

# ... (MIDDLEWARE e ROOT_URLCONF permanecem padrão) ...

TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
        'DIRS': [BASE_DIR / 'templates'],  # Templates globais centralizados
        'APP_DIRS': True,
        'OPTIONS': {
            'context_processors': [
                'django.template.context_processors.debug',
                'django.template.context_processors.request',
                'django.contrib.auth.context_processors.auth',
                'django.contrib.messages.context_processors.messages',
            ],
        },
    },
]

# ... (DATABASES e AUTH_PASSWORD_VALIDATORS permanecem padrão) ...

# Internacionalização (Brasil)
LANGUAGE_CODE = 'pt-br'
TIME_ZONE = 'America/Sao_Paulo'
USE_I18N = True
USE_TZ = True

# Arquivos Estáticos (Boas práticas: separar origem e destino de produção)
STATIC_URL = 'static/'
STATICFILES_DIRS = [BASE_DIR / 'static']
STATIC_ROOT = BASE_DIR / 'staticfiles'

# Arquivos de Mídia (Uploads de usuários)
MEDIA_URL = 'media/'
MEDIA_ROOT = BASE_DIR / 'media'

# Gestão de Login e Logout
LOGIN_URL = 'home:login'
LOGIN_REDIRECT_URL = 'home:index'
LOGOUT_REDIRECT_URL = 'home:login'

DEFAULT_AUTO_FIELD = 'django.db.models.BigAutoField'
```

## 4. Rotas e Views do App `home`

### `core/urls.py`

Inclui as URLs do app `home` e habilita o serviço de arquivos de `media/` em ambiente de desenvolvimento (`DEBUG = True`):

```python
from django.contrib import admin
from django.urls import path, include
from django.conf import settings
from django.conf.urls.static import static

urlpatterns = [
    path('admin/', admin.site.urls),
    path('', include('apps.home.urls')),
]

if settings.DEBUG:
    urlpatterns += static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)
```

### `apps/home/urls.py`

Utiliza as Class-Based Views nativas de autenticação do Django (`LoginView` e `LogoutView`), que já tratam CSRF, validação de senha e sessões com segurança:

```python
from django.urls import path
from django.contrib.auth import views as auth_views
from . import views

app_name = 'home'

urlpatterns = [
    path('', views.index, name='index'),
    path(
        'login/',
        auth_views.LoginView.as_view(
            template_name='home/login.html',
            redirect_authenticated_user=True,
        ),
        name='login',
    ),
    path('logout/', auth_views.LogoutView.as_view(), name='logout'),
]
```

### `apps/home/views.py`


```python
from django.shortcuts import render
from django.contrib.auth.decorators import login_required


@login_required
def index(request):
    return render(request, 'home/index.html')
```

## 5. Templates Base (`templates/`)

### `templates/base.html` 


```html
{% load static %}
<!DOCTYPE html>
<html lang="pt-br">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{% block title %}Meu Projeto{% endblock %}</title>
    <link rel="stylesheet" href="{% static 'css/style.css' %}">
</head>
<body>
    <header>
        {% if user.is_authenticated %}
            <span>Olá, {{ user.username }}</span>
            <form action="{% url 'home:logout' %}" method="post" style="display:inline;">
                {% csrf_token %}
                <button type="submit">Sair</button>
            </form>
        {% endif %}
    </header>

    <main>
        {% block content %}{% endblock %}
    </main>

    <script src="{% static 'js/main.js' %}"></script>
</body>
</html>
```

### `templates/home/login.html`

```html
{% extends 'base.html' %}

{% block title %}Login{% endblock %}

{% block content %}
<h1>Acesso ao Sistema</h1>
<form method="post">
    {% csrf_token %}
    {{ form.as_p }}
    <button type="submit">Entrar</button>
</form>
{% endblock %}
```

### `templates/home/index.html`

```html
{% extends 'base.html' %}

{% block title %}Início{% endblock %}

{% block content %}
<h1>Bem-vindo à página inicial</h1>
<p>Você está autenticado com sucesso.</p>
{% endblock %}
```