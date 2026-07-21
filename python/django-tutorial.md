# Guia de Configuração e Desenvolvimento Django

Este guia prático serve como referência rápida para a criação de projetos Django, configuração de aplicações, arquitetura de templates reutilizáveis (Layout Base) e mapeamento de conceitos para quem vem do Ruby on Rails.

## Sumário
- [1. Configuração do Projeto](#1-configuração-do-projeto)
- [2. Criando um Novo App](#2-criando-um-novo-app)
- [3. Criando Models](#3-criando-models)
- [4. Criar Usuário Admin](#4-criar-usuário-admin)
- [5. Ordem de Desenvolvimento Sugerida](#5-ordem-de-desenvolvimento-sugerida)
- [6. Criando um Layout Base Global](#6-criando-um-layout-base-global)
- [7. Observações e Sintaxe de Templates](#7-observações-e-sintaxe-de-templates)
- [8. Tabela Comparativa: Rails (ERB) vs Django Templates](#8-tabela-comparativa-rails-erb-vs-django-templates)

---

## 1. Configuração do Projeto

### Passo 1: Entrar na pasta do projeto
```bash
cd arsenal
```

### Passo 2: Criar o ambiente virtual
O ambiente virtual isola as dependências do projeto, evitando conflitos com outros projetos Python instalados na máquina. Caso ele já exista, pule para o próximo passo.
```bash
python3 -m venv .venv
```

### Passo 3: Ativar o ambiente virtual
```bash
source .venv/bin/activate
```
ou liberando o acesso aos pacotes globais
```bash
python3 -m venv --system-site-packages .venv
```

*Após a ativação, o terminal deverá exibir algo semelhante a:*
```bash
(.venv) usuario@maquina:~/arsenal$
```

### Passo 4: Instalar o Django
```bash
pip install django
```

### Passo 5: Criar o projeto Django
```bash
django-admin startproject config .
```
> **Nota:** `config` é a pasta que armazenará as configurações globais do projeto. O ponto `.` indica que o projeto será criado na pasta atual, sem gerar uma subpasta extra redundante.

**Estrutura inicial:**
```text
arsenal/
├── .venv/
├── config/
└── manage.py
```

### Passo 6: Executar as migrações
Esse comando cria as tabelas internas utilizadas pelo ecossistema padrão do Django.
```bash
python manage.py migrate
```
*Será criado automaticamente o arquivo de banco de dados:* `db.sqlite3`

### Passo 7: Iniciar o servidor
```bash
python manage.py runserver
```
Acesse no navegador: `[http://127.0.0.1:8000/](http://127.0.0.1:8000/)`

---

## 2. Criando um Novo App

### Criar o app
```bash
python manage.py startapp home
```

**Estrutura criada:**
```text
home/
├── migrations/
├── admin.py
├── apps.py
├── models.py
├── tests.py
└── views.py
```

### Registrar o app no projeto
No arquivo `config/settings.py`, adicione o nome do app em `INSTALLED_APPS`:
```python
INSTALLED_APPS = [
    # ... apps padrões do django
    'home',
]
```

### Vincular as URLs do app ao projeto global
No arquivo `config/urls.py`:
```python
from django.urls import include, path

urlpatterns = [
    path("", include("home.urls")),
]
```

### Criar o arquivo de mapeamento de URLs do App
O Django não cria esse arquivo por padrão. Crie manualmente o arquivo `home/urls.py`:
```python
from django.urls import path
from . import views

urlpatterns = [
    path("", views.index, name="home"),
]
```

### Criar a View
No arquivo `home/views.py`:
```python
from django.shortcuts import render

def index(request):
    return render(request, "home/index.html")
```

### Criar a estrutura de Templates do App
Crie a estrutura de pastas e o arquivo HTML de acordo com a convenção:
```text
home/
└── templates/
    └── home/
        └── index.html
```

### Criar os arquivos estáticos do App (Opcional)
```text
home/
└── static/
    └── home/
        ├── css/
        ├── js/
        └── img/
```

---

## 3. Criando Models

### Criar as classes/modelos
No arquivo `home/models.py`:
```python
from django.db import models

class Processo(models.Model):
    numero = models.CharField(max_length=20)
```

### Criar os arquivos de migração
Analisa as alterações nos models e gera os arquivos de histórico na pasta `migrations/`:
```bash
python manage.py makemigrations
```

### Aplicar as migrações no banco
Efetivamente cria ou altera as tabelas no banco de dados:
```bash
python manage.py migrate
```

---

## 4. Criar Usuário Admin

Gere as credenciais do superusuário administrador:
```bash
python manage.py createsuperuser
```
Depois acesse o painel de controle em: `[http://127.0.0.1:8000/admin](http://127.0.0.1:8000/admin)`

---

## 5. Ordem de Desenvolvimento Sugerida

Para manter um fluxo consistente no desenvolvimento de novos recursos, siga estes passos:
1. Criar o app
2. Registrar no `settings.py`
3. Criar `urls.py` do app e registrar no `urls.py` global
4. Criar as views
5. Criar templates HTML
6. Criar arquivos static (CSS, JS, Imagens)
7. Criar os models
8. Executar `makemigrations`
9. Executar `migrate`
10. Testar no navegador
11. Repetir o processo para os próximos apps

---

## 6. Criando um Layout Base Global

Quando várias páginas compartilham a mesma estrutura (cabeçalho, sidebar, rodapé, CSS e JavaScript), criamos um template base global.

### Passo 1: Criar uma pasta global de templates na raiz do projeto
Crie a pasta `templates` no mesmo nível de `manage.py`:
```text
arsenal/
├── config/
├── home/
├── templates/
│   ├── base.html
│   └── partials/
│       ├── header.html
│       ├── sidebar.html
│       └── footer.html
└── manage.py
```

### Passo 2: Configurar o Django para encontrar os templates globais
No arquivo `config/settings.py`, localize a chave `TEMPLATES` e altere a opção `DIRS`:
```python
'DIRS': [BASE_DIR / 'templates'],
```

### Passo 3: Configurar os arquivos estáticos globais
Ainda no arquivo `config/settings.py`, adicione a seguinte configuração logo abaixo de `STATIC_URL = 'static/'`:
```python
STATICFILES_DIRS = [
    BASE_DIR / 'static-assets',
]
```
> Isso informa ao Django que arquivos globais de CSS, JS e imagens do design do sistema ficam armazenados na pasta raiz `arsenal/static-assets/`.

### Passo 4: Criar o layout principal
No arquivo `templates/base.html`:
```html
{% load static %}

<!DOCTYPE html>
<html lang="pt-BR">

<head>
    <meta charset="UTF-8">
    <link rel="stylesheet" href="{% static 'stylesheets/style.css' %}">
</head>

<body>

    {% include "partials/header.html" %}
    {% include "partials/sidebar.html" %}

    <main>
        {% block content %}
        {% endblock %}
    </main>

    {% include "partials/footer.html" %}

    <script src="{% static 'js/bootstrap.min.js' %}"></script>

</body>

</html>
```

### Passo 5: Herdar o layout nas páginas filhas
No arquivo do seu app (exemplo: `home/templates/home/index.html`), basta estender o arquivo base:
```html
{% extends "base.html" %}

{% block content %}

<h1>Home</h1>
<p>Este conteúdo específico entra direto na tag main do base.html.</p>

{% endblock %}
```

---

## 7. Observações e Sintaxe de Templates

- **`{% load static %}`**: Carrega a tag utilitária do Django responsável por resolver caminhos de arquivos estáticos.
- **`{% static '...' %}`**: Gera dinamicamente a URL absoluta correta para folhas de estilo, scripts e mídias.
- **`{% include "caminho" %}`**: Renderiza e insere o pedaço de código de outro template (partials) exatamente onde a tag foi declarada.
- **`{% block nome %}` e `{% endblock %}`**: Cria marcadores substituíveis que serão preenchidos pelos templates filhos que usam a herança (`{% extends %}`).

---

## 8. Tabela Comparativa: Rails (ERB) vs Django Templates

Se você tem familiaridade com o padrão arquitetural do Ruby on Rails, use o mapeamento abaixo para associar as tecnologias:

| Rails (ERB) | Django Templates | Descrição / Função |
| :--- | :--- | :--- |
| `<% ... %>` | `{% ... %}` | Tags lógicas (condicionais, loops, blocos) |
| `<%= ... %>` | `{{ ... }}` | Exibição / Print de variáveis na tela |
| `application.html.erb` | `base.html` | Arquivo de layout estrutural master da aplicação |
| `<%= yield %>` | `{% block content %}{% endblock %}` | Espaço reservado para injetar o conteúdo de páginas específicas |
| `render partial:` | `{% include %}` | Reaproveitamento de pedaços de interface menores (como navbar) |
