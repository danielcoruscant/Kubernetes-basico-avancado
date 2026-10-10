## Comandos para certificado digital HTTPS no ingress

    ### Certificado de ingress para conexão https ###

    openssl req -x509 \
        -nodes \
        -days 365 \
        -newkey rsa:2048 \
        -keyout tls.key \
        -out tls.crt \
        -subj '/CN=*.demo.com/O=systemfactory' \
        -addext 'subjectAltName = DNS:*.demo.com'

    openssl x509 -in tls.crt -text

## Comandos para criação de usuário e RBAC

    ### Certificado de cliente ###

    openssl req \
        -nodes \
        -days 365 \
        -newkey rsa:2048 \
        -keyout estagiario.key \
        -out estagiario.csr \
        -subj '/CN=estagiario/O=systemfactory' \
        -addext 'subjectAltName = DNS:estagiario.systemfactory.com'

    cat estagiario.csr| base64 -w 0

    kubectl certificate approve estagiario-csr

    kubectl get csr estagiario-csr -ojson |\
		jq -r .status.certificate |\
		base64 -d > estagiario.crt

    openssl x509 -in estagiario.crt -text

    kubectl config set-credentials estagiario \
		--client-certificate estagiario.crt \
		--client-key estagiario.key

    kubectl config set-context estagiario \
		--cluster kind-kind \
		--user estagiario

    kubectl config use-context estagiario

    kubectl get pods

## Comandos para criação de secret

    kubectl create secret tls demo-secret --key tls.key --cert tls.crt
    
## Comandos utilitários

    echo -n "senhateste1234" | base64 --> Criar uma string em base64

    docker run -it nicolaka/netshoot sh --> criar um pod para debug com várias ferramentas para throbleshooting

## Comandos para atualização do cluster

### Atualizar o kubeadm

    curl -fsSL <https://pkgs.k8s.io/core:/stable:/v1.25/deb/Release.key> | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring-1.25.gpg
        echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] <https://pkgs.k8s.io/core:/stable:/v1.25/deb/> /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
    apt update
    apt list -a kubeadm
    apt-mark unhold kubeadm
    apt install kubeadm
    apt-mark hold kubeadm

### Verificando o plano de upgrade e aplicando no Control Plane

**Obs.: Instalar o kubent primeiramente para verificar os componentes depreciados do kubernetes antes do upgrade. Instruções para instalação vide tips.md**

    kubent -t <version_to_upgrade>
    kubeadm upgrade apply v1.25.14
    kubectl drain k8s-master-1 --ignore-daemonsets
    apt-mark unhold kubectl kubelet
    apt install kubectl kubelet -y
    apt-mark hold kubectl kubelet
    systemctl daemon-reload
    systemctl restart kubelet
    kubectl uncordon k8s-master-1
    kubectl get nodes

### Verificando o plano de upgrade e aplicando dos Workers

    **Primeiro atualiza o kubeadm vide passo anterior**
    kubeadm upgrade node
    kubectl drain k8s-master-1 --ignore-daemonsets
    apt-mark unhold kubectl kubelet
    apt install kubectl kubelet -y
    apt-mark hold kubectl kubelet
    systemctl daemon-reload
    systemctl restart kubelet
    kubectl uncordon k8s-master-1
    kubectl get nodes

### Taints & Tolerations

**Utilizado para restrigir deployments para um determinado node, só pods com o toleration nodegroup=backend:NoSchedule será deployado no node.

    kubectl taint node comunidade-devops-worker nodegroup=backend:NoSchedule
    kubectl create deploy -f deployment-taints-toleration