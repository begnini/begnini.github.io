---
title: "Quebrando CAPTCHA com Deep Learning"
layout: post
date:   2020-02-23 10:00:00
---

Olá,


estou ~~apanhando~~ aprendendo a trabalhar com Pytorch e resolvi contar a experiência com um experimento simples. Até então tinha trabalhado apenas com keras e tensorflow para desenvolver aplicações com deep learning e estava mais que na hora de sair da zona de conforto.

Nesse post vou passar pelo problema de quebrar captchas de um site usando pytorch e apontando as diferenças entre ele e keras. O post vai ser dividido em alguns passos, típicos na resolução de problemas envolvendo aprendizado de maquina supervisionado.

<br />

### 1. Definir o problema

O problema, como nos posts passados, é o de quebrar um CAPTCHA, ou seja, ler os caracteres escritos em uma imagem. Para facilitar, vou usar o mesmo captcha de um [post anterior](/2016/02/08/quebrando-outro-captcha-com-opencv-e-python.html). Abaixo alguns exemplos de imagens desse captcha:

<br />

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

<br />

O objetivo então é, dado uma imagem RGB com 284x46 pixels de entrada, treinar uma rede neural que de como saída os 4 caracteres que estão escritos nela.

<br />

### 2. Criar o dataset

No mesmo [post anterior](/2016/02/08/quebrando-outro-captcha-com-opencv-e-python.html) que citei agorinha, era possível um acerto de 84% dos captchas. Usando esse algoritmo para classificar as imagens e fazendo a conferência na mão, foi possível gerar um dataset de treino de 31.550 imagens.

Para padronizar o dataset, coloquei no nome do arquivo os 4 caracteres contidos em cada imagem e separei as imagens em 2 diretórios: um diretório de treino contendo 8.000 imagens e um diretório de teste contendo as 23.550 imagens restantes.

O tamanho do dataset de treino ser bem menor do que o dataset de testes é incomum. Geralmente separa-se 80% do dataset inicial para treino, 10% para validação e 10% para testes, porem, como o problema é simples e não necessita de muitos dados, preferi deixar o treino pequeno.

Para carregar um dataset no pytorch, a maneira idiomática é estender a classe `torch.utils.data.Dataset` com o código necessário para ler as informações cada elemento. Nesse dataset, a classe ficará assim:

<br />

{% highlight python linenos %}
class CaptchaDataset(torch.utils.data.Dataset):

    def __init__(self, root_dir, transform=None):

        self.root_dir  = root_dir
        self.files     = glob.glob(root_dir + '/*.png')
        self.transform = transform

        # todas consoantes + numeros
        chars = string.ascii_uppercase + string.digits

        self.labels = sorted(list(set(chars)))
        self.idx_classes = dict(enumerate(self.labels))
        self.classes_idx = dict([(c, i) for (i, c) in enumerate(self.labels)])

    def __len__(self):
        return len(self.files)

    def __getitem__(self, idx):
        if torch.is_tensor(idx):
            idx = idx.tolist()

        filename = self.files[idx]
        image    = cv2.imread(filename)
        image    = image / 255.0

        captcha  = os.path.basename(filename)[:4]
        captcha  = list(captcha)
        captcha  = self.hot_encoding(captcha)

        sample = {
            'image': image,
            'captcha': captcha
        }

        if self.transform:
            sample = self.transform(sample)

        return sample

    def hot_encoding(self, captcha):
        hot = np.zeros(4, dtype=np.int32)

        for (i, c) in enumerate(captcha):
            pos = self.classes_idx[c]
            hot[i] = pos

        return hot
{% endhighlight %}

<br />

Nesse código, 2 métodos são importantes, a `__len__` e a `__getitem__`. O método mágico `__len__` retorna o tamanho do dataset, nesse caso a quantidade de arquivos PNG em um diretório.

Já o método `__getitem__` carrega um item do dataset. Ele retorna um dicionário com 2 informações, a imagem em pixels (height, width, 3) e o captcha "hotencoded". Hot encoded entre aspas porque, ao contrário do keras, que espera o output sendo hotencoded (vetor com n posições, contendo o valor 1 para a classe correta e zero para o restante), o pytorch espera que a saída seja o id da classe (isso para o loss crossentropy). Na primeira versão, eu estava retornando como o keras espera e durante o treino o pytorch retornava o seguinte erro: `RuntimeError: multi-target not supported`

