---
title: "Placarduino: placar eletrônico com Arduino"
date: 2017-10-22T15:08:02-02:00
featuredImage: /images/placarduino-1.0-final.jpg
tags: [arduino, eletronica, placarduino]
series: [placarduino]
---

Recentemente eu comprei um kit de eletrônica com Arduino e pensei em algum bom
projeto para começar, que pudesse ser útil e, ao mesmo tempo, bem simples. Assim,
nasceu o Placarduino, um mini placar eletrônico de uso geral para controlar a
pontuação de qualquer tipo de jogo disputado entre dois competidores.

---

Esse projeto deve começar bem simples. Talvez com o tempo eu adicione mais
algumas funcionalidades interessantes nele. Mas como MVP deve servir.

Basicamente o Placarduino será composto de um display LCD 16x2 e dois botões
para contar os pontos de cada jogador. É, só isso mesmo. Por enquanto.

Os componentes que eu utilizei nesse projeto foram:

- 1 Arduino Uno
- 1 protoboard de 830 pontos
- 1 display LCD 16x2
- 2 PushButton de 4 terminais 6x6x5
- 1 resistor 300Ω
- 1 resistor 1kΩ
- 2 resistores 10kΩ
- Alguns cabos jumper

## Conhecendo o display

O display LCD é controlado utilizando 16 pinos mostrados na imagem a seguir.

![Pinos do display LCD 16x2][]

- **VSS**: GND de alimentação
- **VDD**: 5V de alimentação
- **V0**: ajuste de contraste
- **RS**: seletor de registrador (comandos ou dados)
- **RW**: seletor da operação (leitura ou escrita)
- **E**: inicia a operação escolhida com RS e RW
- **D0 a D7**: comandos/dados
- **A**: 4.2V para o LED de backlight
- **K**: GND do LED de backlight

O Arduino possui a biblioteca LiquidCrystal para comunicação com o controlador do
display LCD, sendo necessário apenas informar os pinos utilizados. Essa biblioteca
facilita bastante a vida e o código-fonte dela é bem interessante para estudos.
Para quem estiver interessado: [código-fonte da biblioteca][] e [datasheet do
controlador][].

## Ligando o display

Está chegando a parte prática! A primeira coisa a ser feita é conectar o display
LCD diretamente na protoboard:

![LCD na protoboard][]

Depois disso, os pinos GND e 5V do Arduino são conectados às trilhas de alimentação
na lateral da protoboard (com o Arduino desconectado da alimentação, por enquanto).
A alimentação do LCD e do backlight é ligada nessas trilhas utilizando jumpers:
os pinos VSS e K são ligados no GND e o pino VDD é ligado em 5V.

O pino A (alimentação do LED de backlight) é ligado com um resistor de 300Ω em
série nos 5V. Se você não tiver um resistor desse valor pode utilizar outro
parecido, mas quanto maior o valor, menor será o brilho do LCD.

O próximo pino a ser ligado é o V0 com o GND. O V0 controla o contraste dos
caracteres do LCD. Alguns tutoriais da internet utilizam um potenciômetro para
poder ajustar o contraste dinamicamente. Mas como eu quero manter o projeto
simples, optei por utilizar um resistor de 1kΩ no lugar do potenciômetro. De
novo, se você não tiver um resistor de 1kΩ, pode utilizar outro semelhante.

O projeto deve estar parecido com isso até agora:

![Alimentação no LCD na protoboard][]

Os próximos pinos a serem ligados são os que serão utilizados para a comunicação
do Arduino com o LCD. São utilizados 2 pinos de controle (RS e E) e 4 de dados
(D4 a D7), além de mais um jumper na protoboard (RW com GND). Podem ser escolhidos
quaisquer pinos digitais do Arduino para essas ligações, desde que eles sejam
informados corretamente no código. Eu utilizei os pinos 8 a 13.

Voltando um pouco à teoria, a biblioteca LiquidCrystal suporta comunicação
utilizando 4 ou 8 bits. Aparentemente, [não existe nenhuma vantagem ao utilizar
8 bits][], e essa opção é mantida apenas por compatibilidade. Por isso são
utilizados apenas 4 pinos para dados. E o pino RW é ligado diretamente em GND
para escolher o modo de escrita, que é o único utilizado pela biblioteca,
economizando mais um pino do Arduino.

