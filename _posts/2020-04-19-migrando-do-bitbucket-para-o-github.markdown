---
title: "Migrando do Bitbucket para o Github"
layout: post
date:   2020-04-19 10:00:00
---

Olá,

semana passada o [Github disponibilizou de graça](https://github.blog/2020-04-14-github-is-now-free-for-teams/) acesso a repositórios privados com quantidade ilimitada de colaboradores. Além disso, para os usuários que precisarem de recursos mais avançados, o valor por usuário caiu de U$ 9.00 para U$ 4.00 por mês.

Na empresa, somos usuários do Bitbucket há pelo menos 5 anos, principalmente pela facilidade da integração com outros softwares da Atlassian que usamos. Mas algumas coisas incomodavam nele, como não ter suporte a commits assinados, ou não ter syntax highlight no diff dos commits e dos pull requests.

Com a pandemia dando uma folga na demanda e com esse empurrão do Github, resolvemos migrar os projetos do Bitbucket para o Github. O problema é que depois de anos usando o mesmo serviço, com centenas de repositórios criados e instalados, o trabalho de migrar tudo de um lugar para outro é gigantesco.

Sabendo que tanto o Bitbucket quanto o Github fornecem APIs nos seus serviços, permitindo interagir com os repositórios, resolvi automatizar o trabalho.


## Clonando um repositório

Para migrar um repositório de um serviço para outro com todo o histórico de commits, branches, tags e notes, primeiro é preciso clonar ele com o parametro `--mirror`. Por exemplo, o comando:

`git clone --mirror git@bitbucket.org:begnini/example.git`

irá clonar o repositório `begnini/example.git` com todos os metadados dele, replicando o seu estado atual.

Depois de clonado, é necessário criar o repositório novo no github (via interface web, por exemplo), configurando as informações necessárias (nome, descrição, privado/publico, etc.).

Feito isso, é possível enviar o repositório para o Github, com o seguinte comando, executado dentro do diretório do repositório:

`git push --mirror git@github.com:begnini/example.git`

Com isso temos uma cópia fiel do repositório que estava no Bitbucket agora armazenada no Github.

Lembrando que para esses comandos funcionarem, é necessário ter adicionado uma chave publica ssh para o usuário.

## Migrando todos repositórios

No meu caso, como tenho mais de 250 repositórios armazenados no Bibucket, seguir esse passo a passo para cada repositório demoraria dias, por isso resolvi automatizar o processo.

Para fazer o trabalho sujo do git, irei utilizar a biblioteca [GitPython](https://gitpython.readthedocs.io/). Ela abstrai os comandos git em python e permite saber se tudo deu certo com a execução.

Abaixo como executar os comandos descritos acima, usando python:

### 1. Clonando um repositório

{% highlight python linenos %}
import git

def clone(user, repository):
    url = 'git@bitbucket.org:' + user + '/' + repository
    directory = 'repositories/' + repository

    repository = git.Repo.clone_from(
        url,
        directory,
        multi_options=['--mirror'])

{% endhighlight %}

### 2. Criando o repositório no Github

Depois de clonado o repositório localmente, é necessário criar ele no Github. Para isso vamos usar a [API do Github](https://developer.github.com/v3/repos/#create).

{% highlight python linenos %}
def create_repository(organization, repository):
    data = {
        'name': repository,
        'private': True,
        'has_issues': True,
        'has_projects': True,
        'has_wiki': True
    }

    headers = {
        'Authorization': 'token ' + GITHUB_TOKEN
    }

    url = 'https://api.github.com/orgs/%s/repos' % (organization)
    response = requests.post(url, data=json.dumps(data), headers=headers)
    return response.ok
{% endhighlight %}


Note que existe uma variável no método, `GITHUB_TOKEN`. Ela é o token de autenticação necessário para se comunicar com a API. É possível obte-la na página [https://github.com/settings/tokens/new](https://github.com/settings/tokens/new?scopes=repo&description=migrando%20repositórios%20bitbucket%20github). Para que o código acima funcione, é necessário liberar o controle completo dos repositórios para esse token. Após gerar o token, o Github irá mostrar uma hash de 40 caracteres, que deverá ser usada nessa variável.

### 3. Enviando o repositório para o Github

Criado o repositório, ele ainda estará vazio. Agora vamos enviar os dados do repositório clonado no passo 1.

{% highlight python linenos %}
def send(organization, repository):
    url = 'git@github.com:' + organization + '/' + repository
    directory = 'repositories/' + repository

    repository = git.repo.base.Repo(directory)

    remote = repository.remote()
    remote.set_url(url, push=True)

    pushInfo = remote.push(mirror=True)
    for info in pushInfo:
        if info.ERROR & info.flags != 0:
            return False

    return True
{% endhighlight %}

### 4. Listando os repositórios do bitbucket

Agora que podemos migrar 1 repositório, vamos pegar todos os que estão armazenados no bitbucket e em um laço envia-los para o github.

Para se comunicar com o bitbucket vamos usar o cliente oficial do Bitbucket para Python, o [PyBitbucket](https://pypi.org/project/pybitbucket/).


{% highlight python linenos %}
from pybitbucket.bitbucket import Client
from pybitbucket.auth import BasicAuthenticator
from pybitbucket.repository import Repository

def get_repositories(username, password, email):
    bitbucket = Client(
        BasicAuthenticator(
            username,
            password,
            email))

    repositories = Repository.find_repositories_by_owner_and_role(
        owner=username,
        role='admin',
        client=bitbucket)

    return list(repositories)
{% endhighlight %}

### 5. Juntando tudo

Agora com todas as partes necessárias criadas, basta juntar tudo em uma função só para fazer a migração:


{% highlight python linenos %}
def migrate(username, password, email, organization):
    repositories = get_repositories(username, password, email)

    for repo in repositores:
        print ('Migrating', repo.slug)

        clone(username, repo.slug)
        create_repository(organization, repo.slug)
        send(organization, repo.slug)
{% endhighlight %}

## Conclusão

Com isso, em pouco mais de 1 hora foi possível mover todos os 250 repositórios de um serviço para o outro. Ainda existem vários outros lugares para mudar (Slack, CI, etc.), que serão feitos conforme a demanda, mas o importante é que todo o código está pronto para ser trabalhado pelos desenvolvedores.

Algumas bases deram erro, devido ao Github ter limitação de tamanho de arquivo e de repositório. Nesses casos, infelizmente, ainda iremos usar o bitbucket por um bom tempo.