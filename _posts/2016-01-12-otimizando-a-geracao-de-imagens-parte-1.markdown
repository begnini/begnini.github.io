---
title: "Otimizando a geração de imagens - Parte 1"
layout: post
date:   2016-01-12 00:00:00
---


Olá,

há anos trabalho num sistema de gerenciamento eletrônico de documentos, no qual a principal característica é controlar o fluxo de digitalização e qualidade de milhões de imagens. Nesse sistema, o digitalizador usando um scanner de alta produção digitaliza as imagens em 300 dpi com a ajuda de um software desktop e depois envia em lotes essas imagens para o servidor onde está o sistema WEB.

O sistema, por sua vez, gera vários formatos de arquivos para visualização e arquivamento dos documentos. Aqui eu irei dar foco apenas em 3 tipos de processamento que o sistema faz para gerar esses arquivos, geração de thumbnail, rotação e limpeza de imagem usando lat - latent adaptive threshold.

A ideia de revisitar esses processamentos foi por eles serem os maiores consumidores de CPU e devido a percepção que o sistema demora demais para gerar essas imagens, enquanto outros sistemas na web (gmail, flickr) geram quase instantaneamente. As máquinas servidoras são bem parrudas para esse tipo de serviço (cada servidor tem 2 processadores Intel Xeon E5-2650, 64 GB de RAM), por isso esses processamentos não deveriam demorar tanto.

Esse post eu irei dividir em 3 partes para não ficar muito maçante. Essa primeira parte fala sobre geração de thumbnails.


## Thumbnails

