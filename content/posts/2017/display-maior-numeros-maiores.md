---
title: Display maior, números maiores
date: 2017-11-05T18:18:55-02:00
featuredImage: /images/placarduino-big-numbers-lcd.jpg
tags: [arduino, eletronica, placarduino]
series: [placarduino]
---

No [último post sobre o projeto do Placarduino][], eu adicionei um display de 4
linhas e 20 colunas, mas não alterei o formato de exibição do placar. Eu reservei
essa mudança para este post: exibir números maiores no placar.

Indo direto ao ponto, o resultado será como a imagem acima.

## A teoria

Os displays LCD utilizam uma matriz de pixels de 8 linhas e 5 colunas para cada
caractere. Os pixels que devem ser ligados são representados por um bit 1 e os
pixels desligados por 0. Cada caractere é formado por um vetor de 8 bytes, onde
apenas os 5 bits menos significativos de cada byte representam uma linha de pixels.

Por exemplo, a letra "E" é exibida no display que eu tenho como:

![Representação da letra E no display LCD][]

A representação em bits desse caractere é:

![Representação em bits da letra E no display LCD][]

O que equivale ao seguinte código no Arduino:

```c
byte letraE[8] = {
    B11111,
    B10000,
    B10000,
    B11110,
    B10000,
    B10000,
    B11111,
    B00000
};
```

Essas configurações de exibição são salvas na memória do controlador do LCD
utilizando o código ASCII do caractere como endereço. Por exemplo, o código da
letra E na tabela ASCII é 0x45, então esse é o endereço desse caractere na
memória.

Para exibir um caractere no display, o microcontrolador (nesse caso, o Atmega328
do Arduino) envia para o controlador do LCD o código ASCII do caractere e o
controlador busca na memória interna como ele deve ser exibido. Infelizmente
essa memória do controlador é somente leitura. Felizmente o controlador possui
uma outra área de memória que pode ser escrita - _pelo menos é o que eu encontrei
[nesse post][post-engineersgarage]_.

Essa área de memória secundária possui 64 bytes. Como cada caractere utiliza 8
bytes, isso dá 64 / 8 = 8 caracteres. A faixa de endereços dos caracteres que
podem ser configurados é de 0 a 7, que na tabela ASCII são caracteres de controle
e não tem nenhuma exibição por padrão.

Um detalhe importante a ser lembrado é que os caracteres que estão sendo exibidos
no LCD estão ligados diretamente a essa memória. Então, se você reconfigurar um
caractere, todos os caracteres com o mesmo código que já estão no LCD serão
"atualizados" para a nova exibição (descobri isso na prática).

## Criando os caracteres para exibir números grandes

O display LCD exibe caracteres de apenas 8x5 pixels. Mas, com um pouco de
criatividade, é possível combinar vários caracteres de 8x5 para formar letras,
números ou até símbolos maiores. No caso do Placarduino, isso será utilizado
apenas para exibir os números da pontuação em um tamanho maior, então só é
preciso criar os números de 0 a 9.

Eu defini que o tamanho de cada número seria 3x3 caracteres. Com isso, eu
mantenho uma linha livre para exibir o nome do jogador sobre o placar e também
posso exibir dois números lado a lado para pontuações de até 99.

O processo que eu utilizei para criar os caracteres foi bem simples: comecei do
0 e fui criando cada número até chegar no 9. Criei os caracteres customizados
conforme a necessidade, e no fim acabei utilizando exatamente os 8 disponíveis.

Para criar cada parte do número, um site que pode ajudar é esse [gerador de
caracteres customizados][]. Ele só edita um caractere por vez, então para
visualizar o resultado eu criei um sketch separado no Arduino para juntar as
partes e ver o número completo.

O resultado disso foi a criação de uma biblioteca LcdBigNumbers dentro do projeto.
Usando a [estrutura do PlatformIO][], a biblioteca ficou bem isolada do resto do
código e bem simples de utilizar, apenas com um arquivo `.h` e um `.cpp`.

