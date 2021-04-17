---
title: Cartões de acesso para o Placarduino
date: 2018-01-25T15:36:05-02:00
featuredImage: /images/placarduino-com-rfid.jpg
tags: [arduino, eletronica, placarduino]
series: [placarduino]
---

O Placarduino já está funcionando bem. Porém, os únicos jogadores ainda são o
"Jogador 1" e o "Jogador 2". Se o "Fulano" quiser jogar contra o "Beltrano",
eles terão que escolher entre 1) usarem os nomes "Jogador 1" e "Jogador 2" ou
2) trocar os nomes e recompilar o código ou 3) cada um deles pode ter um "cartão
de acesso" para identificação. Vamos de alternativa 3!

## Teoria

Para quem nunca trabalhou com RFID antes, não adianta começar montando o circuito
e nem escrevendo o código. A teoria é uma parte obrigatória para poder usar o
RFID sem surpresas. Eu poderia escrever um monte sobre o que eu aprendi aqui,
mas eu prefiro deixar o link para um dos melhores artigos que eu encontrei e que
me ajudou a entender o funcionamento das tags RFID: [MiFare Cards & Tags][]
(artigo do Adafruit, em inglês).

Uma coisa que eu acho importante deixar claro é a nomenclatura. RFID é um nome
popular, mas é genérico. Significa apenas "identificação por radiofrequência" e
pode se referir a muitas coisas (NFC é um tipo de RFID, por exemplo). Mas,
geralmente, no âmbito do Arduino, RFID se refere à tecnologia de _smart cards_
que operam a 13.56 MHz, também conhecidos apenas como [MIFARE][]. Ainda assim,
muitas lojas anunciam apenas "Leitor e Tags RFID".

## Componentes e montagem do circuito

Os componentes necessários são:

- Um módulo leitor/gravador de tags MIFARE modelo RC522. Eu estou utilizando
[este][leitor-rfid-usinainfo] - _disclaimer: não tenho nenhum vínculo com a loja_
- Algumas tags MIFARE, uma para cada jogador (geralmente são vendidas no formato
de cartão ou chaveiro, você pode usar o que preferir)
- Cabos jumper

A primeira coisa a ser feita é conectar o leitor na protoboard:

![Leitor RFID conectado na protoboard][]

Depois disso, precisamos ligar os jumpers da protoboard para o Arduino. O leitor
RFID se comunica com o microcontrolador utilizando o protocolo [SPI] _(Serial
Peripheral Interface)_. O SPI utiliza 4 linhas de comunicação: MOSI, MISO, SCK
e SS/SDA. Para as três primeiras, o Arduino possui pinos específicos que devem
ser utilizados ([assim como acontece com o I²C][post-i2c]).

- MOSI: pino 11 no Arduino Uno
- MISO: pino 12 no Arduino Uno
- SCK: pino 13 no Arduino Uno

A linha SS (marcada como SDA no leitor RFID que eu estou utilizando) pode ser
conectada em qualquer pino do Arduino. O leitor ainda possui um pino de reset,
que também pode ser conectado em qualquer pino do Arduino. Para deixar as
conexões próximas das outras, eu irei utilizar os pinos 10 para SS/SDA e 9 para
o reset.

Até o momento, as ligações devem estar assim:

![Jumpers ligando leitor RFID ao Arduino][]

Agora, ainda falta ligar a alimentação do módulo RFID. O que eu estou utilizando
(e parece que a maioria dos módulos que existem) é alimentado com 3.3V, mas
verifique o seu módulo para ter certeza. Após ligar a alimentação, o resultado
final deve ser este:

![Jumpers ligando leitor RFID ao Arduino com alimentação 3.3V][]

E na vida real fica mais ou menos assim:

![Foto do Placarduino com módulo RFID][]

## Configuração dos jogadores

