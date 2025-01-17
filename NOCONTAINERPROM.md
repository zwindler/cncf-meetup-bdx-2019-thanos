# CNCF Meetup Bordeaux 2019 Thanos

## Besoin de métriques Prometheus à long terme ? Thanos fera des Marvels !

Prometheus, aussi puissant soit il, repose sur plusieurs postulats pas forcément compatibles avec tous les cas d'usage. Les métriques doivent être stockées sur un medium local et rapide, il faut un serveur par zone (failure domain) et donc potentiellement un grand nombre de serveur Prometheus indépendants, difficile à corréler, ...

Arrive alors Thanos (rien à voir avec le bonhomme violet avec un gant doré)

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

## Bibliographie

* [https://linuxfr.org/news/decouverte-de-l-outil-de-supervision-prometheus#configuration-du-serveur-prometheus]
