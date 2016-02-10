---
title: "Otimizando a geração de imagens - Parte 2"
layout: post
date:   2016-02-10 10:00:00
---


Olá,

essa é a segunda parte do post sobre otimização de geração de imagens ([primeiro post aqui](http://www.begnini.net/2016/01/12/otimizando-a-geracao-de-imagens-parte-1.html)). Nesse post eu vou falar sobre otimização de rotação de imagens.

Atualmente no sistema a rotação de imagens é a operação mais delicada porque, ao contrário das outras operações que são feitas em lote, ela é feita apenas quando o usuário e ele precisa esperar que a operação acabe para continuar com o trabalho.


## Rotação

O sistema gera rotação apenas em 3 ângulos fixos, de 90, 180 e 270 graus. Isso torna algumas coisas mais simples, já que existem otimizações para rotação nesses ângulos. Como no post anterior, todos os testes foram feitos rodando o mesmo comando três vezes em 50 imagens com média de tamanho de 2500x3300 pixels cada. O tempo de processamento mostrado é a média do tempo de processamento de cada imagem.

Atualmente, o comando usado pelo sistema para rotacionar uma imagem em 90 graus é o seguinte:

`
gm convert $file -rotate 90 rotated/$file
`

Como já tem o GraphicsMagick compilado do post passado, o primeiro experimento foi ver se existe algum ganho rodando nos compilados.


<iframe src="/images/2016-02-10-otimizando-a-geracao-de-imagens-parte-2/compilation.html" width="100%" height="340" seamless frameBorder="0" scrolling="no"></iframe>


Nenhuma diferença significativa aconteceu compilando o GraphicsMagick. Todas as rotações demoraram em média 750ms cada, um tempo muito alto para o usuário esperar.

### VIPS

A opção foi então buscar alternativas ao GraphicsMagick. Nas buscas pelo google, uma biblioteca que chamou a atenção foi a [libvips](http://www.vips.ecs.soton.ac.uk/index.php?title=Libvips).

A libvips é otimizada para processar imagens grandes, justamente o problema que tenho. O que faz com que a libvips tenha melhor performance que outras bibliotecas é a maneira com que ela carrega os dados da imagem na memória, carregando apenas o suficiente para o processamento e tornando a parte de I/O multithread, lendo e salvando os dados em paralelo com o processamento.

No debian existe o pacote `libvips-tools`, com um conjunto de ferramentas baseadas na libvips. O Debian 7 (meu ambiente de testes) vem com a versão 7.28 disponível. As versões mantidas atualmente são a versão 7.x, sendo a última release a [7.42.3](http://www.vips.ecs.soton.ac.uk/supported/7.42/) e a versão 8.x, sendo a última release a [8.2.2](http://www.vips.ecs.soton.ac.uk/supported/8.2/).

Por padrão, a libvips usa todos os processadores disponíveis para fazer o processamento. Nos testes, quanto mais processadores, pior o resultado. Como é possível controlar o número de processadores usados através do parâmetro `--vips-concurrency`, fixei todos os comandos rodados passando o parâmetro `--vips-concurrency=1`.

No gráfico abaixo, a comparação do processamento entre o GraphicsMagick atual e as 3 versões do libvips:


<iframe src="/images/2016-02-10-otimizando-a-geracao-de-imagens-parte-2/vips.html" width="100%" height="340" seamless frameBorder="0" scrolling="no"></iframe>

Houve alguma melhora, mas extremamente tímida. O melhor desempenho, da libvips 7.42.3, é apenas 7% mais rápida que o GraphicsMagick atual. Não era o esperado. O que pode ter acontecido?

Uma feature interessante da vips é a possibilidade de mostrar o tempo gasto em cada etapa do processamento da imagem. O parâmetro `--vips-progress` pode dar uma dica do que está acontecendo e onde está o gargalo de processamento.

Rodando em uma imagem de teste, o output do comando vips com `--vips-progress` é:

{% highlight bash linenos %}

$ vips rot DO22551040119990020_0001.jpg tmp/DO22551040119990020_0001.jpg \
  d90 --vips-concurrency=1 --vips-progress

vips temp-3: 3310 x 2489 pixels, 1 threads, 128 x 128 tiles, 256 lines in buffer
vips temp-4: 2489 x 3310 pixels, 1 threads, 2489 x 16 tiles, 256 lines in buffer
vips temp-4: done in 0,19s
vips temp-3: done in 0,502s

{% endhighlight %}

Isso quer dizer que o programa está rodando 2 estágios. No primeiro estágio, ele aloca a imagem de saída na memória em pedaços de 128 x 128 pixels, organizados em blocos de 256 linhas. No segundo estágio, ele descomprime a imagem de entrada em pedaços de 2489 x 16 pixels, organizados em 256 linhas. Essas operações todas demoram 502ms, sendo o tempo total de processamento dessa imagem de 584ms.

Isso quer dizer que a maior parte do processamento de rotação não está na rotação em si, mas sim na compressão e descompressão da imagem (JPEG).

### libjpeg-turbo

Se o objetivo é acelerar o processamento da rotação, o investimento então tem quer ser todo na aceleração da leitura e escrita do JPEG. Para ajudar com isso, vou testar a biblioteca libjpeg-turbo.

A [libjpeg-turbo](http://libjpeg-turbo.virtualgl.org/) é uma biblioteca derivada da libjpeg mas que usa operações SIMD (MMX, SSE2, NEON) para acelerar a compressão e descompressão das imagens. Os benchmarks do site mostram que ela é de 2 a 5 vezes mais rápida que a libjpeg. O interessante dela é que como ela é compatível com a libjpeg, a troca de uma lib por outra é extremamente simples, não precisa modificar código para usá-la.

Para os programas usarem a libjpeg-turbo, basta mudar o LD_LIBRARY_PATH para apontar para o diretório em que ela for instalada caso o programa já venha compilado na distribuição ou, nas versões compiladas, apontar para o diretório da libjpeg-turbo durante a compilação.

Vou baixar e trabalhar com a versão mais atual da libjpeg-turbo (1.4.2). Apenas um detalhe, para usar a libjpeg-turbo junto com a libvips, precisa adicionar o suporte a jpeg8 durante a compilação. Os comandos para configurar, compilar e instalar a libjpeg-turbo são os abaixo:

{% highlight bash %}

$ ./configure --with-jpeg8
$ make
$ sudo make install

{% endhighlight %}


Por padrão ela se instala no diretório `/opt/libjpeg-turbo/`. Isso é bom porque a biblioteca não vai interferir em outros programas que usem a libjpeg, caso exista algum receio de incompatibilidade.

Agora eu irei recompilar a vips 7.42.3 e 8.2.2, linkando elas com a libjpeg-turbo. Lembrando que o diretório a ser passado no parâmetro é o `/opt/libjpeg-turbo/lib64/`, no meu caso que estou numa arquitetura 64 bits (ou `/opt/libjpeg-turbo/lib/` para arquitetura 32 bits).

Para compilar a vips 7.42.3 vou usar o seguinte comando:

{% highlight bash %}

 $ cd vips-7.42.3/
 $ ./configure --with-jpeg-libraries=/opt/libjpeg-turbo/lib64/
 $ make

{% endhighlight %}

Para compilar a vips 8.2.2 é basicamente os mesmos passos:

{% highlight bash %}

 $ cd vips-8.2.2/
 $ ./configure --with-jpeg-libraries=/opt/libjpeg-turbo/lib64/
 $ make

{% endhighlight %}

Agora vou rodar de novo o benchmark acima e ver se os resultados melhoraram. Lembrando que para os programas usarem a libjpeg-turbo, tem que mudar o LD_LIBRARY_PATH para eles enxergarem a biblioteca. Nesse caso, o comando vai ser o abaixo:

`export LD_LIBRARY_PATH=/opt/libjpeg-turbo/lib64/`

<iframe src="/images/2016-02-10-otimizando-a-geracao-de-imagens-parte-2/libjpeg.html" width="100%" height="340" seamless frameBorder="0" scrolling="no"></iframe>


Deixei a primeira coluna com o tempo de processamento atual, como base de comparação. Sem precisar recompilar, a libjpeg-turbo diminuiu em 280ms o tempo de processamento da rotação com GraphicsMagick. E agora com a libjpeg-turbo é que a libvips se destacou, sendo quase 2 vezes mais rápida que o processamento atual e 20% mais rápida que o GraphicsMagick com libjpeg-turbo.


Agora que chegamos num tempo razoável, vou medir o tempo de processamento de todas as 3 rotações: 90, 180 e 270 graus.

<iframe src="/images/2016-02-10-otimizando-a-geracao-de-imagens-parte-2/rotations.html" width="100%" height="340" seamless frameBorder="0" scrolling="no"></iframe>

No geral os tempos foram bem parecidos entre os ângulos. A campeã foi a vips 7.42.3 por muito pouco, dá para dizer que as 3 versões tem o mesmo tempo de processamento se levar em consideração as possíveis oscilações no sistema durante a medição.


### JPEGTRAN

Mas o tempo ainda está alto para o usuário esperar e a libjpeg-turbo ainda tem mais uma surpresinha para a gente. A surpresa é o comando jpegtran.

O jpegtran é um utilitário desenvolvido primeiramente pelo [JPEG Club](http://jpegclub.org/) com o objetivo de permitir a manipulação de JPEGs. Uma dessas manipulações é justamente o que preciso, rotacionar 90, 180 e 270 graus. Como a libjpeg-turbo é toda otimizada para a arquitetura que estou usando, pode ser que os resultados sejam interessantes.

Abaixo a comparação entre o tempo atual de processamento, o tempo usando GraphicsMagick com libjpeg-turbo, o tempo da versão mais rápida da libvips e o tempo da jpegtran nas rotações:

<iframe src="/images/2016-02-10-otimizando-a-geracao-de-imagens-parte-2/jpegtran.html" width="100%" height="340" seamless frameBorder="0" scrolling="no"></iframe>

Fantástico. O jpegtran foi mais de 3 vezes mais rápido que o GraphicsMagick atual e quase 2 vezes mais rápido que a libvips. Levando em consideração que o tempo de leitura e gravação  está em torno de 90ms por imagem, dificilmente dá para diminuir esse tempo com o mesmo hardware.

No final deu para manter abaixo de 230ms o tempo médio de rotação de uma imagem em qualquer ângulo.

### Conclusão

Esse post foi um post um pouco complicado. No começo eu não sabia se seria possível diminuir o tempo de processamento e a primeira e mais obvia escolha, compilar tudo com um monte de otimizações (força-bruta no compilador) se mostrou a menos frutífera, mostrando que não existe (ainda) compilador que faça mágica.

Mas pensando um pouco fora da caixa, estudando e testando várias soluções, foi possível resolver o desafio proposto.

Espero mais uma vez que você tenha aprendido um pouco do que eu aprendi escrevendo esse post e que possa usar as dicas descritas aqui para melhorar sua empresa, seu software ou sua vida em geral. Qualquer duvida ou sugestão, estou a disposição.

Até a próxima.
