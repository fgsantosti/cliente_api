# Criando diretorio do projeto
```python
mkdir cliente_api
cd cliente_api
```

# Criando ambiente virtual para os pacotes do projeto
```python
virtualenv venv
. venv/bin/activate  
```

### No Windows use
 `env\Scripts\activate`

# Instalando o Django e e Django REST framework no ambiente virtual
```python
pip install django
pip install djangorestframework
pip install markdown       
pip install django-filter  # Filtering support
```

# Criando o projeto e a aplicação
```python
django-admin startproject core .  

django-admin startapp clientes
```

# Configurando o settings.py

```python
## Application definition

    INSTALLED_APPS = [
        ...
        'rest_framework',
        'clientes',
    ]
```
# Criando os modelos para nosso clientes

No arquivo ```clientes/models.py``` definimos todos os objetos chamados Modelos -- este é um lugar em que vamos definir os relacionamentos entre as classes que estaram presentes na nossa livraria defindos no nosso diagrama e classes.

Vamos abrir ```clientes/models.py``` no editor de código, apagar tudo dele e escrever o seguinte código:

```python
from django.db import models

class Cliente(models.Model):
    nome = models.CharField(max_length=100)
    email = models.EmailField(blank=False, max_length=30, )
    cpf = models.CharField(max_length=11)
    rg = models.CharField(max_length=9)
    celular = models.CharField(max_length=14)
    ativo = models.BooleanField()

    def __str__(self):
        return self.nome

```

```python
python manage.py makemigrations clientes
```

O Django preparou um arquivo de migração que precisamos aplicar ao nosso banco de dados. 


```python
python manage.py migrate 
```
# Django Admin

```python
python manage.py createsuperuser
```

Vamos abrir ```clientes/admin.py``` no editor de código, apagar tudo dele e escrever o seguinte código:

```python
from django.contrib import admin
from clientes.models import Cliente

class Clientes(admin.ModelAdmin):
    list_display = ('id', 'nome', 'email','cpf', 'rg', 'celular', 'ativo')
    list_display_links = ('id', 'nome')
    search_fields = ('nome',)
    list_filter = ('ativo',)
    list_editable = ('ativo',)
    list_per_page = 25
    ordering = ('nome',)

admin.site.register(Cliente, Clientes)

```

# Serializers

```python
from rest_framework import serializers
from clientes.models import Cliente

class ClienteSerializer(serializers.ModelSerializer):
    class Meta:
        model = Cliente
        fields = '__all__'
    
```

# Views 
Vamos abrir ```clientes/views.py``` no editor de código, apagar tudo dele e escrever o seguinte código:

```python
from rest_framework import viewsets
from clientes.serializers import ClienteSerializer
from clientes.models import Cliente

class ClientesViewSet(viewsets.ModelViewSet):
    """Listando clientes"""
    queryset = Cliente.objects.all()
    serializer_class = ClienteSerializer
```

# Routers
Vamos editar ```core/urls.py``` no editor de código:

```python
from django.contrib import admin
from django.urls import path, include
from rest_framework import routers
from clientes.views import ClientesViewSet

router = routers.DefaultRouter()
router.register('clientes', ClientesViewSet)

urlpatterns = [
    path('admin/', admin.site.urls),
    path('', include(router.urls)),
]

```

# Testando

Vamos startar o servidor web 


```python
python manage.py runserver #startando o servidor
```


```python
http://127.0.0.1:8000/
```

E você irá vizualizar a página de API Root da nossa escola API. 

# Valores unicos

No atributo cpf iremos colocar unique=True. 

```python
from django.db import models

class Cliente(models.Model):
    nome = models.CharField(max_length=100)
    email = models.EmailField(blank=False, max_length=30, )
    cpf = models.CharField(max_length=11, unique=True)
    rg = models.CharField(max_length=9)
    celular = models.CharField(max_length=14)
    ativo = models.BooleanField()

    def __str__(self):
        return self.nome

```

# Validando no Serializer
Vamos criar validações para os atributos apartir do serializers.

