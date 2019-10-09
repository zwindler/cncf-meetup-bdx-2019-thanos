# CNCF Meetup Bordeaux 2019 Thanos

## Besoin de métriques Prometheus à long terme ? Thanos fera des Marvels !

Prometheus, aussi puissant soit il, repose sur plusieurs postulats pas forcément compatibles avec tous les cas d'usage. Les métriques doivent être stockées sur un medium local et rapide, il faut un serveur par zone (failure domain) et donc potentiellement un grand nombre de serveur Prometheus indépendants, difficile à corréler, ...

Arrive alors Thanos (rien à voir avec le bonhomme violet avec un gant doré, je vous rassure)

Celui ci va nous permettre de stocker nos métriques infiniment, tout en les rassemblant au sein d'une même source de données et en améliorant la performance de nos requêtes les plus coûteuses.

## Installer Prometheus sans containerisation

```bash
wget https://github.com/prometheus/prometheus/releases/download/v2.12.0/prometheus-2.12.0.linux-amd64.tar.gz
tar xzf prometheus-2.12.0.linux-amd64.tar.gz
sudo mv prometheus-2.12.0.linux-amd64/ /usr/share/prometheus
sudo useradd -u 3434 -d /usr/share/prometheus prometheus
sudo mkdir -p /var/lib/prometheus/data
```