### Definição da classe

O arquivo de cabeçalho da biblioteca ficou localizado em `lib/LcdBigNumbers/LcdBigNumbers.h`.
Esse arquivo é bem simples e contém a definição de uma classe com um construtor
e três métodos públicos.

```cpp
// lib/LcdBigNumbers/LcdBigNumbers.h

#ifndef PLACARDUINO_LCD_BIG_NUMBERS_H
#define PLACARDUINO_LCD_BIG_NUMBERS_H

class LiquidCrystal_I2C;

class LcdBigNumbers
{
public:
    LcdBigNumbers(LiquidCrystal_I2C *lcd);

    void init();
    void printNumber(const unsigned long int number, uint8_t column, uint8_t row);
    void printDigit(const uint8_t digit, uint8_t column, uint8_t row);

private:
    LiquidCrystal_I2C *lcd;
};

#endif // PLACARDUINO_LCD_BIG_NUMBERS_H
```

Vendo o construtor, ele recebe um ponteiro para o objeto do LCD. _[Dependency
Injection][]_ no Arduino 😄. Infelizmente, não existe uma interface padrão entre
as bibliotecas do LiquidCrystal, então foi necessário utilizar a classe
LiquidCrystal_I2C diretamente.

Além do construtor, existem também um método `init()` que será utilizado para
criar os caracteres na memória do controlador do LCD, um método `printNumber()`
para exibir um número em uma posição específica do display, e o método
`printDigit()` que exibe um único dígito no LCD também em uma posição específica.

### Configuração dos caracteres

O arquivo que possui toda a configuração dos caracteres customizados, bem como
a lógica para exibir os números maiores compostos de 9 caracteres (3x3) é
`lib/LcdBigNumbers/LcdBigNumbers.cpp`.

No começo do arquivo são incluídos os arquivos de cabeçalho necessários:

```cpp
#include <inttypes.h>
#include <binary.h>

#include <LiquidCrystal_I2C.h>

#include "LcdBigNumbers.h"
```

Logo depois, eu criei algumas constantes para identificar os caracteres
customizados, além de outros caracteres do próprio controlador do LCD que foram
utilizados:

```cpp
#define CHAR_TOP_LEFT       0
#define CHAR_TOP_RIGHT      1
#define CHAR_BOTTOM_LEFT    2
#define CHAR_BOTTOM_RIGHT   3
#define CHAR_HALF_TOP       4
#define CHAR_HALF_BOTTOM    5
#define CHAR_TOP_BOTTOM     6
#define CHAR_MIDDLE         7
#define CHAR_EMPTY        254
#define CHAR_FULL         255
```

Esses são os endereços dos caracteres na memória do controlador LCD. Os últimos
dois, `CHAR_EMPTY` e `CHAR_FULL`, já existem por padrão na memória do LCD e não
é necessário criá-los. A definição de cada pixel desses caracteres vem logo
depois, em uma matriz de bytes no formato já mencionado, utilizando o endereço
do caractere como índice do vetor:

```cpp
static uint8_t customChars[8][8] =
{
    // CHAR_TOP_LEFT
    {
        B00001,
        B00111,
        B01111,
        B01111,
        B11111,
        B11111,
        B11111,
        B11111
    },

    // CHAR_TOP_RIGHT
    {
        B10000,
        B11100,
        B11110,
        B11110,
        B11111,
        B11111,
        B11111,
        B11111
    },

    // CHAR_BOTTOM_LEFT
    {
        B11111,
        B11111,
        B11111,
        B11111,
        B01111,
        B01111,
        B00111,
        B00001
    },

    // CHAR_BOTTOM_RIGHT
    {
        B11111,
        B11111,
        B11111,
        B11111,
        B11110,
        B11110,
        B11100,
        B10000
    },

    // CHAR_HALF_TOP
    {
        B11111,
        B11111,
        B11111,
        B11111,
        B00000,
        B00000,
        B00000,
        B00000
    },

    // CHAR_HALF_BOTTOM
    {
        B00000,
        B00000,
        B00000,
        B00000,
        B11111,
        B11111,
        B11111,
        B11111
    },

    // CHAR_TOP_BOTTOM
    {
        B11111,
        B11111,
        B11111,
        B00000,
        B00000,
        B11111,
        B11111,
        B11111
    },

    // CHAR_MIDDLE
    {
        B00000,
        B00000,
        B00000,
        B11111,
        B11111,
        B00000,
        B00000,
        B00000
    }
};
```

