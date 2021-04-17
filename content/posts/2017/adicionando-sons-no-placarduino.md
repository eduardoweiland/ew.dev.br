---
title: Adicionando sons no Placarduino
date: 2017-10-27T21:51:27-02:00
featuredImage: /images/musical-notes.jpg
tags: [arduino, eletronica, placarduino]
series: [placarduino]
---

Este post é a continuação do projeto Placarduino. A última postagem foi a
[criação da primeira versão][]. Até o momento, o placar já exibe dois jogadores
e conta os pontos de cada um. Mas seria interessante se também houvesse algum
tipo de som de abertura e a cada ponto marcado. É isso que vamos fazer!

---

Para fazer isso será necessário:

- 1 Placarduino v1.0
- 1 buzzer passivo de 5V
- 1 resistor de 220Ω
- Alguns cabos jumper
- Uma música simples para tocar (ou mais de uma)

Precisa de pouco material mesmo. Mas o resultado será muito melhor do que a soma
das partes 🙂

## Montagem

A partir da montagem já feita no primeiro post do Placarduino, para adicionar a
música só precisamos conectar um buzzer de 5V ao circuito. O buzzer possui dois
pinos: o positivo (que deve estar marcado com um símbolo de +) deve ser ligado a
um pino analógico do Arduino (eu usei o pino 5) com um resistor de 220Ω, e o
pino negativo deve ser ligado ao GND. No final, o circuito deve ficar assim:

![Montando o buzzer no Placarduino][]

## O código

Para produzir os sons eu utilizei a biblioteca [ToneLibrary][]. Ela pode ser
instalada pelo Gerenciador de bibliotecas do Arduino (_Sketch > Incluir Biblioteca >
Gerenciar Bibliotecas..._). Depois de instalada, ela deve ser incluída no código
do projeto:

```c
#include <Tone.h>
```

Precisamos, também, configurar o pino utilizado para o buzzer. Para isso, eu
coloquei mais uma constante no início do arquivo, que será utilizada na
inicialização da biblioteca:

```c
#define PIN_BUZZER        5
```

Em seguida, o objeto Tone é criado no escopo global. Diferente da biblioteca do
LCD, o pino não é configurado no construtor do objeto, e sim no método `begin()`
dele.

```c
// Objeto para controle de um buzzer
Tone buzzer;
```

A função de setup do buzzer:

```c
void setupBuzzer()
{
    buzzer.begin(PIN_BUZZER);
}
```

Essa função deve ser chamada no `setup()` principal:

```c
void setup()
{
    setupInputs();
    setupLCD();
    setupBuzzer();

    printWelcome();

    delay(3000);
}
```

Agora já está tudo configurado para começar a emitir sons pelo Arduino.

### _Feedback_ sonoro ao somar um ponto

O primeiro som que eu adicionei foi um simples aviso sonoro após a marcação de
um ponto. Para fazer isso, eu criei uma função que eu nomeei de `playFeedback()`
e chamei nos dois lugares que somam um ponto ao placar:

```c
void playFeedback()
{
    buzzer.play(NOTE_C6, 120);
    delay(140);

    buzzer.play(NOTE_C7, 420);
    delay(500);
}
```

O primeiro parâmetro do método `play()` é a frequência que deve ser tocada (aqui
eu utilizei algumas constantes incluídas na biblioteca), e o segundo é a duração
em milissegundos. Um detalhe importante a ser observado é que esse método é
assíncrono, por isso eu adicionei o `delay()` entre as notas com um tempo um
pouco maior do que o que a nota vai ficar tocando.

Depois, essa função é chamada no `checkButtons()`:

```c
void checkButtons()
{
    int newButtonState;

    newButtonState = digitalRead(PIN_BTN_1);
    if (newButtonState != firstButtonState && newButtonState == HIGH) {
        // Se o botão mudou de estado e o novo estado é pressionado, soma um ponto
        firstPlayerScore++;
        playFeedback();                 // <<<<<--------------- ADICIONADO
    }
    firstButtonState = newButtonState;

    newButtonState = digitalRead(PIN_BTN_2);
    if (newButtonState != secondButtonState && newButtonState == HIGH) {
        // Se o botão mudou de estado e o novo estado é pressionado, soma um ponto
        secondPlayerScore++;
        playFeedback();                 // <<<<<--------------- ADICIONADO
    }
    secondButtonState = newButtonState;
}
```

É um som bem simples, mas o resultado fica bem legal 🙂

{{< video src="/videos/play-feedback.webm" width="1280" height="720" >}}

### Música de abertura

Nos requisitos do começo desse post eu listei uma música. Será a música de
abertura! Infelizmente, não tem como tocar um música em [OGG][] apenas com um
buzzer, então terá que ser um som mais simples. Eu elaborei três opções, mas,
pesquisando na internet (de preferência no [DuckDuckGo][]), é fácil encontrar
outros exemplos de músicas com um buzzer.

