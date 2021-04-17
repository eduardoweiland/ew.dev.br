---
title: Usando o Ansible para compilar uma biblioteca
date: 2017-09-11T22:08:35-03:00
featuredImage: /images/ansible-logo.png
tags: [ansible]
---

O Ansible é uma ferramenta muito útil para várias tarefas, devido, principalmente,
à sua flexibilidade. A grande quantidade de módulos existentes torna fácil
automatizar qualquer tipo de processo, mesmo quando o Ansible não seja a opção
mais óbvia.

Por exemplo, quando você precisa compilar um pacote que necessita de etapas
adicionais de configuração: isso pode ser automatizado pelo Ansible.

Vou mostrar aqui um exemplo de como eu criei um playbook para compilar uma
biblioteca de autenticação para o Mosquitto, o [mosquitto-auth-plug][].

---

Para começar, a compilação precisa de algumas dependências. Vamos criar uma
role no Ansible para instalar as dependências desse pacote (aqui eu utilizei
apenas o DNF para instalar os pacotes no Fedora).

```yaml
# roles/install-dependencies/tasks/main.yml
---
- name: Install build dependencies
  become: true
  dnf:
    name: "{{ item }}"
    state: latest
  with_items:
    - mosquitto-devel
    - openssl-devel
    - curl-devel
    - git
    - "@development-tools"
```

Agora vamos criar o playbook e adicionar essa role para testar.

```yaml
# build.xml
---
- hosts: local
  roles:
    - install-dependencies
```

A execução é feita com o comando `ansible-playbook`, utilizando a flag `-K` para
pedir a senha de `sudo`, que é necessária para instalar os pacotes com o DNF. O
inventário utilizado aqui possui apenas o `localhost` configurado:

```
$ ansible-playbook -K -i inventory build.yml
```

Se tudo der certo, agora você já possui todas as dependências necessárias
instaladas. Hora de configurar a compilação, que envolve três etapas:

1. Baixar o código-fonte do repositório git
2. Adicionar o arquivo de configuração da compilação (config.mk)
3. Rodar o make para compilar a biblioteca

Colocando tudo isso em uma segunda role separada:

```yaml
# roles/build/tasks/main.yml
---
- name: Clone source code repository
  git:
    repo: https://github.com/jpmens/mosquitto-auth-plug.git
    dest: /tmp/mosquitto-auth-plug

- name: Render build options
  template:
    src: config.mk.j2
    dest: /tmp/mosquitto-auth-plug/config.mk

- name: Build library
  make:
    chdir: /tmp/mosquitto-auth-plug
```

O arquivo de configuração é renderizado pelo módulo template do Ansible, então
é possível utilizar variáveis, condicionais, laços, e outras estruturas para
"montar" esse arquivo dinamicamente. Eu não precisei utilizar nada disso, então
o template é apenas copiado como está para o caminho especificado. A
configuração que eu utilizei foi essa:

```jinja2
# roles/build/templates/config.mk.j2
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
MOSQUITTO_SRC = /usr

# Specify the path the OpenSSL here
OPENSSLDIR = /usr

# Specify optional/additional linker/compiler flags here
CFG_LDFLAGS =
CFG_CFLAGS =
```

Depois de adicionar essa role no playbook `build.yml`, o comando `ansible-playbook`
pode ser executado novamente com os mesmos parâmetros. Pronto, agora a biblioteca
deve estar compilada em `/tmp/mosquitto-auth-plug/auth-plug.so` 😀.

Esse foi um exemplo bem básico de como o Ansible pode ser utilizado para compilar
um pacote utilizando os módulos git, template e make. Muita coisa ainda pode ser
melhorada, mas pelo menos não será mais necessário realizar todas essas etapas
manualmente 🙂.


[mosquitto-auth-plug]: https://github.com/jpmens/mosquitto-auth-plug
