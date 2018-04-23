---
title: "Logando acesso SSH no Hipchat"
layout: post
date:   2017-01-29 10:00:00
---

Olá,

existem várias maneiras de adicionar segurança ao acesso SSH dos sistemas. Algumas básicas, como remover acesso root, forçar autenticação via chave, limitar o acesso para endereços IP específicos e por ae vai. Mas mesmo com tudo isso implementado, alguém determinado o suficiente pode conseguir acesso a algum servidor e, com acesso local a maquina, o jogo está perdido.


Perdido porque se o usuário conseguir elevar o acesso dele a superusuário, ele consegue também apagar qualquer rastro que esteve ali, além de poder adicionar camuflagens para encobrir qualquer novo acesso que ele faça.


Uma das formas de impedir que o invasor apague os logs depois de ter acesso ao servidor é configurar o syslog para que salve em outra maquina as informações. Mesmo assim, ainda existe a possibilidade da maquina que comporta o syslog remoto ser invadida. No passado, para contornar isso, os administradores chegavam a ligar a saída padrão do syslog no dispositivo de impressão (`/dev/lp`), imprimindo todos os logs assim que chegassem a maquina remota. Curiosidade: segundo o livro [The Fugitive Game](https://www.amazon.com/gp/product/0316528692/), que relata a história de Kevin Mitnick antes da prisão, [Tsutomu Shimomura](https://en.wikipedia.org/wiki/Tsutomu_Shimomura) descobriu que estava sendo invadido porque a quantidade de papel saindo da impressora matricial que imprimia os logs estava diminuindo.


Como não estamos mais nos anos 90 e imprimir todos os logs que dezenas de servidores produzem é inviável, uma solução é enviar o log para algum serviço de terceiro. No nosso caso o hipchat.

### Pluggabble Authentication Module


Puggable Authentication Module ou PAM é um mecanismo que integra múltiplos esquemas de autenticação em uma API única. A ideia por trás da criação dele é que, caso você queira implementar alguma forma de autenticação, você não precisa reescrever todos os programas que irão usar ela (su, sudo, ssh, etc.), basta implementar um módulo de autenticação usando a API do PAM e plugar esse módulo novo no sistema para que todos enxerguem a sua forma de autenticação.


A API do PAM divide as regras de negócio em 4 grupos independentes entre si:

1. Autenticação e aquisição de credenciais
1. Gerenciamento de contas
1. Gerenciamento de sessão
1. Atualização de token de autenticação (password)


Essa divisão permite a você se preocupar apenas com as regras que esteja precisando manipular.

#### Configuração do PAM

A parte mais interessante do PAM é a facilidade de plugar um módulo novo, sem precisar compilar ou reiniciar o sistema para que ele entre em efeito.

Existem 2 modos de configurar o PAM. Modificar o `/etc/pam.conf` ou modificar algum dos arquivos em `/etc/pam.d/`. No `pam.conf`, cada linha é composta dos seguintes tokens, separados por espaço:

`
service type control module-path module-arguments
`

**service** é o nome do serviço ou comando que será configurado. Por exemplo: su, sudo, login, ssh.

**type** é o grupo de regras para qual o módulo será aplicado. Pode ser auth, account, session e password.

**control** é o tipo de comportamento que o PAM deve ter quando o módulo de alguma forma falhar. Os comportamentos podem ser:

* **required** - quando um módulo do PAM falha, ele retorna um erro para a aplicação, mas apenas depois de todos os módulos daquele grupo serem executados para aquele serviço.
* **requisite** - parecido com **required**, com a diferença que quando o módulo falha, ele retorna imediatamente o controle para a aplicação, sem passar pelos outros módulos que podem haver.
* **sufficient** - basta que esse módulo retorne sucesso, para que a aplicação retorne com sucesso. Mas caso outro módulo com **required** falhe antes, esse sucesso é ignorado.
* **optional** - o sucesso ou a falha desse módulo só é importante se ele for o único módulo configurado para esse tipo e serviço.
* **include** - adiciona toda a configuração a partir de outro arquivo de configuração.
* **substack** - parecido com o **include**, com a diferença que o sucesso ou a falha que acontecer com essa configuração não atinge os comandos fora da configuração.


Os arquivos que ficam no diretório `/etc/pam.d/` seguem a mesma sintaxe, com a diferença que as regras para cada serviço ficam dentro de um arquivo com o mesmo nome do serviço e, por isso, na configuração de cada regra o serviço é removido.


#### pam_exec

O módulo **pam_exec** permite executar comandos em qualquer grupo do PAM. É ele quem vai executar o shell que iremos criar para enviar as informações que precisamos para o Hipchat.

Um detalhe importante desse comando é que por padrão ele executa o comando como o usuário que chamou o executável. Para forçar o módulo a chamar o comando como o usuário logado, temos que passar a opção `seteuid` na configuração dele.

Esse módulo expõe nas variáveis de ambiente as seguintes informações:

* **PAM_RHOST** - Contem o endereço remoto do usuário acessando o sistema.
* **PAM_RUSER** - Contem o usuário remoto acessando o sistema.
* **PAM_SERVICE** - É o serviço ou comando que a regra foi aplicada.
* **PAM_TTY** - É o terminal que o usuário conectou.
* **PAM_USER** - É o usuário logado via pam.
* **PAM_TYPE** - É o tipo de módulo chamado, pode ser **account**, **auth**, **password**, **open_session** e **close_session**.


Nosso shell irá usar essas variáveis para enviar as informações para o Hipchat.


### Hipchat API

Iremos utilizar a [API do Hipchat](https://www.hipchat.com/docs/apiv2) para enviar as mensagens para o canal que escolhermos. Para isso usaremos o endpoint [send_message da room API](https://www.hipchat.com/docs/apiv2/method/send_message). Esse endpoint precisa de 3 informações:

* **AUTH_TOKEN** - O token da aplicação com acesso ao envio de mensagens no canal.
* **ROOM_ID** - o nome ou id da sala para qual vai a mensagem.
* **MESSAGE** - A mensagem a ser enviada, podendo ser texto puro ou um html com algumas formatações.

Além das informações que vem do PAM, podemos colocar outras informações que quisermos na mensagem, como o host acessado, o ip do host, etc.

O script final ficaria assim:


{% highlight bash linenos %}
#!/bin/bash
#
# Send SSH Login information to hipchat

ROOM_ID=ssh_notifications
AUTH_TOKEN=YOUR_TOKEN_HERE

HOST=$(hostname)
HOSTIP=$(hostname -I)

MESSAGE="<strong>$PAM_USER</strong> connected to <strong>$HOST</strong> ($HOSTIP) from <strong>$PAM_RHOST</strong>"

if [ "$PAM_TYPE" == "open_session" ]; then
	curl --header "content-type: application/json" \
		--header "Authorization: Bearer $AUTH_TOKEN" \
		-X POST \
		-d "{\"from\":\"Security Bot\", \"notify\": true, \"color\": \"red\", \"message_format\": \"html\", \"message\": \"$MESSAGE\"}" \
		https://api.hipchat.com/v2/room/$ROOM_ID/notification &
fi

{% endhighlight %}

Os parâmetros passados no corpo do POST permitem algumas customizações:

* **from**: É o usuário que irá aparecer como remetente da mensagem
* **notify**: se true, a mensagem deve gerar uma notificação no canal, senão o hipchat não notifica que uma mensagem nova chegou.
* **color**: é a cor que o fundo da mensagem irá aparecer no canal, aceita qualquer cor html.
* **message_format**: é o formato da mensagem, pode ser text ou html

Um detalhe, estamos enviando o curl para rodar em background com a opção `&`. Isso é necessário porque senão o pam irá esperar o comando terminar antes de liberar acesso ao terminal para o usuário e criar uma espera de alguns segundos.


### Configuração

Agora basta salvar esse script em algum lugar do sistema. Por padrão eu coloco ele em `/usr/local/bin/notify_login.sh`, dar permissão de execução para todo mundo e configurar o pam do sshd para chamar ele toda vez que logar.

Para configurar o pam, insira a seguinte linha no arquivo `/etc/pam.d/sshd`.


`
session optional pam_exec.so seteuid quiet /usr/local/bin/notify_login.sh
`

e reinicie o serviço sshd para que a configuração entre em efeito. Agora, a cada login com sucesso, você será avisado no hipchat, com todas as informações que precisa para auditar um possível ataque.


### Conclusão

Esse tipo de auditoria pode ser extendido para outros serviços também, como su, sudo e passwd. Os comandos podem ser gravados e enviados quando o **PAM_TYPE** for `close_session`. São todas implementações simples que aumentam muito a confiança na segurança do sistema.

Até a proxima.



#### Fontes

[man pam.conf](https://linux.die.net/man/5/pam.conf)

[man pam_exec](https://linux.die.net/man/8/pam_exec)

[Pluggable Authentication Module for Linux](http://www.linuxjournal.com/article/2120)

[Posting successful ssh logins to slack](http://sandrinodimattia.net/posting-successful-ssh-logins-to-slack/)

[Hipchat V2 API](https://www.hipchat.com/docs/apiv2)

