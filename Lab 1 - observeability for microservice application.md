# Infrastructure Monitoring using Prometheus and Grafana on AWS

This guide provides a detailed, step-by-step approach to set up observability for a microservices application using Prometheus and Grafana on AWS.

---

## Prerequisites

- AWS account and access to AWS Console
- AWS CLI installed
- User account with `sudo` privileges
- Sufficient storage and internet connectivity

---

## Step 1: Launch EC2 Instance

1. **Login to AWS Console and navigate to EC2.**
2. Click **Launch Instance**.
3. Provide a name for your instance.
4. Select **Ubuntu (t2.micro)** as the AMI.
5. Create a key pair for SSH access.
6. Create a security group and add inbound rules for:
   - SSH (22): For remote access
   - Prometheus (9090): For Prometheus web UI
   - Grafana (3000): For Grafana dashboard
   - Payment Service (8001): For microservice endpoint
   - Balance Service (8001): For microservice endpoint
7. Launch the instance and wait for it to be running.

**SSH into your instance:**

```sh
chmod 400 SRE_Demo.pem
ssh -i "SRE_Demo.pem" ubuntu@ec2-<public-ip>.us-east-2.compute.amazonaws.com
```

---

## Step 2: Update System

Update the package index to ensure you have the latest repositories:

```sh
sudo apt update -y
```

---

## Step 3: Install Prometheus

1. **Create Prometheus user and directories:**
   - Prometheus should run as a non-login system user for security.
   ```sh
   sudo useradd --no-create-home --shell /bin/false prometheus
   sudo mkdir /etc/prometheus
   sudo mkdir /var/lib/prometheus
   sudo chown prometheus:prometheus /var/lib/prometheus
   ```
2. **Download and extract Prometheus:**
   - Download the latest Prometheus release and extract it.
   ```sh
   cd /tmp
   wget https://github.com/prometheus/prometheus/releases/download/v2.46.0/prometheus-2.46.0.linux-amd64.tar.gz
   tar -xvf prometheus-2.46.0.linux-amd64.tar.gz
   cd prometheus-2.46.0.linux-amd64
   sudo mv consoles* /etc/prometheus
   sudo mv prometheus.yml /etc/prometheus
   sudo chown -R prometheus:prometheus /etc/prometheus
   sudo mv prometheus /usr/local/bin/
   sudo chown prometheus:prometheus /usr/local/bin/prometheus
   ```
3. **Create Prometheus systemd service:**

   - This allows Prometheus to run as a service and restart automatically.

   ```ini
   # /etc/systemd/system/prometheus.service
   [Unit]
   Description=Prometheus
   Wants=network-online.target
   After=network-online.target

   [Service]
   User=prometheus
   Group=prometheus
   Type=simple
   ExecStart=/usr/local/bin/prometheus \
     --config.file /etc/prometheus/prometheus.yml \
     --storage.tsdb.path /var/lib/prometheus/ \
     --web.console.templates=/etc/prometheus/consoles \
     --web.console.libraries=/etc/prometheus/console_libraries

   [Install]
   WantedBy=multi-user.target
   ```

4. **Start Prometheus:**
   - Reload systemd and start Prometheus.
   ```sh
   sudo systemctl daemon-reload
   sudo systemctl start prometheus
   sudo systemctl enable prometheus
   sudo systemctl status prometheus
   ```
5. **Access Prometheus UI:**
   - Open `http://<server-ip>:9090` in your browser to verify Prometheus is running.

---

## Step 4: Install Node Exporter

Node Exporter exposes hardware and OS metrics for Prometheus to scrape.

1. **Download and extract Node Exporter:**
   ```sh
   cd /tmp
   wget https://github.com/prometheus/node_exporter/releases/download/v1.6.1/node_exporter-1.6.1.linux-amd64.tar.gz
   sudo tar xvfz node_exporter-1.6.1.linux-amd64.tar.gz
   sudo mv node_exporter-1.6.1.linux-amd64/node_exporter /usr/local/bin/
   ```
2. **Create node_exporter user:**
   - Run Node Exporter as a system user for security.
   ```sh
   sudo useradd -rs /bin/false node_exporter
   ```
3. **Create Node Exporter systemd service:**

   ```ini
   # /etc/systemd/system/node_exporter.service
   [Unit]
   Description=Node Exporter
   After=network.target

   [Service]
   User=node_exporter
   Group=node_exporter
   Type=simple
   ExecStart=/usr/local/bin/node_exporter

   [Install]
   WantedBy=multi-user.target
   ```

4. **Start Node Exporter:**
   ```sh
   sudo systemctl daemon-reload
   sudo systemctl start node_exporter
   sudo systemctl enable node_exporter
   sudo systemctl status node_exporter
   ```
5. **Configure Prometheus to scrape Node Exporter:**
   - Edit `/etc/prometheus/prometheus.yml` and add:
   ```yaml
   - job_name: "Node_Exporter"
     scrape_interval: 5s
     static_configs:
       - targets: ["<Server_IP>:9100"]
   ```
   - Restart Prometheus:
   ```sh
   sudo systemctl restart prometheus
   ```
   - Check metrics in browser: `http://<server-ip>:9100/metrics`

---

## Step 5: Install Grafana

Grafana provides dashboards and visualization for your metrics.

