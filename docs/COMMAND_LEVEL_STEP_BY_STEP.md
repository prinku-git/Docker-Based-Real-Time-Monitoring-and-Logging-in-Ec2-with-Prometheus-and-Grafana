## Docker Based Real Time Monitoring and Logging in Ec2 with Prometheus and Grafana

---

## 1. EC2 Instance Setup

### 1.1 Launch EC2

* AMI: Amazon Linux
* Instance type: t2.micro
* Auto-assign public IP: Enabled
* Security Group: Allow inbound ports for:

  * 22 (SSH)
  * 3000 (Grafana)
  * 9090 (Prometheus)
  * 9100 (Node Exporter)
  * 3100 (Loki)
  * 9093 (Alertmanager)

---

## 2. Connect to EC2 Instance

```bash
ssh -i your-key.pem ec2-user@<EC2_PUBLIC_IP>
```

Verify login:

```bash
whoami
```

---

## 3. System Update

```bash
sudo yum update -y
```

---

## 4. Install Docker

```bash
sudo yum install docker -y
```

Start Docker:

```bash
sudo systemctl start docker
```

Enable Docker on boot:

```bash
sudo systemctl enable docker
```

Add user to Docker group:

```bash
sudo usermod -aG docker ec2-user
```

Logout and login again.

Verify Docker:

```bash
docker --version
docker ps
```

---

## 5. Create Working Directory

```bash
mkdir monitoring
cd monitoring
```

---

## 6. Run Node Exporter (System Metrics)

```bash
docker run -d \
--name node-exporter \
-p 9100:9100 \
--restart unless-stopped \
prom/node-exporter
```

Verify:

```bash
docker ps
curl http://localhost:9100/metrics
```

---

## 7. Prometheus Setup

### 7.1 Create Prometheus Config

```bash
vi prometheus.yml
```

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: ['localhost:9090']

  - job_name: node_exporter
    static_configs:
      - targets: ['localhost:9100']
```

---

### 7.2 Run Prometheus

```bash
docker run -d \
--name prometheus \
-p 9090:9090 \
-v $(pwd)/prometheus.yml:/etc/prometheus/prometheus.yml \
--restart unless-stopped \
prom/prometheus
```

Verify:

```bash
docker ps
```

Access:

```
http://<EC2_PUBLIC_IP>:9090
```

Check **Status → Targets → UP**

---

## 8. Loki Setup (Log Aggregation)

### 8.1 Create Loki Config

```bash
vi loki-config.yml
```

```yaml
auth_enabled: false

server:
  http_listen_port: 3100

common:
  path_prefix: /loki
  storage:
    filesystem:
      chunks_directory: /loki/chunks
      rules_directory: /loki/rules
  replication_factor: 1
  ring:
    kvstore:
      store: inmemory

schema_config:
  configs:
    - from: 2020-10-24
      store: boltdb-shipper
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 24h
```

---

### 8.2 Run Loki

```bash
docker run -d \
--name loki \
-p 3100:3100 \
-v $(pwd)/loki-config.yml:/etc/loki/config.yml \
--restart unless-stopped \
grafana/loki:2.9.0 \
-config.file=/etc/loki/config.yml
```

Verify:

```bash
docker logs loki
```

---

## 9. Promtail Setup (Log Shipping)

### 9.1 Create Promtail Config

```bash
vi promtail-config.yml
```

```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://localhost:3100/loki/api/v1/push

scrape_configs:
  - job_name: system-logs
    static_configs:
      - targets:
          - localhost
        labels:
          job: varlogs
          __path__: /var/log/*.log
```

---

### 9.2 Run Promtail

```bash
docker run -d \
--name promtail \
-v /var/log:/var/log \
-v $(pwd)/promtail-config.yml:/etc/promtail/config.yml \
--restart unless-stopped \
grafana/promtail:2.9.0 \
-config.file=/etc/promtail/config.yml
```

Verify:

```bash
docker ps
docker logs promtail
```

---

## 10. Alertmanager Setup

### 10.1 Create Alertmanager Config

```bash
vi alertmanager.yml
```

```yaml
global:
  resolve_timeout: 5m

route:
  receiver: 'default-receiver'

receivers:
  - name: 'default-receiver'
```

---

### 10.2 Run Alertmanager

```bash
docker run -d \
--name alertmanager \
-p 9093:9093 \
-v $(pwd)/alertmanager.yml:/etc/alertmanager/alertmanager.yml \
--restart unless-stopped \
prom/alertmanager
```

Verify:

```bash
docker ps
```

Access:

```
http://<EC2_PUBLIC_IP>:9093
```

---

## 11. Grafana Setup

```bash
docker run -d \
--name grafana \
-p 3000:3000 \
--restart unless-stopped \
grafana/grafana
```

Verify:

```bash
docker ps
```

Access:

```
http://<EC2_PUBLIC_IP>:3000
```

Login:

* Username: admin
* Password: admin

(Change password after login)

---

## 12. Grafana Configuration

### 12.1 Add Prometheus Data Source

* URL: `http://localhost:9090`
* Save & Test

### 12.2 Add Loki Data Source

* URL: `http://localhost:3100`
* Save & Test

---

## 13. Validation Commands

```bash
docker ps
```

Check logs:

```bash
docker logs prometheus
docker logs loki
docker logs promtail
docker logs grafana
```

---

## 14. Final Validation Checklist

* Prometheus targets: **UP**
* Loki status: **Ready**
* Logs visible in Grafana Explore
* Metrics dashboards loading
* Alertmanager status: **Normal**
* All containers running

---

## 15. Conclusion

This documentation demonstrates a **complete command-level implementation** of a real-time monitoring and logging system on AWS EC2 using Docker, Prometheus, Grafana, Loki, Promtail, and Alertmanager.

---

**Priyanka Rajagopal**
📧 [priyankaraj0919@gmail.com](mailto:priyankaraj0919@gmail.com)
🔗 LinkedIn: [https://www.linkedin.com/in/priyankaraj09](https://www.linkedin.com/in/priyankaraj09)