Os caracteres que são criados com essas configurações são os seguintes:

![Todos os caracteres utilizados na biblioteca LcdBigNumbers][]

Depois de ter esses caracteres criados, cada número é configurado como uma matriz
3x3 com os códigos desses caracteres. Como a matriz tem números de 0 a 9, o
próprio número foi utilizado como índice:

```cpp
static uint8_t numberChars[10][3][3] =
{
    // 0
    {
        { CHAR_TOP_LEFT, CHAR_HALF_TOP, CHAR_TOP_RIGHT },
        { CHAR_FULL, CHAR_EMPTY, CHAR_FULL },
        { CHAR_BOTTOM_LEFT, CHAR_HALF_BOTTOM, CHAR_BOTTOM_RIGHT }
    },

    // 1
    {
        { CHAR_HALF_BOTTOM, CHAR_FULL, CHAR_EMPTY },
        { CHAR_EMPTY, CHAR_FULL, CHAR_EMPTY },
        { CHAR_HALF_BOTTOM, CHAR_FULL, CHAR_HALF_BOTTOM }
    },

    // 2
    {
        { CHAR_HALF_TOP, CHAR_HALF_TOP, CHAR_HALF_BOTTOM },
        { CHAR_HALF_BOTTOM, CHAR_MIDDLE, CHAR_HALF_TOP },
        { CHAR_FULL, CHAR_HALF_BOTTOM, CHAR_HALF_BOTTOM }
    },

    // 3
    {
        { CHAR_HALF_TOP, CHAR_HALF_TOP, CHAR_HALF_BOTTOM },
        { CHAR_EMPTY, CHAR_MIDDLE, CHAR_TOP_BOTTOM },
        { CHAR_HALF_BOTTOM, CHAR_HALF_BOTTOM, CHAR_HALF_TOP }
    },

    // 4
    {
        { CHAR_FULL, CHAR_EMPTY, CHAR_HALF_BOTTOM },
        { CHAR_FULL, CHAR_HALF_BOTTOM, CHAR_FULL },
        { CHAR_EMPTY, CHAR_EMPTY, CHAR_FULL }
    },

    // 5
    {
        { CHAR_FULL, CHAR_HALF_TOP, CHAR_HALF_TOP },
        { CHAR_HALF_TOP, CHAR_MIDDLE, CHAR_HALF_BOTTOM },
        { CHAR_HALF_BOTTOM, CHAR_HALF_BOTTOM, CHAR_BOTTOM_RIGHT }
    },

    // 6
    {
        { CHAR_TOP_LEFT, CHAR_HALF_TOP, CHAR_HALF_TOP },
        { CHAR_FULL, CHAR_HALF_TOP, CHAR_TOP_RIGHT },
        { CHAR_BOTTOM_LEFT, CHAR_HALF_BOTTOM, CHAR_BOTTOM_RIGHT }
    },

    // 7
    {
        { CHAR_HALF_TOP, CHAR_HALF_TOP, CHAR_FULL },
        { CHAR_EMPTY, CHAR_HALF_BOTTOM, CHAR_HALF_TOP },
        { CHAR_EMPTY, CHAR_FULL, CHAR_EMPTY }
    },

    // 8
    {
        { CHAR_HALF_BOTTOM, CHAR_HALF_TOP, CHAR_HALF_BOTTOM },
        { CHAR_TOP_BOTTOM, CHAR_MIDDLE, CHAR_TOP_BOTTOM },
        { CHAR_HALF_TOP, CHAR_HALF_BOTTOM, CHAR_HALF_TOP }
    },

    // 9
    {
        { CHAR_TOP_LEFT, CHAR_HALF_TOP, CHAR_TOP_RIGHT },
        { CHAR_BOTTOM_LEFT, CHAR_HALF_BOTTOM, CHAR_FULL },
        { CHAR_HALF_BOTTOM, CHAR_HALF_BOTTOM, CHAR_BOTTOM_RIGHT }
    }
};
```