1. **Add Grafana GPG key and repository:**
   ```sh
   wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
   sudo add-apt-repository "deb https://packages.grafana.com/oss/deb stable main"
   sudo apt update
   sudo apt install grafana
   ```
   _If you face issues, run:_
   ```sh
   sudo apt install -y software-properties-common
   sudo wget -q -O /usr/share/keyrings/grafana.key https://packages.grafana.com/gpg.key
   echo "deb [signed-by=/usr/share/keyrings/grafana.key] https://packages.grafana.com/oss/deb stable main" | sudo tee /etc/apt/sources.list.d/grafana.list > /dev/null
   sudo apt update
   sudo apt install grafana -y
   ```
2. **Start Grafana:**
   ```sh
   sudo systemctl start grafana-server
   sudo systemctl enable grafana-server
   sudo systemctl status grafana-server
   ```
3. **Access Grafana UI:**
   - Open `http://<server-ip>:3000` in your browser
   - Default login: `admin` / `admin`

---

## Step 6: Create Microservices Node.js App

This app exposes endpoints for payment and balance, and a `/metrics` endpoint for Prometheus.

1. **Install Node.js and dependencies:**
   ```sh
   cd /tmp
   mkdir microservice-monitoring && cd microservice-monitoring
   sudo apt install npm
   npm init -y
   npm install express prom-client cors
   ```
2. **Create `server.js`:**

   ```js
   const express = require("express");
   const client = require("prom-client");
   const cors = require("cors");

   const app = express();
   app.use(cors());

   // Prometheus metrics setup
   const register = new client.Registry();
   client.collectDefaultMetrics({ register });

   const httpRequestDurationMicroseconds = new client.Histogram({
     name: "http_request_duration_seconds",
     help: "Duration of HTTP requests in seconds",
     labelNames: ["service", "method"],
     buckets: [0.1, 0.5, 1, 2, 5],
   });
   register.registerMetric(httpRequestDurationMicroseconds);

   // Payment Service
   app.get("/pay", (req, res) => {
     const start = Date.now();
     setTimeout(() => {
       const duration = (Date.now() - start) / 1000;
       httpRequestDurationMicroseconds
         .labels("payment", "GET")
         .observe(duration);
       res.send("Payment processed!");
     }, Math.random() * 1000); // Simulate latency
   });

   // Balance Service
   app.get("/balance", (req, res) => {
     const start = Date.now();
     setTimeout(() => {
       const duration = (Date.now() - start) / 1000;
       httpRequestDurationMicroseconds
         .labels("balance", "GET")
         .observe(duration);
       res.send("Balance: $1000");
     }, Math.random() * 500); // Simulate latency
   });

   // Prometheus Metrics Endpoint
   app.get("/metrics", async (req, res) => {
     res.set("Content-Type", register.contentType);
     res.end(await register.metrics());
   });

   // Start server on port 8001
   app.listen(8001, () => {
     console.log("Server running on port 8001");
   });
   ```

3. **Run the application:**
   ```sh
   node server.js
   # To run in background:
   nohup node server.js &
   # incase of nohup: ignoring input and appending output to 'nohup.out'
   nohup node server.js </dev/null &>/dev/null &
   ```
4. **Configure Prometheus to scrape microservices:**
   - Edit `/etc/prometheus/prometheus.yml` and add:
   ```yaml
   - job_name: "microservices"
     static_configs:
       - targets: ["<hostname-service_ip>:8001"]
   ```
   - Restart Prometheus:
   ```sh
   sudo systemctl restart prometheus
   ```
   - Test metrics endpoint:
   ```sh
   curl http://localhost:8001/metrics
   ```

---

## Step 7: Visualize Metrics in Grafana

1. **Log in to Grafana:**
   - Open `http://localhost:3000` (default: admin/admin)
2. **Add Prometheus as a data source:**
   - Go to **Configuration** (gear icon)
   - Click **Data Sources** > **Add data source**
   - Select **Prometheus**
   - Set URL: `http://localhost:9090`
   - Click **Save & Test**
3. **Import dashboard:**
   - Click **+** > **Import**
   - Enter Dashboard ID: `14513` (for Linux server metrics)
   - Click **Load**
4. **Create custom dashboard for microservices:**
   - Click **+** > **Dashboard** > **Add new panel**
   - In query section, use:
     - `http_requests_total_count`
     - or `http_request_duration_seconds_count`

---

## Troubleshooting

- Check open ports:
  ```sh
  sudo netstat -tulnp | grep 3001
  ```
- Check running node processes:
  ```sh
  ps aux | grep node
  ```
- Test metrics endpoint:
  ```sh
  curl -v http://localhost:8001/metrics
  ```
- Debug Node.js app:
  ```sh
  DEBUG=* node server.js
  ```
- View Grafana logs:
  ```sh
  sudo journalctl -u grafana-server -f
  ```
- Validate Prometheus config:
  ```sh
  prometheus --config.file=/etc/prometheus/prometheus.yml --log.level=debug
  ```
- View Prometheus logs:
  ```sh
  journalctl -u prometheus --no-pager | tail -20
  ```

---

**You have now successfully implemented infrastructure monitoring using Prometheus and Grafana on AWS!**

**Reference:** https://github.com/AshishSharmaPrivate/Lab/blob/main/Observability_for_Microservices_Applications.txt
