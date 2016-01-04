---
title: "Quatro anos de ataques SSH"
layout: post
date:   2016-01-02 10:00:00
---


Olá,

desde 2008 eu administro alguns servidores alocados na rede da UFMS, de responsabilidade da RNP. Em alguns desses servidores alunos podiam acessar e, como não havia nenhuma politica de segurança, atacantes conseguiram invadir uma conta através de força bruta via SSH.

Depois de toda dor de cabeça de reinstalar o servidor, resolvi implementar o Fail2ban nos servidores para mitigar esses ataques. Para quem não conhece, o Fail2ban é um serviço que fica analisando os arquivos de logs a procura ataques, como falhas de autenticação, tentativas de exploits, etc. e, quando encontra, adiciona uma regra no firewall automaticamente banindo o ip responsável. Ele também pode fazer outras ações, como mandar um e-mail cada vez que um ip é banido.

Essa possibilidade de mandar e-mail é interessante, pois muitas vezes o reconhecimento de um ataque não é realmente um ataque. Um usuário pode ter esquecido o login ou a senha dele, por exemplo. Com isso em mente, configurei o Fail2ban para me mandar e-mail toda vez que um ip fosse banido, com informações sobre o ataque.

Além disso, configurei ele para fazer DNS reverso no endereço ip e através do WHOIS do dominio, descobrir quem é o responsavel pela rede. A [RFC 2142](https://www.ietf.org/rfc/rfc2142.txt) define que todo dominio deve ter a conta de e-mail `abuse` para receber alertas de comportamento inapropriado das maquinas dessa rede. Por isso o Fail2ban foi configurado para mandar um e-mail para `abuse@dominio` toda vez que um ataque acontecesse.

Infelizmente, devido a falta de espaço no meu gmail, eu acabei deletando todos os e-mails do Fail2ban no começo de 2012. Como logo depois eu fiquei milionário, pude pagar U$ 5.00 por ano por mais 20 GB de e-mail, podendo guardar todos os ataques desde 2012 até hoje.

Cada e-mail contêm o ip do atacante, o horário do ataque e o login dos usuários nas tentativas. No total são 171.866 alertas de ataque durante esses 4 anos, o suficiente para tirar algumas estatísticas interessantes.

Primeiramente, desenhando a quantidade de ataques por ano, dá pra notar que de 2013 para 2014 os ataques aumentaram mais de 4 vezes.

<iframe src="/images/2016-01-02-quatro-anos-de-ataques-ssh/by_year.html" width="100%" height="340" seamless frameBorder="0" scrolling="no"></iframe>


Para ter uma idéia do que aconteceu e porque houve essa explosão de ataques durante 2014 e 2015, resolvi desenhar por mês a quantidade de ataques.


<iframe src="/images/2016-01-02-quatro-anos-de-ataques-ssh/by_month.html" width="100%" height="350" seamless frameBorder="0" scrolling="no"></iframe>


Parece haver um aumento normal no número de ataques, tirando 3 meses em que a coisa ficou preta. Dezembro de 2014, maio e junho de 2015 foram os meses que mais tiveram ataques, fugindo completamente do padrão. Como um bom estatistico, quando vejo outliers assim, logo vem a cabeça que existe algum bug na extração ou na consolidação dos dados, por isso resolvi plotar os dados diários da quantidade de ataques desses 3 meses.


Primeiro, postando a quantidade de ataques diários de dezembro de 2014:

<iframe src="/images/2016-01-02-quatro-anos-de-ataques-ssh/by_day_dez2014.html" width="100%" height="350" seamless frameBorder="0" scrolling="no"></iframe>

Incrivel. Nos dias 09, 10 e 11 de dezembro os servidores foram atacados com extrema força, 6 vezes mais ataques que a média do ano de 2014. Para se ter uma idéia do tamanho do volume do ataque, nesses 4 anos nenhum outro dia sofreu tantos ataques quanto o dia 10 e o dia 09 e 11 ficaram em segundo e terceiros lugares, respectivamente. Descontando os 4 mil ataques desses 3 dias, Dezembro de 2014 fica um tanto normal, bem parecido com Novembro e Outubro.

Olhando agora para Maio de 2015:


<iframe src="/images/2016-01-02-quatro-anos-de-ataques-ssh/by_day_may2015.html" width="100%" height="350" seamless frameBorder="0" scrolling="no"></iframe>

Diferentemente de Dezembro de 2014, os ataques estão mais distribuidos entre os dias. Mesmo tirando os dias 20 e 28 de Maio, a média de ataques por dia é de 425, 2.2 vezes maior que a média do ano.

E olhando os dados de Junho de 2015:

<iframe src="/images/2016-01-02-quatro-anos-de-ataques-ssh/by_day_jun2015.html" width="100%" height="350" seamless frameBorder="0" scrolling="no"></iframe>

É basicamente uma continuação da onda de ataques do mês anterior, dando uma tregua apenas no dia 28. Parece então que não existe nada errado com a análise, foram ataques de grande magnitude mesmo.


Continuando a análise, antigamente existia o conceito de que hackers eram na maioria adolescentes no ensino médio e por isso os ataques eram maiores nos fins de semana e na época de férias escolares. Pelos gráficos já analisados é dificil dizer se realmente existem ataques maiores nas férias escolares.

Olhando então o gráfico de ataques por dia da semana:


<iframe src="/images/2016-01-02-quatro-anos-de-ataques-ssh/by_weekday.html" width="100%" height="350" seamless frameBorder="0" scrolling="no"></iframe>

Realmente não existe um padrão, estáo bem distribuidos entre os dias da semana. Isso porque os ataques hoje em dia são na maioria automatizados e botnets não tem fim de semana nem feriado.


### Análise por país de origem


Já sabemos que os ataques são feitos de maneira automática, massiva e por isso não existe um padrão de data dos ataques. Também vimos que existem ondas de ataques, que podem durar alguns dias ou meses. Podemos agora começar a aprofundar a análise olhando o país de origem de cada um dos ataques. Para montar os gráficos eu usei a lib `geoip` do python onde posso consultar através do endereço IP qual o país dono.

O gráfico abaixo mostra os 10 países com mais origens de ataque:

<iframe src="/images/2016-01-02-quatro-anos-de-ataques-ssh/by_country.html" width="100%" height="350" seamless frameBorder="0" scrolling="no"></iframe>


A China é campeã absoluta, 55% dos ataques partem de uma maquina chinesa, mostrando a insegurança das redes chinesas. Em segundo lugar os Estados Unidos, provavelmente mais pela quantidade de maquinas na rede do que pela qualidade da segurança delas. O Brasil aparece em terceiro, mas imagino que seja mais porque os servidores atacados estejam também no Brasil e os bots devem procurar as maquinas em um range de ips próximo.

Para ter uma ideia da evolução dos ataques durante os anos, separei eles por país e agrupei por ano, normalizando pela quantidade de ataques no ano.

<iframe src="/images/2016-01-02-quatro-anos-de-ataques-ssh/by_country_by_year.html" width="100%" height="350" seamless frameBorder="0" scrolling="no"></iframe>

O gráfico mostra a China um pouco mais comportada em 2012 e 2013, com os Estados Unidos não muito distante em quantidade de ataques. Mas 2014 e 2015 a China disparou em quantidade de ataques, de forma que ofuscou a participação dos outros países.

O gráfico mostra também um preocupante crescimento de Hong Kong, Holanda e França.

#### Made in China

Como a China é responsavel pela maioria dos ataques é interessante ter uma visualização apenas dos ataques originários de lá.

<iframe src="/images/2016-01-02-quatro-anos-de-ataques-ssh/cn_by_month.html" width="100%" height="350" seamless frameBorder="0" scrolling="no"></iframe>

Interessante, enquanto esse gráfico mostra claramente que a onda de ataques de Maio e Junho de 2015 vieram principalmente da China, mostra também que o pico de ataques de Dezembro de 2014 não foi responsabilidade dela, pelo menos não diretamente (spoiler: Estados Unidos e Alemanha foram os vilões).

Com isso surge uma questão, será que os ataques vindos de outros paises tem origem na China? Existiria como rastrear o "paciente zero", a primeira maquina a ser infectada? Infelizmente, apenas com as informações do Fail2ban, não é possivel inferir de onde vem as primeiras infecções.

### Análise por login

Cada alerta de ataque tem até 7 tentativas de logins. No total foram 1.098.087 tentativas de login incorretas.

O gráfico abaixo mostra os 20 logins mais tentados nos ataques:

<iframe src="/images/2016-01-02-quatro-anos-de-ataques-ssh/by_login.html" width="100%" height="350" seamless frameBorder="0" scrolling="no"></iframe>

A maioria das tentativas são de logins administradores do sistema, root e admin, quase 74% do total.

Os logins tentados podem ser separados em 5 grupos:

1. logins padrão de sistema: root, admin, bin, adm, log, staff, guest, support
1. logins de serviço: nagios, oracle, ftpuser, kadmin, apache, postgres, ftp, cron, git
1. logins padrão de roteadores: ubnt, PlcmSpIp, vyatta, MGR, cisco, operator
1. logins padrão de distribuições: xbian, vagrant, ubuntu, kodi
1. palavras comuns:
    1. chinesas: zhangyan
    1. inglesas: test, sales, info, default, aaron, fluffy, restaurant
    1. espanholas: pruebas, ventas, reportes, prueba, soporte
    1. portuguesas: usuario, sistemas, teste


#### Distribuição dos ataques por tipo de login

Para cada tipo de login descrito acima, descrevi um gráfico mostrando os ataques mês a mês. Removi os usuários root e bin que bagunçam com os gráficos.

Abaixo a distribuição das tentativas com usuários padrão de sistema, com exceção dos usuários root e admin:

<iframe src="/images/2016-01-02-quatro-anos-de-ataques-ssh/login_default_by_month.html" width="100%" height="350" seamless frameBorder="0" scrolling="no"></iframe>

Agora a distribuição das tentativas levando em consideração os logins de serviço:

<iframe src="/images/2016-01-02-quatro-anos-de-ataques-ssh/login_system_by_month.html" width="100%" height="350" seamless frameBorder="0" scrolling="no"></iframe>

Os ataques com usuários padrão dos roteadores desenhados abaixo. A partir de Setembro de 2014, os ataques aumentaram significativamente:

<iframe src="/images/2016-01-02-quatro-anos-de-ataques-ssh/login_router_by_month.html" width="100%" height="350" seamless frameBorder="0" scrolling="no"></iframe>

Interessante é olhar a origem de ataques usando esses logins. A China não é fica mais em primeiro lugar e os ataques são bem distribuidos entre paises do primeiro mundo.

<iframe src="/images/2016-01-02-quatro-anos-de-ataques-ssh/router_by_country.html" width="100%" height="350" seamless frameBorder="0" scrolling="no"></iframe>


Ataques usando logins padrões de distribuições Linux:

<iframe src="/images/2016-01-02-quatro-anos-de-ataques-ssh/login_distro_by_month.html" width="100%" height="350" seamless frameBorder="0" scrolling="no"></iframe>

Ataques usando palavras comuns:

<iframe src="/images/2016-01-02-quatro-anos-de-ataques-ssh/login_common_by_month.html" width="100%" height="350" seamless frameBorder="0" scrolling="no"></iframe>

Uma palavra interessante, `zhangyan`, apareceu pela primeira vez em Maio de 2014 apenas. Depois disso os ataques usando essa palavra começaram a ser frequentes.

Outra coisa interessante são as tentativas de login das palavras espanholas, a maioria vem de paises de lingua espanhola. Abaixo os 10 principais paises de origem de ataques com palavras de lingua espanhola:


<iframe src="/images/2016-01-02-quatro-anos-de-ataques-ssh/spanish_by_country.html" width="100%" height="350" seamless frameBorder="0" scrolling="no"></iframe>


### Conclusão

Apresentei algumas estatísticas dos 4 anos de ataques sofridos via SSH pelos servidores que administro. Esse tipo de ataque deve ter pelo menos 15 anos de idade e nos ultimos tempos tem crescido muito. Os e-mails enviados automaticamente para a caixa postal `abuse@` dos domínios não tem gerado muito resultado, nesses 4 anos apenas 87 responderam aos e-mails dizendo que tomariam alguma ação sobre a invasão, a maioria (42 respostas) do CERT.br.

Acredito que a melhor forma de combater esses ataques é conscientizar os sysadmins para que barrem na ponta esse tipo de ataque. Algumas dicas:

* Criar whitelists com faixas de IP que são acessadas é uma forma bem efetiva.
* Instalar o fail2ban é interessante para saber como os ataques estão evoluindo.
* Desligar o acesso a root via ssh é fundamental.
* Remover acesso via senha as vezes é complicado demais, mas não tem desculpa para não ter uma politica de senhas fortes