Para configurar os jogadores, é necessário escrever os dados (nesse caso, apenas
o nome do jogador) nos cartões MIFARE. Inicialmente, eu pensei (e até tentei)
encaixar isso dentro do projeto principal do Placarduino, mas um problema com a
leitura do nome dos jogadores da Serial me fez trocar de ideia e utilizar um
Sketch separado, pelo menos por enquanto. (Para os curiosos: o problema foi
conseguir ler os dados da serial sem bloquear o funcionamento normal do jogo e
ainda manter o código organizado).

Como a configuração de novos jogadores não será muito utilizada (pelo menos no
meu caso), eu simplifiquei tudo e utilizei um simples sketch do Arduino (sem o
[PlatformIO][post-platformio]). A ideia ainda é tentar integrar isso no projeto
principal, mas isso pode ser feito em outro momento. Um simples sketch é o
suficiente por enquanto.

O sketch desenvolvido está no [repositório][], no diretório _tools_. Eu não vou
explicar todo o código dele aqui porque é bem semelhante ao utilizado para
leitura, e este eu vou explicar detalhadamente a seguir.

_[Link direto para commit][commit-tool] que adicionou o sketch de configuração
de jogadores (pode estar desatualizado)._

## Leitura dos nomes dos jogadores

A lógica que foi utilizada para realizar a leitura dos cartões é bem simples, em
teoria: no `loop()`, é verificado se existe um cartão na frente do leitor e, se
existir, é feita a leitura do bloco onde deve estar salvo o nome.

Como você deve saber se leu o artigo sobre a teoria dos cartões MIFARE, os dados
são armazenados e acessados em blocos de 16 bytes. Como o nome dos jogadores é
menor do que isso, só é utilizado um bloco para salvar o nome no cartão. O bloco
que foi utilizado é o bloco 1.

Além disso, o acesso aos dados do cartão exige um processo de autenticação com
uma chave de 6 bytes. Como esse não é um projeto ultrassecreto, foi mantida a
chave padrão dos cartões (0xFF 0xFF 0xFF 0xFF 0xFF 0xFF).

A alteração no código envolve, basicamente, três aspectos, descritos a seguir.

### A biblioteca SmartCard

Foi criada uma nova biblioteca do projeto, SmartCard, para simplificar o acesso
aos _smart cards_. Por baixo dos panos, é utilizada a biblioteca [MFRC522][],
mas oferecendo uma interface mais simples para a aplicação.

A classe SmartCard disponibiliza um método `read()` que realiza todas as
operações necessárias para a leitura dos dados do cartão: verifica se existe um
cartão disponível, autentica o acesso ao bloco, lê os dados, "limpa" os dados
recebidos, e finaliza o acesso. O método é autoexplicativo:

```cpp
bool SmartCard::read(
    const uint8_t blockAddr,
    uint8_t *buffer,
    uint8_t *bufferSize
)
{
    if (!this->isCardAvailable()) {
        return false;
    }

    if (!this->authenticate(blockAddr)) {
        return false;
    }

    bool success = this->readBlock(blockAddr, buffer, bufferSize);

    this->deauthenticate();

    return success;
}
```

Os outros métodos chamados pelo `read()` também são, em sua maioria,
autoexplicativos:

```cpp
bool SmartCard::isCardAvailable()
{
    // Verifica se existe um cartão na frente do leitor
    if (!this->mfrc522.PICC_IsNewCardPresent()) {
        return false;
    }

    // Verifica se é possível ler o UID do cartão
    return this->mfrc522.PICC_ReadCardSerial();
}

bool SmartCard::authenticate(const uint8_t blockAddr)
{
    MFRC522::StatusCode status;

    status = this->mfrc522.PCD_Authenticate(
        MFRC522::PICC_CMD_MF_AUTH_KEY_A,
        blockAddr,
        &(this->key),
        &(this->mfrc522.uid)
    );

    if (status != MFRC522::STATUS_OK) {
        Serial.print(F("PCD_Authenticate failed: "));
        Serial.println(this->mfrc522.GetStatusCodeName(status));

        return false;
    }

    return true;
}

void SmartCard::deauthenticate()
{
    MFRC522::StatusCode status;

    status = this->mfrc522.PICC_HaltA();

    if (status != MFRC522::STATUS_OK) {
        Serial.print(F("PICC_HaltA failed: "));
        Serial.println(this->mfrc522.GetStatusCodeName(status));
    }

    this->mfrc522.PCD_StopCrypto1();
}
```

