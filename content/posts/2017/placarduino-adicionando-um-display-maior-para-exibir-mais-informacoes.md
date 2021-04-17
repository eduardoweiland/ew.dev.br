---
title: "Placarduino++: adicionando um display maior para exibir mais informações"
date: 2017-11-02T20:10:24-02:00
featuredImage: /images/placarduino-display-20x4.png
tags: [arduino, eletronica, placarduino]
series: [placarduino]
---

Com várias ideias para o projeto do Placarduino, o display de 16x2 não será
suficiente para exibir todas as informações. E além disso, da forma como o
display está montado hoje, são utilizados muitos pinos do Arduino, que poderiam
ser aproveitados para outras funções. Então, nesse post, eu vou mostrar como eu
fiz a substituição do display 16x2 por um display 20x4 com um módulo adaptador
I²C.

---

Antes de mais nada: a escolha do display. Eu utilizei um LCD que já veio com o
controlador I²C soldado na parte de trás do display. Existe a possibilidade de
comprar apenas o módulo adaptador e soldar ao display ou montar em uma protoboard.

Apenas para deixar claro: eu estou trocando o tamanho do display e também adicionando
o controlador I²C ao mesmo tempo por conveniência. Isso não significa que o display
20x4 sempre deve ser utilizado com I²C: você pode utilizar o display 20x4 ligado
diretamente ao Arduino como foi feito antes com o display 16x2, e também pode
utilizar o controlador I²C com o display 16x2.

Utilizando um controlador I²C para ligar o LCD é possível reduzir consideravelmente
os pinos utilizados: sem ele são necessários 6 pinos digitais do Arduino; com ele
são apenas 2 pinos analógicos.

O I²C (sigla de Inter-Integrated Circuit) é um barramento com comunicação serial
que utiliza apenas duas linhas: uma para o clock (SCL - Serial Clock) e outra
para os dados seriais (SDA - Serial Data). Vários periféricos podem ser conectados
ao mesmo tempo no barramento, mas sempre deve existir um nodo master para gerar
o clock e controlar a comunicação. Os outros nodos são os slaves e cada um possui
um endereço único dentro do barramento.

## Montagem

Antes de começar, o antigo display 16x2 pode ser removido da protoboard, junto
com todos os cabos e resistores que foram utilizados para ele. Como o novo display
possui o controlador soldado na parte de trás, ele não precisa ser montado na
protoboard. Vou utilizar apenas 4 cabos macho-fêmea ligados nos pinos do controlador.

![Display LCD utilizado][]
_Display LCD que estou utilizando, com os cabos conectados no controlador I²C_

O Arduino possui dois pinos analógicos específicos para comunicação I²C: a linha
SDA deve ser ligada no pino A4 e a linha SCL deve ser ligada no pino A5 (no
Arduino UNO). Como nesse momento eu só terei o display LCD conectado no barramento
I²C, eu irei conectar esses pinos do Arduino diretamente aos pinos da placa
controladora do display. Fora isso, só é necessário ligar o GND e o Vcc (5V) e
a montagem está pronta:

![Esquema de ligação do display com I²C][]

## Atualizando o código para I²C

Com a montagem finalizada, resta atualizar o código. A primeira alteração que
precisa ser feita é incluir uma biblioteca que suporte a comunicação com o LCD
por I²C. Como [agora eu estou utilizando o PlatformIO][], eu apenas precisei
editar o `platformio.ini` e substituir a biblioteca `LiquidCrystal@1.3.4` por
`LiquidCrystal_I2C@1.1.2`(eu testei algumas outras bibliotecas, mas essa pareceu
funcionar melhor). Como foi alterada a biblioteca, também é necessário alterar
o `#include` dela.

**Diff das alterações:**

```diff
-#include <LiquidCrystal.h>
+#include <LiquidCrystal_I2C.h>
```