```python
...
class ClienteSerializer(serializers.ModelSerializer):
    class Meta:
        model = Cliente
        fields = '__all__'
    
    def validar_cpf(self, cpf):
        if(len(cpf) != 11):
            raise serealizers.ValidationError("O cpf deve ter 11 dígitos")
        return cpf
    
    def validar_nome(self, nome):
        if not nome.isalpha():
            raise serealizers.ValidationError("Não inclua números neste campo")
        return nome 
    
    def validar_rg(self, rg):
        if(len(rg) != 9):
            raise serealizers.ValidationError("O rg deve ter 9 dígitos")
        return rg

    def validar_celular(self, celular):
        if(len(celular) < 11):
            raise serealizers.ValidationError("O celular deve ter no mínimo 11 dígitos")
        return celular

```

## Melhorando as validações

Vamos criar um arquivo chamado de ```clientes/validadores.py``` onde deixaremos a forma de verificar a validação de forma mais generalizada. 

```python
def validar_cpf(numero_cpf):
    return len(numero_cpf) != 11
```
No arquivo ```clientes/serializers.py``` fica dessa forma:

```python
...
#fazendo a importação das funções de validação dos atributos
from clientes.validadores import *

class ClienteSerializer(serializers.ModelSerializer):
    class Meta:
        model = Cliente
        fields = '__all__'
    
    def validador(self, data):
        if not validar_cpf(data['cpf']):
            raise serealizers.ValidationError({'cpf':"O cpf deve ter 11 dígitos"})

        return data
```
Dessa forma podemos ir adicionando validadores aos arquivo 
```clientes/validadores.py``` e ir acrescentando condicionais em cascata na função validador no arquivo ```clientes/serializers.py```. 


# Expressões regulares

No arquivo ```clientes/validadores.py``` vamos criar uma função para validarmos os celulares no padrão que conhecemos no Brasil. Para isso iremos criar uma função que utilizará expressões regulares para verificarmos esse formato. 

```python
import re
def validar_celular(numero_celular):
    """Verificando se o formato do celular 89 99410-1010"""
    modelo = '[0-9]{2} [0-9]{5}-[0-9]{4}'
    resposta = re.findall(modelo, numero_celular)
    return resposta
```

No arquivo ```clientes/serializers.py``` fica dessa forma:

```python
...
class ClienteSerializer(serializers.ModelSerializer):
    class Meta:
        model = Cliente
        fields = '__all__'
    
    def validador(self, data):
        if not validar_cpf(data['cpf']):
            raise serealizers.ValidationError({'cpf':"O cpf deve ter 11 dígitos"})
        ...
        if not validar_celular(data['celular']):
            raise serealizers.ValidationError({'celular':"O celular não está no formato aceito pelo sistema. Ex. 89 99410-1010"})
        return data
```

Atualize os códigos e teste nossa aplicação novamente para verificar se as modificações foram realmente implementadas. 


## Melhorando as validações 2 ")

Para aumentarmos o nível das validações da nossa api podemos utilizar o ```validate-docbr```, pacote python para validação de documentos brasileiros, no ```https://pypi.org/project/validate-docbr/```. Para instalar o pacote: 

```python 
pip install validate-docbr
```

No arquivo ```clientes/validadores.py``` iremos utilizar o pacote que acabamos de instalar para nos ajudar. 

```python
#importacao do pacote instalado
from validate_docbr import CPF

def validar_cpf(numero_cpf):
    cpf = CPF() #instancia 
    # Validar CPF
    #cpf.validate("012.345.678-90")  # True
    return cpf.validate(numero_cpf)
```

No arquivo ```clientes/serializers.py``` na funcao validador fica dessa forma:

```python
...
    def validador(self, data):
        if not validar_cpf(data['cpf']):
            raise serealizers.ValidationError({'cpf':"O cpf não é válido."})
        
        ...

        return data
```

# Populando o banco de dados

Para aumentarmos o número de clientes do nosso banco usaremos um script para inserir dados de forma mais rápida no nosso banco de dados.

Antes de usaremos o script vamos instalar  podemos utilizar o ```Faker’s```, o Faker é um pacote Python que gera dados falsos para você, a documentação você pode ver no site ```https://faker.readthedocs.io/en/master/```. 

Para instalar o pacote ao nosso projeto: 

```python 
pip install Faker
```

Vamos criar um arquivo chamado ```cliente_api/populando_script.py```