Talvez o mais complicado seja o `readBlock()`, que realiza alguns procedimentos
adicionais. Quando os dados são lidos do cartão, a biblioteca MFRC522 adiciona
2 bytes de [CRC][] no final do buffer de destino dos dados. Mas o método de
leitura já valida esse CRC, então não é mais necessário mantê-lo no buffer.

```cpp
bool SmartCard::readBlock(
    const uint8_t blockAddr,
    uint8_t *buffer,
    uint8_t *bufferSize
)
{
    MFRC522::StatusCode status;

    status = this->mfrc522.MIFARE_Read(blockAddr, buffer, bufferSize);

    if (status != MFRC522::STATUS_OK) {
        Serial.print(F("MIFARE_Read failed: "));
        Serial.println(this->mfrc522.GetStatusCodeName(status));

        return false;
    }

    // Limpa CRC adicionado pela biblioteca no buffer, não é mais necessário
    // (CRC já é validado dentro do MIFARE_Read, que retorna
    // MFRC522::STATUS_CRC_WRONG em caso de erro)
    buffer[16] = '\0';
    buffer[17] = '\0';

    return true;
}
```

A classe também inclui um método `write()`, quando eu estava tentando integrar
a configuração dos cartões dentro do Placarduino. Não vou explicar esse método
aqui porque 1) não está mais sendo utilizado; e 2) é quase igual o `read()`.
Ele continua no repositório, caso você queira consultá-lo ou utilizá-lo no seu
projeto.

_[Link direto para o commit][commit-smartcard] que adicionou a biblioteca
SmartCard (pode estar desatualizado)._

### A classe PlayerCard

A biblioteca SmartCard facilita o acesso aos dados de qualquer _smart card_
RFID, mas ela é genérica. A classe PlayerCard entra no meio do caminho entre a
aplicação e essa biblioteca, lendo os dados do cartão e validando os valores
encontrados, para depois configurá-los no placar.

```cpp
bool PlayerCard::readPlayerNameFromCard(PlayerControl *player)
{
    byte buffer[18];
    byte length = sizeof(buffer);

    if (!this->smartCard->read(this->blockNumber, buffer, &length)) {
        return false;
    }

    Serial.print(F("Dados lidos do cartão: "));
    Serial.println((char*)buffer);

    if (!this->isValidPlayerName(buffer, strnlen((const char*)buffer, length))) {
        Serial.println(F("Nome contém caracteres inválidos, ignorando"));
        return false;
    }

    player->setName((const char*)buffer);
    return true;
}
```

Esse método, sempre que for chamado, irá tentar ler um nome de um cartão e, se
conseguir, irá colocar esse nome no PlayerControl recebido por parâmetro (é uma
classe que [foi criada durante a refatoração][post-refatoracao]).

A validação do nome lido (método `isValidPlayerName()`) é bem simples, aceitando
apenas letras e números `/^[a-z0-9]*$/i`. Isso é feito apenas como um _sanity
check_, para evitar que cartões com outros tipos de informações sejam utilizados
no placar.

### Alterações no Placarduino.ino

O arquito principal ainda precisa de algumas alterações para fazer a leitura dos
cartões RFID. Inicialmente, os objetos para acessar o módulo RFID e ler os
cartões devem ser declarados:

```cpp
#define PIN_RFID_SS      10
#define PIN_RFID_RESET    9

// Leitor RFID
byte rfidKey[] = { 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF };
SmartCard smartCard(rfidKey, PIN_RFID_SS, PIN_RFID_RESET);
PlayerCard playerCard(&smartCard, 1);
PlayerControl *playerToConfigure = &player1;
```

Após isso, é necessário inicializar a biblioteca no `setup()`. A Serial também
está sendo configurada porque é utilizada para mensagens de log:

```cpp
void setup()
{
    // ...
    // no final do setup():

    smartCard.begin();

    Serial.begin(9600);
}
```

Para realizar a leitura dos cartões em qualquer momento do jogo, também é
necessária uma pequena alteração no `loop()`, com a criação de uma nova função
`readPlayerCard()`:

```cpp
void loop()
{
    readPlayerCard();
    checkButtons();
}

// ...

void readPlayerCard()
{
    if (playerCard.readPlayerNameFromCard(playerToConfigure)) {
        if (playerToConfigure == &player1) {
            playerToConfigure = &player2;
        }
        else {
            playerToConfigure = &player1;
        }

        printScore();
    }
}
```

Essa função chama um método da classe PlayerCard criada anteriormente para fazer
a leitura do cartão. Mas, além disso, essa função também troca entre ler o nome
do jogador 1 ou do jogador 2, alterando o objeto apontado pelo ponteiro
`playerToConfigure`. Com isso, o primeiro cartão lido será o nome do jogador 1;
o segundo cartão será o nome do jogador 2; ao ler o terceiro cartão, o nome lido
será do jogador 1 novamente; e assim continua indefinidamente.

_[Link para o commit][commit-playercard] que adicionou a classe PlayerCard e as
alterações no Placarduino.ino (pode estar desatualizado)._

## Finalizando

Com essa alteração, o Placarduino chega na versão 2.0!

- Versão deste post: https://github.com/eduardoweiland/placarduino/tree/v2.0
- Versão mais recente: https://github.com/eduardoweiland/placarduino


[post-i2c]: /posts/2017/11/placarduino-adicionando-um-display-maior-para-exibir-mais-informacoes/
[post-platformio]: /posts/2017/10/substituindo-a-ide-do-arduino-pelo-platformio/
[post-refatoracao]: /posts/2018/01/refatoracao-do-placarduino/
[Leitor RFID conectado na protoboard]: /images/leitor-rfid-conectado-protoboard.png
[Jumpers ligando leitor RFID ao Arduino]: /images/jumpers-leitor-rfid-arduino.png
[Jumpers ligando leitor RFID ao Arduino com alimentação 3.3V]: /images/jumpers-alimentacao-leitor-rfid-arduino.png
[Foto do Placarduino com módulo RFID]: /images/placarduino-com-rfid.jpg
[MiFare Cards & Tags]: https://learn.adafruit.com/adafruit-pn532-rfid-nfc/mifare
[MIFARE]: https://en.wikipedia.org/wiki/MIFARE
[CRC]: https://en.wikipedia.org/wiki/Cyclic_redundancy_check
[esse leitor RFID]: http://www.acura.com.br/edge-50.php
[leitor-rfid-usinainfo]: https://www.usinainfo.com.br/rfid-arduino-e-ibutton/kit-rc522-leitor-rfid-tags-chaveiro-cartao-2582.html
[SPI]: https://www.arduino.cc/en/Reference/SPI
[repositório]: https://github.com/eduardoweiland/placarduino/
[commit-tool]: https://github.com/eduardoweiland/placarduino/commit/73e326ff9e76d27a81b5f4f75825490a281a7335
[commit-smartcard]: https://github.com/eduardoweiland/placarduino/commit/6ce647faa7de3954815deb50a8fa4df86c5fbb24
[commit-playercard]: https://github.com/eduardoweiland/placarduino/commit/a3a134dbea6cd2b06375bd2a23715088431d1fbc
[MFRC522]: https://github.com/miguelbalboa/rfid