A única configuração necessária para comunicar com o display I²C é o endereço
dele no barramento. Os controladores já vem com um endereço configurado que pode
ser diferente entre diferentes modelos e fabricantes, mas pode ser descoberto
com um I²C scanner como mostrado em um [tutorial no site do Arduino][I2cScanner]
(em inglês). O display LCD que eu estou utilizando possui o endereço `0x3F`, mas
se o seu for diferente, lembre-se de alterar no código abaixo.

**Diff das alterações:**

```diff
-#define PIN_LCD_D7        8
-#define PIN_LCD_D6        9
-#define PIN_LCD_D5       10
-#define PIN_LCD_D4       11
-#define PIN_LCD_EN       12
-#define PIN_LCD_RS       13
+#define LCD_I2C_ADDR   0x3F
+#define LCD_COLS         20
+#define LCD_ROWS          4
```

Com a antiga biblioteca, os pinos utilizados na comunicação eram configurados no
construtor do objeto e o tamanho do display era configurado com o método `begin()`.
Com essa nova biblioteca tudo isso é feito no construtor.

**Diff das alterações:**

```diff
-// Inicializa o objeto LCD configurando os pinos de comunicação
-LiquidCrystal lcd(
-    PIN_LCD_RS,
-    PIN_LCD_EN,
-    PIN_LCD_D4,
-    PIN_LCD_D5,
-    PIN_LCD_D6,
-    PIN_LCD_D7
-);
+// Inicializa o objeto LCD configurando o endereço de comunicação e tamanho
+LiquidCrystal_I2C lcd(LCD_I2C_ADDR, LCD_COLS, LCD_ROWS);
```

A função `setupLCD()` também precisa ser alterada. Em vez de chamar o método
`lcd.begin()` com o tamanho do display, é necessário chamar o método `init()`
sem parâmetros, que irá inicializar a comunicação I²C e também o display. Além
disso, também é necessário habilitar a luz de fundo do LCD explicitamente via
código.

**Diff das alterações:**

```diff
 void setupLCD()
 {
-    // Configura o tamanho do LCD: 16 colunas e 2 linhas
-    lcd.begin(16, 2);
+    lcd.init();
+    lcd.backlight();
 }
```

Com isso feito, o display já deve estar funcionando. Mas os dados que são
exibidos nele ainda estão formatados para um display 16x2, então vamos ocupar
melhor o espaço, começando pela tela de início.

**Diff das alterações:**

```diff
 void printWelcome()
 {
     lcd.clear();

-    // Posiciona o cursor na primeira coluna da primeira linha
-    lcd.setCursor(0, 0);
+    // Posiciona o cursor na terceira coluna da segunda linha
+    lcd.setCursor(2, 1);
     lcd.print("PLACARDUINO v1.0");

-    // Posiciona o cursor na terceira coluna da segunda linha
-    lcd.setCursor(2, 1);
+    // Posiciona o cursor na quinta coluna da terceira linha
+    lcd.setCursor(4, 2);
     lcd.print("a.k.a. DAMN");
 }
```

A parte do placar que exibe a pontuação eu vou manter como está - por enquanto 😉

## Roadmap

Agora que a protoboard tem bastante espaço livre para novos componentes e o
display é grande o suficiente para exibir muitas informações, o projeto pode
evoluir muito mais. Eu já fiz uma grande mudança na tela que exibe a pontuação
(que merece um post separado) e já estou pesquisando a tecnologia de tags RFID
para identificar os jogadores. Ideias não faltam, e o conhecimento se adquire.

Como sempre, o código-fonte e o projeto do Fritzing estão no GitHub:

- Versão desse post: https://github.com/eduardoweiland/placarduino/tree/v1.2
- Versão mais recente: https://github.com/eduardoweiland/placarduino


[Display LCD utilizado]: /images/display-lcd-utilizado.jpg
[Esquema de ligação do display com I²C]: /images/placarduino-display-20x4.png
[agora eu estou utilizando o PlatformIO]: /posts/2017/10/substituindo-a-ide-do-arduino-pelo-platformio/
[I2cScanner]: http://playground.arduino.cc/Main/I2cScanner