Para tocar essas músicas na abertura eu criei uma função `playWelcome()` e
adicionei a chamada dela no `setup()`, após o `printWelcome()`:

```c
void setup()
{
    setupInputs();
    setupLCD();
    setupBuzzer();

    printWelcome();
    playWelcome();

    delay(3000);
}
```

A primeira opção é uma simples escala de Dó maior (dó, ré, mi, fá, sol, lá, si,
dó), variando o tempo de duração das notas:

```c
void playWelcome()
{
    int i;
    int notes[] = {
        NOTE_C6,
        NOTE_D6,
        NOTE_E6,
        NOTE_F6,
        NOTE_G6,
        NOTE_A6,
        NOTE_B6,
        NOTE_C7
    };

    for (i = 0; i < 8; ++i) {
        buzzer.play(notes[i], 30 + (i * 5));
        delay(35 + (i * 5));
    }
}
```

{{< video src="/videos/placarduino-musica-1.webm" width="1280" height="720" >}}


A segunda opção já é um pouco mais elaborada. E conhecida também 😉

```c
void playWelcome()
{
    buzzer.play(NOTE_E6, 120);
    delay(140);
    buzzer.play(NOTE_E6, 120);
    delay(140);
    delay(140);
    buzzer.play(NOTE_E6, 120);
    delay(140);
    delay(140);
    buzzer.play(NOTE_C6, 120);
    delay(140);
    buzzer.play(NOTE_E6, 240);
    delay(260);
    buzzer.play(NOTE_G6, 240);
    delay(260);
    delay(260);
    buzzer.play(NOTE_G5, 240);
    delay(260);
}
```

Em algumas partes desse código aparecem duas chamadas do `delay()` seguidas.
**Isso é assim mesmo**. Eu deixei separado porque o primeiro `delay()` é para
aguardar a nota anterior, e o segundo é uma pausa entre as notas. E o resultado
é esse:

{{< video src="/videos/placarduino-musica-2.webm" width="1280" height="720" >}}

E a última música também é famosa. Na verdade, é só a abertura da música (com
algumas adaptações), mas já dá para reconhecer:

```c
void playWelcome()
{
    int i, j;
    int firstNote[] = { NOTE_CS5, NOTE_DS5, NOTE_FS5 };

    for (i = 0; i < 3; ++i) {
        for (j = 0; j < 2; ++j) {
            buzzer.play(firstNote[i], 240);
            delay(260);
            buzzer.play(NOTE_CS6, 240);
            delay(260);
            buzzer.play(NOTE_GS5, 240);
            delay(260);
            buzzer.play(NOTE_FS5, 240);
            delay(260);
            buzzer.play(NOTE_FS6, 240);
            delay(260);
            buzzer.play(NOTE_GS5, 240);
            delay(260);
            buzzer.play(NOTE_F6, 240);
            delay(260);
            buzzer.play(NOTE_GS5, 240);
            delay(260);
        }
    }

    buzzer.play(NOTE_CS5, 240);
    delay(260);
}
```

{{< video src="/videos/placarduino-musica-3.webm" width="1280" height="720" >}}

## Outras melhorias

No espírito da refatoração e melhoria contínuas, outros pontos foram melhorados
no código do projeto. Um deles foi fazer a atualização do LCD apenas quando algum
ponto é marcado. A melhoria mais visível dessa alteração é a redução do _flicker_
no LCD, que era causada pela atualização muito frequente.

Para fazer essa alteração, é só adicionar o `printScore()` no final do método
`setup()` e antes das duas chamadas do `playFeedback()` na função `checkButtons()`.
Além disso, o `printScore()` e o `delay(100)` devem ser removido do `loop()`,
deixando apenas a chamada do `checkButtons()`.

## Finalizando

Se você quiser acessar o código-fonte completo do que foi feito nesse post, está
tudo no GitHub:

- Versão dessa postagem: https://github.com/eduardoweiland/placarduino/tree/v1.1
- Versão mais recente: https://github.com/eduardoweiland/placarduino

Os próximos itens do _roadmap_ são:

- Substituir o display 16x2 por um de 20x4 com uma interface I2C
- Identificação de jogadores por cartões RFID
- Configuração de modalidades de jogo (com condição para definir o vencedor)
- Suporte a jogos com _sets_ (contar pontuação do set e também os sets)
- Campeonatos de pontos corridos


[criação da primeira versão]: /2017-10-22-placarduino-placar-eletronico-com-arduino
[Montando o buzzer no Placarduino]: /images/placarduino-buzzer.png
[ToneLibrary]: https://github.com/daniel-centore/arduino-tone-library
[OGG]: https://www.gnu.org/philosophy/why-audio-format-matters.en.html
[DuckDuckGo]: https://duckduckgo.com