* [https://prometheus.io/download/]

```bash
sudo chown prometheus:prometheus /var/lib/prometheus/data
sudo chown -R prometheus:prometheus /usr/share/prometheus
```

Lancer le binaire à la main pour vérifier que ça fonctionne

```bash
/usr/share/prometheus/prometheus --config.file=/usr/share/prometheus/prometheus.yml
[...]
level=info ts=2019-09-20T14:56:18.244Z caller=main.go:768 msg="Completed loading of configuration file" filename=/usr/share/prometheus/prometheus.yml
level=info ts=2019-09-20T14:56:18.244Z caller=main.go:623 msg="Server is ready to receive web requests."
```

Créer un script systemd pour démarrage automatique

```bash
cat > /etc/systemd/system/prometheus.service << EOF
[Unit]
Description=Prometheus Server
Documentation=https://prometheus.io/docs/introduction/overview/
After=network-online.target

[Service]
User=prometheus
Restart=on-failure
WorkingDirectory=/usr/share/prometheus
ExecStart=/usr/share/prometheus/prometheus --config.file=/usr/share/prometheus/prometheus.yml

[Install]
WantedBy=multi-user.target
EOF
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable prometheus
sudo systemctl start prometheus
sudo systemctl status prometheus
```

## Installer le prometheus exporter pour Proxmox VE

*[https://github.com/znerol/prometheus-pve-exporter]

### Sur un des serveurs Proxmox

Créer un utilisateur pour tout le cluster, ainsi qu'un groupe et y affecter les bons privilèges

```bash
pveum groupadd monitoring -comment 'Monitoring group'
pveum aclmod / -group monitoring -role PVEAuditor
pveum user add pve_exporter@pve
pveum usermod pve_exporter@pve -group monitoring
pveum passwd pve_exporter@pve
```

### Sur les serveurs Proxmox

Installer le prometheus exporter

```bash
apt-get install python-pip
pip install prometheus-pve-exporter
```

Créer le fichier de configuration associé

```
mkdir -p /usr/share/pve_exporter/
cat > /usr/share/pve_exporter/pve_exporter.yml << EOF
default:
    user: pve_exporter@pve
    password: myawesomepassword
    verify_ssl: false
EOF
```

Lancer le binaire à la main pour vérifier que tout fonctionne

```bash
pve_exporter /usr/share/pve_exporter/pve_exporter.yml &
```

TODO un script de démarrage systemd

Ajouter le job dans la conf prometheus et redémarrer le démon

```YAML
vi /usr/share/prometheus/prometheus.yml 
[...]
scrape_configs:
[...]
  - job_name: 'pve'
    static_configs:
      - targets:
        - 10.10.10.1:9221  # Proxmox VE node with PVE exporter.
        - 10.10.10.4:9221  # Proxmox VE node with PVE exporter.
    metrics_path: /pve
    params:
      module: [default]
```

```bash
systemctl restart prometheus.service 
```

## Installer Grafana en baremetal

* [https://grafana.com/grafana/download]

```bash
wget https://dl.grafana.com/oss/release/grafana_6.3.6_amd64.deb
sudo dpkg -i grafana_6.3.6_amd64.deb
```

```bash
systemctl start grafana-server
systemctl enable grafana-server
```

Une fois sur l’interface de Grafana, authentifiez‐vous en tant qu’administrateur (compte admin, mot de passe admin).

Seconde étape : la configuration d’une source de données. Au niveau des champs, vous pourrez rentrer les valeurs suivantes :

Name : nom de la connexion (ex : Prometheus) ;
Type : choisir Prometheus ;
URL : rentrer http://localhost:9090 ;
Access : laisser la valeur proxy.
Laissez les autres champs à la valeur par défaut et cliquez sur Add.

Récupérer le dashboard pour le "prometheus proxmox VE exporter"

*[https://grafana.com/grafana/dashboards/10347]

## Prometheus dans un kube

### Prérequis

Créer un namespace à part

```bash
kubectl --context=zwindlerk8s create ns monitoring
namespace/monitoring created
```

Déployer Tiller pour utiliser Helm (créer un compte de service, lui donenr les droits sur le namespace monitoring).

```bash
kubectl --context=zwindlerk8s --namespace=kube-system create sa tiller

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

kubectl --context=zwindlerk8s apply -f rb-tiller-admin.yaml 

helm init --kube-context=zwindlerk8s --service-account tiller
```

### Déployer un Ingress Controller (Traefik ici)

* [https://github.com/helm/charts/tree/master/stable/traefik]

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
helm install --kube-context=zwindlerk8s stable/traefik --name traefik-ingress-controller --namespace kube-system -f traefik-values.yaml
```

* [https://github.com/containous/traefik]
* [https://supergiant.io/blog/using-traefik-as-ingress-controller-for-your-kubernetes-cluster/]

### Déployer Prometheus

* [https://github.com/helm/charts/tree/master/stable/prometheus]

Déployer Prometheus via helm

```bash
helm install --kube-context=zwindlerk8s --namespace monitoring --name prometheus  stable/prometheus -f prom-values.yaml
```

Créer un Ingress pour accéder à prometheus

```bash
cat > ingress-traefik.yaml << EOF
ssl:
   enabled: true
kubernetes:
  ingressClass: "traefik"
ssl.generateTLS: true
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
  challengeType: http-01
rbac:
  enabled: true
EOF

kubectl --context=zwindlerk8s apply -f ingress-traefik.yaml
```

* [Pour gérer les certifs Let's Encrypt](https://docs.traefik.io/v2.0/user-guides/crd-acme/)

## Déployer le Thanos sidecar pour Prometheus dans Kubernetes

* [https://github.com/thanos-io/thanos/blob/master/tutorials/kubernetes-helm/README.md]

Pour déployer Thanos dans un Kubernetes qui a déjà Prometheus, on va d'abord ajouter Thanos en tant que sidecar du Prometheus server et modifier les valeurs de rétention de Prometheus pour économiser de l'espace disque local.

```bash
helm upgrade --kube-context=zwindlerk8s --namespace monitoring prometheus stable/prometheus -f prom-values-with-thanos-retention.yaml
```

## Déployer une storage Gateway par cluster Kubernetes

configurer les variables suivantes pour un cluster donné

```bash
K8SCLUSTER=zwindlerk8s
AZURERG=${K8SCLUSTER}_rg
AZURESTORAGEACCOUNT=thanos${K8SCLUSTER}
AZURESUBSCRIPTION='Visual Studio Professional'
```

Créer un compte de stockage chez un cloud provider (ou autre solution compatible avec thanos pour le stockage à long terme)

```bash
az storage account create -n ${AZURESTORAGEACCOUNT} -g ${AZURERG} --sku Standard_LRS --subscription "${AZURESUBSCRIPTION}"

AZURESTORAGEACCOUNTKEY=`az storage account keys list --account-name "${AZURESTORAGEACCOUNT}" -g ${AZURERG} --output tsv  --subscription "${AZURESUBSCRIPTION}" | grep key1 | cut -f3`

cat > cm-thanos-store-${K8SCLUSTER}.yaml << EOF
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

kubectl --context=zwindlerk8s apply -f cm-thanos-store-${K8SCLUSTER}.yaml
```

Créer un service pour le Thanos Store

```bash
cat > svc-thanos-store-${K8SCLUSTER}.yaml << EOF
apiVersion: v1
kind: Service
metadata:
  name: thanos-store-${K8SCLUSTER}
  namespace: monitoring
spec:
  ports:
  - name: grpc
    port: 10901
    protocol: TCP
    targetPort: grpc
  selector:
    app: thanos-store
    cluster: ${K8SCLUSTER}
  sessionAffinity: None
  type: ClusterIP
EOF

kubectl --context=zwindlerk8s apply -f svc-thanos-store-${K8SCLUSTER}.yaml
```

Déployer le Thanos Store

```bash
kubectl --context=zwindlerk8s apply -f deploy-thanos-store-${K8SCLUSTER}.yaml
```

Déployer le Thanos Query

```bash
kubectl --context=zwindlerk8s apply -f deploy-thanos-query.yaml
```

## Bibliographie

* [https://linuxfr.org/news/decouverte-de-l-outil-de-supervision-prometheus#configuration-du-serveur-prometheus]