Os números que serão criados com isso são:

![Números de 1 a 5 no display][]
![Números de 6 a 9 no display][]

Isso finaliza a parte trabalhosa. Agora resta apenas lógica.

## A lógica de exibição dos números

A primeira parte da lógica começa com a criação dos caracteres:

```cpp
LcdBigNumbers::LcdBigNumbers(LiquidCrystal_I2C *lcd)
{
    this->lcd = lcd;
}

void LcdBigNumbers::init()
{
    for (uint8_t i = 0; i < 8; i++) {
        this->lcd->createChar(i, customChars[i]);
    }
}
```

O método `init()` irá percorrer todo o vetor de caracteres customizados, criando
todos eles no controlador do LCD. O primeiro parâmetro do `createChar()` é o
endereço que o o caractere deve ocupar na memória, e o segundo é um vetor com os
bytes que foram o padrão de exibição.

O método `printNumber()` é utilizado para decompor um número em vários dígitos e
depois exibir cada dígito separadamente, com um pequeno espaço entre eles. O
método recebe por parâmetro a coluna e a linha do primeiro "pixel" (topo esquerdo)
onde será exibido o número.

```cpp
void LcdBigNumbers::printNumber(
    const unsigned long int number,
    uint8_t column,
    uint8_t row
)
{
    uint8_t digits[10];  // Hard limit de 10 dígitos
    uint8_t count = 0, i;
    unsigned long int moreNumbers = number;

    do {
        digits[count++] = moreNumbers % 10;
        moreNumbers /= 10;
    } while (count < 10 && moreNumbers > 0);

    for (i = 0; i < count; i++) {
        this->printDigit(digits[count - i - 1], column + 4 * i, row);
    }
}
```

**Nota: revisando o código acima durante a atualização do blog para o empress,
encontrei uma pequena grande falha de _[buffer overflow][]_ na condição do
`while`: `count <= 10` deveria ser `count < 10`. O código acima já está corrigido.**

Esse talvez seja o código mais complexo em todo o projeto até agora, mas até que
é simples. O número recebido vai sendo dividido por 10 até chegar a zero, e o
resto da divisão (que é o mesmo que o último dígito do número) é adicionado a
uma lista. No fim, essa lista terá todos os dígitos do número em ordem inversa.

Depois de decompor o número, a lista de dígitos é percorrida do final para o começo
e, para cada dígito, é chamado o outro método da biblioteca, o `printDigit()`.
Para cada dígito exibido, são avançadas 4 colunas para exibir o próximo. Como
cada número sempre ocupa 3 colunas, isso deixa uma coluna de espaço entre os
dígitos (_e isso também pode causar um pequeno bug, que não afeta o Placarduino,
porque essa coluna de espaço entre os dígitos não é apagada, então se havia outra
coisa escrita nesse espaço antes, irá continuar aparecendo entre os dígitos_).

Por fim, a lógica para exibir um dígito no display LCD. Como a definição de quais
caracteres formam o número já foi feita antes na matriz `numberChars`, o único
trabalho desse método é percorrer essa matriz, posicionar o cursor e exibir o
caractere:

```cpp
void LcdBigNumbers::printDigit(const uint8_t digit, uint8_t column, uint8_t row)
{
    if (digit > 9) {
        return;
    }

    uint8_t r, c;

    for (r = 0; r < 3; r++) {
        this->lcd->setCursor(column, row + r);

        for (c = 0; c < 3; c++) {
            this->lcd->write(numberChars[digit][r][c]);
        }
    }
}
```

## Integrando o LcdBigNumbers com o Placarduino

O uso da biblioteca para exibir os pontos dos jogadores no placar vai simplificar
a função `printScore()` atual. Mas, primeiro, a biblioteca deve ser incluída e
inicializada.

No começo do arquivo, depois dos outros `#include`'s:

```cpp
#include "LcdBigNumbers.h"
```

Um objeto LcdBigNumbers deve ser criado logo depois do LiquidCrystal_I2C, passando
um ponteiro para a variável `lcd`:

```
LiquidCrystal_I2C lcd(LCD_I2C_ADDR, LCD_COLS, LCD_ROWS);
LcdBigNumbers bigNumbers(&lcd);
```

O método `init()`, que irá criar todos os caracteres customizados, deve ser
chamado logo após a inicialização do LCD:

```cpp
void setupLCD()
{
    lcd.init();
    bigNumbers.init();

    lcd.backlight();
}
```

Por fim, a função `printScore()` é alterada para exibir os nomes dos dois
jogadores na primeira linha, um alinhado à esquerda e outro à direita. A
exibição dos nomes ainda é feita utilizando `lcd.print()`, mas a exibição dos
pontos agora é feita com `bigNumbers.printNumber()`, alinhando, como os nomes,
à esquerda e à direita.

```cpp
void printScore()
{
    lcd.clear();

    lcd.setCursor(0, 0);
    lcd.print(firstPlayerName);

    // Alinha o nome do jogador à direita
    lcd.setCursor(LCD_COLS - strlen(secondPlayerName), 0);
    lcd.print(secondPlayerName);

    bigNumbers.printNumber(firstPlayerScore, 0, 1);

    // Altera coluna inicial de acordo com valor da pontuação para
    // manter alinhado à direita
    if (secondPlayerScore >= 10) {
        bigNumbers.printNumber(secondPlayerScore, LCD_COLS - 7, 1);
    }
    else {
        bigNumbers.printNumber(secondPlayerScore, LCD_COLS - 3, 1);
    }
}
```

## Resultado

![Números grandes no Placarduino][]

Como sempre, o código está no GitHub:

- A versão desse post: https://github.com/eduardoweiland/placarduino/tree/v1.3
- Versão mais atual: https://github.com/eduardoweiland/placarduino


[último post sobre o projeto do Placarduino]: /2017-11-02-placarduino-adicionando-um-display-maior-para-exibir-mais-informacoes
[estrutura do PlatformIO]: /2017-10-31-substituindo-a-ide-do-arduino-pelo-platformio
[Números grandes no Placarduino]: /images/placarduino-big-numbers-lcd.jpg
[Representação da letra E no display LCD]: /images/display-lcd-letra-E.png
[Representação em bits da letra E no display LCD]: /images/display-lcd-letra-E-bits.png
[Todos os caracteres utilizados na biblioteca LcdBigNumbers]: /images/custom-chars-lcd-big-number.png
[Números de 1 a 5 no display]: /images/lcd-big-numbers-1-a-5.jpg
[Números de 6 a 9 no display]: /images/lcd-big-numbers-6-a-9.jpg
[post-engineersgarage]: https://www.engineersgarage.com/embedded/arduino/how-to-create-custom-characters-in-lcd-using-arduino
[gerador de caracteres customizados]: https://omerk.github.io/lcdchargen/
[Dependency Injection]: https://en.wikipedia.org/wiki/Dependency_injection
[buffer overflow]: https://pt.wikipedia.org/wiki/Transbordamento_de_dados
