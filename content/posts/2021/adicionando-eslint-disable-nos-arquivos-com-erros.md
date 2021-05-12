---
title: "Adicionando eslint-disable nos arquivos com erros"
date: 2021-05-12T14:09:49-03:00
tags: [eslint, javascript, ember]
---

Recentemente eu precisei atualizar um projeto em Ember que estava na versão 3.6
(bem desatualizado) para a versão 3.24 (atual LTS). Para quem conhece o Ember,
sabe que muita coisa mudou entre essas versões (Glimmer, classes nativas etc.).
E, com as alterações, o Ember também atualizou o plugin para o ESLint,
incluindo novas regras para identificar código antigo e reforçar as novas boas
práticas.

Mas, mesmo com tantas alterações, quase todo o código antigo ainda segue
funcionando (exceto onde APIs privadas eram utilizadas 🤷), graças ao [Semantic
Versioning][]. Ele não *precisa* ser atualizado para a nova sintaxe por
enquanto, isso só será necessário ao atualizar para o Ember 4.0, quando ele
for lançado.

Só que agora o ESLint está acusando erros em quase todos os arquivos 😟!

O Ember fornece alguns [codemods][Ember Codemods] para ajudar a atualizar o
código para a nova sintaxe. Mas o problema é que **nem tudo é atualizado**.
Algumas alterações precisam ser feitas manualmente, o que não é uma solução
muito viável quando existem 259 erros para serem corrigidos manualmente, mesmo
depois de rodar o `eslint --fix` e os codemods 😱.

A solução: adicionar comentários `/* eslint-disable rule-name */` em todos
os arquivos que estão com erros, especificando apenas as regras que estão
sendo violadas naquele arquivo. Dessa forma, os arquivos antigos não irão
acusar nenhum erro, mas todo código novo deverá passar no lint com as regras
novas 👌.

Mas fazer isso manualmente ainda seria muito trabalhoso. Deve existir uma
forma de automatizar isso 🤔...

Em primeiro lugar, eu precisava de um output do ESLint que fosse fácil de
parsear em outras ferramentas. O formato padrão é bom para ser lido por
humanos, mas não por máquias. Felizmente, o ESLint suporta [vários formatos
diferentes][ESLint Formatters]. Eu optei por utilizar o formato `compact`,
porque reporta cada erro em uma única linha, em um formato bem definido, de
onde é fácil extrair as informações necessárias (caminho do arquivo e nome da
regra).

Um exemplo de erro reportado no formato `compact`:

```
/home/eduardo/my-project/app/instance-initializers/global-loading-route.js: line 8, col 24, Error - Don't access `router:main` as it is a private API. Instead use the public 'router' service. (ember/no-private-routing-service)
```

É fácil de identificar que a linha começa com o caminho do arquivo, seguido por
dois-pontos, números da linha e da coluna, nível e mensagem de erro, terminando
com o nome da regra entre parênteses. Traduzindo isso para um `sed`:

```sh
$ eslint -f compact . | sed -nr 's/^([^:]+).*\((.+)\)$/\1\t\2/p'
```

O resultado disso é uma lista mais "limpa", apenas com o caminho do arquivo e o
nome da regra que falhou, separados por um tab. Como o mesmo erro pode ser
reportado mais de uma vez no mesmo arquivo, é importante adicionar a dupla
`sort | uniq`:

```sh
$ eslint -f compact . | sed -nr 's/^([^:]+).*\((.+)\)$/\1\t\2/p' | sort | uniq
```

Só o que falta fazer agora é adicionar os comentários `/* eslint-disable */` em
todos os arquivos. Eu poderia tentar agrupar todas as regras e colocar um único
comentário no começo do arquivo, porém 1) o comentário poderia ultrapassar o
limite de caracteres da linha e causar novos erros; 2) o ESLint permite vários
comentários separados, não é necessário agrupar; e 3) é mais fácil adicionar
um comentário por regra, a partir do formato de saída `compact`.

Para fazer isso, eu direcionei a saída do comando acima para um loop com
`while read` e um sed para adicionar o comentário no começo do arquivo. O
comando ficou assim:

```sh
$ eslint -f compact . | sed -nr 's/^([^:]+).*\((.+)\)$/\1\t\2/p' \
  | sort | uniq | while IFS=$'\t' read file rule ; do \
  sed -i "1s;^;/* eslint-disable $rule */\n;" "$file" ; done
```

Nesse comando, o `IFS=$'\t'` serve para separar os campos no `read` apenas
com o `tab` e não com espaços, então, mesmo se existir algum espaço no caminho
do arquivo, ele será lido corretamente. O `read file rule` vai ler uma linha
da entrada padrão (que é a saída do `uniq`), e colocar o nome do arquivo na
variável `$file` e o nome da regra na variável `$rule`.  Essas variáveis são,
então, utilizadas no `sed`, que edita o arquivo inserindo uma nova linha com o
comentário `/* eslint-disable $rule */`.

O resultado depois disso: zero falhas! 😎


[Semantic Versioning]: https://semver.org/
[Ember Codemods]: https://github.com/ember-codemods
[ESLint Formatters]: https://eslint.org/docs/user-guide/formatters/