```python
import os, django

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'core.settings')
django.setup()

from faker import Faker
from validate_docbr import CPF
import random
from clientes.models import Cliente

def criando_pessoas(quantidade_de_pessoas):
    fake = Faker('pt_BR')
    Faker.seed(10)
    for _ in range(quantidade_de_pessoas):
        cpf = CPF()
        nome = fake.name()
        email = '{}@{}'.format(nome.lower(),fake.free_email_domain())
        email = email.replace(' ', '')
        cpf = cpf.generate()
        rg = "{}{}{}{}".format(random.randrange(10, 99),random.randrange(100, 999),random.randrange(100, 999),random.randrange(0, 9) ) 
        celular = "{} 9{}-{}".format(random.randrange(10, 21), random.randrange(4000, 9999), random.randrange(4000, 9999))
        ativo = random.choice([True, False])
        p = Cliente(nome=nome, email=email, cpf=cpf, rg=rg, celular=celular, ativo=ativo)
        p.save()

criando_pessoas(50)
print('executado com sucesso!')
```

Podemos agora executar o nosso script:

```python
python populando_script.py
```

# Paginação 

No nosso arquivo settings.py iremos adicionar uma configuração para a nossa paginação na nossa api. ```Ref. https://www.django-rest-framework.org/api-guide/pagination/```

```python
REST_FRAMEWORK = {
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.LimitOffsetPagination',
    'PAGE_SIZE': 10
}
```
Para colocarmos um estilo de paginação que aceita um único número de página nos parâmetros de consulta da solicitação usamos o ```PageNumberPagination```. Dessa forma ```https://api.example.org/accounts/?page=4```
```python
REST_FRAMEWORK = {
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
    'PAGE_SIZE': 10
}
```

# Configurando o settings.py
## Application definition
Já baixamos o nosso pacote ```pip install django-filter``` agora precisamos instalar ele no nosso projeto.

```python
    INSTALLED_APPS = [
        ...
        'django_filters',
        ...
    ]
```

# Ordenação 

A nossa api ainda não realiza uma ordenação dos clientes inseridos na nosso banco de dados, esses ainda estão apenas exibidos pela ordem de inserção. Vamos usar um pouco do pacote instalado anteriormente.

Vamos modificar um pouco mais o nosso arquivo ```core/settings.py```, vamos incrementar a nossa configuração do atributo ```REST_FRAMEWORK``` uma nova linha, não esqueça de colocar a virgula depois das configurações de paginação como é apresentado no código abaixo.

```python
REST_FRAMEWORK = {
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
    'PAGE_SIZE': 10,

    'DEFAULT_FILTER_BACKENDS': ['django_filters.rest_framework.DjangoFilterBackend'] #novo
}
```

Vamos modificar um pouco mais o nosso arquivo ```clientes/views.py```, que ficará dessa forma:

```python
from rest_framework import viewsets, filters
from clientes.serializers import ClienteSerializer
from clientes.models import Cliente
from django_filters.rest_framework import DjangoFilterBackend #novo

class ClientesViewSet(viewsets.ModelViewSet):
    """Listando clientes"""
    queryset = Cliente.objects.all()
    serializer_class = ClienteSerializer
    filter_backends = [DjangoFilterBackend, filters.OrderingFilter]#novo
    ordering = ['nome'] #ordenar por nome

```
A biblioteca django-filter inclui uma classe ```DjangoFilterBackend``` que suporta filtragem de campo altamente personalizável para framework REST.
A classe ```OrderingFilter``` oferece suporte à ordenação de resultados controlada por parâmetro de consulta simples.

# Buscar e filtrar por atributo

Vamos modificar um pouco mais o nosso arquivo ```clientes/views.py```, que ficará dessa forma:

```python
from rest_framework import viewsets, filters #novo
from clientes.serializers import ClienteSerializer
from clientes.models import Cliente

class ClientesViewSet(viewsets.ModelViewSet):
    """Listando clientes"""
    queryset = Cliente.objects.all()
    serializer_class = ClienteSerializer
    filter_backends = [DjangoFilterBackend, filters.OrderingFilter, filters.SearchFilter] #novo
    ordering_fields = ['nome',] #ordenar por nome
    search_fields = ['nome', 'cpf']#buscar por nome e cpf
    filterset_fields = ['ativo']
```

Você pode definir um atributo ```filterset_fields``` na exibição, ou viewset, listando o conjunto de campos que deseja filtrar. 

