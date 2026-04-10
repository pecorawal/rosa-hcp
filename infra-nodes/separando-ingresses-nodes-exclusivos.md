
# Criando infra nodes exclusivos no Rosa HCP

***Exemplo de API ROSA***
https://api.rosa.xxxx.p3.openshiftapps.com:443


1. Criar Machine Pool - infra

```bash
  rosa create machinepool --cluster=rosa \
  --name=infra-pool \
  --instance-type="m6a.xlarge" \
  --replicas=2 \
  --labels="node-role.kubernetes.io/infra=" \
  --taints="node-role.kubernetes.io/infra=:NoSchedule"
``` 

# Separando ingress em node pool distinto

2. Cria o patch para o ingressController/default

```yaml
cat <<EOF > ingress-patch.yaml
spec:
  nodePlacement:
    nodeSelector:
      matchLabels:
        node-role.kubernetes.io/infra: ""
    tolerations:
    - effect: NoSchedule
      key: node-role.kubernetes.io/infra
      operator: Exists
EOF
```

3. Aplica o patch

```
oc patch ingresscontroller/default -n openshift-ingress-operator --type=merge --patch-file=ingress-patch.yaml
```

4. Valida a migracao de ingress
```oc get pods -n openshift-ingress -o wide```

