## Reaproveitar PVs dinâmicos com policy retain

- Editar a storageclass ou criar uma nova com a propriedade reclaimPolicy: Retain
- Criar uma pvc apontando para storageclass com o propriedade reclaimPolicy: Retain
- Criar um deployment apontando para o volume dinâmico já criado
- Após apagar o deployment e o pvc
- Edita o pvc e deixa o storageclass vazio e inclui a propriedade volumeName com o nome do pv já criado
- Editar o pv e remover o uid da propriedade claimRef para forçar um novo bind do pvc com pv
- Recria o pvc e o deployment

    Obs: A propriedade reclaimPolicy fica no storageclass.

## Para funcionamento de VPA e HPA

- Instalação do componente metrics server: https://kubernetes-sigs.github.io/metrics-server/
- Intalação do VPA: https://kubernetes.io/docs/concepts/workloads/autoscaling/vertical-pod-autoscale/