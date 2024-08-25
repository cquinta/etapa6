# Criando um Cluster

Vamos utilizar o Docker Playground para criar um cluster com 3 managers e 2 workers. 
para utilizar o Docker Payground é necessário criar uma conta no [Docker Hub](https://hub.docker.com/). 

## Iniciando o cluster

Na nossa arquitetura teremos os seguintes nós com os seguintes papeis: 

| Nome  | Endereço IP     | Função             |
|-------|-----------------|--------------------|
| Node1 | 192.168.0.28    | Manager (Líder)    |
| Node2 | 192.168.0.27    | Manager (Seguidor) |
| Node3 | 192.168.0.26    | Manager (Seguidor) |
| Node4 | 192.168.0.25    | Worker 1           |
| Node5 | 192.168.0.24    | Worker 2           |


Vamos iniciar o nosso Cluster: 

```bash

docker swarm init \
  --advertise-addr 192.168.0.28:2377 \
  --listen-addr 192.168.0.28:2377

```
O advertise-addr e o listen-addr geralmente são os mesmos, porém é possível anuciar o endereço em um loadbalancer neste caso o listen-addr deve ser o endereço do nó. 

O resultado será semelhante ao seguinte: 

```bash
$ docker swarm init \
  --advertise-addr 192.168.0.28:2377 \
  --listen-addr 192.168.0.28:2377
Swarm initialized: current node (7pn9os1z6y3wuryr4vivy8tse) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-5sat99b6kt3u6fbebqlxwqfosvvtgq07reybmvjbktt14q3ark-0tir78hisqyll741fp1d3n4r0 192.168.0.28:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.

$ docker node ls
ID                            HOSTNAME   STATUS    AVAILABILITY   MANAGER STATUS   ENGINE VERSION
7pn9os1z6y3wuryr4vivy8tse *   node1      Ready     Active         Leader           24.0.7

```

Para ver os tokens de worker e de manager no futuro os comandos são os seguintes: 

```bash
$ docker swarm join-token worker

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-5sat99b6kt3u6fbebqlxwqfosvvtgq07reybmvjbktt14q3ark-0tir78hisqyll741fp1d3n4r0 192.168.0.28:2377


$ docker swarm join-token manager

To add a manager to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-5sat99b6kt3u6fbebqlxwqfosvvtgq07reybmvjbktt14q3ark-5lq5ir0grnsph7wtu2e2xy6lq 192.168.0.28:2377

```

Vamos logar nos dois nós workers e executar o comando para fazer o join no cluster como worker. 

```bash
$ docker swarm join --token SWMTKN-1-5sat99b6kt3u6fbebqlxwqfosvvtgq07reybmvjbktt14q3ark-0tir78hisqyll741fp1d3n4r0 192.168.0.28:2377
This node joined a swarm as a worker.

```

Agora vamos logar nos outros dois managers(seguidores) e executar o comando para fazer o join no cluster como manager.

```bash

$ docker swarm join --token SWMTKN-1-5sat99b6kt3u6fbebqlxwqfosvvtgq07reybmvjbktt14q3ark-5lq5ir0grnsph7wtu2e2xy6lq 192.168.0.28:2377
This node joined a swarm as a manager.

```
Liste os nós 
```bash
$ docker node ls
ID                            HOSTNAME   STATUS    AVAILABILITY   MANAGER STATUS   ENGINE VERSION
7pn9os1z6y3wuryr4vivy8tse     node1      Ready     Active         Leader           24.0.7
0uw61mkbd9mo57mtl31yvz4cr     node2      Ready     Active         Reachable        24.0.7
0krh5prna2uh35jxiqd4cgren *   node3      Ready     Active         Reachable        24.0.7
lh7go0agmbb5j86cv7mqtit1o     node4      Ready     Active                          24.0.7
3yp3y7fq3zhxcegtq0esj9k5h     node5      Ready     Active                          24.0.7
```

Todos os três managers estão marcados como Reachable ou Leader. Os nós que não têm nada nessa coluna são os workers. O manager com o asterisco após seu nome é aquele de onde está sendo executado o comando.

Pronto, nosso cluster está criado. 

## Boas Práticas

### Trancar o cluster

O comando abaixo faz com que managers que sejam reiniciados precisem apresentar uma chave de segurança para voltar ao cluster: 

```bash

$ docker swarm update --autolock=true
Swarm updated.
To unlock a swarm manager after it restarts, run the `docker swarm unlock`
command and provide the following key:

    SWMKEY-1-2wc7kaazfKi/jcoOSMqYkTpJC10Ic2dOVty8fWQ+Gs8

Please remember to store this key in a password manager, since without it you
will not be able to restart the manager.

```

Para recuperar a chave de acesso execute o  comando docker swarm unlock-key  para visualizar sua chave de desbloqueio.


### Nós manager dedicados

Por padrão, o Swarm executa aplicativos de usuário tanto em workers quanto em managers. 
Para  executar aplicativos de usuário apenas em workers, execute o seguinte comando em qualquer manager para impedir que ele execute aplicativos de usuário. 

```bash

$ docker node update --availability drain 


```
Você verá o impacto disso em etapas posteriores quando implantar serviços com várias réplicas.

Agora que você construiu um swarm e entende os conceitos de líderes e HA de managers, vamos passar para o lado dos aplicativos.

```bash
[node1] (local) root@192.168.0.28 ~
$ docker node update --availability drain node2
Error response from daemon: node node2 is ambiguous (2 matches found)
[node1] (local) root@192.168.0.28 ~
$ docker node update --availability drain node2
node2
[node1] (local) root@192.168.0.28 ~
$ docker node update --availability drain node3
node3

docker node ls
ID                            HOSTNAME   STATUS    AVAILABILITY   MANAGER STATUS   ENGINE VERSION
7pn9os1z6y3wuryr4vivy8tse *   node1      Ready     Drain          Leader           24.0.7
0uw61mkbd9mo57mtl31yvz4cr     node2      Ready     Drain          Reachable        24.0.7
0krh5prna2uh35jxiqd4cgren     node3      Ready     Drain          Reachable        24.0.7
lh7go0agmbb5j86cv7mqtit1o     node4      Ready     Active                          24.0.7
3yp3y7fq3zhxcegtq0esj9k5h     node5      Ready     Active                          24.0.7

```

## Fazendo Deploy de Aplicações no Swarm

### De modo Imperativo 

```bash
docker service create --name web-fe \
   -p 8080:8080 \
   --replicas 5 \
   nigelpoulton/ddd-book:web0.1
z8xi3qhhkpltdk0r3omuqfup4
overall progress: 5 out of 5 tasks 
1/5: running   [==================================================>] 
2/5: running   [==================================================>] 
3/5: running   [==================================================>] 
4/5: running   [==================================================>] 
5/5: running   [==================================================>] 

```

O comando docker service create instrui o Docker a implantar um novo serviço. A flag --name atribui o nome "web-fe" a este serviço, a flag -p mapeia a porta 8080 em cada nó do swarm para a porta 8080 dentro de cada réplica do serviço (container), e a flag --replicas instrui o Swarm a implantar cinco réplicas do serviço. A última linha informa ao Swarm qual imagem deve ser usada para construir as réplicas.

Em segundo plano, o Swarm executa um loop de reconciliação que constantemente compara o estado observado do cluster com o estado desejado (informado no comando). Quando os dois estados coincidem, tudo está em ordem, e nenhuma ação adicional é necessária. Quando eles não coincidem, o Swarm toma medidas para alinhar o estado observado com o estado desejado.

Por exemplo, se um worker que hospeda uma das cinco réplicas falhar, o estado observado do cluster cairá de 5 réplicas para 4 e não coincidirá mais com o estado desejado de 5. Assim que o swarm percebe a diferença, ele inicia uma nova réplica para alinhar o estado observado com o estado desejado. Chamamos isso de reconciliação ou self healing.

Os dois comnandos abaixo permitem verificar o serviço rodando: 

```bash

$ docker service ls
ID             NAME      MODE         REPLICAS   IMAGE                          PORTS
z8xi3qhhkplt   web-fe    replicated   5/5        nigelpoulton/ddd-book:web0.1   *:8080->8080/tcp
[node1] (local) root@192.168.0.28 ~
$ docker service ps web-fe
ID             NAME       IMAGE                          NODE      DESIRED STATE   CURRENT STATE           ERROR     PORTS
30c8wpdvnn9g   web-fe.1   nigelpoulton/ddd-book:web0.1   node5     Running         Running 6 minutes ago             
ucds3klfvqdl   web-fe.2   nigelpoulton/ddd-book:web0.1   node4     Running         Running 6 minutes ago             
wkdbdp0g2qit   web-fe.3   nigelpoulton/ddd-book:web0.1   node5     Running         Running 6 minutes ago             
sh7ccxcpxrpv   web-fe.4   nigelpoulton/ddd-book:web0.1   node5     Running         Running 6 minutes ago             
bnpqmlfqur6c   web-fe.5   nigelpoulton/ddd-book:web0.1   node4     Running         Running 6 minutes ago 

```
Para mais detalhes utilize a instrução inspect: 

```bash
$ docker service inspect --pretty web-fe

ID:             z8xi3qhhkpltdk0r3omuqfup4
Name:           web-fe
Service Mode:   Replicated
 Replicas:      5
Placement:
UpdateConfig:
 Parallelism:   1
 On failure:    pause
 Monitoring Period: 5s
 Max failure ratio: 0
 Update order:      stop-first
RollbackConfig:
 Parallelism:   1
 On failure:    pause
 Monitoring Period: 5s
 Max failure ratio: 0
 Rollback order:    stop-first
ContainerSpec:
 Image:         nigelpoulton/ddd-book:web0.1@sha256:3f5b281b914b1e39df8a1fbc189270a5672ff9e98bfac03193b42d1c02c43ef0
 Init:          false
Resources:
Endpoint Mode:  vip
Ports:
 PublishedPort = 8080
  Protocol = tcp
  TargetPort = 8080
  PublishMode = ingress 

```

### Replicas x Global

O swarm tem 2 modos de entregar réplicas nos nós

* replicas -> entrega quantas réplicas são informadas
* global -> entrega uma réplica em cada nó elegível ( respeitando os drains...)

Para utilizar o global baster setar ``` --mode global ``` ao criar o serviço. 

### Escalando serviços 

Outra funcionalidade  dos serviços é a capacidade de escalá-los para cima ou para baixo, adicionando ou removendo réplicas  com o comando docker service scale.

```bash

$ docker service scale web-fe=10
web-fe scaled to 10
overall progress: 10 out of 10 tasks 
1/10: running   [==================================================>] 
2/10: running   [==================================================>] 
3/10: running   [==================================================>] 
4/10: running   [==================================================>] 
5/10: running   [==================================================>] 
6/10: running   [==================================================>] 
7/10: running   [==================================================>] 
8/10: running   [==================================================>] 
9/10: running   [==================================================>] 
10/10: running   [==================================================>] 
verify: Service converged 


$ docker service ps web-fe
ID             NAME        IMAGE                          NODE      DESIRED STATE   CURRENT STATE            ERROR     PORTS
30c8wpdvnn9g   web-fe.1    nigelpoulton/ddd-book:web0.1   node5     Running         Running 14 minutes ago             
ucds3klfvqdl   web-fe.2    nigelpoulton/ddd-book:web0.1   node4     Running         Running 14 minutes ago             
wkdbdp0g2qit   web-fe.3    nigelpoulton/ddd-book:web0.1   node5     Running         Running 14 minutes ago             
sh7ccxcpxrpv   web-fe.4    nigelpoulton/ddd-book:web0.1   node5     Running         Running 14 minutes ago             
bnpqmlfqur6c   web-fe.5    nigelpoulton/ddd-book:web0.1   node4     Running         Running 14 minutes ago             
4l5d4fsi5sso   web-fe.6    nigelpoulton/ddd-book:web0.1   node4     Running         Running 51 seconds ago             
lsaj3dudcrex   web-fe.7    nigelpoulton/ddd-book:web0.1   node5     Running         Running 51 seconds ago             
sw9akcwk6j3w   web-fe.8    nigelpoulton/ddd-book:web0.1   node4     Running         Running 51 seconds ago             
qa1qeodsn7u2   web-fe.9    nigelpoulton/ddd-book:web0.1   node4     Running         Running 51 seconds ago             
lqfo7n1pn83z   web-fe.10   nigelpoulton/ddd-book:web0.1   node5     Running         Running 52 seconds ago

$ docker service scale web-fe=5
web-fe scaled to 5
overall progress: 5 out of 5 tasks 
1/5: running   [==================================================>] 
2/5: running   [==================================================>] 
3/5: running   [==================================================>] 
4/5: running   [==================================================>] 
5/5: running   [==================================================>] 
verify: Service converged 
[node1] (local) root@192.168.0.28 ~
$ docker service ps web-fe
ID             NAME        IMAGE                          NODE      DESIRED STATE   CURRENT STATE            ERROR     PORTS
30c8wpdvnn9g   web-fe.1    nigelpoulton/ddd-book:web0.1   node5     Running         Running 15 minutes ago             
ucds3klfvqdl   web-fe.2    nigelpoulton/ddd-book:web0.1   node4     Running         Running 15 minutes ago             
wkdbdp0g2qit   web-fe.3    nigelpoulton/ddd-book:web0.1   node5     Running         Running 15 minutes ago             
sh7ccxcpxrpv   web-fe.4    nigelpoulton/ddd-book:web0.1   node5     Running         Running 15 minutes ago             
bnpqmlfqur6c   web-fe.5    nigelpoulton/ddd-book:web0.1   node4     Running         Running 15 minutes ago             
4l5d4fsi5sso   web-fe.6    nigelpoulton/ddd-book:web0.1   node4     Remove          Running 11 seconds ago             
lsaj3dudcrex   web-fe.7    nigelpoulton/ddd-book:web0.1   node5     Remove          Running 11 seconds ago             
sw9akcwk6j3w   web-fe.8    nigelpoulton/ddd-book:web0.1   node4     Remove          Running 11 seconds ago             
qa1qeodsn7u2   web-fe.9    nigelpoulton/ddd-book:web0.1   node4     Remove          Running 11 seconds ago             
lqfo7n1pn83z   web-fe.10   nigelpoulton/ddd-book:web0.1   node5     Remove          Running 11 seconds ago

```
### Deletando os Serviços

Agora vamos deletar o serviço criado através do ```docker service rm```

```bash
$ docker service rm web-fe
web-fe
[node1] (local) root@192.168.0.28 ~
$ docker service ls
ID        NAME      MODE      REPLICAS   IMAGE     PORTS
[node1] (local) root@192.168.0.28 ~
$ docker service ps web-fe
no such service: web-fe

```

### Rollouts

Vamos criar um serviço em uma rede específica. 
Primeiro vamos criar a rede
```bash
$ docker network create -d overlay uber-net
runxrkhddej9g76tfrs2i3htg

```
A nova foi criada como uma rede overlay abrangendo todo o swarm. Todos os containers na mesma rede overlay podem se comunicar, mesmo que estejam implantados em nós diferentes.

Agora vamos criar o serviço conectado à nova rede: 

```bash

$ docker service create --name uber-svc \
   --network uber-net \
   -p 8080:8080 --replicas 12 \
   nigelpoulton/ddd-book:web0.1
to9cvt3dtkqf3q49qx81fur8x
overall progress: 12 out of 12 tasks 
1/12: running   [==================================================>] 
2/12: running   [==================================================>] 
3/12: running   [==================================================>] 
4/12: running   [==================================================>] 
5/12: running   [==================================================>] 
6/12: running   [==================================================>] 
7/12: running   [==================================================>] 
8/12: running   [==================================================>] 
9/12: running   [==================================================>] 
10/12: running   [==================================================>] 
11/12: running   [==================================================>] 
12/12: running   [==================================================>] 
verify: Service converged 
[node1] (local) root@192.168.0.28 ~
$ docker service ls
ID             NAME       MODE         REPLICAS   IMAGE                          PORTS
to9cvt3dtkqf   uber-svc   replicated   12/12      nigelpoulton/ddd-book:web0.1   *:8080->8080/tcp
[node1] (local) root@192.168.0.28 ~
$ docker service ps uber-svc
ID             NAME          IMAGE                          NODE      DESIRED STATE   CURRENT STATE            ERROR     PORTS
xcbx4tomiyx6   uber-svc.1    nigelpoulton/ddd-book:web0.1   node5     Running         Running 19 seconds ago             
kwt1884qflcq   uber-svc.2    nigelpoulton/ddd-book:web0.1   node4     Running         Running 18 seconds ago             
mlk3tyzwb8m1   uber-svc.3    nigelpoulton/ddd-book:web0.1   node5     Running         Running 18 seconds ago             
w6rphfgk6j63   uber-svc.4    nigelpoulton/ddd-book:web0.1   node5     Running         Running 18 seconds ago             
qqap9ywaaloq   uber-svc.5    nigelpoulton/ddd-book:web0.1   node4     Running         Running 18 seconds ago             
dpplstkjlaiw   uber-svc.6    nigelpoulton/ddd-book:web0.1   node4     Running         Running 18 seconds ago             
zalv92gywt62   uber-svc.7    nigelpoulton/ddd-book:web0.1   node5     Running         Running 18 seconds ago             
fqaxrwc9n739   uber-svc.8    nigelpoulton/ddd-book:web0.1   node4     Running         Running 17 seconds ago             
m8hlbyu62r0w   uber-svc.9    nigelpoulton/ddd-book:web0.1   node5     Running         Running 19 seconds ago             
c343chwl6lxu   uber-svc.10   nigelpoulton/ddd-book:web0.1   node4     Running         Running 18 seconds ago             
js9y742tzku0   uber-svc.11   nigelpoulton/ddd-book:web0.1   node4     Running         Running 17 seconds ago             
17rbe7yrseqa   uber-svc.12   nigelpoulton/ddd-book:web0.1   node5     Running         Running 19 seconds ago

```

Esse modo de publicar um serviço em cada nó do swarm, incluindo nós que não estão executando réplicas, é chamado de modo ingress e é o padrão. A alternativa é o modo host, que publica um serviço apenas nos nós que estão executando réplicas.

Conecte-se ao serviço abrindo um navegador da web e apontando-o para o endereço IP de qualquer nó do swarm na porta 8080. 

No modo ingress é possível acessar um serviço de qualquer nó no cluster, mesmo aqueles que não estejam executando réplicas.

Agora vamos atualizar o serviço utilizando a imagem no mesmo repositório do Docker Hub com a tag web0.2. 

Para isso vamos executar um rollout  de maneira escalonada, atualizando duas réplicas por vez com um intervalo de 20 segundos entre cada atualização. 

```bash

$ docker service update \
   --image nigelpoulton/ddd-book:web0.2 \
   --update-parallelism 2 \
   --update-delay 20s \
   uber-svc
uber-svc
overall progress: 2 out of 12 tasks 
1/12: running   [==================================================>] 
2/12: running   [==================================================>] 
3/12: ready     [======================================>            ] 
4/12: ready     [======================================>            ] 
5/12:   

```