Outro detalhe dessa classe é o último parâmetro do construtor, `transform`. É um callback para processar modificações em um item do dataset, antes de envia-lo para o treino. Depois vamos mostrar a utilidade dele.

<br />

### 3. Definir a arquitetura

<br />

#### Um pouco de teoria


Ian GoodFellow et. al [propos uma arquitetura](https://arxiv.org/abs/1312.6082) capaz de reconhecer múltiplos dígitos a partir dos pixels de uma imagem, usando uma rede neural convolucional. Essa arquitetura foi usada para reconhecimento de números de casas em imagens do Google Street View ([dataset usado](http://ufldl.stanford.edu/housenumbers/)). Nesse paper ele também mostra que a rede pode reconhecer captchas, mesmo os distorcidos pelo reCAPTCHA, com 99.8% de acerto.

Ele cita ainda outros papers [1](https://arxiv.org/abs/1311.2524), [2](https://papers.nips.cc/paper/5207-deep-neural-networks-for-object-detection), [3](https://arxiv.org/abs/1310.1811) mostrando como redes convolucionais podem ser usadas para detecção e localização de objetos, em particular texto, dentro de uma imagem.

Baseado nesses artigos, vou gerar uma variação mais simples da rede descrita. A rede vai ter basicamente varias camadas compostas de uma camada convolucional seguida de um max pooling com stride de 2. Em cada camada dobro o número de filtros (8, 16, 32...) enquanto o max pooling diminui por 2 a altura e a largura da imagem. No final da rede terão algumas camadas densas para classificar a entrada.

A principal diferença entre esse modelo e um modelo de classificação normal ([LeNet](http://yann.lecun.com/exdb/lenet/), [VGGNet](https://arxiv.org/abs/1409.1556)) é que essa rede não terá 1, mas sim 4 saídas, uma para cada carácter reconhecido.


<br />

#### Mão na massa


No Pytorch, a maneira idiomática de se definir um modelo é criando uma classe que estende a classe base `torch.nn.Module`. Nessa classe as camadas que vão ser usadas são definidas no construtor e no método `forward` é formada a sequência em que a entrada será processada.

<br />

{% highlight python linenos %}
class Model(torch.nn.Module):

    def __init__(self):
        super(Model, self).__init__()

        self.conv1 = torch.nn.Conv2d(3, 8, 3)
        self.conv2 = torch.nn.Conv2d(8, 16, 3)
        self.conv3 = torch.nn.Conv2d(16, 32, 3)
        self.conv4 = torch.nn.Conv2d(32, 64, 3)

        self.pool  = torch.nn.MaxPool2d(2, 2)

        self.fc1 = torch.nn.Linear(64 * 15, 120)
        self.fc2 = torch.nn.Linear(120, 84)

        # classificadores - 1 por char
        self.fc3 = torch.nn.Linear(84, 36)
        self.fc4 = torch.nn.Linear(84, 36)
        self.fc5 = torch.nn.Linear(84, 36)
        self.fc6 = torch.nn.Linear(84, 36)

    def forward(self, x):
        x = self.pool(torch.nn.functional.relu(self.conv1(x)))
        x = self.pool(torch.nn.functional.relu(self.conv2(x)))
        x = self.pool(torch.nn.functional.relu(self.conv3(x)))
        x = self.pool(torch.nn.functional.relu(self.conv4(x)))

        x = x.view(-1, 64 * 15)

        x = torch.nn.functional.relu(self.fc1(x))
        x = torch.nn.functional.relu(self.fc2(x))

        x0 = self.fc3(x)
        x1 = self.fc4(x)
        x2 = self.fc5(x)
        x3 = self.fc6(x)

        # 4 outputs, 1 pra cada char
        return [x0, x1, x2, x3]
{% endhighlight %}

<br />

Diferentemente do Keras, o Pytorch necessita que se defina quantos canais terão a entrada de cada camada convolucional e de cada camada densa. É um pouco chato fazer os cálculos, mas pode-se ver qual o tamanho do output de uma camada anterior imprimindo `x.size()` dentro do método `forward` antes de chamar a camada que não se sabe o tamanho. Algo como:

<br />

{% highlight python %}
class Model(torch.nn.Module):
    ...
    def forward(self, x):
        ...
        x = self.pool(torch.nn.functional.relu(self.conv4(x)))

        # nao sei o tamanho do output
        print (x.size())

        x = x.view(-1, 10 * 20)
{% endhighlight %}

<br />

Definido o modelo, é hora de a função de loss e o otimizador. Como falei acima, vou usar como loss o método `crossentropy` e `Adam` como otimizador.

<br />

{% highlight python %}
criterion = torch.nn.CrossEntropyLoss()
optimizer = torch.optim.Adam(model.parameters())
{% endhighlight %}

<br />

Onde `model` é uma instância da classe `Model`.

<br />

#### Detalhes importantes

Primeiro, diferentemente do Keras/Tensorflow que espera o canal de cores das imagens estar na última dimensão (na ordem `(BATCH_SIZE, HEIGHT, WIDTH, CHANNELS)`), no Pytorch as imagens precisam estar com o canal de cores na segunda dimensão (na ordem `(BATCH_SIZE, CHANNELS, HEIGHT, WIDTH)`).

Outra diferença é que o pytorch espera que a entrada e a saida dos algoritmos seja um Tensor em vez de ser um vetor numpy. Por último, tanto o modelo quanto o dataset tem um tipo (int, float, double, etc.) e não fica feliz se esses tipos se misturam na hora do treino. Nesse exemplo, o Model é do tipo Float e a imagem carregada pelo OpenCV está retornando como Double.

Para resolver isso, é preciso criar uma classe que transforma cada entrada para seguir o padrão do pytorch e castear o tipo da imagem para float. Os ids dos caracteres do captcha também precisam ser casteados para long para funcionar. Caso não seja feito esses castings, na hora de treinar o pytorch deve dar um erro do tipo:

`RuntimeError: Expected object of scalar type Long but got scalar type Int for argument #2 'target'`

<br />

{% highlight python linenos %}
class ToTensor(object):
    """Convert ndarrays in sample to Tensors."""

    def __call__(self, sample):
        image, captcha = sample['image'], sample['captcha']

        # swap color axis because
        # numpy image: H x W x C
        # torch image: C X H X W
        image = image.transpose((2, 0, 1))

        return {
            'image': torch.from_numpy(image).to(torch.float),
            'captcha': torch.from_numpy(captcha).to(torch.long)
        }
{% endhighlight %}

<br />

Essa classe recebe um elemento (dict) do dataset carregado, faz as transformações necessárias para adaptar para o pytorch e retorna um dict com as mesmas chaves. Agora, basta passar uma instância dessa classe no parâmetro `transform` no dataset.

<br />

```python
dataset = CaptchaDataset('train', transform=ToTensor())
```

<br />

O transform pode ser usado para outras transformações na imagem (rotação, brilho, etc.), mas não são necessários nesse tutorial.

<br />

### 4. Treinar o modelo

Antes de começar o treino é preciso instanciar um `DataLoader`, que definirá para o Pytorch como os dados do `Dataset` serão iterados. Essa
classe cuida de coisas como o tamanho do batch size e se os elementos do dataset irão em ordem ou serão escolhidos randomicamente em cada época.

<br />

```python
BATCH_SIZE = 32
dataloader = torch.utils.data.DataLoader(dataset, batch_size=BATCH_SIZE, shuffle=True)
```

<br />

O dataloader é um iterador que irá retornar uma lista de entradas e saídas do tamanho do BATCH_SIZE para cada iteração. Para treinar uma época, o loop ficara da seguinte forma:

<br />

{% highlight python linenos %}
def fit(dataloader, model, optimizer, criterion):
    running_loss = 0.0

    for data in dataloader:
        images_batch = data['image']
        labels_batch = data['captcha']

        optimizer.zero_grad()

        outputs = model(images_batch)
        loss = 0
        for i in range(4):
            loss += criterion(outputs[i], labels_batch[:, i])

        loss.backward()
        optimizer.step()

        running_loss += loss.item()

    return running_loss
{% endhighlight %}

<br />

Explicando um pouco o código dessa função, tem-se:

Na linha 4, é feita a iteração em cima dos dados de treino que declaramos no dataloader. Nas linhas 5 e 6 separa-se o input e o output, usando como índice a chave que foi data no Dataset. O bacana dessa organização é que fica fácil distinguir os dados quando precisar de outros inputs e outputs (por exemplo, um problema que o input é um texto e uma imagem) e passar por partes diferentes da rede.

No pytorch os gradientes são somados a cada mini batch, por isso deve-se zera-los quando começar uma passada na rede. A linha 8 faz esse trabalho.

A linha 10 é a responsável pela inferência do modelo nas imagens. Essa inferência vai fazer as imagens passarem pela rede e retornar uma lista com a classificação dos 4 outputs definidos no modelo.

Para cada output então, será calculada a diferença entre o que a rede inferiu e o que realmente é, usando o critério (crossentropy) definido. Como o objetivo é que a rede acerte as 4 letras, calcula-se o erro para cada letra e soma-se eles (linha 13).

Com o erro calculado, o método `loss.backwards()` é chamado para fazer a propagação do erro pela rede, recalculando os pesos dela.

A próxima linha, `optimizer.step()`, atualiza os parâmetros de otimização a partir da quantidade de passos. Por último, na linha 18, o erro obtido em cada mini batch é somado, para retorna-lo e imprimi-lo depois.

Com keras, esse loop não existe no treino. Em vez desse código basta chamar o método `model.fit(X, Y, batch_size=BATCH_SIZE)` e o framework cuida de tudo isso. O pytorch tem uma biblioteca separada que abstrai boa parte desse trabalho, chamada [Ignite](https://github.com/pytorch/ignite/).

Agora é treinar 10 épocas nesse dataset. O loop é simples:

<br />

```python
for epoch in range(10):
    loss = fit(dataloader, model, optimizer, criterion)
    print (epoch, loss)
```

<br />

O resultado é ... ok. Não dá pra saber muito apenas olhando o loss, só dá pra saber que a rede está aprendendo.

<br />

|Época|Loss|
|-----|----:|
|0| 3360.5179271698|
|1| 2377.05939245224|
|2| 957.0550020933151|
|3| 484.01244044303894|
|4| 285.9964772462845|
|5| 198.75198429822922|
|6| 137.1452361792326|
|7| 121.65961718559265|
|8| 85.87324553355575|
|9| 84.99568557739258|

<br />

E agora? Como saber se o modelo está realmente aprendendo ou apenas está decorando os resultados? Para isso precisa-se de 2 coisas:


Primeiro, dividir o dataset em treino e validação. O Pytorch tem a função `random_split` do pacote `torch.utils.data` que faz isso em um dataset.

<br />

```python
train_size = int(len(dataset) * 0.9)
validation_size = len(dataset) - train_size

sizes = [train_size, validation_size]

train_dataset, validation_dataset = torch.utils.data.random_split(dataset, sizes)

# transforma em dataloaders
train_dataloader = torch.utils.data.DataLoader(train_dataset, batch_size=BATCH_SIZE, shuffle=True)
validation_dataloader = torch.utils.data.DataLoader(validation_dataset, batch_size=BATCH_SIZE, shuffle=False)
```

<br />

Com isso, o treino fica com 90% dos dados enquanto a validação fica com o restante. Como a separação é randômica, cada vez que esse código é rodado, deve servir datasets diferentes, o que deve impactar na reprodutibilidade do processamento, mas os resultados não devem ser muito diferentes dos expostos aqui.

O segundo passo é criar um método de avaliação do modelo, olhando o dataset de validação.

<br />

{% highlight python linenos %}
def evaluate(dataloader, model, criterion):

    model.eval()
    dataloader.init_epoch()

    correct_chars = 0
    correct_captchas = 0
    validation_loss = 0

    with torch.no_grad():
        for data in dataloader:
            images = data['image']
            labels = data['captcha']

            outputs = model(images)

            loss = 0
            for i in range(4):
                loss += criterion(outputs[i], labels[:, i])

            validation_loss = loss.item()

            predicted = [torch.max(output, 1)[1] for output in outputs]

            # transforma o predicted p/ ter as mesmas dimensoes dos labels
            # transforma a lista (python) de lista de tensors (4, X)
            # em uma lista de tensors (4 * X)
            predicted = torch.cat(predicted)

            # transforma a matriz (4, X) numa matriz (X, 4), mesmo formato
            # da labels
            predicted = predicted.view(4, -1).transpose(0, 1)

            # calcula qtos chars acertou, por imagem
            per_image = (predicted == labels).sum(1)

            # se acertou os 4 chars, acertou o captcha
            captchas_ok = (per_image == 4).sum().item()
            correct_captchas += captchas_ok

            # quantos chars acertou, no total
            correct = per_image.sum().item()
            correct_chars += correct

    return (validation_loss, correct_chars, correct_captchas)

{% endhighlight %}

<br />

Aqui o pytorch mostra o seu diferencial. Trabalhar com os dados (Tensors) vindo do framework é tão simples quanto se estivessem em numpy.

Agora com o código de validação 100%, é hora de mudar o loop de treino para rodar a validação a cada época. Apenas mais um detalhe, o método `model.eval()` está sendo chamado dentro da função `evaluate` para que o modelo entre em estado de predição e o pytorch não atualize seus pesos. Por isso, a função `fit` deve ser mudada para chamar o método `model.train()` que voltará o modelo ao estado de treinamento e atualizará os pesos durante o processamento do dataset. A mudança é simples, basta apenas adicionar essa linha no começo da `fit` e deixar o resto como está:

<br />

{% highlight python linenos %}
def fit(dataloader, model, optimizer, criterion):
    model.train()
    ...
{% endhighlight %}

<br />


Pronto, agora é treinar e validar o modelo no mesmo loop, com o seguinte código:

<br />

{% highlight python linenos %}
# tamanho do dataset de validacao, usado p/ imprimir as porcentagens
val_images = len(validation_dataset)
val_chars  = val_images * 4

for epoch in tqdm.tqdm_notebook(range(10)):
    loss = fit(train_dataloader, model, optimizer, criterion)

    # adicionado avaliacao do modelo
    acc  = evaluate(validation_dataloader, model, criterion)

    val_loss, chars_acc, captchas_acc = acc
    print (epoch, loss, val_loss, chars_acc, captchas_acc,
        (chars_acc / val_chars) * (100.0),
        (captchas_acc / val_images) * (100.0))

{% endhighlight %}

<br />

Rodando novamente o modelo, têm-se os seguintes resultados para o loop acima:

<br />

|Época|Train Loss|Validation Loss|Chars corretos (% )|Captchas corretos (% )|
|-----|----:|----:|----:|----:|
|0|	3034.937|	13.325|	127 (3.97 %)|	 0 (0.00 %)|
|1|	2770.484|	9.053|	1064 (33.25 %)|	 2 (0.25 %)|
|2|	1380.163|	3.922|	2085 (65.16 %)|	 143 (17.88 %)|
|3|	657.645|	2.855|	2532 (79.12 %)|	 314 (39.25 %)|
|4|	371.933|	1.833|	2755 (86.09 %)|	 436 (54.50 %)|
|5|	225.808|	1.139|	2931 (91.59 %)|	 574 (71.75 %)|
|6|	155.626|	0.983|	2940 (91.88 %)|	 586 (73.25 %)|
|7|	119.391|	0.707|	3003 (93.84 %)|	 633 (79.12 %)|
|8|	 91.694|	0.701|	3010 (94.06 %)|	 640 (80.00 %)|
|9|	 82.021|	0.742|	3022 (94.44 %)|	 647 (80.88 %)|

<br />

Apenas com 10 épocas foi possível chegar a um acerto de 80% dos captchas, nada mal. Rodando com 100 épocas e imprimindo de 5
em 5 épocas, o resultado é ainda melhor:

<br />

|Época|Train Loss|Validation Loss|Chars corretos (% )|Captchas corretos (% )|
|-----|----:|----:|----:|----:|
|0|	3033.076|	13.387|	117 (3.66 %)|	 0 (0.00 %)|
|5|	341.210|	1.508|	2852 (89.12 %)|	 513 (64.12 %)|
|10|	89.716|	0.732|	3047 (95.22 %)|	 665 (83.12 %)|
|15|	48.777|	0.461|	3076 (96.12 %)|	 691 (86.38 %)|
|20|	40.647|	0.370|	3110 (97.19 %)|	 717 (89.62 %)|
|25|	25.351|	0.318|	3133 (97.91 %)|	 739 (92.38 %)|
|30|	16.067|	0.192|	3111 (97.22 %)|	 719 (89.88 %)|
|35|	17.410|	0.394|	3111 (97.22 %)|	 720 (90.00 %)|
|40|	17.856|	0.472|	3126 (97.69 %)|	 732 (91.50 %)|
|45|	15.941|	0.147|	3145 (98.28 %)|	 747 (93.38 %)|
|50|	5.851|	0.317|	3164 (98.88 %)|	 768 (96.00 %)|
|55|	17.309|	0.133|	3153 (98.53 %)|	 756 (94.50 %)|
|60|	12.965|	0.130|	3168 (99.00 %)|	 770 (96.25 %)|
|65|	8.558|	0.125|	3165 (98.91 %)|	 767 (95.88 %)|
|70|	13.959|	0.398|	3162 (98.81 %)|	 764 (95.50 %)|
|75|	8.709|	0.087|	3170 (99.06 %)|	 772 (96.50 %)|
|80|	13.800|	0.193|	3168 (99.00 %)|	 772 (96.50 %)|
|85|	12.236|	0.348|	3140 (98.12 %)|	 744 (93.00 %)|
|90|	11.994|	0.115|	3178 (99.31 %)|	 780 (97.50 %)|
|95|	6.976|	0.079|	**3184 (99.50 %)**|	 **786 (98.25 %)**|
|100|	**3.860**|	**0.025**|	3181 (99.41 %)|	 783 (97.88 %)|

<br />

O reconhecimento é fantastico, comparado ao acerto da [quebra na mão](/2016/02/08/quebrando-outro-captcha-com-opencv-e-python.html) (96% de acerto de letras e 84% de captchas).

<br />

### 5. Testar o modelo no dataset de teste

Para ter certeza de que a taxa de acerto é realmente tão boa quanto a validação, é hora de rodar o `evaluate` no dataset de 23 mil imagens que tinham sido separadas. O código é bem parecido com o evaluate do loop de treino.

<br />

{% highlight python linenos %}

testDataset = CaptchaDataset('test', transform=ToTensor())
testloader = torch.utils.data.DataLoader(testDataset,
                batch_size=32, shuffle=False)

test_acc = evaluate(testloader, model, criterion)

test_images = len(testDataset)
test_chars  = test_images * 4

loss, chars_acc, captchas_acc = test_acc

print ('%.3f\t%d (%.2f %%)\t %d (%.2f %%)' % (loss,
            chars_acc, (chars_acc / test_chars) * (100.0),
            captchas_acc, (captchas_acc / test_images) * (100.0)))

{% endhighlight %}

<br />

Abaixo o resultado do processamento no dataset de teste:

<br />


```python
> loss  chars ok (%)     captchas ok (%)
> 0.082	93745 (99.52 %)	 23146 (98.28 %)
```
<br />

O acerto é melhor ainda que o da validação.

<br />


### Conclusão

Com esse experimento, foi possível tirar 2 conclusões:

1. Pytorch é um framework fácil de se trabalhar. Os problemas que aconteceram durante a construção desse POST geralmente eram resolvidos colando o erro no Google e a maioria das respostas estava na documentação ou no [fórum do pytorch](https://discuss.pytorch.org). A API dos tensores ser bem parecida com a API do numpy é um diferencial que faz a manipulação dos dados ser bem agradável. Outra diferença é, apesar da API ser mais verbosa, ela não é enfadonhamente verbosa (Java anyone?) e existe mais controle e visibilidade do que está sendo feito na rede sem ter que recorrer a um método "faz tudo" com um monte de parâmetros.

2. Deep Learning é muito poderosa no processamento de imagens e reconhecimento de padrões. Parece obvio, mas quando se compara as técnicas anteriores de visão computacional com as atuais (ou nem tão atuais - as técnicas usadas aqui tem mais de 5 anos), é possível ver a superioridade tanto na acurácia quanto na simplicidade para resolver um problema.

Como sempre que possível, está disponivel os [arquivos](/files/2020-02-23-quebrando-captcha-com-deep-learning/images.tar.gz) com que trabalhei para descrever o post.

<br />

### Apendice - Uma rede mais eficiente

A rede descrita nesse tutorial usa uma arquitetura antiga para extrair os dados. Baseada na VGG, ela usa 2 camadas densas depois das camadas convolucionais e antes das camadas de reconhecimento (com softmax). Essas camadas são extremamente grandes em relação a quantidade de parametros, sendo as 2 responsáveis por 77% dos pesos da rede.

Uma forma de simplificar a rede é substituir as camadas densas por uma camada que calcula a média global dos filtros, reduzindo o tamanho da saída de (batch, channels, height, width) para (batch, channels).

Outra forma de melhorar o resultado é adicionar mais uma camada de convolução. Para isso é preciso adicionar padding nas convoluções, pois na arquitetura atual o height da última convolução já está em 1.

A nova rede ficará então assim:

<br />

{% highlight python linenos %}
class Model2(torch.nn.Module):

    def __init__(self):
        super(Model2, self).__init__()

        self.conv1 = torch.nn.Conv2d(3, 8, 3, padding=1)
        self.pool  = torch.nn.MaxPool2d(2, 2)

        self.conv2 = torch.nn.Conv2d(8, 16, 3, padding=1)
        self.conv3 = torch.nn.Conv2d(16, 32, 3, padding=1)
        self.conv4 = torch.nn.Conv2d(32, 64, 3, padding=1)

        # adicionado + 1 conv devido ao padding
        self.conv5 = torch.nn.Conv2d(64, 128, 3, padding=1)

        self.gap = torch.nn.AvgPool2d((1, 8))

        self.fc3 = torch.nn.Linear(128, 36)
        self.fc4 = torch.nn.Linear(128, 36)
        self.fc5 = torch.nn.Linear(128, 36)
        self.fc6 = torch.nn.Linear(128, 36)

    def forward(self, x):
        x = self.pool(torch.nn.functional.relu(self.conv1(x)))
        x = self.pool(torch.nn.functional.relu(self.conv2(x)))
        x = self.pool(torch.nn.functional.relu(self.conv3(x)))
        x = self.pool(torch.nn.functional.relu(self.conv4(x)))
        x = self.pool(torch.nn.functional.relu(self.conv5(x)))

        x = self.gap(x)

        # a saida do GAP eh a quantidade de canais da ultima convolucao
        x = x.view(-1, 128)

        x0 = self.fc3(x)
        x1 = self.fc4(x)
        x2 = self.fc5(x)
        x3 = self.fc6(x)

        return [x0, x1, x2, x3]
{% endhighlight %}

<br />

Com essa rede, a quantidade de paramêtros treináveis caiu de 162.252 para 116.960 (em torno de 28% menor). Mais importante que isso é a acuracia da rede,
como ela se comporta nesse problema? Rodando o mesmo treino e validação nessa nova rede, tem-se o seguinte resultado:

<br />

|Época|Train Loss|Validation Loss|Chars corretos (% )|Captchas corretos (% )|
|-----|----:|----:|----:|----:|
|0|	3029.617|	13.342|	106 (3.31 %)|	 0 (0.00 %)|
|5|	315.077|	1.273|	2862 (89.44 %)|	 499 (62.38 %)|
|10|	23.163|	0.520|	3104 (97.00 %)|	 708 (88.50 %)|
|15|	20.484|	0.232|	3124 (97.62 %)|	 727 (90.88 %)|
|20|	12.864|	0.023|	3157 (98.66 %)|	 757 (94.62 %)|
|25|	0.074|	0.001|	3184 (99.50 %)|	 784 (98.00 %)|
|30|	0.032|	0.001|	3188 (99.62 %)|	 788 (98.50 %)|
|35|	0.017|	0.001|	3189 (99.66 %)|	 789 (98.62 %)|
|40|	0.009|	0.001|	3191 (99.72 %)|	 791 (98.88 %)|
|45|	0.005|	0.001|	3192 (99.75 %)|	 792 (99.00 %)|
|50|	0.003|	0.001|	3192 (99.75 %)|	 792 (99.00 %)|
|55|	0.002|	0.001|	3192 (99.75 %)|	 792 (99.00 %)|
|60|	0.001|	0.002|	3192 (99.75 %)|	 792 (99.00 %)|
|65|	0.000|	0.001|	3191 (99.72 %)|	 791 (98.88 %)|
|70|	0.000|	0.002|	3190 (99.69 %)|	 790 (98.75 %)|
|75|	0.000|	0.001|	**3192 (99.75 %)**|	 **792 (99.00 %)**|
|80|	0.000|	0.001|	3190 (99.69 %)|	 790 (98.75 %)|
|85|	0.000|	0.001|	3191 (99.72 %)|	 791 (98.88 %)|
|90|	0.000|	0.001|	3190 (99.69 %)|	 790 (98.75 %)|
|95|	0.000|	0.001|	3189 (99.66 %)|	 789 (98.62 %)|
|100|	0.000|	0.000|	3188 (99.62 %)|	 788 (98.50 %)|

<br />

A rede, apesar de menor, convergiu muito mais rapidamente batendo o loss da validação já na época 20 e chegou a 99.75% de acerto nos carácteres da validação. O resultado no dataset de teste, como esperado, também é melhor:

<br />

```python
> loss  chars ok (%)     captchas ok (%)
> 0.000	93895 (99.68 %)	 23250 (98.73 %)
```
