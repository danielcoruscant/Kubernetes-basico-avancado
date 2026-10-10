# Processos de Troubleshooting

## Diagnóstico de pods

Diariamente na sua experiência com Kubernetes, você verá dois erros enquanto os pods ficam crashando: `ImagePullBackOff` e `CrashLoopBackOff`.

Vamos falar um pouco sobre cada um deles e quais comandos utilizar para analisar o problema.

### ImagePullBackOff

Durante o startup do pod, o node ao qual está atribuído vai fazer o pull da imagem de um registry. Esse processo pode falhar por vários motivos:

- autenticação (mais comum)
- tag inexistente
- rede (timeout / latência)
- TLS (ex.: certificado expirado)

No fim das contas, o erro no pod vai ser o mesmo: `ImagePullBackOff`. Nesses casos, o comando que precisa ser usado é `kubectl describe`. Nesse momento o pod nem startou ainda, então logs ou exec não são úteis.

Vamos simular o problema e executar o describe. Vou colocar uma tag inexistente.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: alpine
  name: alpine
spec:
  replicas: 1
  selector:
    matchLabels:
      app: alpine
  template:
    metadata:
      labels:
        app: alpine
    spec:
      containers:
        - image: alpine:vraus
          name: alpine
          command:
            - sleep
            - infinity
```

E vamos dar describe no pod falhando.

```bash
$ kubectl describe pod <pod>
```

Você verá os eventos gravados para o startup do pod, e um deles será a causa do BackOff.

```text
Warning  Failed     21s (x2 over 42s)  kubelet            Failed to pull image "alpine:vraus": rpc error: code = NotFound desc = failed to pull and unpack image "docker.io/library/alpine:vraus": failed to resolve reference "docker.io/library/alpine:vraus": docker.io/library/alpine:vraus: not found
```

Claramente o erro é: essa imagem não existe.

### CrashLoopBackOff

O `CrashLoopBackOff` é o erro mais comum, eu diria, e é essencialmente erro na aplicação. Os outros times vão vir quebrando falando que é problema no Kubernetes e bláblá, quando na verdade o processo principal do container está finalizando com código de erro diferente de zero.

Então aqui os comandos invertem: o `describe` não vai trazer muitos detalhes, porque o pod já startou, e o erro é na aplicação dentro do pod. O log é o subcomando mais efetivo para estes casos.

Vamos simular aqui, sobrescrevendo o `CMD` da imagem para um `ls -l` em uma pasta que não existe. O `ls -l` vai retornar um código diferente de zero e vai crashar o pod.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: alpine
  name: alpine
spec:
  replicas: 1
  selector:
    matchLabels:
      app: alpine
  template:
    metadata:
      labels:
        app: alpine
    spec:
      containers:
        - image: alpine:latest
          name: alpine
          command:
            - ls
            - -l
            - vraus
```

Então vamos usar o logs:

```bash
$ kubectl logs alpine-6894474c59-6kfqz
```

```text
ls: vraus: No such file or directory
```

Isso explicaria o porquê do pod estar crashando. O mesmo serve para qualquer outra aplicação: nos logs, você entenderá olhando a exceção de erro o motivo do crash.

## Diagnóstico de nós

### Simulando NodeNotReady

O kubelet é o agente que roda em cada node e é responsável pelos pods que lá estão. Isso é: realizar os healthchecks dos probes, fazer pull de imagem, etc.

Além do gerenciamento normal, ele é o cara que fala diretamente com o API server para o node entrar no cluster. Para relembrar, veja a imagem dessa doc:

`Kubernetes Components`

Então, para que toda essa comunicação entre os worker nodes e os master nodes ocorra da melhor forma possível, a rede e a segurança precisam estar propriamente configuradas.

Aqui dá para ter uma ideia de todas as portas que precisam estar abertas:

`Ports and Protocols`

Em Cloud, seriam as route tables, security groups, NACLs, etc.

Em um ambiente on-premises, por exemplo, o IPTables pode ser um impeditivo se algo foi configurado erroneamente.

Vamos simular um problema para que você entenda quando e onde conseguir esse log.

Vou acessar via SSH os 3 master nodes do meu cluster Kubernetes, e dentro de cada um deles, vou dropar todo o tráfego vindo de um worker específico.

```bash
$ iptables-save > /tmp/iptables.bkp # backup por garantia
$ iptables -I INPUT -s 192.168.0.41 -j DROP
```

No caso acima, usei o `-I INPUT` para jogar a regra no topo da cadeia, acima até mesmo de regras de CNI e kube-proxy.

Em alguns minutos, o worker node vai mudar o status de `Ready` para `NotReady`.

