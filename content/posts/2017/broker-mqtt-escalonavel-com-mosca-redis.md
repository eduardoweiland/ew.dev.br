---
title: Broker MQTT escalonável com Mosca + Redis
date: 2017-10-27T22:35:40-02:00
featuredImage: /images/mosca.png
tags: [docker, kubernetes, mosca, mqtt, nodejs]
---

Em uma postagem anterior, eu mostrei como configurar um [broker MQTT no Kubernetes
com o Mosquitto][post-mosquitto-k8s]. Mas o suporte do Mosquitto para a [criação
de clusters usando bridges][] é muito limitado. Felizmente, existe uma solução
muito mais simples, que utiliza o Redis para criar o cluster e o [Mosca][] para
suporte ao protocolo MQTT.

Essa ideia surgiu quando eu encontrei um artigo no Medium descrevendo basicamente
a mesma arquitetura que eu irei mostrar aqui. O artigo original (em inglês) pode
ser acessado em _[How to Build an High Availability MQTT Cluster for the Internet
of Things][medium-link]_.

## Criação do Broker

De forma resumida, pode-se dizer que o Mosca é um _framework_ para criação de um
broker MQTT. O Mosca em si não implementa nenhum suporte a pub/sub (que é
necessário para o funcionamento do MQTT), mas, em vez disso, permite utilizar
outras implementações como backend (incluindo Mosquitto, Redis, MongoDB e outros).
O Mosca se preocupa apenas com o protocolo MQTT.

Bom, com o funcionamento do Mosca bem entendido, o próximo passo é criar o broker.
No caso do Mosca, isso significa criar uma aplicação NodeJS e instalar as
dependências pelo NPM. E as única dependências necessárias são o `mosca` e um
backend, nesse caso o `redis`.

```
$ npm i -s mosca redis
```

Esse comando deve instalar as duas dependências e adicioná-las ao `package.json`.
Em seguida, é criado o arquivo principal do projeto, que vai iniciar o Mosca e
configurar o backend utilizado. Eu chamei esse arquivo de `app.js`.

```js
const url = require('url');
const mosca = require('mosca');
const authorizer = require('./authorizer');

const redis = url.parse(process.env.REDIS_URL);

const settings = {
  port: parseInt(process.env.MQTT_PORT) || 1883,
  backend: {
    type: 'redis',
    redis: require('redis'),
    host: redis.hostname,
    port: redis.port,
    db: parseInt(redis.path.slice(1)),
    password: redis.password,
  },
};

console.log('Broker MQTT iniciando...');

const server = new mosca.Server(settings);

server.on('ready', () => {
  // Configura callbacks de autenticação e autorização
  server.authenticate       = authorizer.authenticate;
  server.authorizePublish   = authorizer.authorizePublish;
  server.authorizeSubscribe = authorizer.authorizeSubscribe;

  console.log('Broker MQTT aguardando conexões na porta', settings.port);
});

server.on('published', (packet, client) => {
  console.log(
    'Message published on topic',
    packet.topic,
    ':',
    packet.payload.toString(),
  );
});
```

O módulo `authorizer` que foi incluído nesse arquivo possui a lógica de autenticação
e autorização dos clientes. Como autenticação não é o foco desse post, irei deixar
aqui apenas um template de como os métodos podem ser implementados:

```js
module.exports.authenticate = function authenticate(
  client,
  username,
  passwordBuffer,
  callback,
) {
  if (usuarioESenhaEstaoCorretos) {
    // Você pode adicionar informações (ex.: permissões)
    // sobre o cliente no objeto client, se quiser:
    client.superuser = true;
    callback(null, true);
  }
  else {
    callback(null, false);
  }
};

module.exports.authorizePublish = function authorizePublish(
  client,
  topic,
  payload,
  callback,
) {
  // Você pode verificar dados adicionados
  // no objeto cliente durante autenticação:
  if (client.superuser) {
    callback(null, true);
  }
  else {
    callback(null, false);
  }
};

module.exports.authorizeSubscribe = function authorizeSubscribe(
  client,
  topic,
  callback,
) {
  // Você pode verificar dados adicionados
  // no objeto cliente durante autenticação:
  if (client.superuser) {
    callback(null, true);
  }
  else {
    callback(null, false);
  }
};
```

Agora, para executar o broker, basta rodá-lo como uma aplicação NodeJS comum (sem
esquecer de passar a variável de ambiente com a URL do Redis):

```
$ REDIS_URL=redis://localhost:6379/0 node app.js
```

## Mosca em um container

Um Dockerfile bem simples para adicionar essa aplicação do Mosca para um container
Docker seria assim:

```dockerfile
FROM node:6-alpine

EXPOSE 1883
WORKDIR /app
ADD package.json /app/package.json

RUN apk add --update --no-cache --virtual .build-deps \
        build-base zeromq-dev && \
    npm install && \
    apk del --purge .build-deps && \
    rm -rf /var/cache/apk

ADD . /app

CMD node app.js
```

## Mosca no Kubernetes

Tendo a aplicação com o Mosca rodando dentro de um container, para publicar em um
cluster do Kubernetes não é muito diferente do que [publicar o Mosquitto][post-mosquitto-k8s].
E, para escalonar o cluster, um _[Horizontal Pod Autoscaler][]_ deve resolver.


[post-mosquitto-k8s]: /posts/2017/09/deploy-do-mosquitto-com-autenticacao-no-kubernetes/
[criação de clusters usando bridges]: https://stackoverflow.com/questions/36283197/mqtt-mosquitto-bridge-horizontal-scaling/36283565#36283565
[Mosca]: http://www.mosca.io
[medium-link]: https://medium.com/@lelylan/how-to-build-an-high-availability-mqtt-cluster-for-the-internet-of-things-8011a06bd000
[Horizontal Pod Autoscaler]: https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/
