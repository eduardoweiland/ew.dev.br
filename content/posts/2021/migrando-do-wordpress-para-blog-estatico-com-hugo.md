---
title: Migrando do Wordpress para blog estático com Hugo
date: 2021-04-17T16:48:50-03:00
tags: [wordpress, hugo, blog]
---

Migrei meu blog do Wordpress para um blog estático com [Hugo][]. Quer saber
como foi o processo? Continue lendo 😁

## Motivação

Eu sempre tive _vontade_ de migrar meu blog para um site estático, mas sempre me
faltava um incentivo a mais. Eu cheguei a iniciar projetos com várias ferramentas
de geração de sites estáticos, como [Hexo][], [Eleventy][] e [Empress Blog][],
mas eu sempre acabava desistindo no meio do caminho.

Porém, com a recente alta do dólar, o custo de manter um servidor no ar aumentou
consideravelmente. E, considerando que existem várias alternativas que permitem
hospedar um site estático gratuitamente, pagar para manter um blog (que quase
nunca é atualizado) no ar não faz nenhum sentido.

## Por que Hugo?

Fácil de responder. Só entrei no [Jamstack] e peguei o primeiro que **não** era
feito em JS. Por que **não** em JS? Se eu precisar customizar alguma coisa, já
aproveito para aprender algo novo 😉.

Além disso, quando comecei a pesquisar mais sobre o Hugo, para decidir se
usaria ele ou não, descobri que é muito fácil de instalar e começar a usar, é
simples e prático para fazer customizações no tema, tem uma boa documentação e,
principalmente, bons temas. Estava decidido, ia ser Hugo.

## Criando o projeto

Para começar a usar o Hugo, basta baixar o executável da página de Releases do
GitHub, extrair e colocar em um diretório que esteja na $PATH. Em seguida, rodar
os comandos:

```
$ hugo new nome-do-site
$ cd nome-do-site
```

Em seguida, é necessário instalar um tema (o Hugo não vem com nenhum tema padrão).
É só escolher um dos [temas disponíveis][hugo-themes] e seguir o passo-a-passo da
instalação do tema. Em geral, os temas são instalados como um [submódulo][git-submodule]
no repositório Git. Mas os arquivos também podem ser copiados para a pasta de temas
sem problemas (embora não seja o recomendado).

No meu caso, estou usando o excelente tema [Hugo Coder][], que é instalado com o
comando:

```
$ git submodule add https://github.com/luizdepra/hugo-coder.git themes/hugo-coder
```

Agora só falta o conteúdo.

## Migrando os posts do Wordpress

O conteúdo no Hugo é escrito em Markdown, mas os posts do Wordpress não estão
em Markdown. 🤔 Será necessária alguma ferramenta pra converter isso...

Não encontrei muitas ferramentas para fazer essa conversão, mas [essa aqui][wp-to-md]
funcionou razoavelmente bem, convertendo os cabeçalhos, links, imagens, listas de
itens, negrito, itálico... exceto os código-fontes 😒. No fim, precisei revisar
todos os posts manualmente¹ e acabei descartando alguns que eram muito antigos,
lá de 2012, 2013 e 2014. Afinal, ninguém precisa mais de posts falando sobre
desenvolvimento para Android 2.3 usando [ActionBarSherlock][]!

> _¹ Fiz essa exportação dos posts para markdown e o review dos textos em 2020,
> quando eu estava testando o Empress Blog. Abandonei o Empress Blog, mas pelo
> menos os posts puderam ser aproveitados 👍._

No fim, migrei apenas 13 posts do Wordpress, então revisá-los manualmente não
foi um problema. Até encontrei um possível [buffer overflow][] no código do
Placarduino 😬.

Comentários eu simplesmente descartei. Talvez, futuramente, eu adicione um
sistema de comentários no blog, e, talvez, exista alguma forma de importar os
comentários anteriores. Mas, se não houver, não vejo problemas, posso ficar sem
os comentários antigos.

De métricas, que, antes, eu usava apenas o que Jetpack do Wordpress oferecia,
não pretendo adicionar nada nesse novo blog. Zero cookies, zero tracking, 100%
privacidade 💯.

## Build e Deploy

Decidi hospedar o repositório desse blog e também a página web publicada no
GitLab. Eu poderia ter usado o GitHub também, mas preferi o GitLab, porque ele
é Software Livre.

A configuração do GitLab CI foi fácil. A [documentação do Hugo][hugo-deploy-gitlab]
já possui um modelo do arquivo `.gitlab-ci.yml` pronto para uso, é só copiar e
colar no projeto e está funcionando. Eu só precisei alterar a imagem Docker
para utilizar a versão _extended_ do Hugo, que suporta SCSS, que é necessário
para o tema que eu estou utilizando.

## Finalizando

Historicamente, sempre que eu faço atualizações no blog, eu começo uma "temporada"
de atualizações. Foi assim em 2012-2014, quando eu migrei das páginas de usuário
do SourceForge para o OpenShift, e em 2017-2018 quando eu migrei do OpenShift
para o DigitalOcean. Agora em 2021, migrando do DigitalOcean para o GitLab,
só o tempo dirá quanto tempo essa nova temporada irá durar...


[Hugo]: https://gohugo.io/
[Hexo]: https://hexo.io/
[Eleventy]: https://www.11ty.dev/
[Empress Blog]: https://github.com/empress/empress-blog
[Jamstack]: https://jamstack.org/generators/
[hugo-themes]: https://gohugo.io/
[git-submodule]: https://git-scm.com/book/en/v2/Git-Tools-Submodules
[Hugo Coder]: https://github.com/luizdepra/hugo-coder/
[wp-to-md]: https://github.com/lonekorean/wordpress-export-to-markdown
[ActionBarSherlock]: http://actionbarsherlock.com/
[buffer overflow]: /posts/2017/11/display-maior-numeros-maiores/
[hugo-deploy-gitlab]: https://gohugo.io/hosting-and-deployment/hosting-on-gitlab/
