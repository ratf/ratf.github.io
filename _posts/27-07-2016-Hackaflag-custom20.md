---
layout: post
title: Custom 20
categories:
  - Hackaflag SP Classificatória
published: true
---
<div class="message"> O PROBLEMA </div>
Temos um modelo de como a flag é gerada:
{% highlight js %}

name: admin 97100109105110
key: gjsot 103106115111116

serial: DESEC{200206224216226:MjEyMzJmMjk3YTU3YTVhNzQzODk0YTBlNGE4MDFmYzM=}
{% endhighlight %}

Para obter a flag você precisa gerar com base nos seguintes valores:
{% highlight js %}

name: hueboot
key:

serial: desec{format:format}
{% endhighlight %}

<div class="message"> A SOLUÇÃO </div>

A dificuldade dessa flag é conseguir reconhecer padrões de encodings conhecidas. A resolução apresentada não é única, é um exemplo de como eu fiz.

A primeira coisa que tentei foi decodificar a segunda parte do serial de exemplo, parecia ser um base64 então usei pra isso o base64 do sistema. Pode ser utilizado alguma ferramenta online. o resultado foi: 
{% highlight js %}

21232f297a57a5a743894a0e4a801fc3
{% endhighlight %}

Em um primeiro momento não soube como tratar isso, mas por alguma razão desconhecida, resolvi chutar que era MD5 (depois que usei até faz sentido, essa string de resultado tem cara de MD5 =P )

Nesse ponto eu resolvi partir do exemplo pro serial, pra garantir que eu tinha as ferramentas corretas para gerar a chave do CTF. A primeira coisa que tentei codificar foi o name (admin)

Pra isso tentei com o md5sum do linux, não passou nem perto:
{% highlight js %}

tidereis13@yoda ~ $ echo "admin"|md5sum
456b7016a916a4b178dd72b947c152b7 -
{% endhighlight %}

Acredito que não é a mesma coisa de outros encodings MD5, depois vou pesquisar como fazer isso via shell.

E tentei com o MD5 Encode do Yellowpipe:

http://www.yellowpipe.com/yis/tools/encrypter/index.php
{% highlight js %}

Result:

admin
converts to: 21232f297a57a5a743894a0e4a801fc3
{% endhighlight %}


Rolou demais! o mesmo md5 hash. Converti pra base64 usando o Yellowpipe e bingo!

A segunda parte do serial tava feita:
{% highlight js %}

md5(hueboot) = 18c90f1c5789f25ce8186426c5856be8
base64( md5(hueboot)) = MThjOTBmMWM1Nzg5ZjI1Y2U4MTg2NDI2YzU4NTZiZTg=

serial: desec{format:MThjOTBmMWM1Nzg5ZjI1Y2U4MTg2NDI2YzU4NTZiZTg=}
{% endhighlight %}

<div class="message">IMPORTANTISSIMO!!!!</div>

Nessa parte eu estava resolvendo com o Yellowpipe, porque vi que usando o base64 do linux estava gerando um resultado diferente da página. Descobri depois que foi a forma de passar a string pro base64 no linux.
Normalmente fazemos: 
{% highlight js %}

echo "STRING" | base64 
{% endhighlight %}

No entanto o echo sempre envia uma quebra de linha '\n' por padrão. Portanto para gerar o hash correto você precisa utilizar a opção '-n' 
{% highlight js %}

echo -n "STRING" | base64
{% endhighlight %}

Uma vez que descobri essa parte gritei por ajuda. Isso é dica de ouro: NUNCA DEIXE DE PEDIR AJUDA SEMPRE QUE PRECISAR, QUISER, ACHAR QUE ALGUÉM PODE AJUDAR. SEMPRE PEÇA RÁPIDO!!!!! lembre-se que o tempo do ctf é contadinho, então quanto mais rápido melhor.

Faltava a primeira parte e ai entrou o Manoel (R.A.T.F.). eu ja tinha percebido que a key (gjsot) era uma criptografia de Cesar. Em um primeiro momento eu tentei no yellowpipe mas ele me confundiu com o fator de rotacionamento, o Manoel sacou que era um ROT+6 e a ferramenta que conseguiu me fornecer esse mesmo resultado foi a do link abaixo:

http://www.dcode.fr/caesar-cipher

Essa ferramenta não mostra as rotações para transformar o texto criptografado para o original, mas sim quantas rotações são feitas do original para o criptografado.
{% highlight js %}

Assim a ROT+6(hueboot) = nakhuuz
{% endhighlight %}

Além disso ele sacou que o código numérico na frente do name e da key era a representação decimal do código ASCII de cada letra:
{% highlight js %}

DEC(a) = 97
DEC(ad)= 97100
DEC(admin) = 97100109105110
{% endhighlight %}

Pra essa conversão usei o site abaixo:

https://www.branah.com/ascii-converter

Uma vez convertido os códigos, bastava somar os dois inteiros formados, mas era preciso usar Python, ou outra linguagem, ou fazer na mão, porque são números grandes demais para a maioria das calculadoras.

Um site que pode ser utilizado é https://defuse.ca/big-number-calculator.htm

97100109105110 + 103106115111116 = 200206224216226

e assim :
{% highlight js %}

DEC(hueboot) = 10411710198111111116
DEC(nakhuuz) = 11097107104117117122
Soma = 21508817302228228238
{% endhighlight %}

Dessa forma temos o resultado final:
{% highlight js %}
DESEC{21508817302228228238:MThjOTBmMWM1Nzg5ZjI1Y2U4MTg2NDI2YzU4NTZiZTg=}

FLAG: DESEC{21508817302228228238:MThjOTBmMWM1Nzg5ZjI1Y2U4MTg2NDI2YzU4NTZiZTg=}
{% endhighlight %}
