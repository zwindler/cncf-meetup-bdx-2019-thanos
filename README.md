# CNCF Meetup Bordeaux 2019 Thanos

## Besoin de métriques Prometheus à long terme ? Thanos fera des Marvels !

Prometheus, aussi puissant soit il, repose sur plusieurs postulats pas forcément compatibles avec tous les cas d'usage. Les métriques doivent être stockées sur un medium local et rapide, il faut un serveur par zone (failure domain) et donc potentiellement un grand nombre de serveur Prometheus indépendants, difficile à corréler, ...

Arrive alors Thanos (rien à voir avec le bonhomme violet avec un gant doré)

Celui ci va nous permettre de stocker nos métriques infiniment, tout en les rassemblant au sein d'une même source de données et en améliorant la performance de nos requêtes les plus coûteuses.

## Prérequis

Avoir 2 clusters Kubernetes

```bash
kubectl config get-contexts
CURRENT   NAME                CLUSTER             AUTHINFO                                 NAMESPACE
          aks1-eu             aks1-eu             clusterUser_aks1-eu-rg_aks1-eu
*         aks2-us             aks2-us             clusterUser_aks2-us-rg_aks2-us  

for K8SCLUSTER in ${CLUSTERLIST}; do kubectl --context=${K8SCLUSTER} get nodes; done
NAME                       STATUS   ROLES   AGE   VERSION
aks-agentpool-28179912-0   Ready    agent   20m   v1.14.6
NAME                       STATUS   ROLES   AGE     VERSION
aks-agentpool-41438152-0   Ready    agent   3m33s   v1.14.6
```

Renseigner la liste de tous les clusters dans une variable d'environnement

```bash
CLUSTERLIST="aks1-eu aks2-us"
```

Créer un namespace à part pour tous les tools de monitoring

```bash
for K8SCLUSTER in ${CLUSTERLIST}; do kubectl --context=${K8SCLUSTER} create ns monitoring; done
```

Déployer Tiller pour utiliser Helm (créer un compte de service, lui donner les droits sur le namespace monitoring).

```bash
for K8SCLUSTER in ${CLUSTERLIST}; do kubectl --context=${K8SCLUSTER} --namespace=kube-system create sa tiller; done

cat > rb-tiller-admin.yaml << EOF
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  annotations: {}
  name: monitoring-admin
  namespace: monitoring
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: admin
subjects:
- kind: ServiceAccount
  name: tiller
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  annotations: {}
  name: kube-system-admin
  namespace: kube-system
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: tiller
  namespace: kube-system
EOF

for K8SCLUSTER in ${CLUSTERLIST}; do kubectl --context=${K8SCLUSTER} apply -f rb-tiller-admin.yaml; done

for K8SCLUSTER in ${CLUSTERLIST}; do helm init --kube-context=${K8SCLUSTER} --service-account tiller; done
```

### Déployer un Ingress Controller (Traefik ici)

