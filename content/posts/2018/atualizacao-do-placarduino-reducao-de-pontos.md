---
title: "Atualização do Placarduino: redução de pontos"
date: 2018-01-13T11:19:50-02:00
featuredImage: /images/placarduino-push-buttons-diminuir-pontos.png
tags: [arduino, eletronica, placarduino]
series: [placarduino]
---

Faz algum tempo que eu não publico nada sobre o projeto do [Placarduino][], mas
ele não ficou parado esse tempo todo. Foram feitas duas atualizações, mas eu não
tive tempo de organizar uma postagem sobre essas alterações. Então nesse post eu
vou mostrar a modificação mais simples, que foi adicionar um botão para reduzir
os pontos de um jogador (e.g. se o ponto foi adicionado por engano, agora existe
um "Ctrl+Z").

## Planejamento

Antes dessa alteração, existiam apenas dois botões no circuito, um para cada
jogador. Ao pressionar um dos botões, era somado 1 aos pontos do respectivo
jogador. Agora serão adicionados mais dois botões, também um para cada jogador,
que serão utilizados para subtrair 1 dos pontos do respectivo jogador.

Para essa alteração, serão utilizados:

- 2 push buttons
- 2 resistores 10 kΩ
- Alguns cabos jumper

## Ligação dos pushbuttons

A ligação dos novos pushbuttons no circuito é a mesma dos dois botões já
existentes: 5V em um dos terminais e o resistor de 10 kΩ entre o outro terminal
e o GND. Nesse terminal com o resistor também é adicionado um fio ligado a uma
das entradas digitais do Arduino.

Para facilitar as ligações, os pinos utilizados foram 4, 5, 6 e 7. Antes,
eram utilizados apenas os pinos 6 e 7, e o pino 5 estava em uso para ligação do
[buzzer adicionado para emitir sons][]. O buzzer foi movido para o pino 3 e os
os botões foram ligados conforme a imagem a seguir.

![Esquema de ligação dos novos push buttons][]

Isso é tudo que precisa ser alterado no circuito. O resto é apenas código.

## Alterações no código

A primeira alteração necessária, como já foi comentado, é a alteração dos pinos
utilizados:

```cpp
// Configurações de pinos
#define PIN_BTN_SUB_PL1   7
#define PIN_BTN_ADD_PL1   6
#define PIN_BTN_SUB_PL2   5
#define PIN_BTN_ADD_PL2   4

#define PIN_BUZZER        3
```

A função `setupInputs()` também precisa ser alterada para utilizar as novas
constantes:

```cpp
void setupInputs()
{
    pinMode(PIN_BTN_SUB_PL1, INPUT);
    pinMode(PIN_BTN_ADD_PL1, INPUT);
    pinMode(PIN_BTN_SUB_PL2, INPUT);
    pinMode(PIN_BTN_ADD_PL2, INPUT);
}
```

Além de alterar os pinos utilizados no Arduino, as variáveis utilizadas para
salvar o estado dos botões também precisam ser modificadas:

```cpp
// Configurações dos jogadores e placar
const char firstPlayerName[] = "Jogador 1";
int firstPlayerScore = 0;
int subPlayer1ButtonState = LOW;
int addPlayer1ButtonState = LOW;

const char secondPlayerName[] = "Jogador 2";
int secondPlayerScore = 0;
int subPlayer2ButtonState = LOW;
int addPlayer2ButtonState = LOW;
```

Agora a parte mais trabalhosa (mas não muito) dessa alteração: a verificação dos
botões e alteração do placar. A função `checkButtons()`, que antes fazia tudo,
foi refatorada: foi adicionada a função `checkScoreInput()` para verificar cada
botão, e a função `checkButtons()` chama essa nova função 4 vezes, uma para cada
botão do circuito.

`checkScoreInput()` recebe 5 parâmetros:

- `int inputPin` é o número do pino que deve ser lido (uma das constantes `PIN_BTN_*`)
- `int *stateStore` é um ponteiro para a variável onde deve ser salvo o estado
do botão (uma das variáveis `(add|sub)PlayerNButtonState`)
- `unsigned int *scoreStore` é um ponteiro para a variável onde deve ser salvo
o placar do jogador (uma das variáveis `(first|second)PlayerScore`)
- `int op` é a operação a ser executada, ou seja, quantos pontos deve ser
adicionados ou subtraídos do placar (+1 ou -1)
- `void(*feedbackFunction)()` é um ponteiro para uma função de feedback, que
são as funções que tocam um som ao alterar o placar (mostradas mais adiante)

Talvez a parte mais complicada seja o ponteiro de função. Não é realmente um
código bonito, mas funciona. Para aprender mais sobre ponteiros de funções,
pesquise no ~~Google~~ DuckDuckGo.

O código completo da função até que é bem simples, muito semelhante ao que já
era feito antes:

```cpp
void checkScoreInput(
    int inputPin,
    int *stateStore,
    int *scoreStore,
    int op,
    void(*feedbackFunction)()
)
{
    int newButtonState = digitalRead(inputPin);

    if (newButtonState != *stateStore && newButtonState == HIGH) {
        // Placar não deve ficar negativo, então a operação
        // só é feita se o placar for continuar >= 0
        if (*scoreStore + op >= 0) {
            *scoreStore += op;
            printScore();
            feedbackFunction();
        }
    }

    *stateStore = newButtonState;
}
```

E mais simples ainda ficou o `checkButtons()`, que apenas chama essa função com
os parâmetros corretos:

```cpp
void checkButtons()
{
    checkScoreInput(PIN_BTN_ADD_PL1, &addPlayer1ButtonState, &firstPlayerScore, +1, playFeedbackPositive);
    checkScoreInput(PIN_BTN_SUB_PL1, &subPlayer1ButtonState, &firstPlayerScore, -1, playFeedbackNegative);
    checkScoreInput(PIN_BTN_ADD_PL2, &addPlayer2ButtonState, &secondPlayerScore, +1, playFeedbackPositive);
    checkScoreInput(PIN_BTN_SUB_PL2, &subPlayer2ButtonState, &secondPlayerScore, -1, playFeedbackNegative);
}
```

Para finalizar, a última alteração é nas funções de _feedback_. Antes existia
apenas uma, que era utilizada ao incrementar o placar de um jogador. Agora são
duas, uma ao adicionar e outra ao subtrair um ponto do placar.

A antiga função `playFeedback()` apenas foi renomeada para `playFeedbackPositive()`,
e a função `playFeedbackNegative()` foi criada com a ordem inversa das notas:

```cpp
void playFeedbackPositive()
{
    buzzer.play(NOTE_C6, 120);
    delay(140);

    buzzer.play(NOTE_C7, 420);
    delay(500);
}

void playFeedbackNegative()
{
    buzzer.play(NOTE_C7, 120);
    delay(140);

    buzzer.play(NOTE_C6, 420);
    delay(500);
}
```

## Finalizando

Isso é tudo, pessoal!™

Como eu comentei no início do post, foram feitas duas alterações no Placarduino
que não foram postadas aqui. Essa foi uma delas, e foi bem simples. Mas a segunda
alteração foi a melhor de todas e merece um post separado: identificação dos
jogadores com cartões RFID. Em breve eu publico essa postagem aqui no blog.

Enquanto isso, não deixe de conferir o projeto no GitHub:

- Versão desse post: https://github.com/eduardoweiland/placarduino/tree/v1.4
- Versão mais recente: https://github.com/eduardoweiland/placarduino


[Placarduino]: /2017-10-22-placarduino-placar-eletronico-com-arduino
[buzzer adicionado para emitir sons]: /2017-10-27-adicionando-sons-no-placarduino
[Esquema de ligação dos novos push buttons]: /images/placarduino-push-buttons-diminuir-pontos.png