```text
NAME           STATUS     ROLES           AGE    VERSION
k8s-master-1   Ready      control-plane   273d   v1.28.2
k8s-master-2   Ready      control-plane   273d   v1.28.2
k8s-master-3   Ready      control-plane   273d   v1.28.2
k8s-worker-1   Ready      <none>          273d   v1.28.2
k8s-worker-2   NotReady   <none>          273d   v1.28.2  -----> aqui
k8s-worker-3   Ready      <none>          273d   v1.28.2
k8s-worker-4   Ready      <none>          273d   v1.28.2
```

Sempre que você se deparar com uma situação assim, em vez de cegamente começar a testar um caminhão de coisas, acesse o worker node via SSH ou outra forma e colete o log do kubelet.

```bash
$ journalctl -fu kubelet
```

```text
Jun 26 11:54:29 k8s-worker-2 kubelet[1333]: E0626 11:54:29.538499    1333 controller.go:146] "Failed to ensure lease exists, will retry" err="Get \"<https://localhost:6443/apis/coordination.k8s.io/v1/namespaces/kube-node-lease/leases/k8s-worker-2?timeout=10s\>": context deadline exceeded - error from a previous attempt: EOF" interval="7s"
Jun 26 11:54:32 k8s-worker-2 kubelet[1333]: E0626 11:54:32.438807    1333 kubelet_node_status.go:540] "Error updating node status, will retry" err="error getting node \"k8s-worker-2\": Get \"<https://localhost:6443/api/v1/nodes/k8s-worker-2?resourceVersion=0&timeout=10s\>": context deadline exceeded - error from a previous attempt: EOF"
```

Isso é: o node não consegue nem chegar ao API server na porta 6443. Nesse momento é onde você começa a usar ferramentas como netcat/curl para testar a comunicação. Eventualmente você vai chegar na solução, que nesse caso é regra de firewall.

PS: para voltar ao normal, usei:

```bash
$ iptables -D INPUT 1
```

## Testes de rede

Diariamente precisamos testar a comunicação pod-to-pod, passando por service e todos os recursos do ecossistema do Kubernetes.

Como a comunicação é totalmente interna, precisamos de um container de teste com ferramentas de rede para validar a comunicação.

### Pod de teste

Para esses casos, eu costumo iniciar um pod de teste com alpine e instalar o que preciso. Nem sempre isso é possível, mas é comum o time ter uma imagem customizada com ferramentas de teste.

Assim, eu rapidamente gero um manifesto para um pod do alpine:

```bash
$ kubectl run --image alpine --dry-run -oyaml alpine | kubectl neat > alpine.yaml
```

Então eu só adiciono um `command` com `sleep` para travar o pod e posso acessá-lo.

```bash
$ vim alpine.yaml
```

```yaml
command:
  - sleep
  - infinity
```

```bash
$ kubectl apply -f alpine
$ kubectl exec -it alpine sh
# apk add curl
```

E aí é só correr para o abraço nos testes.

### Sidecar container

Porém, nem sempre isso é possível, ou a melhor opção. Há vezes em que precisamos inspecionar a network namespace do pod, mas não temos as ferramentas necessárias para isso, então se faz necessário injetar um sidecar para isso.

Imagina que temos um deployment com NGINX rodando.

```bash
$ kubectl create deploy --image nginx nginx
```

Podemos alterar o deployment, incluindo um container sidecar do alpine também.

```bash
$ kubectl edit deploy nginx
```

```yaml
spec:
  containers:
    - image: alpine
      name: alpine
      command:
        - sleep
        - infinity
```

Quando o novo pod for iniciando, basta darmos o mesmo `exec`, informando o container.

```bash
$ kubectl exec -it nginx-5784699764-8qnbh -c alpine sh
```

```bash
/ # apk add curl
fetch https://dl-cdn.alpinelinux.org/alpine/v3.20/main/x86_64/APKINDEX.tar.gz
fetch https://dl-cdn.alpinelinux.org/alpine/v3.20/community/x86_64/APKINDEX.tar.gz
...
...
OK: 13 MiB in 24 packages

/ # curl localhost
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
...
```

Canivete suiço!

E também há casos em que você não tem nem permissão para instalar pacotes; então a imagem do alpine não vai ajudar muito.

Se você não tem nenhuma imagem pronta para troubleshooting, recomendo a `netshoot` como um bom ponto para você iniciar.

[GitHub - nicolaka/netshoot: a Docker + Kubernetes network troubleshooting swiss-army container](https://github.com/nicolaka/netshoot)

```bash
$ docker run -it nicolaka/netshoot sh
```

Eu duvido que nenhuma das ferramentas incluídas não te ajude. A imagem já vem com `tcpdump`, `netstat`, `iperf`, etc.