* [Chart Helm officiel de Traefik en tant qu'Ingress Controller](https://github.com/helm/charts/tree/master/stable/traefik)

```bash
cat > traefik-values.yaml << EOF
ssl:
   enabled: true
   enforced: true
kubernetes:
  ingressClass: "traefik"
acme:
  enabled: true
  email: zwindl3r@gmail.com
  onHostRule: true
  staging: true
  logging: false
  # Configure a Let's Encrypt certificate to be managed by default.
  # This is the only way to request wildcard certificates (works only with dns challenge).
  domains:
    enabled: false
EOF
```

Déployer Traefik

```bash
for K8SCLUSTER in ${CLUSTERLIST}; do helm install --kube-context=${K8SCLUSTER} stable/traefik --name traefik-ingress-controller --namespace kube-system -f traefik-values.yaml; done
```

* [Github de traefik](https://github.com/containous/traefik)
* [Tutoriel configuration de Traefik comme Ingress Controller par Supergiant](https://supergiant.io/blog/using-traefik-as-ingress-controller-for-your-kubernetes-cluster/)

Récupérer l'IP d'entrée pour créer des entrées DNS correspondantes

```bash
for K8SCLUSTER in ${CLUSTERLIST}; do echo "$K8SCLUSTER `kubectl describe svc traefik-ingress-controller --context=$K8SCLUSTER --namespace kube-system | grep Ingress | awk '{print $3}'`"; done
aks1-eu 52.137.9.241
aks2-us 104.41.141.41
```

## Déployer Prometheus dans nos clusters Kubernetes

* [Helm chart officiel de Prometheus](https://github.com/helm/charts/tree/master/stable/prometheus)
* [Tutoriel pour déployer Thanos sidecar en utilisant Helm](https://github.com/thanos-io/thanos/blob/master/tutorials/kubernetes-helm/README.md)

Déployer Prometheus via helm avec thanos en sidecar

```bash
for K8SCLUSTER in ${CLUSTERLIST}; do cat > prom-values-with-thanos-${K8SCLUSTER}.yaml << EOF
rbac:
  create: true
serviceAccounts:
  alertmanager:
    create: true
  kubeStateMetrics:
    create: true
  nodeExporter:
    create: true
  pushgateway:
    create: true
server:
  create: true
  retention: 12h
  extraArgs:
    storage.tsdb.min-block-duration: 2h
    storage.tsdb.max-block-duration: 2h
  sidecarContainers:
  - name: thanos-sidecar
    image: quay.io/thanos/thanos:v0.7.0
    args:
    - "sidecar"
    - "--log.level=debug"
    - "--tsdb.path=/data/"
    - "--prometheus.url=http://127.0.0.1:9090"
    - "--objstore.config-file=/etc/thanos-config/azure-storage.yml"
    ports:
    - name: sidecar-http
      containerPort: 10902
    - name: grpc
      containerPort: 10901
    - name: cluster
      containerPort: 10900
    volumeMounts:
    - name: storage-volume
      mountPath: /data
    - name: config-volume
      mountPath: /etc/prometheus-config
      readOnly: false
    - name: thanos-config-volume
      mountPath: /etc/thanos-config
  global:
    scrape_interval: 5s
    scrape_timeout: 4s
    external_labels:
      cluster: ${K8SCLUSTER}
      replica: 1
    evaluation_interval: 5s
  extraVolumes:
  - configMap:
      defaultMode: 420
      name: thanos-storage-${K8SCLUSTER}
    name: thanos-config-volume
  extraVolumeMounts:
  - name: thanos-config-volume
    mountPath: /etc/thanos-config
EOF
done

for K8SCLUSTER in ${CLUSTERLIST}; do helm install --kube-context=${K8SCLUSTER} --namespace monitoring --name prometheus stable/prometheus -f prom-values-with-thanos-${K8SCLUSTER}.yaml ; done

#ou, si il est déjà déployé
for K8SCLUSTER in ${CLUSTERLIST}; do helm upgrade --kube-context=${K8SCLUSTER} --namespace monitoring prometheus stable/prometheus -f prom-values-with-thanos-${K8SCLUSTER}.yaml ; done
```

Créer un service pour accéder au sidecar Thanos

```bash
for K8SCLUSTER in ${CLUSTERLIST}; do cat > svc-prom-thanos-sidecar-${K8SCLUSTER}.yaml << EOF
apiVersion: v1
kind: Service
metadata:
  name: prom-thanos-sidecar-${K8SCLUSTER}
  namespace: monitoring
spec:
  ports:
  - name: grpc
    port: 10901
    nodePort: 30901
    protocol: TCP
    targetPort: grpc
  selector:
    app: prometheus
    component: server
    release: prometheus
  sessionAffinity: None
  type: NodePort
EOF
kubectl --context=${K8SCLUSTER} apply -f svc-prom-thanos-sidecar-${K8SCLUSTER}.yaml
done
```

Créer un Ingress pour accéder à prometheus

```bash
for K8SCLUSTER in ${CLUSTERLIST}; do cat > ingress-traefik-prometheus-${K8SCLUSTER}.yaml << EOF
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: prometheus
  namespace: monitoring
  annotations:
    kubernetes.io/ingress.class: traefik
spec:
  rules:
  - host: prometheus-${K8SCLUSTER}.zwindler.fr
    http:
      paths:
      - path: /
        backend:
          serviceName: prometheus-server
          servicePort: 80
EOF
kubectl --context=${K8SCLUSTER} apply -f ingress-traefik-prometheus-${K8SCLUSTER}.yaml
done
```

TODO

* [Pour gérer les certifs Let's Encrypt](https://docs.traefik.io/v2.0/user-guides/crd-acme/)

## Créer un nouveau replica de Prometheus sur nos clusters Kubernetes

Déployer Prometheus via helm avec thanos en sidecar

```bash
for K8SCLUSTER in ${CLUSTERLIST}; do cat > prom-values-with-thanos-${K8SCLUSTER}-replica.yaml << EOF
rbac:
  create: true
alertmanager:
  enabled: false
alertmanagerFiles:
  alertmanager.yml: ""
kubeStateMetrics:
  enabled: false
nodeExporter:
  enabled: false
pushgateway:
  enabled: false
server:
  create: true
  retention: 12h
  extraArgs:
    storage.tsdb.min-block-duration: 2h
    storage.tsdb.max-block-duration: 2h
  sidecarContainers:
  - name: thanos-sidecar
    image: quay.io/thanos/thanos:v0.7.0
    args:
    - "sidecar"
    - "--log.level=debug"
    - "--tsdb.path=/data/"
    - "--prometheus.url=http://127.0.0.1:9090"
    - "--objstore.config-file=/etc/thanos-config/azure-storage.yml"
    ports:
    - name: sidecar-http
      containerPort: 10902
    - name: grpc
      containerPort: 10901
    - name: cluster
      containerPort: 10900
    volumeMounts:
    - name: storage-volume
      mountPath: /data
    - name: config-volume
      mountPath: /etc/prometheus-config
      readOnly: false
    - name: thanos-config-volume
      mountPath: /etc/thanos-config
  global:
    scrape_interval: 5s
    scrape_timeout: 4s
    external_labels:
      cluster: ${K8SCLUSTER}
      replica: 2
    evaluation_interval: 5s
  extraVolumes:
  - configMap:
      defaultMode: 420
      name: thanos-storage-${K8SCLUSTER}
    name: thanos-config-volume
  extraVolumeMounts:
  - name: thanos-config-volume
    mountPath: /etc/thanos-config
EOF
done

for K8SCLUSTER in ${CLUSTERLIST}; do helm install --kube-context=${K8SCLUSTER} --namespace monitoring --name prometheus-replica stable/prometheus -f prom-values-with-thanos-${K8SCLUSTER}-replica.yaml; done

#ou, si il est déjà déployé
helm upgrade --kube-context=${K8SCLUSTER} --namespace monitoring prometheus-replica stable/prometheus -f prom-values-with-thanos-${K8SCLUSTER}-replica.yaml
```

Créer un service pour accéder au sidecar Thanos

```bash
for K8SCLUSTER in ${CLUSTERLIST}; do cat > svc-prom-thanos-sidecar-${K8SCLUSTER}-replica.yaml << EOF
apiVersion: v1
kind: Service
metadata:
  name: prom-thanos-sidecar-${K8SCLUSTER}-replica
  namespace: monitoring
spec:
  ports:
  - name: grpc
    port: 10901
    nodePort: 30911
    protocol: TCP
    targetPort: grpc
  selector:
    app: prometheus
    component: server
    release: prometheus-replica
  sessionAffinity: None
  type: NodePort
EOF
kubectl --context=${K8SCLUSTER} apply -f svc-prom-thanos-sidecar-${K8SCLUSTER}-replica.yaml
done
```

Créer un Ingress pour accéder à prometheus

```bash
for K8SCLUSTER in ${CLUSTERLIST}; do cat > ingress-traefik-prometheus-${K8SCLUSTER}-replica.yaml << EOF
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: prometheus-replica
  namespace: monitoring
  annotations:
    kubernetes.io/ingress.class: traefik
spec:
  rules:
  - host: prometheus-${K8SCLUSTER}-replica.zwindler.fr
    http:
      paths:
      - path: /
        backend:
          serviceName: prometheus-replica-server
          servicePort: 80
EOF
kubectl --context=${K8SCLUSTER} apply -f ingress-traefik-prometheus-${K8SCLUSTER}-replica.yaml
done
```

## Déployer Grafana

On va utiliser une nouvelle fois Helm pour déployer Grafana. Crer un fichier values dans lequel on modifiera le mot de passe de l'admin.

```bash
cat > grafana-values.yaml << EOF
adminUser: admin
adminPassword: myawesomepassword
EOF
```

```bash
for K8SCLUSTER in ${CLUSTERLIST}; do helm install --kube-context=${K8SCLUSTER} --namespace monitoring --name grafana stable/grafana -f grafana-values.yaml; done

##ou si on le met à jour
for K8SCLUSTER in ${CLUSTERLIST}; do helm upgrade  --kube-context=${K8SCLUSTER} --namespace monitoring grafana stable/grafana -f grafana-values.yaml; done
```

Créer un Ingress pour accéder à grafana

```bash
for K8SCLUSTER in ${CLUSTERLIST}; do cat > ingress-traefik-grafana-${K8SCLUSTER}.yaml << EOF
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: grafana
  namespace: monitoring
  annotations:
    kubernetes.io/ingress.class: traefik
spec:
  rules:
  - host: grafana-${K8SCLUSTER}.zwindler.fr
    http:
      paths:
      - path: /
        backend:
          serviceName: grafana
          servicePort: 80
EOF
kubectl --context=${K8SCLUSTER} apply -f ingress-traefik-grafana-${K8SCLUSTER}.yaml
done
```

## Déployer une storage Gateway par cluster Kubernetes

configurer les variables suivantes pour un cluster donné, puis créer un compte de stockage chez un cloud provider (ou autre solution compatible avec thanos pour le stockage à long terme)

```bash
AZURERG=thanos-rg
AZURESTORAGEACCOUNT=thanosstorageaccount
AZURESUBSCRIPTION='Essai gratuit'
AZURELOCATION=francecentral

az group create -l ${AZURELOCATION} -n ${AZURERG}

az storage account create -n ${AZURESTORAGEACCOUNT} -g ${AZURERG} --kind StorageV2 --sku Standard_LRS --subscription "${AZURESUBSCRIPTION}"

AZURESTORAGEACCOUNTKEY=`az storage account keys list --account-name "${AZURESTORAGEACCOUNT}" -g ${AZURERG} --output tsv  --subscription "${AZURESUBSCRIPTION}" | grep key1 | cut -f3`

```

Pour chaque cluster, créer une configmap contenant les informations pour accéder aux buckets

```bash
for K8SCLUSTER in ${CLUSTERLIST}; do cat > cm-thanos-store-${K8SCLUSTER}.yaml << EOF
apiVersion: v1
data:
  azure-storage.yml: |
    config:
      container: ${K8SCLUSTER}
      storage_account: ${AZURESTORAGEACCOUNT}
      storage_account_key: ${AZURESTORAGEACCOUNTKEY}
    type: AZURE
kind: ConfigMap
metadata:
  labels:
    app: prometheus-thanos
    cluster: ${K8SCLUSTER}
  name: thanos-storage-${K8SCLUSTER}
  namespace: monitoring
EOF

kubectl --context=${K8SCLUSTER} apply -f cm-thanos-store-${K8SCLUSTER}.yaml
done
```

Créer un service pour le Thanos Store

```bash
for K8SCLUSTER in ${CLUSTERLIST}; do cat > svc-thanos-store-${K8SCLUSTER}.yaml << EOF
apiVersion: v1
kind: Service
metadata:
  name: thanos-store-${K8SCLUSTER}
  namespace: monitoring
spec:
  ports:
  - name: grpc
    port: 10901
    nodePort: 31901
    protocol: TCP
    targetPort: grpc
  selector:
    app: thanos-store
    cluster: ${K8SCLUSTER}
  sessionAffinity: None
  type: NodePort
EOF

kubectl --context=${K8SCLUSTER} apply -f svc-thanos-store-${K8SCLUSTER}.yaml
done

```

Déployer le Thanos Store sur chaque cluster pour gérer les métriques à long terme

```bash
for K8SCLUSTER in ${CLUSTERLIST}; do cat > deploy-thanos-store-${K8SCLUSTER}.yaml << EOF
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    app: thanos-store
    cluster: ${K8SCLUSTER}
  name: thanos-store-${K8SCLUSTER}
  namespace: monitoring
spec:
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: thanos-store
      cluster: ${K8SCLUSTER}
  strategy:
    type: Recreate
  template:
    metadata:
      annotations:
        prometheus.io/port: "10902"
        prometheus.io/scrape: "true"
      creationTimestamp: null
      labels:
        app: thanos-store
        cluster: ${K8SCLUSTER}
    spec:
      containers:
      - args:
        - store
        - --log.level=info
        - --http-address=0.0.0.0:10902
        - --grpc-address=0.0.0.0:10901
        - --objstore.config-file=/etc/config/azure-storage.yml
        - --index-cache-size=50GB
        - --chunk-pool-size=50GB
        - --data-dir=/data
        image: quay.io/thanos/thanos:v0.7.0
        imagePullPolicy: Always
        name: thanos-store
        ports:
        - containerPort: 10902
          name: http
          protocol: TCP
        - containerPort: 10901
          name: grpc
          protocol: TCP
        - containerPort: 10900
          name: cluster
          protocol: TCP
        volumeMounts:
        - mountPath: /etc/config
          name: config-volume
        - mountPath: /data
          name: mnt
      initContainers:
      - command:
        - sh
        - -c
        - rm -rf /data/*
        image: busybox
        imagePullPolicy: Always
        name: init-rm
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /data
          name: mnt
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
      volumes:
      - configMap:
          defaultMode: 420
          name: thanos-storage-${K8SCLUSTER}
        name: config-volume
      - hostPath:
          path: /mnt/thanos-storage-${K8SCLUSTER}/data
          type: DirectoryOrCreate
        name: mnt
EOF
done

for K8SCLUSTER in ${CLUSTERLIST}; do kubectl --context=${K8SCLUSTER} apply -f deploy-thanos-store-${K8SCLUSTER}.yaml; done
```

### Thanos Query

Déployer Thanos Query en eregistrant le chemin vers tous les Thanos (sidecar) pour les métriques instantannées et les Thanos Store pour les métriques long terme.

```bash
cat > deploy-thanos-query.yaml << EOF
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    app: thanos-query
  name: thanos-query
  namespace: monitoring
spec:
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: thanos-query
  template:
    metadata:
      annotations:
        prometheus.io/port: "10902"
        prometheus.io/scrape: "true"
      creationTimestamp: null
      labels:
        app: thanos-query
    spec:
      containers:
      - name: thanos-query
        image: quay.io/thanos/thanos:v0.7.0
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 10902
          name: http
          protocol: TCP
        - containerPort: 10901
          name: grpc
          protocol: TCP
        - containerPort: 10900
          name: cluster
          protocol: TCP
        args:
        - query
        - --log.level=info
        - --http-address=0.0.0.0:10902
        - --query.replica-label=replica
        - --query.auto-downsampling
EOF

for K8SCLUSTER in ${CLUSTERLIST}; do cat >> deploy-thanos-query.yaml << EOF
        - --store=prom-thanos-sidecar-${K8SCLUSTER}.zwindler.fr:30901
        - --store=prom-thanos-sidecar-${K8SCLUSTER}-replica.zwindler.fr:30911
        - --store=thanos-store-${K8SCLUSTER}.zwindler.fr:31901
EOF
done

for K8SCLUSTER in ${CLUSTERLIST}; do cat > ingress-traefik-thanos-query-${K8SCLUSTER}.yaml << EOF
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: thanos-query
  namespace: monitoring
  annotations:
    kubernetes.io/ingress.class: traefik
spec:
  rules:
  - host: thanos-${K8SCLUSTER}.zwindler.fr
    http:
      paths:
      - path: /
        backend:
          serviceName: thanos-query
          servicePort: 10902
EOF
done

for K8SCLUSTER in ${CLUSTERLIST}
do kubectl --context=${K8SCLUSTER} apply -f deploy-thanos-query.yaml
kubectl --context=${K8SCLUSTER} apply -f svc-thanos-query.yaml
kubectl --context=${K8SCLUSTER} apply -f ingress-traefik-thanos-query-${K8SCLUSTER}.yaml
done
```
