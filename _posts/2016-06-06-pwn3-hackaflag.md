---
layout: post
title: XPL Pwn3
categories:
  - Hackaflag_Aracaju
published: true
---
<div class="message">Link para pwn3:https://s3-us-west-1.amazonaws.com/contattafiles/ratf/dRi0rQk6fm53a8r/pwn3</div>
Ao se executar o pwn pela primeira vez e tentar fazer um overflow enviando um monte de A's para a pergunta "Qual seu nome?", percebi que ele limitava a quantidade de caracteres que eram impressos.

Uma proteção contra overflow? Agora o negócio tava mais complicado que os outros dois pwn.

Mas por acaso, antes de analisar o pwn no gdb ou objdump, percebi que ele substituia qualquer "k" (apenas minusculo) por "lol" e com k's suficientes os lol estouravam o espaço reservado para eles e passariam por cima de tudo, causando um buffer overflow. (Acho que o conhecimento que isso passa é de que sempre se deve testar qualquer tipo de INPUT para verificar falhas em um programa)

Abrindo o pwn no gdb e mandando um info functions, achei 4 funções 
que eram interessantes:
{% highlight js %}

0x08048f13  get_flag
0x08048f21  replace
0x080491af  vuln
0x0804932d  main

{% endhighlight %}
Coloquei breakpoints em todas (b replace, b vuln, b main, b get_flag) e coloquei o pwn pra rodar (run).

Primeiro breakpoint para em main, função principal do programa (pouca novidade aqui). Uso o comando disas main e tento analisar o código (tô longe de saber muito de asm), entendi que main, basicamente só serve pra chamar a função vuln.

{% highlight js %}
   0x08049333 <+6>:     call   0x80491af <vuln>
{% endhighlight %}

Com o comando c, mando pwn continuar. E, como imaginei, ele para em vuln. Tento analisar o código da função usando disas vuln, mas a coisa fica um pouco complicada por causa do meu pouco conhecimento de asm. 
Mesmo assim, compreendo algumas linhas e conclui que vuln era responsável por imprimir a mensagem perguntando do nome, copiar o que a função replace retornar, para o buffer vulnerável usando a perigosa função strcpy, e então imprimir na tela o resultado.
Só lembrando que isso tudo eram suposições (boa parte baseadas nas chamadas para funções que vi no código, tipo printf, fgets, call para replace, strcpy...)., já que meu conhecimento em asm (AINDA) não é suficiente.

Como o código de vuln já mostrava, quando mandei o GDB continuar, surgiu a mensagem pedindo para digitar o nome. Depois de digitar qualquer coisa e teclar enter, o breakpoint em replace faz efeito.

Analisar a função replace foi mais complicado. A conclusão que tirei, por achismo, é que ela fica responsável por varrer a string em busca de K's e, caso encontre, substitua por lol. E provavelmente retorne a string modificada.

Instruindo o gdb a continuar, vi que o programa finaliza exibindo a mensagem só isso.

Então executei o programa novamente no gdb (usando o comando run), mas dessa vez meti um monte de k's no lugar do meu nome. Aí, depois de mandar o pwn continuar após o breakpoint em replace, o gdb me mostra:
{% highlight js %}

Program received signal SIGSEGV, Segmentation fault.
0x6f6c6c6f  in ?? ()

{% endhighlight %}
6f6c6c6f = ollo em hex, o que significa que temos meios lol's em eip, o que significa que podemos sobreescreve-lo com coisa mais útil. Tipo o endereço de get_flag.

Eu aposto que deve haver um jeito mais prático, mas no calor do ctf, fui manualmente testando quantos k's eram necessários para chegar em EIP. De k em k, descobri que 22 k's corrompem eip com um ol, meio lol. Então usei 21 k's mais um caractere qualquer (usei o l no dia) e 4 A's.
{% highlight js %}

Oi, lollollollollollollollollollollollollollollollollollollollollollAAAA


Program received signal SIGSEGV, Segmentation fault.
0x41414141 in ?? ()
{% endhighlight %}

Achei onde pôr o endereço de get_flag ;D

Para testar a solução, usei o python para criar um arquivo que serviria de entrada para pwn: 
{% highlight js %}
python -c 'print "k"*21 + "l" + "\x13\x8f\x04\x08"' > input
{% endhighlight %}

Obs: eu já tinha anotado o endereço de get_flag (0x08048f13) depois de analisar as funções no gdb, imaginei que seria útil. E, como sabemos que a regra usada nesse caso para manipulação de armazenamento na memória é Little Endian, invertemos o endereço: 08 04 8f 13 vira 13 8f 04 08.

Para ver se tudo funcionava, no gdb com pwn3, coloquei um breakpoint apenas em get_flag. Então rodei o programa e redirecionei a entrada para o arquivo input, assim, ele iria ler o conteúdo do arquivo e usar como resposta para a pergunta "Qual seu nome?".
{% highlight js %}

(gdb) run < input
Starting program: /home/update/pwn3 < input
Qual seu nome? Oi, lollollollollollollollollollollollollollollollollollollollolloll▒


Breakpoint 1, 0x08048f13 in get_flag ()
{% endhighlight %}

Missão cumprida. EIP apontava para get_flag.
{% highlight js %}

(gdb) c
Continuando.
cat: flag.txt: Arquivo ou diretório não encontrado

Program received signal SIGSEGV, Segmentation fault.
0x08048f1f in get_flag ()
{% endhighlight %}

Agora bastava explorar o pwn que estava rodando em algum ip dentro da rede do hack a flag e ver o contéudo do arquivo flag.txt ;)

EOF
