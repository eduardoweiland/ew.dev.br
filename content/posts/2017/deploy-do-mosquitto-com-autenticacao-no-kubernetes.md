---
title: Deploy do Mosquitto com autenticação no Kubernetes
date: 2017-09-22T11:10:45-03:00
featuredImage: /images/mosquitto-kubernetes.png
tags: [docker, kubernetes, mosquitto, mqtt, nodejs]
---

O [Mosquitto][] é um broker MQTT de código aberto e desenvolvido com apoio da
Eclipse Foundation. O MQTT é um protocolo de comunicação no estilo [pub/sub][],
muito utilizado em aplicações de Internet das Coisas. Neste post, vou mostrar
como configurar e publicar o Mosquitto com suporte a criptografia, autenticação
e autorização em um cluster do Kubernetes.

Esse artigo assume que você já tem alguma experiência com Docker e possui
acesso a um cluster Kubernetes para fazer o deploy do Mosquitto.

---

Antes de mais nada, precisamos colocar o Mosquitto em um container. Já existem
algumas imagens prontas no Docker Hub, mas, além dele, também vamos precisar de
um plugin para fornecer a autenticação. Vou utilizar o [mosquitto-auth-plug][],
que é um dos plugins mais completos. Para isso, eu criei esse Dockerfile que
instala o Mosquitto e também o plugin:

```dockerfile
FROM alpine:3.6

EXPOSE 1883

RUN apk add --update --no-cache mosquitto libcurl libssl1.0 && \
    rm -rf /var/cache/apk

ADD auth-plug-build-config.mk /

RUN apk add --update --no-cache --virtual build-deps \
        mosquitto-dev git build-base curl-dev openssl-dev && \
    git clone git://github.com/jpmens/mosquitto-auth-plug.git \
        /mosquitto-auth-plug && \
    cd /mosquitto-auth-plug && \
    mv /auth-plug-build-config.mk config.mk && \
    make && \
    cp auth-plug.so /usr/local/lib/auth-plug.so && \
    cd / && \
    rm -rf /mosquitto-auth-plug && \
    apk del --purge build-deps && \
    rm -rf /var/cache/apk

ADD mosquitto.conf /etc/mosquitto/mosquitto.conf
ADD auth-plug.conf /etc/mosquitto.d/auth-plug.conf

CMD ["mosquitto", "-c", "/etc/mosquitto/mosquitto.conf"]
```

Esse Dockerfile começa com a imagem do Alpine 3.6. A porta 1883 é exposta, que
é a porta padrão do MQTT sem SSL. O MQTT com criptografia utiliza a porta 8883
por padrão, mas vamos configurar o SSL mais adiante, então, por enquanto, vamos
deixar assim.

Depois disso, são instalados os pacotes do Mosquitto e outras dependências de
runtime para o auth-plug. O plugin suporta vários backends de autenticação,
mas iremos utilizar apenas o HTTP, então essas são as dependências necessárias.
Se você quiser utilizar outros backends, as dependências podem ser diferentes.

O comando seguinte adiciona para dentro do container o arquivo de configuração
da compilação do plugin. Esse é o arquivo que configura quais backends serão
suportados. Um modelo desse arquivo é incluído no projeto ([config.mk.in][]).
As únicas alterações que eu fiz foi habilitar apenas o HTTP e configurar o
caminho para as bibliotecas do Mosquitto em /usr. Resumindo, o arquivo ficou
assim:

```makefile
# Select your backends from this list
BACKEND_CDB ?= no
BACKEND_MYSQL ?= no
BACKEND_SQLITE ?= no
BACKEND_REDIS ?= no
BACKEND_POSTGRES ?= no
BACKEND_LDAP ?= no
BACKEND_HTTP ?= yes
BACKEND_JWT ?= no
BACKEND_MONGO ?= no
BACKEND_FILES ?= no

# Specify the path to the Mosquitto sources here
# MOSQUITTO_SRC = /usr/local/Cellar/mosquitto/1.4.12
MOSQUITTO_SRC = /usr

# Specify the path the OpenSSL here
OPENSSLDIR = /usr

# Specify optional/additional linker/compiler flags here
# On macOS, add
#	CFG_LDFLAGS = -undefined dynamic_lookup
# as described in https://github.com/eclipse/mosquitto/issues/244
#
# CFG_LDFLAGS = -undefined dynamic_lookup  -L/usr/local/Cellar/openssl/1.0.2l/lib
# CFG_CFLAGS = -I/usr/local/Cellar/openssl/1.0.2l/include -I/usr/local/Cellar/mosquitto/1.4.12/include
CFG_LDFLAGS =
CFG_CFLAGS =
```

