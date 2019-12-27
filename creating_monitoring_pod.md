### Pod creation

```
yum -y install podman
podman pod create --name monitoring -p 3000 -p 9090 -p 9093 -p 80 -p 443
mkdir -p /var/www
mkdir -p /var/nginx/conf
podman run --pod monitoring --name nginx -v /var/www:/usr/share/nginx/html:ro -v /var/nginx/conf:/etc/nginx:ro -d nginx
podman run --pod monitoring --restart always --name prometheus -d prom/prometheus
podman run --pod monitoring --restart always --name alertmanager -d prom/alertmanager
podman volume create grafana-storage
podman run --pod monitoring --restart always -d --name=grafana \
-v grafana-storage:/home/ec2-user/.local/share/containers/storage/volumes/grafana-storage \
-e "GF_SERVER_ROOT_URL=http://localhost/grafana" \
-e "GF_INSTALL_PLUGINS=camptocamp-prometheus-alertmanager-datasource,grafana-piechart-panel" \
grafana/grafana 
podman generate kube monitoring > monitoring.yaml
```

podman will autogenerate a kube file. It's not perfect so we have to manually edit it.

- Remove the double port entries, so the port is only opened at the correct container
- Remove the "/bin/alertmanager" and "/bin/prometheus" entries
- Add the "--web.external-url=" entries so we can use nginx in combination

 