Com mais essas ligações, o LCD está totalmente conectado. Você deve ter algo
parecido com isso:

![Ligação do LCD completa: expectativa][]

## Testando o LCD

Agora que todas as ligações do LCD estão feitas é hora de testar. Esse teste já
vai ser parte do Placarduino, será a tela inicial. Na primeira linha será impresso
o nome e a versão do projeto, e na segunda linha será exibido o apelido sugerido
por alguns amigos :)

```c
#include <LiquidCrystal.h>

/*
 * CONSTANTES
 */

// Configurações de pinos
#define PIN_LCD_D7        8
#define PIN_LCD_D6        9
#define PIN_LCD_D5       10
#define PIN_LCD_D4       11
#define PIN_LCD_EN       12
#define PIN_LCD_RS       13

/*
 * GLOBAIS
 */

// Inicializa o objeto LCD configurando os pinos de comunicação
LiquidCrystal lcd(
    PIN_LCD_RS,
    PIN_LCD_EN,
    PIN_LCD_D4,
    PIN_LCD_D5,
    PIN_LCD_D6,
    PIN_LCD_D7
);

/*
 * SETUP
 */

void setup() {
    setupLCD();
    printWelcome();
}

/*
 * LOOP
 */

void loop() {
    // Não faz nada por enquanto
}

/*
 * FUNÇÕES
 */

void setupLCD()
{
    // Configura o tamanho do LCD: 16 colunas e 2 linhas
    lcd.begin(16, 2);
}

void printWelcome()
{
    lcd.clear();

    // Posiciona o cursor na primeira coluna da primeira linha
    lcd.setCursor(0, 0);
    lcd.print("PLACARDUINO v1.0");

    // Posiciona o cursor na terceira coluna da segunda linha
    lcd.setCursor(2, 1);
    lcd.print("a.k.a. DAMN");
}
```

E o resultado é esse:

![Tela inicial do Placarduino v1.0][]

## Contando os pontos

Muito bom, o LCD já está pronto para exibir o placar. Falta agora ter alguma
forma de contar os pontos para exibir. Aí entram os dois PushButtons e a teoria.

Um PushButton funciona como um simples interruptor. Quando o botão está
pressionado, o circuito é fechado e a corrente pode passar. No caso do PushButton
de 4 terminais existem 2 pares de terminais sempre "ligados" entre si, como pode
ser visto no desenho abaixo. O posicionamento dos terminais entre o desenho do
botão e do diagrama elétrico representa o caso mais comum, mas talvez isso possa
variar (não tenho certeza). Na dúvida, utilize um multímetro para verificar.

![Push Button][]

Esse posicionamento dos terminais é importante para saber como inserir os
PushButtons na protoboard corretamente. E falando nisso, é hora de colocar eles
na protoboard mesmo.

Os dois PushButtons devem ser inseridos no meio da protoboard, sobre o espaçamento
do centro. Dessa forma eles ocupam apenas um ponto de cada linha em vez de quatro,
e sobra espaço para os demais componentes. Em seguida, é ligado um jumper entre a
alimentação de 5V e um dos terminais de cada PushButton. O outro terminal é ligado
com um resistor de 10kΩ ao GND. Novamente, se você não possuir um resistor de 10kΩ
pode utilizar outro, de preferência com um valor _maior_.

![Push buttons na protoboard][]

O resistor ligado ao GND tem como objetivo proteger o circuito quando o botão
estiver pressionado. Como o outro terminal está ligado em 5V, ao pressionar o
botão essa tensão passará para o outro terminal, conectado do GND, causando um
curto-circuito. O resistor serve para evitar esse curto-circuito. Outra forma de
ligação, que não necessitaria de um resistor externo, seria ligar um terminal do
PushButton no GND e o outro em um pino de entrada do Arduino configurado como
INPUT_PULLUP (que usa um resistor interno), mas assim a lógica seria invertida
(HIGH = não pressionado e LOW = pressionado).

Para terminar a ligação, os terminais dos PushButtons ligados ao GND também devem
ser ligados a dois pinos do Arduino. Eu vou utilizar os pinos 7 e 6. A ligação
final ficou assim:

![PushButtons conectados][]

Agora o resto é feito em código. Os seguintes trechos foram adicionados no código
anterior:

Nas contantes de configurações dos pinos:

```c
#define PIN_BTN_1         7
#define PIN_BTN_2         6
```

Depois da declaração do LCD, nas variáveis globais:

```c
// Configurações dos jogadores e placar
const char firstPlayerName[] = "Jogador 1";
unsigned int firstPlayerScore = 0;
int firstButtonState = LOW;

const char secondPlayerName[] = "Jogador 2";
unsigned int secondPlayerScore = 0;
int secondButtonState = LOW;
```

As funções setup e loop ficaram assim:

```c
void setup()
{
    setupInputs();
    setupLCD();
    printWelcome();

    delay(3000);
}

void loop()
{
    checkButtons();
    printScore();

    delay(100);
}
```

E foram adicionadas mais 3 novas funções nesse arquivo: `setupInputs`,
`checkButtons` e `printScore`:

```c
void setupInputs()
{
    pinMode(PIN_BTN_1, INPUT);
    pinMode(PIN_BTN_2, INPUT);
}

void checkButtons()
{
    int newButtonState;

    newButtonState = digitalRead(PIN_BTN_1);
    if (newButtonState != firstButtonState && newButtonState == HIGH) {
        // Se o botão mudou de estado e o novo estado é pressionado, soma um ponto
        firstPlayerScore++;
    }
    firstButtonState = newButtonState;

    newButtonState = digitalRead(PIN_BTN_2);
    if (newButtonState != secondButtonState && newButtonState == HIGH) {
        // Se o botão mudou de estado e o novo estado é pressionado, soma um ponto
        secondPlayerScore++;
    }
    secondButtonState = newButtonState;
}

void printScore()
{
    // Buffer para saída do dtostrf
    // Usado para formatar os pontos alinhados à direita
    char outScore[4];

    lcd.clear();

    lcd.setCursor(0, 0);
    lcd.print(firstPlayerName);
    lcd.setCursor(13, 0);
    lcd.print(dtostrf(firstPlayerScore, 3, 0, outScore));

    lcd.setCursor(0, 1);
    lcd.print(secondPlayerName);
    lcd.setCursor(13, 1);
    lcd.print(dtostrf(secondPlayerScore, 3, 0, outScore));
}
```

## Resultado

Depois da tela de início, o placar é logo exibido. Nesse momento, os dois
PushButtons podem ser pressionados para contar os pontos: o botão da esquerda
soma um ponto para o Jogador 1 e o botão da direita soma um ponto para o
Jogador 2. Bem simples e funcional.

![Placarduino 1.0 final][]

## Novas ideias

A primeira versão do Placarduino é apenas isso mesmo. Mas futuramente podem ser
adicionadas mais coisas, como por exemplo:

- Sons (já está em andamento)
- Indicar vitória/fim de jogo (manualmente ou até automaticamente)
- Identificação dos jogadores (talvez com RFID ou impressão digital, ou pela
Serial mesmo)
- Controle de campeonatos (quem sabe)

Projeto no GitHub com todo o código criado aqui e o esquemático do Fritzing:

- Versão dessa postagem: https://github.com/eduardoweiland/placarduino/tree/v1.0
- Versão mais recente: https://github.com/eduardoweiland/placarduino


[Pinos do display LCD 16x2]: /images/display-lcd.png
[LCD na protoboard]: /images/lcd-na-protoboard.png
[Alimentação no LCD na protoboard]: /images/placarduino-alimentacao-lcd.png
[Ligação do LCD completa: expectativa]: /images/placarduino-ligacao-lcd-completa-expectativa.png
[Tela inicial do Placarduino v1.0]: /images/placarduino-tela-inicial.jpg
[Push Button]: /images/push-button.png
[Push buttons na protoboard]: /images/placarduino-push-buttons.png
[PushButtons conectados]: /images/placarduino-push-buttons-conectados.png
[Placarduino 1.0 final]: /images/placarduino-1.0-final.jpg
[código-fonte da biblioteca]: https://github.com/arduino-libraries/LiquidCrystal/blob/master/src/LiquidCrystal.cpp
[datasheet do controlador]: https://www.sparkfun.com/datasheets/LCD/HD44780.pdf
[não existe nenhuma vantagem ao utilizar 8 bits]: http://forum.arduino.cc/index.php?topic=125347.0