Em seguida, aparece uma única instrução RUN com vários comandos. Isso é assim
para deixar a imagem o menor possível, já que o Docker separa a imagem final em
várias camadas, sendo que cada instrução do Dockerfile cria uma camada separada.
Então, nessa única instrução RUN, são instalados os pacotes necessários para a
compilação, o repositório é clonado, o projeto é compilado, a biblioteca
`auth-plug.so` é copiada para o local adequado e, por fim, tudo o que é
desnecessário é removido. O resultado é que essa camada contém um único arquivo,
o `auto-plug.so`.

No final do Dockerfile, são adicionados os arquivos de configuração do Mosquitto
e do auth-plug. O Mosquitto possui [diversas opções de configuração][mosquitto-config],
que não serão abordadas aqui. O principal a ser garantido é que a opção
`include_dir` aponte para `/etc/mosquitto.d`, para onde é copiado o arquivo de
configuração do auth-plug, que contém o seguinte:

```
auth_plugin /usr/local/lib/auth-plug.so
auth_opt_backends http

auth_opt_http_hostname auth-server
auth_opt_http_port 8000

# POST /auth com username e password
auth_opt_http_getuser_uri /auth

# POST /superuser com username
auth_opt_http_superuser_uri /superuser

# POST /acl com username, clientid, topic e acc(1 = read, 2 = read-write)
auth_opt_http_aclcheck_uri /acl
```

Com o Mosquitto pronto, agora resta criar a API de autenticação. Para esse
exemplo, eu vou utilizar uma simples aplicação em NodeJS com Express. O código
básico da aplicação é mostrado abaixo (os módulos auth e acl não são
incluídos aqui porque a lógica deles depende de cada aplicação):

```js
const express = require('express');
const bodyParser = require('body-parser');

const { checkPassword } = require('./auth');
const { validateTopic } = require('./acl');

const app = express();

app.use(bodyParser.urlencoded({ extended: true }));

function allow(res) {
  res.sendStatus(200);
}

function deny(res) {
  res.sendStatus(403);
}

function error(e, res) {
  console.error(e);
  res.sendStatus(500);
}

// POST /auth com username e password
app.post('/auth', function (req, res) {
  try {
    let { username, password } = req.body;

    console.log('auth: ', { username, password });

    if (checkPassword(username, password)) {
      allow(res);
    }
    else {
      deny(res);
    }
  }
  catch (e) {
    error(e, res);
  }
});

// POST /superuser com username
app.post('/superuser', function (req, res) {
  // Nesse exemplo eu não usei superusuários, então sempre é negado
  deny(res);
});

// POST /acl com username, clientid, topic e acc(1 = read, 2 = read-write)
app.post('/acl', function (req, res) {
  try {
    let { username, clientid, topic, acc } = req.body;

    console.log('acl: ', { username, clientid, topic, acc });

    if (validateTopic(username, topic, acc)) {
      allow(res);
    }
    else {
      deny(res);
    }
  }
  catch (e) {
    error(e, res);
  }
});

app.listen(8000, () => {
  console.log('mosquitto-auth listening on port 8000');
});
```

E o Dockerfile para construir a imagem dessa API é bem simples:

```dockerfile
FROM node:6-alpine

EXPOSE 8000
WORKDIR /app
ADD . /app

RUN npm install

CMD npm start
```

Agora, antes de publicar tudo isso no Kubernetes, vamos rodar localmente para
garantir que tudo está funcionando. Usando um arquivo do docker-compose isso é
uma tarefa bem simples:

```yaml
version: "3"
services:
  auth:
    build:
      context: ./auth

  broker:
    build:
      context: ./broker
    links:
      - auth:auth-server
    ports:
      - 1883:1883
```

Agora, utilizando um cliente MQTT qualquer já é possível verificar se a
autenticação está funcionando, conectando no broker em `localhost:1883`.

```
$ mosquitto_pub -h localhost -p 1883 -t 'topico' \
  -m 'mensagem' -u usuario -P senha_incorreta
Connection Refused: not authorised.
Error: The connection was refused.

$ mosquitto_pub -h localhost -p 1883 -t 'topico' \
  -m 'mensagem' -u usuario -P senha_correta
```

Nenhum erro reportado com a senha correta = sucesso!

---