Atualmente o sistema gera os thumbnails com 300 pixels de largura e a altura variando conforme a razão de aspecto. Para esse processamento é usado o GraphicsMagick, inspirado nesse [post da etsy](https://codeascraft.com/2010/07/09/batch-processing-millions-of-images/). O comando usado hoje para gerar os thumbnails é basicamente o seguinte:

`
gm convert $file -geometry 300x thumbs/${file/.jpg/.png}
`

Nesse comando pega-se um arquivo jpeg na sua resolução original e o transforma num arquivo PNG com largura de 300 pixels.

### Usando Filtros

Estudando o GraphicsMagick, descobri que esse comando esconde um parâmetro muito importante, o `-filter`. Sem ele, o GraphicsMagick escolhe o filtro mais apropriado para uma qualidade boa sem consumir muita CPU. Mudando o filtro usado, é possível economizar CPU perdendo um pouco de qualidade no resultado final.

Para estabelecer os tempos de processamento, foram rodadas 3 vezes o comando abaixo, variando apenas o filtro:

{% highlight bash %}
time for file in *.jpg; do
  gm convert $file -filter $filter -geometry 300x thumbs/${file/.jpg/.png};
done
{% endhighlight %}


em um diretório com 50 imagens com média de tamanho de 2500x3300 pixels cada e calculado a média do tempo de processamento (somando sys e user).


<iframe src="/images/2016-01-12-otimizando-a-geracao-de-imagens-parte-1/filters.html" width="100%" height="340" seamless frameBorder="0" scrolling="no"></iframe>

O primeiro tempo é o tempo de processamento atual do sistema, sem filtro algum, levando uma média de 1500ms de processamento por imagem para gerar um thumbnail. Conforme o filtro, houve um ganho bom de tempo de processamento, com o Point sendo mais de 2 vezes mais rápido que o atual. Mas como fica a qualidade desses thumbs para cada filtro?

Abaixo estão as imagens geradas com cada um dos filtros para comparar a qualidade, as imagens estão na mesma ordem do gráfico (clique para ver a imagem com 300 pixels):


<a href="/images/2016-01-12-otimizando-a-geracao-de-imagens-parte-1/default.png">
	<img src="/images/2016-01-12-otimizando-a-geracao-de-imagens-parte-1/default.png" alt="Sem Filtro" title="Sem Filtro" width="180px" />
</a>
<a href="/images/2016-01-12-otimizando-a-geracao-de-imagens-parte-1/Sinc.png">
	<img src="/images/2016-01-12-otimizando-a-geracao-de-imagens-parte-1/Sinc.png" alt="-filter Sinc" title="-filter Sinc" width="180px" />
</a>
<a href="/images/2016-01-12-otimizando-a-geracao-de-imagens-parte-1/Bessel.png">
	<img src="/images/2016-01-12-otimizando-a-geracao-de-imagens-parte-1/Bessel.png"  alt="-filter Bessel"  title="-filter Bessel"  width="180px" />
</a>
<a href="/images/2016-01-12-otimizando-a-geracao-de-imagens-parte-1/Lanczos.png">
	<img src="/images/2016-01-12-otimizando-a-geracao-de-imagens-parte-1/Lanczos.png"  alt="-filter Lanczos"  title="-filter Lanczos"  width="180px" />
</a>


<a href="/images/2016-01-12-otimizando-a-geracao-de-imagens-parte-1/Cubic.png">
	<img src="/images/2016-01-12-otimizando-a-geracao-de-imagens-parte-1/Cubic.png" alt="-filter Cubic" title="-filter Cubic" width="180px" />
</a>
<a href="/images/2016-01-12-otimizando-a-geracao-de-imagens-parte-1/Mitchell.png">
	<img src="/images/2016-01-12-otimizando-a-geracao-de-imagens-parte-1/Mitchell.png" alt="-filter Mitchell" title="-filter Mitchell" width="180px" />
</a>
<a href="/images/2016-01-12-otimizando-a-geracao-de-imagens-parte-1/Catrom.png">
	<img src="/images/2016-01-12-otimizando-a-geracao-de-imagens-parte-1/Catrom.png" alt="-filter Catrom" title="-filter Catrom" width="180px" />
</a>
<a href="/images/2016-01-12-otimizando-a-geracao-de-imagens-parte-1/Quadratic.png">
	<img src="/images/2016-01-12-otimizando-a-geracao-de-imagens-parte-1/Quadratic.png"  alt="-filter Quadratic"  title="-filter Quadratic"  width="180px" />
</a>


<a href="/images/2016-01-12-otimizando-a-geracao-de-imagens-parte-1/Gaussian.png">
	<img src="/images/2016-01-12-otimizando-a-geracao-de-imagens-parte-1/Gaussian.png" alt="-filter Gaussian" title="-filter Gaussian" width="180px" />
</a>
<a href="/images/2016-01-12-otimizando-a-geracao-de-imagens-parte-1/Hermite.png">
	<img src="/images/2016-01-12-otimizando-a-geracao-de-imagens-parte-1/Hermite.png" alt="-filter Hermite" title="-filter Hermite" width="180px" />
</a>
<a href="/images/2016-01-12-otimizando-a-geracao-de-imagens-parte-1/Triangle.png">
	<img src="/images/2016-01-12-otimizando-a-geracao-de-imagens-parte-1/Triangle.png" alt="-filter Triangle" title="-filter Triangle" width="180px" />
</a>
<a href="/images/2016-01-12-otimizando-a-geracao-de-imagens-parte-1/Hanning.png">
	<img src="/images/2016-01-12-otimizando-a-geracao-de-imagens-parte-1/Hanning.png" alt="-filter Hanning" title="-filter Hanning" width="180px" />
</a>


<a href="/images/2016-01-12-otimizando-a-geracao-de-imagens-parte-1/Hamming.png">
	<img src="/images/2016-01-12-otimizando-a-geracao-de-imagens-parte-1/Hamming.png"  alt="-filter Hamming"  title="-filter Hamming"  width="180px" />
</a>
<a href="/images/2016-01-12-otimizando-a-geracao-de-imagens-parte-1/Blackman.png">
	<img src="/images/2016-01-12-otimizando-a-geracao-de-imagens-parte-1/Blackman.png" alt="-filter Blackman" title="-filter Blackman" width="180px" />
</a>
<a href="/images/2016-01-12-otimizando-a-geracao-de-imagens-parte-1/Box.png">
	<img src="/images/2016-01-12-otimizando-a-geracao-de-imagens-parte-1/Box.png" alt="-filter Box" title="-filter Box" width="180px" />
</a>
<a href="/images/2016-01-12-otimizando-a-geracao-de-imagens-parte-1/Point.png">
	<img src="/images/2016-01-12-otimizando-a-geracao-de-imagens-parte-1/Point.png" alt="-filter Point" title="-filter Point" width="180px" />
</a>



Tirando o thumbnail gerado com o filtro Point, que ficou bem ruim, todos os outros estão quase idênticos, com qualidade excelente para o sistema.

Só que mesmo o menor tempo dos filtros aceitáveis (Box, com 761ms) ainda é um tempo muito alto para o usuário esperar.

### Compilando GraphicsMagick


A próxima ideia então é compilar a última versão do GraphicsMagick (1.3.23) com vários parâmetros de otimização para ver se tem algum ganho comparado com o GraphicsMagick instalado no sistema (1.3.16). Outra ideia é modificar a variável que controla o número de threads do OPENMP (`env OMP_NUM_THREADS`), seguindo a dica do post da etsy. As flags de compilação passadas para o GCC para otimização foram:

`
-m64 -mtune=generic -march=x86-64 -mfpmath=sse -O2 -funroll-loops -fschedule-insns
`

As variações de processamento para a comparação foram as seguintes:

1. Processamento com o gm padrão (1.3.16)
1. Processamento com o gm padrão (1.3.16) desligando o OPENMP em tempo de execução (`env OMP_NUM_THREADS=1`)
1. Processamento com a última versão do gm (1.3.23)
1. Processamento com a última versão do gm (1.3.23) desligando o OPENMP em tempo de execução (`env OMP_NUM_THREADS=1`)
1. Processamento com a última versão do gm (1.3.23) removendo o OPENMP (`./configure --disable-openmp`)
1. Processamento com a última versão do gm (1.3.23) com flags de otimização do GCC
1. Processamento com a última versão do gm (1.3.23) com flags de otimização do GCC desligando o OPENMP em tempo de execução (`env OMP_NUM_THREADS=1`)
1. Processamento com a última versão do gm (1.3.23) com flags de otimização do GCC e removendo o OPENMP

O tempo plotado é igual ao do gráfico anterior, a média das 3 rodadas de processamento somando user + sys, por imagem. O comando rodado foi usando o filter box:

{% highlight bash %}
time for file in *.jpg; do
  gm convert $file -filter Box -geometry 300x thumbs/${file/.jpg/.png};
done
{% endhighlight %}


<iframe src="/images/2016-01-12-otimizando-a-geracao-de-imagens-parte-1/filterbox.html" width="100%" height="340" seamless frameBorder="0" scrolling="no"></iframe>


O gráfico mostra claramente que o vilão do tempo de processamento é o OPENMP. O paralelismo oferecido pelo OPENMP na geração de uma imagem menor causa mais estrago do que ajuda, já que não tem muito o que paralelizar. Os parâmetros de otimização dão um ganho de cerca de 15% no tempo de processamento.

### Downsampling

Mas mesmo o melhor tempo de 359ms por imagem é muito alto. Comecei então a procurar por alternativas, continuando a investir no GraphicsMagick encontrei [esse e-mail do
Bob Friesenhahn](http://graphicsmagick-help.narkive.com/r29TpuIa/gm-help-best-performance-for-resizing#post3) (mantenedor do GraphicsMagick). Ele diz algo que eu não conhecia. Basicamente o decodificador do JPEG, pela forma com que o formato é construído permite que leia-se apenas um pedaço do arquivo, decodificando uma resolução menor da imagem. Isso economiza tempo de leitura de disco e de processamento na geração de imagens de resolução menor.

Para fazer esse downsampling do JPEG com o GraphicsMagick basta passar o tamanho da imagem com o parâmetro `-size` antes do nome do arquivo origem. Para mostrar o impacto desse parâmetro, fiz o gráfico abaixo variando o size:


<iframe src="/images/2016-01-12-otimizando-a-geracao-de-imagens-parte-1/downsampling.html" width="100%" height="340" seamless frameBorder="0" scrolling="no"></iframe>

Passando 2500 como parâmetro faz um upsampling em vez de downsampling, quase triplicando o tempo de processamento. 2000 e 1500 de parâmetro não fizeram diferença, empataram com o tempo sem parâmetro (359ms). A partir do parâmetro 1000, o tempo cai pela metade. Com 500 o tempo é quatro vezes menor que o tempo base.

Mas e a qualidade? Será que existe uma perda muito grande com esse downsampling? Abaixo está um thumbnail gerado com cada um dos parâmetros passados (clique para ver a imagem com 300 pixels):


<a href="/images/2016-01-12-otimizando-a-geracao-de-imagens-parte-1/2500.png">
	<img src="/images/2016-01-12-otimizando-a-geracao-de-imagens-parte-1/2500.png" alt="-size 2500" title="-size 2500" width="140px" />
</a>
<a href="/images/2016-01-12-otimizando-a-geracao-de-imagens-parte-1/2000.png">
	<img src="/images/2016-01-12-otimizando-a-geracao-de-imagens-parte-1/2000.png" alt="-size 2000" title="-size 2000" width="140px" />
</a>
<a href="/images/2016-01-12-otimizando-a-geracao-de-imagens-parte-1/1500.png">
	<img src="/images/2016-01-12-otimizando-a-geracao-de-imagens-parte-1/1500.png" alt="-size 1500" title="-size 1500" width="140px" />
</a>
<a href="/images/2016-01-12-otimizando-a-geracao-de-imagens-parte-1/1000.png">
	<img src="/images/2016-01-12-otimizando-a-geracao-de-imagens-parte-1/1000.png" alt="-size 1000" title="-size 1000" width="140px" />
</a>
<a href="/images/2016-01-12-otimizando-a-geracao-de-imagens-parte-1/500.png">
	<img src="/images/2016-01-12-otimizando-a-geracao-de-imagens-parte-1/500.png"  alt="-size 500"  title="-size 500"  width="140px" />
</a>


Não existe uma diferença visível de qualidade da imagem e com o aumento na velocidade de quatro vezes essa é com certeza a melhor maneira de gerar thumbnails.


### Conclusão

É importante entender bem como o software que a gente trabalha funciona, principalmente quando é algo vital para o produto. Com alguma pesquisa e testando variações de parâmetros consegui diminuir em 15 vezes o tempo de geração de thumbnail do sistema.

Mas para qualquer comparativo ser válido é necessário seguir um método cientifico, ser capaz de reproduzir os mesmos resultados nas mesmas condições da forma mais simples possível. De preferencia montar um script que rode sempre tudo o que precisa, apresentando o resultado final já calculado. Sobra pra você apenas o trabalho mais difícil, pensar.