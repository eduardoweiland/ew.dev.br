---
title: Fim de jogo para o Placarduino
date: 2018-02-04T19:39:28-02:00
featuredImage: /images/placarduino-led-comemoracao.png
tags: [arduino, eletronica, placarduino]
series: [placarduino]
---

O Placarduino está ficando muito bom. Os jogadores já podem se identificar e
contar seus pontos, mas chegou a hora do _game over_: é o fim de jogo para o
Placarduino. Ou seja, um botão para finalizar o jogo e o vencedor comemorar!

---

Há muito tempo eu já estava pensando em como sinalizar o final de uma partida no
Placarduino. Porém, cada tipo de jogo possui regras diferentes, então seria
complicado colocar a lógica de fim de jogo no código. Então, a solução mais
simples é utilizar um botão que pode ser pressionado a qualquer momento,
indicando que a partida acabou e que o jogador que estiver com mais pontos no
momento é o vencedor. E, claro, no fim do jogo também tem a comemoração do
vencedor.

O material utilizado:

- 1 push button
- 1 LED (eu utilizei um de cor verde)
- 1 resistor de 300 Ω (para o LED)
- 1 resistor de 10 kΩ
- Alguns cabos jumper

O uso do LED é opcional, mas foi um teste que eu fiz e achei o resultado
interessante, então eu mantive no projeto.

## Ligação do LED comemorativo

Para ligar o LED no circuito, além do LED são necessários um resistor e um cabo
jumper. E nada de código. Vamos "roubar" a energia do buzzer para acender o LED
de acordo com a música, ligando-os em paralelo conforme o diagrama:

![Ligação do LED de comemoração no Placarduino][]

E é só isso mesmo! Agora, sempre que o buzzer tocar um som, o LED irá acompanhar
e ficar piscando:

{{< video src="/videos/placarduino-led-piscando.webm" width="853" height="480" >}}

## Adicionando o botão para terminar o jogo

O novo botão foi adicionado ao circuito da mesma forma que todos os outros, com
um resistor de pull-down. Esse novo botão foi ligado no pino digital 8 do
Arduino. O resultado final é mostrado na imagem abaixo (o que foi adicionado
está destacado com a elipse vermelha).

![Ligação do botão de fim de jogo no Placarduino][]

E é só isso mesmo! Só falta o código...

Para verificar o pressionamento do botão, será utilizada a [biblioteca
RisingEdgeButton criada anteriormente][post-refatoracao]. E, para evitar que o
jogo continue depois do apito final, será utilizada uma flag booleana.

```cpp
// Linhas adicionadas ao Placarduino.ino nos locais adequados

#include "RisingEdgeButton.h"

#define PIN_BTN_GAME_OVER 8

RisingEdgeButton gameOverButton(PIN_BTN_GAME_OVER);
bool isGameOver = false;
```

Dentro do `loop()`, é evitado que o jogo continue depois do fim (fica na tela de
fim de jogo até reiniciar o Arduino) e verificado o pressionamento do botão. Ao
pressionar o botão, é chamada outra função para tratar o fim de jogo:

```cpp
void loop()
{
    if (isGameOver) {
        return;
    }

    readPlayerCard();
    checkButtons();

    if (gameOverButton.pressed()) {
        gameOver();
    }
}
```

A função `gameOver()` começa alterando a flag de final de jogo e limpando o
display LCD para exibir uma nova mensagem. Em seguida, é verificada a pontuação
dos dois jogadores para identificar se existe ou vencedor ou se o jogo terminou
empatado:

```cpp
void gameOver()
{
    bool hasWinner = false;
    const char *winnerName;
    size_t winnerNameLength;

    isGameOver = true;
    lcd.clear();

    if (player1.getScore() > player2.getScore()) {
        winnerName = player1.getName();
        hasWinner = true;
    }
    else if (player2.getScore() > player1.getScore()) {
        winnerName = player2.getName();
        hasWinner = true;
    }

    // continua...
```

Em seguida é exibida a mensagem do fim de jogo. Caso o jogo termine empatado, é
exibido apenas a mensagem "EMPATE" centralizada (confesso que não dediquei muito
tempo para essa parte de empate).

Já em caso de vitória de algum jogador, é exibido um "emoji" de comemoração \\o/,
além do nome do jogador e a indicação de que ele "GANHOU!".

```cpp
    // ... continuação

    if (!hasWinner) {
        lcd.setCursor(7, 1);
        lcd.print("EMPATE");
    }
    else {
        // \o/
        lcd.setCursor(1, 1); lcd.write(255);
        lcd.setCursor(1, 2); lcd.write(255);
        lcd.setCursor(2, 3); lcd.write(255);

        lcd.setCursor(3, 2); lcd.write(0);
        lcd.setCursor(4, 2); lcd.write(4);
        lcd.setCursor(5, 2); lcd.write(1);
        lcd.setCursor(3, 3); lcd.write(2);
        lcd.setCursor(4, 3); lcd.write(5);
        lcd.setCursor(5, 3); lcd.write(3);

        lcd.setCursor(7, 1); lcd.write(255);
        lcd.setCursor(7, 2); lcd.write(255);
        lcd.setCursor(6, 3); lcd.write(255);

        winnerNameLength = strnlen(winnerName, PlayerControl::MAX_NAME_LENGTH);
        lcd.setCursor(9 + (11 - winnerNameLength) / 2, 1);
        lcd.print(winnerName);

        lcd.setCursor(11, 2);
        lcd.print("GANHOU!");

        musicPlayer.victory();
    }
}
```

![Comemoração ao final do jogo][]

E, para fechar com chave de ouro, o `MusicPlayer` toca um tema da vitória (bem
simples, mas é melhor que nada)!

```cpp
void MusicPlayer::victory()
{
    this->playNote(NOTE_C6, 140);
    this->playNote(NOTE_D6, 140);
    this->playNote(NOTE_E6, 140);
    this->playNote(NOTE_G6, 260);
    this->pause(140);
    this->playNote(NOTE_E6, 140);
    this->playNote(NOTE_G6, 550);
}
```

## Finalizando

E isso é tudo por enquanto. Como sempre, você pode conferir o código-fonte e o
arquivo do Fritzing no repositório do GitHub:

- Versão deste post: https://github.com/eduardoweiland/placarduino/tree/v2.1
- Versão mais atualizada: https://github.com/eduardoweiland/placarduino

Bons jogos e até a próxima! \\o/


[post-refatoracao]: /posts/2018/01/refatoracao-do-placarduino/
[Ligação do LED de comemoração no Placarduino]: /images/placarduino-led-comemoracao.png
[Ligação do botão de fim de jogo no Placarduino]: /images/placarduino-botao-fim-de-jogo.png
[Comemoração ao final do jogo]: /images/placarduino-jogador-ganhou.jpg