Agora está quase tudo pronto para publicar no Kubernetes. Só é necessário
enviar as imagens para um Registry Docker que o cluster consiga acessar. Eu
utilizei o Registry do GitLab.com e criei um [ImagePullSecret][] no Kubernetes
para acessá-lo, mas você pode utilizar o que achar melhor.

Os recursos necessários no Kubernetes para publicar o Mosquitto com autenticação
são, basicamente, dois: um Deployment e um Service. O Mosquitto que iremos
publicar irá rodar apenas como um servidor, sem nenhum tipo de redundância.
Teremos apenas um container rodando o Mosquitto e um container rodando a API.
Para simplificar, vamos deixar tudo isso em um único Pod (não faça isso em
produção).

O Deployment ficou assim:

```yaml
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: mqtt-broker

spec:
  template:
    metadata:
      labels:
        app: mqtt
    spec:
      hostAliases:
        # O plugin de autenticação do Mosquitto conecta na
        # API em `auth-server`. Como os containers estão no
        # mesmo Pod, isso deve ser mapeado para localhost.
        - ip: "127.0.0.1"
          hostnames:
            - "auth-server"

      containers:
        - name: mosquitto-broker
          featuredImage: registry.gitlab.com/namespace/mosquitto:latest
          resources:
            requests:
              cpu: 100m
              memory: 100Mi
          ports:
            - containerPort: 1883

        - name: mosquitto-auth
          featuredImage: registry.gitlab.com/namespace/mosquitto-auth:latest
          resources:
            requests:
              cpu: 100m
              memory: 50Mi
          ports:
            - containerPort: 8000

      imagePullSecrets:
        - name: gitlab-registry
```

E agora, no Service, é onde iremos configurar o SSL. O cluster Kubernetes que eu
estou utilizando está rodando na Amazon Web Services, e a AWS fornece um serviço
de certificados com renovação automática e que pode ser vinculado a um LoadBalancer
de forma bem simples.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mosquitto-broker
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-backend-protocol: tcp
    service.beta.kubernetes.io/aws-load-balancer-ssl-cert: "arn do certificado na AWS"
    service.beta.kubernetes.io/aws-load-balancer-ssl-ports: "mqtts"

spec:
  selector:
    app: mqtt

  type: LoadBalancer

  ports:
    - port: 8883
      name: mqtts
      targetPort: 1883
      protocol: TCP
```

Com essa configuração de serviço, a terminação SSL é feita pelo LoadBalancer da
AWS e a conexão TCP sem criptografia é redirecionada para o Pod do Mosquitto no
Kubernetes. Toda essa "mágica" é configurada pelas anotações adicionadas nos
metadados do serviço.

Agora esses arquivos podem ser aplicados com `kubectl apply -f` e em breve os
containers estarão em execução com o LoadBalancer configurado. Para obter o
endereço do LoadBalancer que deve ser utilizado para conectar no broker podemos
utilizar o comando `kubectl describe service mosquitto-broker`. Na saída desse
comando, uma linha identificada com LoadBalancer Ingress deve apresentar um
endereço parecido com `xxxxxxxxxxxxxxxxxxxxx-xxxxxxxx.sa-east-1.elb.amazonaws.com`.
Depois de configurar o DNS para direcionar o domínio configurado no certificado
SSL para o LoadBalancer, podemos fazer novamente o teste com o mosquitto_pub:

```
$ mosquitto_pub -h mqtt.dominio.com -p 8883 -t 'topico' \
  -m 'mensagem' -u usuario -P senha_incorreta \
  --cafile /etc/pki/tls/certs/ca-bundle.crt
Connection Refused: not authorised.
Error: The connection was refused.

$ mosquitto_pub -h mqtt.dominio.com -p 8883 -t 'topico' \
  -m 'mensagem' -u usuario -P senha_correta \
  --cafile /etc/pki/tls/certs/ca-bundle.crt
```

Nenhum erro reportado com a senha correta = sucesso!

Agora você tem um broker Mosquitto rodando em um cluster Kubernetes com
criptografia SSL, autenticação de usuários e autorização de acesso aos tópicos.


[Mosquitto]: https://mosquitto.org/
[pub/sub]: https://en.wikipedia.org/wiki/Publish%E2%80%93subscribe_pattern
[mosquitto-auth-plug]: https://github.com/jpmens/mosquitto-auth-plug
[config.mk.in]: https://github.com/jpmens/mosquitto-auth-plug/blob/master/config.mk.in
[mosquitto-config]: https://mosquitto.org/man/mosquitto-conf-5.html
[ImagePullSecret]: https://kubernetes.io/docs/concepts/containers/images/#specifying-imagepullsecrets-on-a-pod