# Deploy 

https://www.heroku.com/python

Inicialmente para utilizarmos o heroku precisamos instalar o CLI, que é a interface de linha de comando Heroku (CLI), que permite criar e gerenciar aplicativos Heroku diretamente do terminal. É uma parte essencial do uso do Heroku. Você pode ver a forma de instalar e usar no link: ```https://devcenter.heroku.com/articles/heroku-cli```. 

Instalando no Ubuntu e Debian:
```python
curl https://cli-assets.heroku.com/install-ubuntu.sh | sh
```
Para verificar a versão é so executar o comando ```heroku --version```.

## Comece a usar a CLI do Heroku
Depois de instalar a CLI, execute o comando heroku login. Digite qualquer tecla para acessar seu navegador da Web e concluir o login. A CLI faz login automaticamente.

```python
heroku login 
```

No nosso projeto precisamos instalar uma biblioteca do heroku com o seguinte comando:

```python
pip install django-heroku 
```

Ao executar o comando acima o meu sistema apresentou um pequeno problema com a biblioteca ```psycopg2```, para solucinar o problema de configuração foi instalado os pacotes abaixo.

```python
pip install psycopg2-binary #ao projeto
sudo apt-get install --reinstall libpq-dev #no computador
```

Depois da biblioteca do ```django-heroku``` ser instalada podemos configurar o nosso arquivo ```settings.py.```. Ao arquivo vamos acrescentar a seguinte linha.

```python
import django_heroku #no inicio
django_heroku.settings(locals()) #no fim
```

## Instalando o Gunicorn
Depois disso vamos instalar o gunicorn ao nosso projeto. Gunicorn 'Green Unicorn' é um servidor Python WSGI HTTP para UNIX. É um modelo pré-fork worker. O servidor Gunicorn é amplamente compatível com vários frameworks da web, implementado de forma simples, leve nos recursos do servidor e bastante rápido. ``` https://gunicorn.org/```

```python
pip install gunicorn
pip freeze > requirements.txt #para atualizar no requirements.txt
```

Vamos agora criar um arquivo para configuramos o nosso projeto para que ele possa utilizar os recursos que o Gunicorn nos disponibiliza. Em ``` clientes_api/Procfile```. 

```python
web: gunicorn core.wsgi
```
Desta forma estamos vinculando as responsabilidades de servidor para o gunicorn. 

## Instalar o Git

Para que você possa instalar o git na sua máquina você pode pegar mais informação da sua plataforma no site a seguir: ```https://git-scm.com/```.

No seu projeto iremos acionar os seguinte comandos do git, para que possamos realizar o deploy da nossa aplicação no heroku.

```python
git init 
git add .
git commit -m "primeiro commit do projeto"
heroku login #caso nao tenha realizado o login ainda
heroku git:remote -a fgsclientes
git push heroku master
```


No terminal mostrará o link do site criado pelo heroku, no nosso caso aqui foi ```https://fgsclientes.herokuapp.com/```
```python
#sainda no terminal, mostra que foi feita vinculação entre git local e remoto
set git remote heroku to https://git.heroku.com/fgsclientes.git
```
Ao irmos ao site termos um erro on fala que as nossas tabelas não foram criadas. 

```python 
ProgrammingError at /clientes/
relation "clientes_cliente" does not exist
LINE 1: SELECT COUNT(*) AS "__count" FROM "clientes_cliente"
```
Vamos executar alguns comandos para criamos nossas tabelas resolvermos esse problema.

```python 
heroku run python manage.py migrate
```

```python 
heroku run python manage.py createsuperuser
```

Essas modificações foram realizadas apenas localmente, precisamos atualizar o nosso código também no heroku. Para isso iremos executar os seguintes comandos. 

Faça uma modificação para que os clientes da api precisem realizar uma autenticação antes de visualizar as informações dos clientes. 

```python
git status
git add .
git commit -m "segundo commit atualização na autenticação"
git push heroku master
```

Para populamos nosso banco de dados do nosso sistema online é so executarmos o script ```populando_script.py```

```python
heroku run  python populando_script.py 
```

https://www.elephantsql.com/

https://www.elephantsql.com/docs/pgadmin.html

https://www.pgadmin.org/download/

https://www.pgadmin.org/download/pgadmin-4-apt/
