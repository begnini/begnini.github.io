---
title: "Quebrando outro captcha com OpenCV e Python"
layout: post
date:   2016-02-08 17:00:00
---


Olá,

esse é o segundo post sobre quebra de captchas do blog ([primeiro post aqui](http://www.begnini.net/2015/12/30/quebrando-captcha-com-opencv-e-python.html)). No primeiro post eu dei uma introdução a CAPTCHA e descrevi um passo a passo para quebrar um. Nesse, eu escolhi um captcha um pouco mais complexo e utilizar o máximo possível de métodos do OpenCV para manipular a imagem.

Abaixo alguns exemplos do captcha a serem analisados:


<a href="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/captcha1.png">
    <img src="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/captcha1.png" alt="Exemplo de captcha" title="Exemplo de captcha" width="240px" />
</a>
<a href="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/captcha2.png">
    <img src="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/captcha2.png" alt="Exemplo de captcha" title="Exemplo de captcha" width="240px" />
</a>
<a href="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/captcha3.png">
    <img src="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/captcha3.png" alt="Exemplo de captcha" title="Exemplo de captcha" width="240px" />
</a>

<a href="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/captcha4.png">
    <img src="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/captcha4.png" alt="Exemplo de captcha" title="Exemplo de captcha" width="240px" />
</a>
<a href="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/captcha5.png">
    <img src="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/captcha5.png" alt="Exemplo de captcha" title="Exemplo de captcha" width="240px" />
</a>
<a href="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/captcha6.png">
    <img src="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/captcha6.png" alt="Exemplo de captcha" title="Exemplo de captcha" width="240px" />
</a>

Como dá pra perceber, as cores de fundo são bem variadas e as letras as vezes estão rotacionadas. Além disso, a letras tem alguns riscados para atrapalhar a identificação automática.



### Remoção do Fundo

A primeira coisa que se nota é que as letras são sempre da cor branca. Isso vai ajudar na remoção do fundo colorido. Para isso vou usar uma técnica de segmentação de imagem usando o método do OpenCV [cv2.threshold](http://docs.opencv.org/2.4/modules/imgproc/doc/miscellaneous_transformations.html?highlight=threshold#threshold).

Thresholding é a técnica mais simples de segmentação de imagem que existe. A ideia dela é olhar a intensidade de cada pixel da imagem em comparação com um valor de corte (threshold) e decidir se o pixel pertence ao fundo da imagem ou ao objeto que queremos. O OpenCV tem 5 métodos de threshold:

* **THRESH_BINARY**: para cada pixel da imagem, se a intensidade for maior que o valor de corte, o pixel recebe o valor máximo (parâmetro da função), senão recebe zero;
* **THRESH_BINARY_INV**: é o contrário do **THRESH_BINARY**, se a intensidade do pixel for maior que o valor de corte, o pixel recebe zero, senão recebe o valor máximo;
* **THRESH_TRUNC**: todos os pixels com intensidade maior que o valor de corte são truncados para o valor de corte, o restante permanece inalterado;
* **THRESH_TOZERO**: todos os pixels com intensidade maior que o valor de corte ficam inalterados, enquanto os com intensidade menor são zerados;
* **THRESH_TOZERO_INV**: é o contrário do **THRESH_TOZERO**, os pixels com intensidade maior que o valor de corte são zerados, enquanto os com intensidade menor ficam inalterados.


O grande problema é encontrar o valor de corte exato que divide o que é fundo e o que é o objeto. Se usar um valor muito alto, parte do objeto pode se confundir com o fundo e se ele for muito baixo, parte do fundo pode ser reconhecido como objeto.

Existem vários algoritmos que tentam descobrir esse valor a partir da imagem, o OpenCV implementa o [algoritmo de OTSU de binarização](https://en.wikipedia.org/wiki/Otsu's_method). Para usar o algoritmo de OTSU, basta passar o parâmetro **THRESH_OTSU** combinado com algum dos parâmetros acima no método de threshold.

Abaixo um teste com uma imagem usando o método OTSU como descobridor do valor de corte para a função threshold. Vou tentar com cada um dos tipos de thresholding:

{% highlight python linenos %}
# importacao da biblioteca do opencv
import cv2

# le a imagem
image = cv2.imread('captcha.png')

# transforma ela de colorida (RGB) para tons de cinza
gray = cv2.cvtColor(image, cv2.COLOR_RGB2GRAY)


# aplica o thresholding
for thresh in [cv2.THRESH_BINARY, cv2.THRESH_BINARY_INV, cv2.THRESH_TRUNC, cv2.THRESH_TOZERO, cv2.THRESH_TOZERO_INV]:
    # o segundo parâmetro, 127, é ignorado
    (_, bw) = cv2.threshold(gray, 127, 255, thresh | cv2.THRESH_OTSU)

{% endhighlight %}

A imagem ficou assim para cada método:

<a href="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/bw_captcha_binary.png">
    <img src="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/bw_captcha_binary.png" alt="thresholding com binary" title="threshoding com binary" width="240px" style="border: 1px solid gray"/>
</a>
<a href="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/bw_captcha_binary_inv.png">
    <img src="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/bw_captcha_binary_inv.png" alt="thresholding com binary invertido" title="thresholding com binary invertido" width="240px" style="border: 1px solid gray"/>
</a>
<a href="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/bw_captcha_trunc.png">
    <img src="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/bw_captcha_trunc.png" alt="thresholding com truncagem" title="thresholding com truncagem" width="240px" style="border: 1px solid gray"/>
</a>

<a href="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/bw_captcha_tozero.png">
    <img src="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/bw_captcha_tozero.png" alt="thresholding com zero" title="thresholding com zero" width="240px" style="border: 1px solid gray"/>
</a>
<a href="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/bw_captcha_tozero_inv.png">
    <img src="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/bw_captcha_tozero_inv.png" alt="thresholding com zero invertido" title="thresholding com zero invertido" width="240px" style="border: 1px solid gray"/>
</a>


O OTSU calculou que o valor de corte para essa imagem é 136, um pouco além da metade do espectro (0 a 255). Apesar de ter ficado com algum ruido, a limpeza foi boa o suficiente para destacar as letras.


### Heurísticas

[Heurística](https://en.wikipedia.org/wiki/Heuristic_%28computer_science%29) é a ideia de que muitas vezes a solução ideal é muito difícil de achar ou custa muito caro computacionalmente e outras soluções podem dar uma resultado quase tão bom quanto a solução ideal com um nível de dificuldade bem mais baixo.

Por exemplo, é possível reparar que nos captchas o posicionamento das letras em relação a imagem segue um padrão. As letras sempre estão alinhadas do lado esquerdo da imagem, com uma margem de 30 pixels do canto esquerdo. Com isso, dá pra jogar fora os 30 primeiros pixels da esquerda e pegar os 150 pixels próximos. Para recortar apenas essa fatia das imagens usando OpenCV é muito simples. Como elas são vetores numpy, basta pegar um pedaço do vetor (slice) como no python:


{% highlight python linenos %}
# como é uma matrix, a primeira parte antes da virgula são as linhas
# e a segunda parte são as colunas
crop = image[:,30:180]
{% endhighlight %}


As imagens binarizadas e recortadas ficaram assim:

<a href="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/bw_captcha1_crop.png">
    <img src="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/bw_captcha1_crop.png" alt="Captcha binarizado e recortado" title="Captcha binarizado e recortado" />
</a>
<a href="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/bw_captcha2_crop.png">
    <img src="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/bw_captcha2_crop.png" alt="Captcha binarizado e recortado" title="Captcha binarizado e recortado" />
</a>
<a href="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/bw_captcha3_crop.png">
    <img src="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/bw_captcha3_crop.png" alt="Captcha binarizado e recortado" title="Captcha binarizado e recortado" />
</a>

<a href="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/bw_captcha4_crop.png">
    <img src="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/bw_captcha4_crop.png" alt="Captcha binarizado e recortado" title="Captcha binarizado e recortado"  />
</a>
<a href="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/bw_captcha5_crop.png">
    <img src="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/bw_captcha5_crop.png" alt="Captcha binarizado e recortado" title="Captcha binarizado e recortado"  />
</a>
<a href="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/bw_captcha6_crop.png">
    <img src="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/bw_captcha6_crop.png" alt="Captcha binarizado e recortado" title="Captcha binarizado e recortado" />
</a>

Como é uma heurística, o recorte não é perfeito. Mas remove boa parte da sujeira deixada pela binarização, melhorando a qualidade da imagem.

Outra coisa interessante é que como a imagem é menor, o OTSU é menos influenciado pela sujeira do fundo. Então o ideal é recortar ela primeiro e depois de binarizar, eliminando mais sujeira do lado direito. Abaixo estão as imagens recortadas primeiro e depois binarizadas:

<a href="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/bw_captcha1_crop2.png">
    <img src="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/bw_captcha1_crop2.png" alt="Captcha binarizado e recortado" title="Captcha binarizado e recortado" />
</a>
<a href="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/bw_captcha2_crop2.png">
    <img src="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/bw_captcha2_crop2.png" alt="Captcha binarizado e recortado" title="Captcha binarizado e recortado" />
</a>
<a href="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/bw_captcha3_crop2.png">
    <img src="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/bw_captcha3_crop2.png" alt="Captcha binarizado e recortado" title="Captcha binarizado e recortado" />
</a>

<a href="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/bw_captcha4_crop2.png">
    <img src="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/bw_captcha4_crop2.png" alt="Captcha binarizado e recortado" title="Captcha binarizado e recortado"  />
</a>
<a href="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/bw_captcha5_crop2.png">
    <img src="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/bw_captcha5_crop2.png" alt="Captcha binarizado e recortado" title="Captcha binarizado e recortado"  />
</a>
<a href="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/bw_captcha6_crop2.png">
    <img src="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/bw_captcha6_crop2.png" alt="Captcha binarizado e recortado" title="Captcha binarizado e recortado" />
</a>



### Segmentação

Depois da imagem limpa, a próxima etapa é encontrar os caracteres e separá-los. Assim como no post passado, a ideia é calcular os componentes conectados e criar um bounding box (retângulo em volta) deles. Existe um método que faz exatamente isso implementado na biblioteca skimage, chamado [label](http://scikit-image.org/docs/dev/api/skimage.measure.html#skimage.measure.label). Esse método enumera todos os componentes conectados de uma imagem.

{% highlight python linenos %}
import skimage.measure
import skimage.color

# encontra os componentes conectados
(labels, total) = skimage.measure.label(blackwhite, background=0, return_num=True, connectivity=2)

# pega os componentes com mais de 20 pixels
images = [numpy.uint8(labels==i) * 255 for i in range(total) if numpy.uint8(labels==i).sum() > 20]

# gera uma imagem com cada componente pintado de um cor diferente
img = skimage.color.label2rgb(labels, bg_color=[1, 1, 1])

# pinta retangulos em volta de cada componente
color = (1.0, 0.0, 0.0)

for label in images:
    # encontra os contornos para cada componente
    (countours, _) = cv2.findContours(label, cv2.RETR_TREE, cv2.CHAIN_APPROX_SIMPLE)

    # calcula o retangulo em volta dos contornos
    (x,y,w,h)      = cv2.boundingRect(countours[0])

    # e pinta ele
    cv2.rectangle(img, (x, y), (x+w, y+h), color, 1)

{% endhighlight %}

Apenas um detalhe sobre esse código, a imagem gerada não varia mais entre 0 e 255 cada cor como é costume trabalhar. A partir da chamada do método [skimage.color.label2rbg](http://scikit-image.org/docs/dev/api/skimage.color.html#skimage.color.label2rgb), o código trabalha com a intensidade de vermelho, verde e azul entre 0.0 e 1.0, só notar que a cor do retângulo é uma tupla de floats entre 0.0 e 1.0.

Esse detalhe é importante porque antes de salvar a imagem, ela deve ser convertida novamente para inteiros variando entre 0 e 255. Caso você salve ela do jeito que está, vai ver só preto na imagem salva. O código abaixo faz essa conversão:

{% highlight python linenos %}
# multiplica cada posicao,
# fazendo variar cada intensidade entre 0.0 a 255.0
img2 = img * 255

# transforma em inteiro
img2 = img2.astype(numpy.uint8)

{% endhighlight %}


O resultado é essa bagunça toda ae abaixo:

<a href="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/bw_captcha1_label.png">
    <img src="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/bw_captcha1_label.png" alt="Captcha binarizado e recortado" title="Captcha binarizado e recortado" style="width: 240px; border: 1px solid gray"/>
</a>
<a href="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/bw_captcha2_label.png">
    <img src="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/bw_captcha2_label.png" alt="Captcha binarizado e recortado" title="Captcha binarizado e recortado" style="width: 240px; border: 1px solid gray"/>
</a>
<a href="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/bw_captcha3_label.png">
    <img src="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/bw_captcha3_label.png" alt="Captcha binarizado e recortado" title="Captcha binarizado e recortado" style="width: 240px; border: 1px solid gray"/>
</a>

<a href="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/bw_captcha4_label.png">
    <img src="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/bw_captcha4_label.png" alt="Captcha binarizado e recortado" title="Captcha binarizado e recortado"  style="width: 240px; border: 1px solid gray"/>
</a>
<a href="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/bw_captcha5_label.png">
    <img src="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/bw_captcha5_label.png" alt="Captcha binarizado e recortado" title="Captcha binarizado e recortado"  style="width: 240px; border: 1px solid gray"/>
</a>
<a href="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/bw_captcha6_label.png">
    <img src="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/bw_captcha6_label.png" alt="Captcha binarizado e recortado" title="Captcha binarizado e recortado" style="width: 240px; border: 1px solid gray"/>
</a>

Agora sei a posição de cada caractere nas imagens. Então é só dar fazer uma limpeza geral nessas imagens, eliminando essas sujeirinhas que sobraram.


### Limpeza

Pra ficar bonito o resultado, vou aplicar os labels que foram encontrados acima nas imagens binarizadas e o que tiver fora deles pintar como fundo. Para isso é só pegar os labels encontrados e somá-los.

{% highlight python linenos %}

# soma todos os labels encontrados
cleaned = sum(images)

{% endhighlight %}

O resultado são imagens mais limpas, prontas para serem reconhecidas.


<a href="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/bw_captcha1_cleaned.png">
    <img src="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/bw_captcha1_cleaned.png" alt="Captcha binarizado e recortado" title="Captcha binarizado e recortado" style="width: 240px; border: 1px solid gray"/>
</a>
<a href="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/bw_captcha2_cleaned.png">
    <img src="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/bw_captcha2_cleaned.png" alt="Captcha binarizado e recortado" title="Captcha binarizado e recortado" style="width: 240px; border: 1px solid gray"/>
</a>
<a href="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/bw_captcha3_cleaned.png">
    <img src="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/bw_captcha3_cleaned.png" alt="Captcha binarizado e recortado" title="Captcha binarizado e recortado" style="width: 240px; border: 1px solid gray"/>
</a>

<a href="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/bw_captcha4_cleaned.png">
    <img src="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/bw_captcha4_cleaned.png" alt="Captcha binarizado e recortado" title="Captcha binarizado e recortado"  style="width: 240px; border: 1px solid gray"/>
</a>
<a href="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/bw_captcha5_cleaned.png">
    <img src="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/bw_captcha5_cleaned.png" alt="Captcha binarizado e recortado" title="Captcha binarizado e recortado"  style="width: 240px; border: 1px solid gray"/>
</a>
<a href="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/bw_captcha6_cleaned.png">
    <img src="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/bw_captcha6_cleaned.png" alt="Captcha binarizado e recortado" title="Captcha binarizado e recortado" style="width: 240px; border: 1px solid gray"/>
</a>

## Reconhecimento

Agora que todos os caracteres estão separados e limpos que começa realmente o trabalho.

Para estudar o padrão das letras, baixei mil imagens diferentes e rodei a limpeza e recorte dos caracteres. Lembrando que até agora estava trabalhando com imagens controladas e tudo funcionou perfeitamente bem, mas nem tudo está sob controle ainda. Nessas mil imagens tem vários problemas que ainda não foram resolvidos.

O código abaixo vai fazer o recorte das letras das mil imagens, salvar num diretório e gerar algumas estatísticas para analisar a qualidade do processamento.

{% highlight python linenos %}
def simple_cleanup(file):
    bw = cut_and_binarize(file)

    (labels, total) = skimage.measure.label(bw, background=0, return_num=True, connectivity=2)
    images = [numpy.uint8(labels==i) * 255 for i in range(total) if numpy.uint8(labels==i).sum() > 25]

    cropped = []
    dimensions   = []

    for label in images:
        (countours, _) = cv2.findContours(label.copy(), cv2.RETR_TREE, cv2.CHAIN_APPROX_SIMPLE)
        (x,y,w,h)      = cv2.boundingRect(countours[0])

        # area do bounding box
        area    = w * h

        # quantidade de pixels do caracter
        surface = label.sum() * 255

        label = label[y:y+h, x:x+w]

        cropped.append(label)
        dimensions.append((x,y,w,h,area,surface))

    return (cropped, dimensions)


i = 1

stats = []

for file in glob.glob('images/*.png'):
    (cropped, dimensions) = simple_cleanup(file)
    stats.extend(dimensions)

    # salva cada imagem recortada na pasta crop
    for crop in cropped:
        cv2.imwrite('crop/%04d.png' % (i), crop)
        i = i + 1
{% endhighlight  %}

Vou gerar os histogramas das dimensões das letras das mil imagens.

{% highlight python linenos %}

# desenha os histogramas
def histogram(points):
    hist, bins = numpy.histogram(points, points.max() - points.min() + 1)

    plt.bar(bins[:-1], hist, width = 1)
    plt.xlim(min(bins), max(bins))
    plt.show()


# arrays com todos os x iniciais
xs = numpy.array([s[0] for s in stats])

# arrays com todos os y iniciais
ys = numpy.array([s[1] for s in stats])

# arrays com todas as larguras
widths = numpy.array([s[2] for s in stats])

# arrays com todas as alturas
heights = numpy.array([s[3] for s in stats])

# array com as areas dos bounding boxes
areas = numpy.array([s[4] for s in stats])

# array com as quantidade de pixels de cada letra
surfaces = numpy.array([s[5] for s in stats])

{% endhighlight  %}


<a href="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/histogram_xs.png">
    <img src="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/histogram_xs.png" alt="Histograma de X inicial" title="Histograma de X inicial" style="width: 360px"/>
</a>
<a href="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/histogram_ys.png">
    <img src="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/histogram_ys.png"  alt="Histograma de Y inicial" title="Histograma de Y inicial" style="width: 360px"/>
</a>

<a href="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/histogram_widths.png">
    <img src="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/histogram_widths.png"  alt="Histograma das larguras" title="Histograma das larguras" style="width: 360px"/>
</a>
<a href="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/histogram_heights.png">
    <img src="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/histogram_heights.png" alt="Histograma das alturas" title="Histograma das alturas"  style="width: 360px"/>
</a>

<a href="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/histogram_areas.png">
    <img src="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/histogram_areas.png"  alt="Histograma das larguras" title="Histograma das larguras" style="width: 360px"/>
</a>
<a href="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/histogram_surfaces.png">
    <img src="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/histogram_surfaces.png" alt="Histograma das alturas" title="Histograma das alturas"  style="width: 360px"/>
</a>

### Encontrando outliers

As estatísticas dos gráficos acima estão na tabela abaixo:

| Dimensão       | Mediana | Média   | Desvio Padrão | Mínimo | Máximo | Tamanho |
| ---            | ---:    | ---:    | ---:          | ---:   | ---:   | ---:    |
| **X**          | 44.0    | 46.72   | 38.92         | 1      | 140    | 3755    |
| **Y**          | 10.0    | 9.57    | 7.07          | 1      | 42     | 3755    |
| **Largura**    | 25.0    | 32.02   | 23.78         | 3      | 148    | 3755    |
| **Altura**     | 28.0    | 28.25   | 9.60          | 3      | 44     | 3755    |
| **Área**       | 702.0   | 1041.29 | 1118.35       | 21     | 6512   | 3755    |
| **Superfície** | 258.0   | 470.16  | 710.06        | 26     | 4505   | 3755    |


Primeiramente olhando pro tamanho final, dá pra perceber que algo foi perdido. Deveriam ter quatro mil imagens (quatro letras em cada uma das mil imagens) e tem apenas 3.755. Essas letras provavelmente grudaram-se com outras. Olhando as estatísticas de **X** e da **Largura** dos bounding boxes dá pra ver bem isso.

Explicando melhor, não tem como ter uma letra com largura de 148 pixels se a imagem inteira tem 150 pixels. As letras mais largas (W e M) tem pouco mais de 30 pixels de largura. Outra coisa que dá pra perceber é a distância entre as letras, o desvio padrão de **X** diz isso.

Estatisticamente é possível ver os dados com problema olhando a diferença entre a mediana e a média de cada valor. **X**, **Y** e a **Altura** estão com a média e a mediana bem próximos, enquanto a **Largura**, a **Área** e a **Superfície** estão com uma diferença tremenda entre a mediana e a média.

Para encontrar esse pessoal problemático vamos usar uma técnica chamada [Interquatile Range](https://en.wikipedia.org/wiki/Interquartile_range) que irá dizer os intervalos do que é aceitável e do que é outlier. A técnica consiste em dividir o conjunto de dados em quatro partes iguais usando 3 números como divisores:


* **Q1** - Os valores menores que Q1 correspondem a 25% do conjunto
* **Q2** - Os valores entre Q1 e Q2 correspondem a outros 25% do conjunto. Q2 tambêm é conhecido como a mediana
* **Q3** - Os valores entre Q2 e Q3 correspondem a outros 25% do conjunto. E os valores maiores que Q3 são os 25% restante.


O Interquatile Range (IQR) é a diferença entre Q3 e Q1. Essa distância mostra o quanto a metade central dos dados estão espalhados e a partir desse valor que se calcula os outliers.

[John Tukey](https://en.wikipedia.org/wiki/John_Tukey), que inventou essa forma de análise, definiu que os outliers devem ficar entre Q1 - (1.5 * IQR) e Q3 + (1.5 * IQR).

O seguinte código calcula os intervalos que não considerados outliers usando o método proposto por John Tukey:

{% highlight python linenos %}

def compute_outliers(data):
    q1 = numpy.percentile(data, 25)
    q2 = numpy.percentile(data, 50)
    q3 = numpy.percentile(data, 75)
    iqr = q3 - q1

    lower_bound = max(data.min(), q1 - (1.5 * iqr))
    upper_bound = min(data.max(), q3 + (1.5 * iqr))

    return (q1, q2, q3, iqr, lower_bound, upper_bound)

{% endhighlight %}

Abaixo estão os resultados para cada uma das características:

| Dimensão       | Q1   | Q2  | Q3     | IQR   | Mínimo | Máximo  |
| ---            | ---: | ---:| ---:   | ---:  | ---:   | ---:    |
| **X**          | 2    | 44  | 77.0   | 75.0  | 1      | 140.00  |
| **Y**          | 5    | 10  | 11.0   | 6.0   | 1      | 20.00   |
| **Largura**    | 21   | 25  | 31.0   | 10.0  | 6      | 46.00   |
| **Altura**     | 24   | 28  | 34.0   | 10.0  | 9      | 44.00   |
| **Área**       | 560  | 702 | 1050.0 | 490.0 | 21     | 1785.00 |
| **Superfície** | 170  | 258 | 392.5  | 222.5 | 26     | 726.25  |


Analisando um por um dos números:

* **X** - O menor e o maior valor cobre todo o range de X então ele não tem nenhum outlier
* **Y** - Qualquer bounding box que esteja acima do pixel 20 é problemático
* **Largura** - Caracteres com largura menor que 6 ou maior que 46 tem algum problema
* **Altura**  - Caracteres com altura menor que 9 pixels é problemático
* **Area**    - Caracteres com area menor que 21 pixels e maior que 1785 é problemático
* **Superfície** - Caracteres com superfície maior que 726 pixels também tem algum problema

Algumas dessas coisas são obvias. Como falei ali em cima, o maior caractere tem em torno de 30 pixels de largura, o interquatile range  disse que qualquer com mais de 46 de largura está quebrado, o que não deixar de estar certo. A altura menor de 9 pixels também.

Mas o interessante de tudo isso é que esses valores vieram automaticamente, sem precisar de nenhuma análise manual. Isso pode ser usado para encontrar casos extremos num banco de dados, por exemplo, ou tentativas de fraude numa aplicação.

Voltando ao problema, desenhei os boxplots de cada uma das características para ficar mais fácil visualizar os outliers:


<a href="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/boxplot_xs.png">
    <img src="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/boxplot_xs.png" alt="boxplot de X inicial" title="boxplot de X inicial" style="width: 360px"/>
</a>
<a href="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/boxplot_ys.png">
    <img src="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/boxplot_ys.png"  alt="boxplot de Y inicial" title="boxplot de Y inicial" style="width: 360px"/>
</a>

<a href="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/boxplot_widths.png">
    <img src="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/boxplot_widths.png"  alt="boxplot das larguras" title="boxplot das larguras" style="width: 360px"/>
</a>
<a href="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/boxplot_heights.png">
    <img src="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/boxplot_heights.png" alt="boxplot das alturas" title="boxplot das alturas"  style="width: 360px"/>
</a>

<a href="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/boxplot_areas.png">
    <img src="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/boxplot_areas.png"  alt="boxplot das areas" title="boxplot das areas" style="width: 360px"/>
</a>
<a href="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/boxplot_surfaces.png">
    <img src="/images/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/boxplot_surfaces.png" alt="boxplot das superfícies" title="boxplot das superfícies"  style="width: 360px"/>
</a>



Agora, filtrando os dados e removendo os outliers, as estatísticas ficaram assim:



| Dimensão       | Média     | Desvio Padrão | Mínimo | Máximo | Tamanho |
| ---            | --------: | ---:          | ---:   | ---:   | ---:    |
| **X**          |  42.23    | 31.42         |  1     | 129    | 2692
| **Y**          |  10.08    | 4.17          |  1     | 19     | 2692
| **Largura**    |  25.68    | 5.69          |  8     | 45     | 2692
| **Altura**     |  28.17    | 4.79          | 10     | 44     | 2692
| **Área**       |  732.09   | 236.96        | 80     | 1716   | 2692
| **Superfície** |  261.81   | 102.91        | 36     | 721    | 2692


Como esperado, o desvio padrão de todo mundo caiu, mas infelizmente mais dados foram jogados fora. Agora só tem 2692 letras (67% do total) para classificar.


### Normalização

Para comparar as imagens, primeiro é necessário que elas tenham as mesmas características. Nesse caso, como elas já estão em preto e branco, a única variação entre elas é o tamanho. Pra isso, vou transformar todas as imagens em um tamanho padrão de 64x64, mantendo a razão de aspecto e centralizando a imagem antiga na imagem nova.

Abaixo o método de normalização:

{% highlight python linenos %}

def normalize(image):
    (w, h) = image.shape

    fx = w / 64.0
    fy = h / 64.0

    f = max(fx, fy)

    w_ = int(w / f)
    h_ = int(h / f)

    resized = cv2.resize(image, (w_, h_))

    box = numpy.zeros((64, 64), dtype=numpy.uint8)

    x0  = (64 - w_) / 2
    y0  = (64 - h_) / 2

    box[y0:y0 + h_, x0:x0 + w_] = resized

    return box

{% endhighlight %}


Primeiro é feito o cálculo do fator de variação da altura e da largura em relação ao tamanho que eu defini (linhas 4 e 5), depois é pego a maior variação e baseada nela (linha 7) é calculada a nova altura e a nova largura (linhas 9 e 10). Após isso,  a imagem é redimensionada para ter o novo tamanho (linha 12). Nesse momento eu sei que uma das dimensões (largura ou altura) tem 64 pixels e a outra é menor ou igual a 64 pixels.

Agora é criar uma imagem 64x64 composta de zeros (linha 14) e copiar a imagem redimensionada para ela, calculando a posição inicial da imagem de modo que ela fique centralizada (linhas 16, 17 e 19).

Agora que todas as imagens estão normalizadas, é hora de agrupá-las.

### Agrupamento

Para agrupar as imagens similares primeiro é preciso ter uma métrica de similaridade. Nesse caso vou usar a [Mean Squared Error](https://en.wikipedia.org/wiki/Mean_squared_error) como métrica de similaridade entre as imagens. Basicamente a MSE calcula para cada pixel de uma imagem a diferença entre ele e o pixel da outra imagem na mesma posição, eleva essa diferença ao quadrado e depois soma todos os resultados. Essa soma é então dividida pelo tamanho da imagem.

O código abaixo mostra como fazer isso usando numpy. Nele dividimos o resultado por 255.0 para que os pixels tenham tamanho entre 0 e 1.0:

{% highlight python linenos %}

def mse(image, image2):
    diff  = (image.astype(numpy.float) - image2.astype(numpy.float)) / 255.0
    dist  = numpy.sum(diff ** 2)
    dist /= float(image.shape[0] * image.shape[1])

    return dist

{% endhighlight %}

Esse código vai retornar a distância, ou melhor, a diferença entre as 2 imagens. Caso as imagens sejam exatamente iguais, o método irá retornar 0.0 e caso elas sejam completamente diferentes ele irá retornar 1.0. Mas surge um problema, qual é a margem de erro aceitável? Como escolher uma distância em que 2 caracteres diferentes não sejam confundidos?

Primeiro o código de agrupamento:

{% highlight python linenos %}

def group(limiar):
    groups = collections.defaultdict(list)
    computed = set()

    for (i, image) in enumerate(normalized):
        if i in computed:
            continue

        found = False

        for (j, image2) in enumerate(normalized):

            if i >= j or j in computed:
                continue

            dist = mse(image, image2)

            if dist < limiar:
                groups[i].append(j)
                found = True
                computed.add(j)

        if not found:
            groups[i].append(i)

    return groups

{% endhighlight %}

O que ele faz basicamente é força-bruta, comparando cada imagem com todas as outras e calculando a distância entre elas. Se a distância entre as imagens for menor que um limiar definido, ele adiciona essa imagem a um grupo e remove ela do conjunto de imagens computaveis.

Testando com o limiar de 0.05 nos 2.692 recortes, é gerado 347 grupos. Um bocado alto, já que tem apenas 36 caracteres possiveis.

Primeiro vou salvar os resultados num diretório e ver como estão os grupos:

{% highlight python linenos %}

def save(directory, groups, normalized):
    for i in groups.keys():
        if len(groups[i]) < 3:
            continue

        subdirectory = os.path.join(directory, '%04d' % (i))

        if not os.path.isdir(subdirectory):
            os.makedirs(subdirectory)

        file = os.path.join(subdirectory, '%04d.png' % (i))
        cv2.imwrite(file, normalized[i])

        for j in groups[i]:
            dist = mse(normalized[i], normalized[j])

            file = os.path.join(subdirectory, '%04d_%0.3f.png' % (j, dist))
            cv2.imwrite(file, normalized[j])

{% endhighlight %}

O código acima cria um subdiretório para cada grupo e salva as imagens do grupo nesse subdiretório, colocando no nome a distância entre a imagem principal e a imagem agrupada. Se o grupo tiver menos de 3 imagens, ele é ignorado, um pequeno truque para remover o lixo.

Olhando as imagens no subdiretório dá pra perceber que elas estão bem separadas, não houve mistura de caracteres. Mas muitos caracteres que deveriam estar juntos ficaram separados, sinal de que o limiar está muito baixo. Para ajustar isso, fiz agrupamento com 0.06, 0.07, 0.08, 0.09, 0.1 e 0.15 de limiar.

Agrupando até 0.1, o resultado só foi melhorando. Mas em 0.15 o resultado foi desastroso, vários recortes com caracteres completamente diferentes começaram a ser agrupados, por isso o limiar de 0.1 foi o escolhido final.


### Classificação

Essa parte é completamente manual. Basta pegar os arquivos que estavam no diretório formado pelo agrupamento e dar o nome da letra para eles. Os grupos diferentes que tiverem letra igual devem ser juntados e o que for lixo (caracteres grudados, por exemplo) descartados.

Dos 76 diretórios gerados pelo método de agrupamento, sobraram 28. Mas deveriam haver 36, não? 10 números e 26 letras. Acontece que, ao que parece, nenhuma vogal (A, E, I, O, U e Y) participa do conjunto, pelo menos nenhuma apareceu no conjunto de mil captchas. Além disso, tanto o número 0 quanto a letra Q não apareceram tambem, provavelmente porque deve confundir muito o usuário na hora de digitar.


### Detecção

Mas apesar da redução na quantidade de caracteres, a chance de acertar chutando esse captcha ainda é muito grande (28^4 ou 1 a cada 614.656 tentativas). Mas agora que  os caracteres foram classificados, basta procurar eles nas imagens e imprimir o resultado do que encontrou.

Para isso vou usar o método do OpenCV [matchTemplate](http://docs.opencv.org/2.4/modules/imgproc/doc/object_detection.html?highlight=matchtemplate#matchtemplate). Esse método faz a busca de uma imagem dentro de outra através de [convolução](https://en.wikipedia.org/wiki/Kernel_(image_processing)#Convolution), basicamente comparacando cada pixel da imagem com cada pixel dos recortes. Ela retorna uma matriz com o cálculo do resultado para todas as posições da imagem.  Dependendo do tipo de método usado para a comparação, o melhor valor pode ser o valor mínimo global ou o valor máximo global da matriz (minimizar o erro ou maximizar o ganho).

Os métodos de cálculo da `cv2.matchTemplate` são:

* **TM_SQDIFF**: Calcula a soma do quadrado da diferença entre os pixels, bem parecido com o MSE que discutimos acima. O casamento perfeito entre duas imagens é zero, enquanto um casamento ruim tende a um número bem grande;
* **TM_SQDIFF_NORMED**: Calcula a soma do quadrado da diferenca entre os pixels, como o TM_SQDIFF, mas divide o resultado de forma com que ele fique entre 0.0 para um casamento perfeito e 1.0 para quando a imagem for completamente diferente;
* **TM_CCORR**: Calcula a [correlação](https://en.wikipedia.org/wiki/Cross-correlation) entre os pixels do template e da imagem, multiplicando os pixels de uma imagem com outra e somando os resultados;
* **TM_CCORR_NORMED**: É a versão [normalizada](https://en.wikipedia.org/wiki/Cross-correlation#Normalized_cross-correlation) da TM_CCORR;
* **TM_CCOEFF**: Calcula o coeficiente de correlação entre o template e a imagem. Faz o casamento de uma image, relativa a sua média, com a outra imagem, tambêm relativa a sua média;
* **TM_CCOEFF_NORMED**: É a versão normalizada do coeficiente de correlação.

Os métodos normalizados são interessantes quando se tem imagens de tamanhos ou brilhos diferentes, mas quer que elas tenham o mesmo peso na comparação;

O código abaixo busca os templates recortados e classificados no captcha a ser quebrado:

{% highlight python linenos %}

# Busca o melhor template p/ uma letra
def search_for_letter(image, letter, templates, method):
    best = 2 ** 32

    # p/ SQDIFF o menor valor é o melhor, p/ os outros
    # o maior valor é o melhor
    if method not in [cv2.TM_SQDIFF, cv2.TM_SQDIFF_NORMED]:
        best = 0.0

    pos  = None

    for template in templates:
        # busca o template na imagem, usando o metodo passado
        match = cv2.matchTemplate(image, template, method)

        # encontra a pontuacao e a localizacao do template
        minVal,maxVal,minLoc,maxLoc = cv2.minMaxLoc(match)

        # p/ SQDIFF o menor valor é o melhor, p/ os outros
        # o maior valor é o melhor
        if method not in [cv2.TM_SQDIFF, cv2.TM_SQDIFF_NORMED]:

            if best < maxVal:
                pos = {
                    'error': maxVal,
                    'location': maxLoc,
                    'letter': letter
                }
                best = maxVal
        else:

            if best > minVal:
                pos = {
                    'error': minVal,
                    'location': minLoc,
                    'letter': letter
                }

                best = minVal

    return pos


{% endhighlight %}

{% highlight python linenos %}

# itera em cima de todas as letras para achar
# o melhor resultado
def search(file, templates, method):
    matches = []

    image = cut_and_binarize(file)

    for letter in templates:
        pos = search_for_letter(image, letter, templates[letter], method)

        if pos is not None:
            matches.append(pos)

    reverse = False

    # p/ SQDIFF o menor valor é o melhor, p/ os outros
    # o maior valor é o melhor
    if method not in [cv2.TM_SQDIFF, cv2.TM_SQDIFF_NORMED]:
        reverse = True

    # ordena os melhores casamentos
    matches = sorted(matches, key=lambda x:x['error'],reverse=reverse)

    # pega os 4 melhores casamentos e ordena em X
    return sorted(matches[:4], key=lambda x:x['location'][0])

{% endhighlight %}

Esse método recebe por parâmetro o nome do arquivo que vai ser quebrado, os templates carregados (um dicionário onde cada chave é uma letra e o valor é uma lista de imagens que encontramos para aquela letra) e o método de comparação (da listagem acima) entre o template e a imagem.

Para testar, peguei 100 captchas, nomeei cada arquivo com as letras do captcha e rodei a busca para testar o resultado de cada método de comparação da `cv2.matchTemplate`:


{% highlight python linenos %}

def validation(directory, templates, method):
    captchas = glob.glob(directory + '/*.png')

    corrects = 0

    for file in captchas:
        # tenta quebrar o captcha
        matches = search(file, templates, method)
        letters = [match['letter'] for match in matches]

        # testa se o captcha casa com o nome do arquivo
        filename = os.path.basename(file)
        captcha  = filename.replace('.png', '')
        captcha  = captcha.upper()

        correct = 0

        for letter in letters:
            if letter in captcha:
                correct += 1

        if correct != 4:
            print 'Error', file, correct, letters

        corrects += correct

    print "Success:", corrects, "total:", (4 * len(captchas)), "(", corrects / (400.0 * len(captchas)), "%)"

{% endhighlight %}

Rodando para cada um dos métodos de comparação, o resultado foi o seguinte:


| Método                | Letras certas  | Captchas certos |
| ---                   | --------:      | ---:            |
| **TM_SQDIFF**         | 350 (87.5%)    | 64 (64%)        |
| **TM_SQDIFF_NORMED**  | 378 (94.5%)    | 81 (81%)        |
| **TM_CCORR**          | 140 (35%)      | 1 (1%)          |
| **TM_CCORR_NORMED**   | 381 (95.25%)   | 82 (82%)        |
| **TM_CCOEFF**         | 324 (81%)      | 39 (39%)        |
| **TM_CCOEFF_NORMED**  | **384 (96%)**  | **84 (84%)**    |


O melhor método de comparação foi o **TM_CCOEFF_NORMED**, acertando 84% dos captchas e 96% das letras. Como cada template tem tamanho variável, os métodos não normalizados tiveram um desempenho sofrível.



### Conclusão

Nesse post introduzi várias técnicas de processamento de imagem e estatística que podem ajudar não só a quebrar captchas, mas a descobrir objetos em uma cena, normalizar dados, calcular a diferença entre imagens e outros truques uteis no dia a dia.

Como esse post ficou muito maior do que eu esperava, alguns tópicos foram passados bem por cima. Por exemplo, na normalização eu poderia centralizar o caractere usando o [centro de massa](http://docs.opencv.org/2.4/modules/imgproc/doc/structural_analysis_and_shape_descriptors.html#moments), no agrupamento, além do MSE, eu testei [Normalized Compression Distance](https://en.wikipedia.org/wiki/Normalized_compression_distance) e [Structured Similarity](http://scikit-image.org/docs/dev/api/skimage.measure.html#skimage.measure.compare_ssim) o que já seria material suficiente para outro post. Então caso você tenha alguma dúvida ou sugestão, entre em contato.

Como sempre que possível, está disponivel o [notebook](/notebooks/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/notebook.ipynb) com o código completo do que foi descrito aqui e os [arquivos](/files/2016-02-08-quebrando-outro-captcha-com-opencv-e-python/images.zip) com que trabalhei para descrever o post.

Até a próxima.
