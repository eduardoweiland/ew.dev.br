---
title: Reset automático do Placarduino após o fim do jogo
date: 2018-02-18T13:18:46-03:00
tags: [arduino, eletronica, placarduino]
series: [placarduino]
---

Depois da última modificação do Placarduino, que adicionou um tratamento para
[fim de jogo][post-fim-de-jogo], a única forma de iniciar um novo jogo é
resetando o Arduino ao final do jogo. E se isso fosse automático? Usando o
Watchdog do Atmega isso é possível!

---

Existem outras formas de resetar um Arduino, como usar um jumper entre o pino de
reset e um pino digital para controlar pelo código, ou pular para a primeira
instrução do programa (usando Assembly ou um ponteiro de função declarado com o
endereço zero), ou usando o Watchdog. Esta última me parece ser a solução menos
"gambiarra" (e também é [a solução recomendada pela Atmel][]).

O Watchdog é uma funcionalidade desenvolvida especificamente para reiniciar o
microcontrolador quando ele ficar sem enviar um sinal de _heartbeat_ por um
período de tempo configurado.

O uso do Watchdog com o objetivo de reiniciar o Atmega é muito simples, basta
chamar uma função para habilitá-lo e configurar o tempo limite para reiniciar.
Após isso, qualquer operação que demore mais do que o tempo configurado (um loop
infinito, por exemplo) irá causar um reset do Atmega (já que não foi enviado o
_heartbeat_).

Para utilizar o Watchdog é necessário adicionar o seguinte include no começo do
arquivo Placarduino.ino:

```cpp
#include <avr/wdt.h>
```

E no final da função `gameOver()`:

```cpp
void gameOver()
{
    // ...

    wdt_enable(WDTO_4S);
    while (1);
}
```

A macro `wdt_enable()` é a responsável por habilitar o Watchdog e configurar o
tempo para reset usando uma das constantes disponíveis. Neste caso, foi utilizado
o tempo de 4 segundos, mas existem constantes desde 15ms até 8s. Logo depois de
habilitar o Watchdog, o `while (1)` garante que nada mais aconteça no Placaraduino
até que o microcontrolador seja resetado (o que antes era feito com a flag
`isGameOver`, que agora pode ser removida).

Apenas por curiosidade, já que não é necessário para esse caso, o _heartbeat_ que
evita que o Watchdog resete o Atmega, é "enviado" chamando a macro `wdt_reset()`.
Se o Watchdog fosse utilizado em alguma aplicação de monitoramento ou que não
tenha contato frequente de pessoas e fosse habilitado durante todo o programa,
essa macro deveria ser chamada periodicamente no `loop()` e em outros funções
que podem demorar mais do que o tempo configurado, para indicar que o programa
está funcionando e evitar o reset.

E, apenas para evitar qualquer surpresa: em alguns casos [pode acontecer de o
Watchdog continuar habilitado após o reset][watchdog-timer-reset], causando
outros resets inesperados. Geralmente o _bootloader_ deve desativar o Watchdog
durante a inicialização, mas alguns _bootloaders_ podem não fazer isso. Então,
para não ficar dependente do _bootloader_, eu achei melhor desativar o Watchdog
no `setup()`:

```cpp
void setup()
{
    MCUSR = 0;
    wdt_disable();

    // ...
}
```

E isso é tudo! Ao final do jogo, após 4 segundos o Atmega será reiniciado pelo
Watchdog e um novo jogo será iniciado. Como sempre, o código está no GitHub:

- Esta versão: https://github.com/eduardoweiland/placarduino/tree/v2.2
- Versão mais recente: https://github.com/eduardoweiland/placarduino/


[post-fim-de-jogo]: /posts/2018/02/fim-de-jogo-para-o-placarduino/
[a solução recomendada pela Atmel]: https://arduino.stackexchange.com/a/13424
[watchdog-timer-reset]: https://tushev.org/articles/arduino/5/arduino-and-watchdog-timer
