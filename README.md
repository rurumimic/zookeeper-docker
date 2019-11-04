# zookeeper-docker

Default Zookeeper cluster with docker.

---

## library/zookeeper

Docker Hub: [library/zookeeper](https://hub.docker.com/_/zookeeper/)  
GitHub: [31z4/zookeeper-docker](https://github.com/31z4/zookeeper-docker)

도커 이미지를 받는다.

```bash
docker pull zookeeper
```

---

## Network

[Networking in Compose](https://docs.docker.com/compose/networking/)

주키퍼 서버를 다른 도커 컨테이너에서도 사용하기 편하게 미리 네트워크를 구성한다.

`bridge network`(docker-compose)나 `overlay network`(docker swarm)를 미리 만들어 놓는다.

```bash
# for docker-compose
docker network create zookeeper_net

# for docker swarm
docker network create -d overlay --attachable zookeeper_net
```

도커 네트워크는 기본적으로 `bridge`로 생성된다.  
오버레이 네트워크에 도커 컨테이너를 추가/제거할 수 있도록 `--attachable` 옵션을 사용한다.

***--link***

[Legacy container links](https://docs.docker.com/network/links/)

도커 컨테이너끼리 통신을 할 때는 `--link` 옵션을 사용하지 말고 네트워크를 구성하는 것이 좋다.

---

## Standalone mode

```bash
docker run --name standalone-zookeeper --network zookeeper_net --restart always -d zookeeper
```

도커 이미지의 기본 포트로 `2181`, `2888`, `3888`가 설정되어 있다. (클라이언트 포트, 팔로워 포트, 마스터 선출 포트)

---

## Replicated mode

[Running Replicated ZooKeeper](http://zookeeper.apache.org/doc/r3.5.4-beta/zookeeperStarted.html#sc_RunningReplicatedZooKeeper)

개발을 할 때는 주키퍼 독립모드를 사용하지만, 배포를 할 때는 앙상블을 구성한다.  
복제모드를 구성할 때는 최소 서버 3개가 필요하다.

### docker-compose.yml / stack.yml

```yml
version: '3.1'

services:
  zoo1:
    image: zookeeper
    restart: always
    hostname: zoo1
    ports:
      - 2181:2181
    environment:
      ZOO_MY_ID: 1
      ZOO_SERVERS: server.1=0.0.0.0:2888:3888;2181 server.2=zoo2:2888:3888;2181 server.3=zoo3:2888:3888;2181
    networks:
      - zookeeper

  zoo2:
    image: zookeeper
    restart: always
    hostname: zoo2
    ports:
      - 2182:2181
    environment:
      ZOO_MY_ID: 2
      ZOO_SERVERS: server.1=zoo1:2888:3888;2181 server.2=0.0.0.0:2888:3888;2181 server.3=zoo3:2888:3888;2181
    networks:
      - zookeeper

  zoo3:
    image: zookeeper
    restart: always
    hostname: zoo3
    ports:
      - 2183:2181
    environment:
      ZOO_MY_ID: 3
      ZOO_SERVERS: server.1=zoo1:2888:3888;2181 server.2=zoo2:2888:3888;2181 server.3=0.0.0.0:2888:3888;2181
    networks:
      - zookeeper

networks:
  zookeeper:
    external:
      name: zookeeper_net
```

`networks`를 설정하면 도커는 default 네트워크를 생성하지 않고 이미 구성해둔 네트워크를 사용한다.

### zoo.cfg

주키퍼 설정은 `zoo.cfg` 파일에서 관리한다. `zoo.cfg`는 `/conf` 디렉토리에 있다.
대신 도커를 사용할 때는 환경변수를 사용한다.

- ZOO_MY_ID: 앙상블 안에서 서버는 각자 유일한 1 ~ 255 사이 값을 가진다.
- ZOO_SERVERS: `server.id=host:port:port;client_port` 형태로 서버들을 지정한다.

이 밖에도 다른 [환경변수](https://github.com/31z4/zookeeper-docker/#environment-variables)를 설정할 수 있다.

### Run

두 가지 방법으로 실행할 수 있다.

#### docker-compose

```bash
docker-compose up -d
```

파일 이름을 `docker-compose.yml`로 지정하면, 도커 컴포즈 명령에서 `-f compsoefile.yml`를 생략할 수 있다.

#### Docker Swarm

도커 스웜 클러스터를 사용한다면 `stack` 명령어를 사용한다.

```bash
docker stack deploy -c stack.yml zookeeper
```

---

## Test

### Standalone mode

```bash
docker run --rm \
--network zookeeper_net \
-it zookeeper \
bin/zkCli.sh \
-server standalone-zookeeper
```

`-server standalone-zookeeper:2181`에서 포트 부분을 생략할 수 있다.

### Replicated mode

```bash
docker run --rm \
--network zookeeper_net \
-it zookeeper \
bin/zkCli.sh \
-server zoo1
```

`zoo2`와 `zoo3` 서버도 접속 가능하다.

### Create/Delete node example

[Connecting to ZooKeeper](http://zookeeper.apache.org/doc/r3.5.4-beta/zookeeperStarted.html#sc_ConnectingToZooKeeper)

간단하게 주키퍼에 노드 추가/삭제를 해본다.

```bash
[zk: zoo1(CONNECTED) 0] help
```

주키퍼 명령어를 볼 수 있다.

```bash
[zk: zoo1(CONNECTED) 1] ls /
[zookeeper]
```

노드 목록을 볼 수 있다.

```bash
[zk: zoo1(CONNECTED) 2] create /zk_test my_data
Created /zk_test

[zk: zoo1(CONNECTED) 3] ls /
[zookeeper, zk_test]
```

`my_data`라는 값을 가진 `/zk_test` 노드를 생성한다.

```bash
[zk: zoo1(CONNECTED) 4] get /zk_test
my_data
cZxid = 0x2
...
```

`/zk_test` 노드의 값을 확인한다.

```bash
[zk: zoo1(CONNECTED) 5] set /zk_test junk
cZxid = 0x2
...

[zk: zoo1(CONNECTED) 6] get /zk_test
junk
cZxid = 0x2
...
```

노드의 값을 바꿀 수 있다.

```bash
[zk: zoo1(CONNECTED) 7] delete /zk_test

[zk: zoo1(CONNECTED) 8] ls /
[zookeeper]

[zk: zoo1(CONNECTED) 9]
```

노드를 삭제하고 `Ctl + c`로 도커 컨테이너에서 빠져나온다.
