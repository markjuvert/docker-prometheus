# Using Grafana with Prometheus for Alerting and Monitoring

In this demo, I set up Prometheus and Grafana for alerting and monitoring.

## Environment

In this demonstration, an EC2 instance was creating in AWS Cloud and Ubuntu 20.04 LTS is installed as the OS. All the necessary packages required to run the application will be installed such as docker, node.js, prometheus, grafana, ... 
Assuming you understand the basics of AWS and how to setup a simple VM. To start with, create a user, define roles and policies that the user has permission to. It's best practice to follow Principle of least privilege, giving only the required access to the user or resource. 
There are many ways to spin-up resources in AWS such as using the console, CDK, API, IAC and better still configuration management systems. In this case, AWS console was used to create the EC2 Instance. In the EC2 page, click on launch an instance, enter the name, select an appropriate image, in this case ubuntu 20.04 LTS is selected, architecture, instance type keypair, network settings, security group, storage, and click on launch instance as shown in the screenshot. 
Update the server and install docker, and node using the script provided in my github repo of this demonstration. 

## Application
The application used in this demonstration was created by Miss Ating and she also wrote a block about the application in medium  [Todo App with Node.js](https://medium.com/@atingenkay/creating-a-todo-app-with-node-js-express-8fa51f39b16f) and her [Github](https://github.com/missating/nodejs-todo).

The application is a simple node.js application that let's you add and complete task on a single page, storing both new and completed task in a different array.

### How to run the app locally:

1. Run npm install to install all needed dependencies
2. Then start the server using node index.js
3. Navigate to your browser http://localhost:3000/ to view the app

Dockerizing and running the application.

I created a Dockerfile which from the alpine image which is light weight and install the node package, copy my application to the working directory and finally the CMD command which let's me run the application.

#### To build the image, simple use the command:
```
docker build -t devops-monitoring .
```

#### To view thelist of images:
```
Docker image list


REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
devops-monitoring   latest              532cf27aa4eb        5 minutes ago         74MB
node                10-alpine           fe6ff768f798        6 minutes ago         70.7MB
```

#### To run the image, simply use the command:
```
docker run --name to-do-app -p 80:8000 -d devops-monitoring
```
--name specifies name of the container, -p maps Port 80 of the host/server to port 8080 of the container, -d specifies the image used.
![App][images/app-new.png]


## Installing Prometheus

There are several ways we can setup prometheus, for example, we can deploy it in a container, download the script and run it or set it up as we will do in this demo.
1. Add a new user Prometheus
```
 sudo useradd --no-create-home --shell /bin/false prometheus 
 ```
 2. Create a new directory to store libraries and configuration files
  ```
  sudo mkdir /etc/prometheus
  sudo mkdir /var/lib/prometheus
  ```
3. Set the ownership of /var/lib/prometheus to the prometheus user:
```
sudo chown prometheus:prometheus /var/lib/prometheus
```
4. Cd into the tmp directory and download prometheus using wget
   ```wget https://github.com/prometheus/prometheus/releases/download/v2.40.3/prometheus-2.40.3.linux-amd64.tar.gz
   ```
5. Extract the downloaded file using tar -xvf
  ```
  tar -xvf prometheus-2.40.3.linux-amd64.tar.gz
  ```
6. CD into the new directory 
```
cd prometheus-2.40.3.linux-amd64
```
7. Move the configuration files and directories into the /etc/ folder:
   ```
    sudo mv console* /etc/prometheus
    sudo mv prometheus.yml /etc/prometheus
    ```
8. Verify to make sure prometheus is the owner of these files:
```
sudo chown -R prometheus:prometheus /etc/prometheus
```
9. Move the binaries to the /usr/local/bin/ directory:
```
sudo mv prometheus /usr/local/bin/
sudo mv promtool /usr/local/bin/
sudo chown prometheus:prometheus /usr/local/bin/prometheus
sudo chown prometheus:prometheus /usr/local/bin/promtool
```

10. Finally, we need to set up a systemd service file, letting us use systemctl to manage the starting, stopping, and enabling of Prometheus. To do this, we need to create a file, prometheus.service, in the
/etc/systemd/system/ directory:
```
 sudo vi /etc/systemd/system/prometheus.service

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
11. Save and exit then reload the daemon:
```
sudo systemctl daemon-reload
``` 
12. Start Prometheus
```
sudo systemctl start prometheus
sudo systemctl enable prometheus
```

Access prometheus in the browser using the IP address and port 9090

Noe: in some cases, if it doesn't start, we may wanna change the localhost:9090  to the Ip of the server in the prometheus.yml
```
static_configs:
      - targets: ["44.203.172.184:9090"]
```

## Installing Alert Manager
1. create a new system user, named alertmanager, to run the Alertmanager itself:
```
sudo useradd --no-create-home --shell /bin/false alertmanager
```

2. Create the /etc/alertmanager directory to store the files:
```
sudo mkdir /etc/alertmanager
```
3. Download Alertmanager into our /tmp/ folder:
```
cd /tmp/
wget https://prometheus.io/download/#:~:text=alertmanager%2D0.24.0.linux%2Damd64.tar.gz
```
5. Untar the file
```
tar -xvf alertmanager-0.24.0.linux-amd64.tar.gz
```
6. CD into the alert manager folder and do an ls to view the content
```
ls
alertmanager  alertmanager.yml  amtool  LICENSE  NOTICE
```

7. Move the two binaries into the /usr/local/bin directory and set the ownership to the alertmanager user:
```
sudo mv alertmanager /usr/local/bin/
sudo mv amtool /usr/local/bin/
sudo chown alertmanager:alertmanager /usr/local/bin/alertmanager
sudo chown alertmanager:alertmanager /usr/local/bin/amtool

sudo mv alertmanager.yml /etc/alertmanager/
sudo chown -R alertmanager:alertmanager /etc/alertmanager/
```

8. Create the alertmanager.service file in the /etc/systemd/system directory:
```
sudo vi /etc/systemd/system/alertmanager.service

[Unit]
Description=Alertmanager
Wants=network-online.target
After=network-online.target

[Service]
User=alertmanager
Group=alertmanager
Type=simple
WorkingDirectory=/etc/alertmanager/
ExecStart=/usr/local/bin/alertmanager \
    --config.file=/etc/alertmanager/alertmanager.yml

[Install]
WantedBy=multi-user.target
```
Save and exit

9. Before starting the Alertmanager, we need to update the Prometheus configuration, as well. Stop the prometheus service:
```
sudo systemctl stop prometheus
```

10. And update the /etc/prometheus/prometheus.yml file, uncommenting the Alertmanager configuration and setting the target to localhost:9093:
```
sudo vi /etc/prometheus/prometheus.yml

Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
           #- alertmanager:9093
           Uncomment and change the last line to: (- localhost:9093)
            - localhost:9093

```

11. Reload systemd and start both the prometheus and alertmanager services:
```
sudo systemctl daemon-reload
sudo systemctl start prometheus
sudo systemctl start alertmanager

Let’s also make sure alertmanager starts at boot:
sudo systemctl enable alertmanager
```
To confirm that everything is working, navigate to port 9093 of your server in your web browser.

## Installing Grafana

To install grafana, head to the download page [Grafan setup](https://grafana.com/grafana/download) and follow the instructionsm


```
sudo apt-get install -y adduser libfontconfig1
cd /tmp/
wget https://dl.grafana.com/enterprise/release/grafana-enterprise_9.2.6_amd64.deb
sudo dpkg -i grafana-enterprise_9.2.6_amd64.deb
sudo systemctl daemon-reload
sudo systemctl enable grafana-server (Enable start on reboot)
sudo systemctl start grafana-server
```

Access the web ui by using the IP address and port 3000. The default username and password is admin, make sure you change that as soon as you login.

## Add a datasource and dashboard to Grafana

A datasource in grafana is a place that receives metrics. To add a datasource, click on the add datasource, select prometheus and enter the name, prometheus URL (in this case http://localhost:9090), leave everything else as default and click on safe and test. 
A dashboard is a set of one or more panels organized and arranged into one or more rows. It is where we store all the charts and organizational patterns. In this demonstration, I added two dashboards. Click on dashboards, add new, enter the name of the dashboard and save. To BE Continued ....


## Infrastructure Monitoring
Prometheus is an open source monitoring system. It provides a modern time series database, a robust query language, several metric visualization possibilities, and a reliable alerting solution for traditional and cloud-native infrastructure.
Currently, the monitoring system only monitors itself. To get other metrics, we can use exporters to gather metrics for other servers/services

### Node Exporter
Node Exporter is a Prometheus exporter for hardware and OS metrics exposed by *NIX kernels, written in Go with pluggable metric collectors.

- System metrics for Prometheus (Linux), while WMI Exporter is System metrics for Windows servers

An exporter gathers metrics and creates the endpoint that Prometheus can scrape

The Node Exporter can monitor; CPU, Memory, Disk I/O, Disk space and filesystem metrics, Full list available on the project GitHub

Most metrics enabled bv default
• Select metrics can be added via --collector. NAME flag on
startup

#### Installing Node Exporter
To install the Node Exporter, we’ll be working the same way we did for Prometheus and Alertmanager: By downloading the needed files, adding a system user, and creating a systemd service file so we can start and stop the exporter as needed.

1. Create that system user:
```
sudo useradd --no-create-home --shell /bin/false node_exporter
```
2. cd /tmp/
   
3. Then download and extract the .tar.gz file from https://github.com/prometheus/node_exporter/releases/download/v1.5.0/node_exporter-1.5.0.linux-amd64.tar.gz

4. Extract: tar -xvf node_exporter-1.5.0.linux-amd64.tar.gz

5. Move into the new directory: cd node_exporter-1.5.0.linux-amd64

6. Here we have only one file we need to work with: the node_exporter binary. Let’s move it to the
/usr/local/bin directory:
```
sudo mv node_exporter /usr/local/bin/
```

7. And set the appropriate ownership:
```
sudo chown node_exporter:node_exporter /usr/local/bin/node_exporter
```

8. Next, we need to create the systemd file for the service:
```
sudo vi /etc/systemd/system/node_exporter.service

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

9. We can now reload systemd and start the node_exporter:
```
sudo systemctl daemon-reload
sudo systemctl start node_exporter
```
Note: Instead of 'systemctl start node _exporter', use 'systemctl enable --now node _exporter.service' to make sure the service starts up if the server is restarted.


Finally, we can add the node_exporter to our endpoints so Prometheus can scrape the data:

```
sudo vi /etc/prometheus/prometheus.yml
- job_name: 'node_exporter'
  static_configs:
  - targets: ['localhost:9100']
```
Restart Prometheus
```
sudo systemctl restart prometheus

sudo systemctl status prometheus
```
At the end of the installation, the prometheus.yml file configuration should be as below
```
cat /etc/prometheus/prometheus.yml
# my global config
global:
  scrape_interval: 15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
          # - localhost:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: "prometheus"

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
      - targets: ["localhost:9090"]
  - job_name: 'grafana'
    static_configs:
      - targets: ['localhost:3000']
  - job_name: 'alertmanager'
    static_configs:
      - targets: ['localhost:9093']
  - job_name: 'node_exporter'
    static_configs:
      - targets: ['localhost:9100']
```


Now, if we head back to prometheus and search for node_, we can see several node metrics related to the infrastructure for example disk, cpu, file, memory amongst others. 
There are also some metrics that are not enabled by default that we can enable them. For example the ones in the systemd script specified by using the -- collector and the name of the collector 
(node_memory_MemFree_bytes as shown in the screenshot) 

From the screenshot, we can see that there is a lot of memory from the start and as time goes on, the memory is being used.
To understand more about this, We can use a tool called stress to simulate how the memory is being used, for example if a heavy memory process is running and how it affects the memory of the system, stress can help us gives us a better idea

#### Installing Stress
Install stress
```
sudo apt-get install stress -y
```

stress -m 2 (the -m signifies that we are adding stress to our memory, and this will send out 2 virtual memory hogs)
Wait for a minute and head back to the prometheus dashboard and execute the last node_memory command again.
According to the screenshot, we can see that we have more free memory than when we started, and this is most likely because of excessive caches and buffers that were freed up when the stress test was ran.


## CPU METRICS

Since we already have the Node Exporter setup and ready to record some infrastructure metrics, we will apply some stress to the CPU in the sake of this demonstration for better results. 
To apply stress against the CPU, run the command:
```
stress -c 5
```
Prometheus and most of the monitoring applications retrieve CPU metrics from the /proc/stat file on the host itself. Metrics involving the CPU are prefixed with node_cpu. 
For example the CPU counter is prefixed node_cpu_seconds_total. It keeps a running total of the amount of time the CPU is in each defined state, in seconds. This metric measures the amount of time of each of the following CPU states:
• idle: Time the CPU is doing nothing 
• iowait: Time the CPU is waiting for I/O 
• irq: Time spent on fixing interrupts 
• nice: Time spent on user-level processed with a positive nice value 
• softirq: Time spent fixing interrupted 
• steal: In running a virtual machine, time spent with other VMs “stealing” CPU 
• guest: On servers that host VMs, this value will be guest and contain the amount of CPU usage of the VMs hosted 
• system: Time spent in the kernel 
• user: Time spent in userland 
From the table screenshot, it can be seen that the highest time is the idle mode which is how long the cpu has been idle. If we switch to the 
Prometheus uses a language called PromQL that lets us run queries against this metric data, performing tasks like calculating averages, means, and other mathematical functions. An example of such function is the rate or irate querying function. 
rate calculates the per-second average change of the time series in the range; irate performs the same function, but is intended for particularly fast-moving counters. 
For example
```
irate(node_cpu_seconds_total[30s]) * 100
```