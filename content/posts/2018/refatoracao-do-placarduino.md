---
title: Refatoração do Placarduino
date: 2018-01-14T11:27:54-02:00
tags: [arduino, eletronica, placarduino]
series: [placarduino]
---

Como eu já havia comentado no [último post sobre o Placarduino][], o projeto já
estava precisando de uma refatoração (quase) completa. E eu realmente fiz isso.
Então, neste post eu pretendo descrever resumidamente as principais mudanças que
eu fiz (o post sobre o RFID terá que esperar, já que essa parte também precisa de
uma refatoração).

## Que rufem os tambores

Ou melhor, o buzzer. E foi por essa parte que eu comecei a refatoração. Eu havia
criado 3 "músicas de abertura" para o Placarduino, mas para trocar entre as músicas
era necessário comentar uma parte do código e descomentar outra! E, além disso,
criar uma nova melodia era bastante trabalhoso e repetitivo.

Para resolver tudo isso, eu criei uma biblioteca MusicPlayer que "implementa"
essas músicas como métodos. Por exemplo, se eu quiser tocar o tema do Super Mário
na abertura do Placarduino, eu só preciso chamar `musicPlayer.superMarioTheme()`.
E para trocar a música, basta trocar o nome do método.

Além disso, a classe possui dois métodos auxiliares: `playNote` e `pause`. Com
isso, criar as músicas ficou mais simples e fácil de entender:

![Código para o Tema do Super Mário antes e depois da refatoração][]

Para o futuro, já estou pensando em melhorar um pouco mais essa biblioteca, para
deixá-la mais "artística". Algumas das ideias são: configurar o [andamento][] da
música e usar [duração][] de notas em vez de milissegundos. Contudo, algumas
construções mais complexas não se encaixariam nessa regra (ex.: [quiálteras][]),
então ainda preciso estudar um pouco mais como fazer isso.

**Mais detalhes:** [Refatoração completa do MusicPlayer][]

## Controle dos jogadores

A segunda parte da refatoração foi relacionada aos dados dos jogadores que
estavam em variáveis espalhadas pelo arquivo principal (nome dos jogadores,
pontuação, estado dos botões). Cada vez que era necessário adicionar uma
informação nova para um jogador novo, era trabalho duplicado. E ainda por cima
a leitura dos botões estava muito confusa.

Assim, surgiram duas novas classes: `RisingEdgeButton` _(como biblioteca)_ e
`PlayerControl` _(no projeto mesmo)_.

Começando pelo botão, eu até pesquisei outras bibliotecas prontas para facilitar
uso de botões, mas a maioria era complexa demais e/ou pouco flexível. As melhores
que eu encontrei só funcionavam se o botão fosse configurado como `INPUT_PULLUP`
no Arduino, o que eu não estou utilizando e nem pretendo utilizar.

Então, eu mesmo criei uma biblioteca bem simples e nada flexível, utilizando a
mesma lógica que eu já utilizava, mas movendo a leitura do botão e a variável de
estado dele para dentro de uma classe.

**Mais detalhes:** [Criação do RisingEdgeButton][]

A nova classe `PlayerControl` é, agora, a responsável por fazer a leitura dos
botões (usando o `RisingEdgeButton`) e alterar o placar de acordo com o que foi
pressionado.

No arquivo principal do projeto, a função `checkButtons()` ainda existe, mas ela
apenas chama os métodos `updateScore()` dos dois jogadores (esse é o método que
lê o input do botão e atualiza o placar, se necessário). De acordo com o retorno
da chamada dos métodos (-1 se o placar foi reduzido e +1 se foi aumentado), o
arquivo principal ainda é o responsável por tocar o som de _feedback_ e atualizar
o placar no display.

A legibilidade do código ficou muito melhor com essa alteração:

![Função checkButtons antes e depois da refatoração][]

**Mais detalhes:** [Criação do PlayerControl][]

## "Refactormap" - _roadmap_ de refatoração

Como eu já comentei no começo deste post, o uso do RFID precisa de uma refatoração
antes de ser publicado (como está agora, tudo é feito em uma única função de 60
linhas, literalmente).

Outra refatoração que eu já estou pensando é de mover toda a lógica do Placarduino
para uma classe separada. Com isso, o arquivo principal deve virar basicamente
um arquivo de configuração.

~~A refatoração está sendo feita em um branch separado, você pode conferir o
progresso no GitHub~~ branch já foi integrado - ver o [pull request][].


[último post sobre o Placarduino]: /posts/2018/01/atualizacao-do-placarduino-reducao-de-pontos/
[Código para o Tema do Super Mário antes e depois da refatoração]: /images/refatoracao-musicplayer-super-mario.png
[Função checkButtons antes e depois da refatoração]: /images/refatoracao-check-buttons.png
[andamento]: https://pt.wikipedia.org/wiki/Andamento
[duração]: https://pt.wikipedia.org/wiki/Duração_(música)
[quiálteras]: https://pt.wikipedia.org/wiki/Quiáltera
[Refatoração completa do MusicPlayer]: https://github.com/eduardoweiland/placarduino/commit/8cc6dd42ed49d8c6b172840ae588b72766309236
[Criação do RisingEdgeButton]: https://github.com/eduardoweiland/placarduino/commit/1c424fc206babfc5e2fb2fcf794144af379810cc
[Criação do PlayerControl]: https://github.com/eduardoweiland/placarduino/commit/65a17c702fc61e1eac20e00efa50540a301fca26
[pull request]: https://github.com/eduardoweiland/placarduino/pull/1
