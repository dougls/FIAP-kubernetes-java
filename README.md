# Caixa — Java Turma 2 (2026)

Laboratório de Kubernetes com uma API Spring Boot. O cluster sobe na sua máquina com [KIND](https://kind.sigs.k8s.io/) (Kubernetes IN Docker): cada nó do cluster é um container Docker.

Ao final deste roteiro você consegue:

- instalar o KIND e subir o cluster definido em `infra/kind-cluster.yaml`;
- construir a imagem da API e publicá-la **dentro** do cluster com `kind load docker-image`;
- explicar a diferença entre os Dockerfiles e entre Pod, Deployment e StatefulSet.

## Mapa do repositório

| Caminho | O que é |
| --- | --- |
| `pom.xml` | Projeto Maven. Spring Boot **3.2.4**, Java **21**, dependência `spring-boot-starter-web`. |
| `src/main/java/com/fiap/demo/DemoApplication.java` | Classe principal da aplicação. |
| `src/main/java/com/fiap/demo/HelloController.java` | `GET /` responde `API Java rodando perfeitamente no container!`. |
| `src/main/resources/application.properties` | Nome da aplicação e propriedades de banco. O starter JPA está comentado no `pom.xml`, então essas propriedades de datasource ficam inertes. |
| `Dockerfile` | Imagem de um estágio só. A imagem final ainda é a do Maven. |
| `Dockerfile.multistage` | Build no Maven e execução numa imagem distroless, só com o JAR. |
| `infra/kind-cluster.yaml` | Cluster local: 1 control-plane e 3 workers, nome `k8s-java`. |
| `infra/pod.yaml` | Um Pod solto, sem controlador. |
| `infra/deployment.yaml` | Deployment com 3 réplicas. |
| `infra/statefulset.yaml` | Service headless + StatefulSet com volume por réplica. |

A aplicação escuta na porta **8080**. Os manifestos em `infra/` usam a imagem pública `eclipse-temurin:21-jre` e um comando de diagnóstico da JVM. A imagem da API, construída pelos Dockerfiles, entra no cluster no passo do `kind load`.

Credencial de verdade não mora neste repositório. O `application.properties` traz placeholders de exemplo. Em um projeto real, senha e chave de nuvem vêm de Secret do Kubernetes ou de um cofre, e um `.dockerignore` impede que esse arquivo vá parar dentro da imagem.

## Pré-requisitos

- Docker em execução (Docker Desktop no macOS e no Windows, Docker Engine no Linux). O KIND cria os nós como containers, então sem Docker não existe cluster.
- Cerca de **6 GB de RAM** livres para o Docker. Este cluster tem 4 nós.
- `kubectl`.
- `kind` **v0.33.0** (versão estável na data deste material).

Confira antes de continuar:

```bash
docker info
kubectl version --client
kind version
```

## 1. Instalar o KIND

O KIND só cria e apaga o cluster. Quem conversa com a API do Kubernetes continua sendo o `kubectl`.

### macOS (Homebrew)

```bash
brew install kind kubectl
```

Binário oficial, se você não usa Homebrew:

```bash
# Mac Intel
[ "$(uname -m)" = x86_64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.33.0/kind-darwin-amd64
# Mac Apple Silicon
[ "$(uname -m)" = arm64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.33.0/kind-darwin-arm64

chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```

O `kubectl` no Mac também pode vir do binário oficial: https://kubernetes.io/docs/tasks/tools/install-kubectl-macos/

### Linux

```bash
# AMD64
[ "$(uname -m)" = x86_64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.33.0/kind-linux-amd64
# ARM64
[ "$(uname -m)" = aarch64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.33.0/kind-linux-arm64

chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```

### Windows

Com Chocolatey, no PowerShell:

```powershell
choco install kind kubernetes-cli
```

Binário direto:

```powershell
curl.exe -Lo kind-windows-amd64.exe https://kind.sigs.k8s.io/dl/v0.33.0/kind-windows-amd64
Move-Item .\kind-windows-amd64.exe C:\Windows\System32\kind.exe
```

No Windows o caminho mais estável é Docker Desktop com backend WSL2, e os comandos deste README rodando dentro da distro WSL.

### Quem já tem Go

```bash
go install sigs.k8s.io/kind@v0.33.0
```

O binário cai em `$(go env GOPATH)/bin`. Esse diretório precisa estar no `PATH`.

Documentação usada aqui: https://kind.sigs.k8s.io/docs/user/quick-start/

## 2. Criar o cluster local

Na raiz do repositório:

```bash
kind create cluster --config infra/kind-cluster.yaml
```

O arquivo do cluster é este:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: k8s-java
nodes:
- role: control-plane
- role: worker
- role: worker
- role: worker
```

O que cada campo faz:

| Campo | Efeito |
| --- | --- |
| `kind: Cluster` | Documento de configuração do KIND, não um recurso do Kubernetes. |
| `apiVersion: kind.x-k8s.io/v1alpha4` | Versão do schema dessa configuração. |
| `name: k8s-java` | Nome do cluster. O contexto do `kubectl` vira `kind-k8s-java`. Os containers Docker se chamam `k8s-java-control-plane`, `k8s-java-worker`, `k8s-java-worker2` e `k8s-java-worker3`. |
| `role: control-plane` | Nó que roda a API server, o scheduler, o controller-manager e o etcd. |
| `role: worker` | Nó que só executa carga de trabalho. São três, para os agendamentos de Deployment e StatefulSet terem para onde ir. |

O `kind create cluster` também grava o kubeconfig e deixa o contexto atual apontando para esse cluster.

```bash
kubectl cluster-info --context kind-k8s-java
kubectl get nodes
docker ps --format 'table {{.Names}}\t{{.Image}}\t{{.Status}}'
```

Os quatro nós precisam aparecer `Ready`.

## 3. As duas imagens Docker

As duas imagens geram o mesmo JAR (`target/demo-0.0.1-SNAPSHOT.jar`) e escutam na porta 8080. O que muda é **o que sobra na imagem que vai para produção**.

### `Dockerfile` — um estágio

```dockerfile
FROM maven:3.9-eclipse-temurin-21
WORKDIR /app
COPY . .
RUN mvn clean package -DskipTests
EXPOSE 8080
ENTRYPOINT [ "java", "-jar", "target/demo-0.0.1-SNAPSHOT.jar" ]
```

A imagem que executa a API é a mesma que compilou o projeto. Ela carrega JDK, Maven, código-fonte, cache do Maven e o JAR. Tem shell. Roda, por padrão, como root. Serve para entender o build. É pesada e larga demais para publicar.

`EXPOSE 8080` só documenta a porta. Quem publica a porta no host é o Kubernetes (Service, `kubectl port-forward`) ou um `-p` do Docker.

### `Dockerfile.multistage` — build separado da execução

Estágio `builder`:

- parte de `maven:3.9-eclipse-temurin-21`;
- copia primeiro só o `pom.xml` e roda `mvn dependency:go-offline`, para o cache de dependências sobreviver a mudanças de código;
- copia `src/` e gera o JAR.

Estágio final:

- parte de `gcr.io/distroless/java21-debian12`, uma imagem sem shell, sem gerenciador de pacotes e sem o código-fonte;
- copia apenas o JAR, com o nome `api.jar`;
- fixa o usuário `nonroot`;
- sobe a API com `java -jar api.jar`.

```bash
docker build -t demo-java:local -f Dockerfile.multistage .
docker build -t demo-java:completa -f Dockerfile .
docker images demo-java
```

Compare o tamanho das duas tags. A imagem `demo-java:local` é a que vamos carregar no cluster.

Detalhe que muda o manifesto: a distroless **não tem** `/bin/sh`. Os YAML de `infra/` usam `command: ["sh", "-c"]`. Isso funciona na `eclipse-temurin:21-jre` e na imagem de um estágio. Na imagem distroless esse comando falha com `executable file not found`. A API multistage sobe pelo `ENTRYPOINT` dela; o manifesto não deve substituir o comando.

## 4. Colocar a imagem local no cluster

O Docker da sua máquina e o containerd de cada nó do KIND são armazenamentos diferentes. `docker build` deixa a imagem no daemon do host. O kubelet, dentro do nó, não enxerga essa imagem. Sem um passo extra, o Pod fica em `ErrImagePull` ou `ImagePullBackOff`.

```text
docker build  --->  imagem no Docker do host
                         |
                         |  kind load docker-image
                         v
                    containerd de cada nó do KIND
                         |
                         |  kubelet, com imagePullPolicy IfNotPresent ou Never
                         v
                       container sobe sem ir a um registry
```

Carregue a tag que você acabou de construir. O nome do cluster é obrigatório: o padrão do KIND é um cluster chamado `kind`, e o nosso se chama `k8s-java`.

```bash
kind load docker-image demo-java:local --name k8s-java
```

A tag no comando e a tag no manifesto precisam ser **idênticas**, incluindo o nome. `demo-java:local` e `demo-java:latest` são imagens diferentes para o kubelet.

Confira se a imagem chegou no nó:

```bash
docker exec k8s-java-control-plane crictl images | grep demo-java
```

`kind load docker-image` copia para todos os nós. Para ver um worker:

```bash
docker exec k8s-java-worker crictl images | grep demo-java
```

### Por que a `imagePullPolicy` importa

| Política | Comportamento |
| --- | --- |
| `IfNotPresent` | Usa a imagem que já está no nó. Só tenta baixar se ela não existir. É a política deste repositório e a correta para imagem carregada com `kind load`. |
| `Never` | Nunca baixa. Se o `kind load` não foi feito, o Pod falha na hora. Útil quando você quer ter certeza de que está rodando a imagem local. |
| `Always` | Sempre consulta o registry. Uma tag que só existe na sua máquina falha, porque não há registry para responder. |

Se o campo for omitido e a tag for `latest`, o Kubernetes assume `Always`. Por isso a tag deste laboratório é `:local`, e os manifestos declaram a política.

O `kind load` fala com o mesmo Docker que criou o cluster. Se `docker images` não lista `demo-java:local`, o load também não encontra a imagem. Isso acontece quando o build foi feito em outro contexto (Colima, Podman, outro Docker Desktop).

## 5. Subir a API com a imagem local

Os YAML de `infra/` ensinam Pod, Deployment e StatefulSet com a imagem do Temurin. Para ver a API Spring Boot no ar, use um Deployment cujo `image` é exatamente a tag carregada.

Salve como `infra/app.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-java
  labels:
    app: demo-java
spec:
  replicas: 2
  selector:
    matchLabels:
      app: demo-java
  template:
    metadata:
      labels:
        app: demo-java
    spec:
      containers:
        - name: api
          image: demo-java:local
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: 100m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 512Mi
---
apiVersion: v1
kind: Service
metadata:
  name: demo-java
spec:
  selector:
    app: demo-java
  ports:
    - port: 8080
      targetPort: 8080
```

```bash
kubectl apply -f infra/app.yaml
kubectl rollout status deployment/demo-java
kubectl get pods -l app=demo-java -o wide
```

O Service é do tipo `ClusterIP`: ele só existe dentro do cluster. Da sua máquina, o caminho é o `port-forward`:

```bash
kubectl port-forward svc/demo-java 8080:8080
```

Em outro terminal:

```bash
curl http://localhost:8080/
```

A resposta esperada é `API Java rodando perfeitamente no container!`.

Quando alterar o código, o ciclo inteiro se repete. O Kubernetes não percebe um `docker build` novo se a tag continuar `demo-java:local` e a política for `IfNotPresent`, porque a imagem antiga continua “presente”.

```bash
docker build -t demo-java:local -f Dockerfile.multistage .
kind load docker-image demo-java:local --name k8s-java
kubectl rollout restart deployment/demo-java
```

O `rollout restart` recria os Pods e força o kubelet a abrir a imagem que acabou de ser carregada.

## 6. Diferenças entre os artefatos

Há dois grupos no repositório.

**Imagens** (`Dockerfile` e `Dockerfile.multistage`) empacotam a aplicação. Elas respondem: o que esse container contém e qual processo ele inicia.

**Workloads** (`pod.yaml`, `deployment.yaml`, `statefulset.yaml`) dizem ao Kubernetes como executar containers. Eles respondem: quantas cópias, o que acontece quando uma cai, e se cada cópia tem nome e disco estáveis.

```text
Pod            uma caixa. Se ela some, ninguém cria outra.
Deployment     um conjunto de caixas intercambiáveis. O controlador mantém a quantidade.
StatefulSet    um conjunto de caixas numeradas. Cada número guarda o próprio disco.
```

### Pod — `infra/pod.yaml`

Um Pod é a menor unidade que o Kubernetes agenda. Este manifesto cria **um** Pod chamado `java-nu`.

O container:

- usa `eclipse-temurin:21-jre`;
- pede `100m` de CPU e `512Mi` de memória, com teto de `200m` e `1Gi`;
- imprime a configuração da JVM (`-XshowSettings:vm`) e depois fica vivo com `sleep infinity`.

`100m` significa 100 milicores, isto é, 0,1 CPU. `512Mi` é mebibyte (1024 × 1024 bytes), a unidade que o Kubernetes usa de fato.

O que esse Pod faz sozinho:

- sobe uma vez, com o nome que está no `metadata.name`;
- se o processo termina, o Pod fica `Succeeded` ou `Failed` e permanece assim;
- se você apaga o Pod, ele não volta.

É o artefato certo para um teste curto e para ler o manifesto sem um controlador no meio. Aplicação de verdade pede um controlador.

```bash
kubectl apply -f infra/pod.yaml
kubectl get pod java-nu
kubectl logs java-nu
kubectl delete pod java-nu
kubectl get pod java-nu
```

Depois do `delete`, o `get` não encontra nada. Esse é o ponto da aula.

### Deployment — `infra/deployment.yaml`

Um Deployment declara um estado desejado: “quero 3 Pods com este molde, para sempre”. O controlador cria um ReplicaSet, e o ReplicaSet cria e recria os Pods.

| Trecho | Função |
| --- | --- |
| `replicas: 3` | Quantidade desejada. |
| `selector.matchLabels` | Como o Deployment encontra os Pods que lhe pertencem. Tem que ser igual aos labels do molde. |
| `template` | O molde do Pod. Cada réplica nasce dessa spec. |
| `imagePullPolicy: IfNotPresent` | Aproveita a imagem se ela já estiver no nó. |

Os Pods de um Deployment são intercambiáveis. O nome recebe um sufixo aleatório (`java-nu-7d9f8c6b5d-abcde`). Não há disco próprio por réplica. Apagar um Pod faz outro nascer, com outro nome, para completar as 3 réplicas. Trocar a imagem no manifesto e aplicar de novo dispara uma atualização gradual: um Pod novo sobe antes de um antigo sair.

Olho no label deste arquivo. O nome do Deployment é `java-nu` e o label do objeto Deployment é `app: java-nu`. O selector e o molde dos Pods usam `app: caixaverso`. Isso é válido: o selector precisa bater com o **molde**, e bate. O label do objeto Deployment é outra coisa. Na hora de listar os Pods, o filtro certo é o do molde:

```bash
kubectl apply -f infra/deployment.yaml
kubectl get pods -l app=caixaverso
kubectl delete pod -l app=caixaverso
kubectl get pods -l app=caixaverso
```

O segundo `get` mostra Pods novos. O Deployment recompôs as 3 réplicas.

`kubectl get pods -l app=java-nu` lista o Pod solto do `pod.yaml`, porque aquele manifesto sim etiqueta o Pod com `app: java-nu`. São dois objetos diferentes que compartilham um nome parecido.

### StatefulSet — `infra/statefulset.yaml`

Este arquivo traz dois recursos.

O Service `kafka-headless` tem `clusterIP: None`. Um Service normal recebe um IP virtual e distribui conexões entre os Pods. Um Service headless não recebe IP virtual: o DNS devolve o endereço de cada Pod. O StatefulSet exige esse Service no campo `serviceName`.

O StatefulSet `kafka` pede 3 réplicas. O container ainda é o `eclipse-temurin:21-jre` com o mesmo comando de diagnóstico. O nome `kafka` marca a intenção didática: uma carga em que cada réplica tem identidade. Não há um broker Kafka nessa imagem.

O que muda em relação ao Deployment:

| | Deployment | StatefulSet |
| --- | --- | --- |
| Nome do Pod | sufixo aleatório | ordinal estável: `kafka-0`, `kafka-1`, `kafka-2` |
| DNS | o Service balanceia entre todos | `kafka-0.kafka-headless.default.svc.cluster.local` aponta para aquele Pod |
| Criação | em paralelo | em ordem: `0`, depois `1`, depois `2` |
| Remoção do conjunto | em paralelo | na ordem inversa, do maior ordinal para o menor |
| Disco | nada neste manifesto | um PVC de `1Gi` por Pod, montado em `/dados` |
| Réplicas | qualquer uma atende qualquer pedido | cada ordinal é uma instância reconhecível |

O bloco `volumeClaimTemplates` cria um PersistentVolumeClaim para cada Pod. O nome do volume fica `dados-kafka-0`, `dados-kafka-1` e `dados-kafka-2`. O modo de acesso é `ReadWriteOnce`: um nó por vez monta o volume.

O KIND já traz a StorageClass `standard` (provisioner local-path). Com ela, o PVC fica `Pending` até o Pod ser agendado, e o volume nasce no disco da sua máquina, dentro do container do nó. Apagar o Pod `kafka-0` faz o StatefulSet criar outro `kafka-0`, e o volume `dados-kafka-0` volta para ele. O dado escrito em `/dados` sobrevive à troca do Pod.

O Deployment deste repositório não tem `volumeClaimTemplates`. Arquivo escrito no sistema de arquivos do container desaparece quando o Pod é recriado.

```bash
kubectl apply -f infra/statefulset.yaml
kubectl get pods -l app=kafka
kubectl get pvc
kubectl exec kafka-0 -- sh -c 'echo ola > /dados/mensagem.txt'
kubectl delete pod kafka-0
kubectl wait --for=condition=Ready pod/kafka-0
kubectl exec kafka-0 -- cat /dados/mensagem.txt
```

A saída do último comando é `ola`.

Limites deste manifesto, de propósito mais enxutos que os outros: só há `limits.memory: 1Gi`. Não há request de CPU nem de memória, e não há limite de CPU. O scheduler trata o request ausente como zero, então ele não reserva recurso para esses Pods.

### Os três lado a lado

| Pergunta | Pod | Deployment | StatefulSet |
| --- | --- | --- | --- |
| Arquivo | `infra/pod.yaml` | `infra/deployment.yaml` | `infra/statefulset.yaml` |
| Quantas cópias | 1, com nome fixo | `replicas: 3` | `replicas: 3`, nomes `kafka-0` em diante |
| Quem recria se cair | ninguém | o ReplicaSet | o StatefulSet, com o mesmo ordinal |
| Identidade de rede | o IP do Pod muda se ele for recriado em outro lugar | os Pods ficam atrás de um Service, se você criar um | DNS estável por ordinal, via Service headless |
| Disco estável | não declara volume | não declara volume | PVC de 1Gi em `/dados` |
| Atualização | editar e apagar na mão | rolling update | em ordem, um ordinal por vez |
| Uso típico | experiência e debug | API sem estado, como a nossa | banco, fila, qualquer coisa que guarda dado por réplica |

A API Spring Boot deste repositório é sem estado: qualquer réplica responde `GET /` igual. O artefato que combina com ela é o Deployment. O StatefulSet está aqui para mostrar identidade e volume, usando a JRE como processo de estudo.

## 7. Requests, limits e a JVM

Os três manifestos de estudo sobem a JVM com:

```text
java -XX:MaxRAMPercentage=75.0 -XshowSettings:vm -version
```

`-XshowSettings:vm` escreve no log o tamanho máximo de heap que a JVM calculou. `-XX:MaxRAMPercentage=75.0` pede que esse heap seja 75% da memória que o processo enxerga.

Dentro do container, o Java 21 lê o limite do cgroup. Esse limite é o `resources.limits.memory` do manifesto, aqui `1Gi`. Setenta e cinco por cento de 1 GiB aparecem no log como heap máximo em torno de 768 MB.

O `requests.memory` (`512Mi` no Pod e no Deployment) é outra conta: é o que o scheduler reserva para aceitar o Pod no nó. A JVM olha o **limit**, não o request. Sem `limits.memory`, o processo passa a enxergar a memória do nó inteiro, e o percentual de 75% deixa de ter relação com o tamanho que você queria para aquele container.

Soma rápida, se você aplicar os três arquivos juntos:

- Pod: request de 512Mi;
- Deployment: 3 × 512Mi;
- StatefulSet: sem request.

São pelo menos 2 GiB só de request, mais a memória dos 4 nós. Com o Docker curto de RAM, os Pods ficam `Pending` e o `kubectl describe pod` mostra `Insufficient memory`. Suba um artefato por vez se a máquina apertar.

## 8. Comandos do dia a dia

```bash
# contexto e nós
kubectl config current-context
kubectl get nodes -o wide

# o que está rodando
kubectl get pods -o wide
kubectl get deploy,sts,svc,pvc

# por que um Pod não sobe
kubectl describe pod <nome-do-pod>
kubectl logs <nome-do-pod>
kubectl get events --sort-by=.lastTimestamp

# entrar no container que tem shell (Temurin tem; distroless não tem)
kubectl exec -it <nome-do-pod> -- sh

# encerrar só a carga, e manter o cluster
kubectl delete -f infra/app.yaml
kubectl delete -f infra/pod.yaml
kubectl delete -f infra/deployment.yaml
kubectl delete -f infra/statefulset.yaml
```

Apagar o StatefulSet não apaga os PVCs. Eles continuam de propósito, para o dado sobreviver. Para remover os discos deste laboratório:

```bash
kubectl delete pvc -l app=kafka
```

Se o PVC não tiver esse label, apague pelo nome (`dados-kafka-0` e os seguintes).

## 9. Problemas comuns

| Sintoma | Causa provável | O que fazer |
| --- | --- | --- |
| `kind: command not found` | binário fora do `PATH` | Reinstale pelo passo 1 e abra um terminal novo. |
| `Cannot connect to the Docker daemon` | Docker parado | Abra o Docker Desktop e espere o engine ficar pronto. |
| `ErrImagePull` em `demo-java:local` | imagem não foi carregada, tag diferente, ou política `Always` | Rode `kind load docker-image demo-java:local --name k8s-java` e confira `image` e `imagePullPolicy`. |
| `no nodes found for cluster "kind"` | faltou `--name k8s-java` | O cluster deste repo não usa o nome padrão. |
| Pod distroless em `CreateContainerError` / `CrashLoopBackOff` | manifesto trocou o comando para `sh -c` | Na imagem multistage, deixe o `ENTRYPOINT` original (`java -jar`). |
| Pod `Pending` | request maior que a folga do nó, ou PVC sem StorageClass | `kubectl describe pod`. No KIND a StorageClass `standard` já existe. |
| `curl` em `localhost:8080` recusado | nada publicou a porta no host | Use `kubectl port-forward svc/demo-java 8080:8080`. `EXPOSE` no Dockerfile não publica porta. |
| Build novo e Pod antigo | mesma tag e `IfNotPresent` | Repita o `kind load` e faça `kubectl rollout restart`. |
| `kubectl` olhando outro cluster | contexto mudou | `kubectl config use-context kind-k8s-java`. |

## 10. Derrubar o ambiente

```bash
kind delete cluster --name k8s-java
```

Isso remove os containers dos nós, o kubeconfig desse cluster e os volumes criados dentro dele. As imagens `demo-java:*` continuam no Docker do host até você removê-las:

```bash
docker rmi demo-java:local demo-java:completa
```

## Referências

- KIND, quick start: https://kind.sigs.k8s.io/docs/user/quick-start/
- KIND, carregar uma imagem local: https://kind.sigs.k8s.io/docs/user/quick-start/#loading-an-image-into-your-cluster
- Pod: https://kubernetes.io/docs/concepts/workloads/pods/
- Deployment: https://kubernetes.io/docs/concepts/workloads/controllers/deployment/
- StatefulSet: https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/
- Service headless: https://kubernetes.io/docs/concepts/services-networking/service/#headless-services
- Requests e limits: https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/
