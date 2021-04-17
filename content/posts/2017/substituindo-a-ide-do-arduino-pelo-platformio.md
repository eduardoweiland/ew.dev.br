---
title: Substituindo a IDE do Arduino pelo PlatformIO
date: 2017-10-31T23:35:33-02:00
featuredImage: "/images/platformio-logo.png"
tags: [arduino, eletronica, placarduino]
series: [placarduino]
---

O Arduino é, basicamente, uma plataforma de aprendizado. A IDE oficial é bem
simples e não oferece muitos recursos para usuários mais avançados. Para quem
quer manter as dependências/bibliotecas organizadas, a IDE do Arduino não ajuda
em nada. Felizmente existe o [PlatformIO][]. E, nesse post, eu vou mostrar como
eu atualizei o [Placarduino][] para utilizar a estrutura de um projeto dessa
plataforma.

---

Com as ideias de melhorias para o Placarduino, em breve ele deve se transformar
em um pequeno monstro 🙂. Várias bibliotecas externas utilizadas, em versões
específicas, algumas que poderiam ser instaladas pelo gerenciador de bibliotecas
do Arduino, outras que precisariam ser baixadas, descompactadas e copiadas para
um diretório específico, além das bibliotecas internas para separar as funções
do programa e vários arquivos no projeto principal. É, a organização do código
será fundamental para continuar desenvolvendo esse projeto.

O PlatformIO se descreve como um ecossistema de código-aberto para desenvolvimento
de Internet das Coisas. O [core][platformio-core] dele consiste em uma arquitetura
de compilação multiplataforma, um gerenciador de bibliotecas com gerenciamento
de dependências dos projetos (similar ao NPM e ao Composer), um localizador de
dependências utilizadas (que verifica os `#include`'s para saber quais bibliotecas
devem ser compiladas) e outras ferramentas.

Além do core, o PlatformIO também pode ser [integrado a várias IDEs][platformio-ide].
As IDEs mais recomendadas são o Atom e o VSCode. É essa integração do PlatformIO
com a IDE que eu vou utilizar e mostrar neste post. Para continuar, eu escolhi o
VSCode como IDE por ser mais rápido e responsivo que o Atom (_pelo menos uma vez
na vida a Microsoft fez um software decente..._).

Abrindo a IDE com o plugin instalado, existe uma opção para importar um sketch
existente do Arduino. Essa opção, basicamente, cria um novo projeto baseado no
template padrão e copia todos os arquivos do sketch do Arduino para uma pasta
`src` dentro do novo projeto. Mas não funcionou muito bem com o Placarduino.
Como o sketch atual é um repositório Git, ao fazer essa importação, o repositório
ficou dentro do src e não no projeto inteiro. Então eu decidi fazer a migração
manualmente.

Um projeto do PlatformIO é identificado por um arquivo `platformio.ini` na raiz
do diretório. Nesse arquivo são configurados a placa e o framework utilizados (o
PlatformIO suporta vários, mas eu só estou desenvolvendo o Placarduino com a
placa Arduino UNO e o framework Arduino). Esse mesmo arquivo também é utilizado
para listar as dependências de bibliotecas, que serão instaladas automaticamente.
Eu me baseei no arquivo gerado para um novo projeto e na [documentação do formato
desse arquivo][platformio-projectconf], e acabei com essa configuração:

```ini
; PlatformIO Project Configuration File
;
;   Build options: build flags, source filter
;   Upload options: custom upload port, speed and extra flags
;   Library options: dependencies, extra library storages
;   Advanced options: extra scripting
;
; Please visit documentation for the other options and examples
; http://docs.platformio.org/page/projectconf.html

[env:uno]
platform = atmelavr
board = uno
framework = arduino

lib_deps =
    Wire@1.0
    LiquidCrystal@1.3.4
    ToneLibrary@1.7.1
```

Nesse arquivo, as opções `platform`, `board` e `framework` em conjunto configuram
a placa que está sendo utilizada e como deve ser feita a compilação e "deploy"
para ela. Você pode ver [nessa página][platformio-boards] todas as opções e
combinações possíveis.

A opção `lib_deps` é uma lista das dependências do projeto. Para pesquisar as
bibliotecas disponíveis, você pode utilizar o PIO Home (ícone de casa na barra
interior do VSCode), pesquisar a biblioteca, copiar o nome e versão da aba
"Install" e colocar nesse arquivo. Após salvar esse arquivo, o PlatformIO já
vai baixar as dependências automaticamente e salvar na pasta `.piolibdeps`.

Depois de criar o `platformio.ini`, eu movi alguns arquivos para melhorar a
organização:

```
$ mkdir docs src
$ git mv Placarduino.fzz docs/
$ git mv Placarduino.ino src/
```

O resto dos arquivos necessários, o próprio PlatformIO gera automaticamente
quando o projeto é aberto na IDE.

Agora programar para Arduino já está mais parecido com programação de softwares
no século 21 😉


[Placarduino]: /2017-10-22-placarduino-placar-eletronico-com-arduino
[PlatformIO]: http://platformio.org/
[platformio-core]: http://docs.platformio.org/en/latest/core.html
[platformio-ide]: http://docs.platformio.org/en/latest/ide.html
[platformio-projectconf]: http://docs.platformio.org/en/latest/projectconf.html
[platformio-boards]: http://platformio.org/boards
