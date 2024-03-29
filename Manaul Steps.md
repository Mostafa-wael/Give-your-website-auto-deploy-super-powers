# Configure the RDS
1. From AWS side:
   1. Assign a DB name, master name, and password.
   2. Create a new RDS instance.
   3. Make it publicly accessible.
   4. Add the default port `5432` to the security group inbound and outbound.
2. From CircleCI side:
   1. Add the default port `5432` to the environment variables.
   2. Add the DB name to the environment variables.
   3. Add the password to the environment variables.


# Configure Prometheus
   1. Create a new instance in EC2.
   2. Configure a Security Group. Think of it like firewall rules. We will need port 9090 for Prometheus, port 9100 for Prometheus Node Exporter and finally, port 9093 for the Alertmanager. For this example, we are going to use a single Security Group for all the AWS EC2 instances to keep it simple.
   3. Install Prometheus on AWS EC2, [check this](https://codewizardly.com/prometheus-on-aws-ec2-part1/) or follow the following steps:
      1. Connect to the machine using SSH.
      2. It is recommended to create a different user than root to run specific services. This will help to isolate Prometheus and add protection to the system. I really like this stackexchange answer, it could give you a better explanation of why we should avoid the usage of the root user for everything. Also we need to create a directory to host Prometheus configuration and another one to host its data.
      ``` bash 
         sudo useradd --no-create-home prometheus
         sudo mkdir /etc/prometheus
         sudo mkdir /var/lib/prometheus
      ```
      3. Now we need to install Prometheus.
      ``` bash
      wget https://github.com/prometheus/prometheus/releases/download/v2.19.0/prometheus-2.19.0.linux-amd64.tar.gz
      tar xvfz prometheus-2.19.0.linux-amd64.tar.gz

      sudo cp prometheus-2.19.0.linux-amd64/prometheus /usr/local/bin
      sudo cp prometheus-2.19.0.linux-amd64/promtool /usr/local/bin/
      sudo cp -r prometheus-2.19.0.linux-amd64/consoles /etc/prometheus
      sudo cp -r prometheus-2.19.0.linux-amd64/console_libraries /etc/prometheus

      sudo cp prometheus-2.19.0.linux-amd64/promtool /usr/local/bin/
      rm -rf prometheus-2.19.0.linux-amd64.tar.gz prometheus-2.19.0.linux-amd64
      ```
      4. Initially and as a proof of concept we can configure Prometheus to monitor itself. All what we need to do is create or replace the content of /etc/prometheus/prometheus.yml.
      ``` yaml
      global:
         scrape_interval: 15s
         external_labels:
            monitor: 'prometheus'

      scrape_configs:
         - job_name: 'prometheus'
            static_configs:
               - targets: ['localhost:9090']
      ```
      5. We might want Prometheus to be available as a service. Every time we reboot the system Prometheus will start with the OS. Create /etc/systemd/system/prometheus.service and add to it the following content:
      ``` bash
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
      6. Let’s change the permissions of the directories, files and binaries we just added to our system.
      ``` bash
      sudo chown prometheus:prometheus /etc/prometheus
      sudo chown prometheus:prometheus /usr/local/bin/prometheus
      sudo chown prometheus:prometheus /usr/local/bin/promtool
      sudo chown -R prometheus:prometheus /etc/prometheus/consoles
      sudo chown -R prometheus:prometheus /etc/prometheus/console_libraries
      sudo chown -R prometheus:prometheus /var/lib/prometheus
      ```
      7. Now we need to configure systemd.
      ``` bash
      sudo systemctl daemon-reload
      sudo systemctl enable prometheus
      ```

   4. Connect your backend server to Prometheus:
      1. Edit `/etc/prometheus/prometheus.yml` file.
      2. Add the following: 
      ``` yaml
      global:
        scrape_interval: 15s
        external_labels:
            monitor: 'prometheus'
      scrape_configs:
      - job_name: 'node_exporter'
            static_configs:
            - targets: ['ec2-13-58-127-241.us-east-2.compute.amazonaws.com:9100']
      ``` 
      3. Install Alert manger on your prometheus server. Check this [link](https://codewizardly.com/prometheus-on-aws-ec2-part4/) or follow the following:
         1. Install Alertmanager.
            ``` bash 
            wget https://github.com/prometheus/alertmanager/releases/download/v0.21.0/alertmanager-0.21.0.linux-amd64.tar.gz
            tar xvfz alertmanager-0.21.0.linux-amd64.tar.gz

            sudo cp alertmanager-0.21.0.linux-amd64/alertmanager /usr/local/bin
            sudo cp alertmanager-0.21.0.linux-amd64/amtool /usr/local/bin/
            sudo mkdir /var/lib/alertmanager

            rm -rf alertmanager*
            ```
         2. Configure Alertmanager as a service. `/etc/systemd/system/alertmanager.service`
            ```
            [Unit]
            Description=Alert Manager
            Wants=network-online.target
            After=network-online.target

            [Service]
            Type=simple
            User=prometheus
            Group=prometheus
            ExecStart=/usr/local/bin/alertmanager \
            --config.file=/etc/prometheus/alertmanager.yml \
            --storage.path=/var/lib/alertmanager

            Restart=always

            [Install]
            WantedBy=multi-user.target
            ```
         3. Update Prometheus configuration file. Edit `/etc/prometheus/prometheus.yml`
            ``` yaml
            global:
               scrape_interval: 1s
               evaluation_interval: 1s

            rule_files:
            - /etc/prometheus/rules.yml
            alerting:
               alertmanagers:
               - static_configs:
                  - targets:
                     - localhost:9093
            ``` 
         1. Add Alertmanager’s configuration `/etc/prometheus/alertmanager.yml`.
            ``` yaml
            route:
               group_by: [Alertname]
               receiver: email-me

            receivers:
            -  name: email-me
               email_configs:
               - to: EMAIL_YO_WANT_TO_SEND_EMAILS_TO
                  from: YOUR_EMAIL_ADDRESS
                  smarthost: smtp.gmail.com:587
                  auth_username: YOUR_EMAIL_ADDRESS
                  auth_identity: YOUR_EMAIL_ADDRESS
                  auth_password: YOUR_EMAIL_PASSWORD
            ```
         2. Create a Rule: This is just a simple alert rule. In a nutshell it alerts when an instance has been down for more than 3 minutes. Add this file at `/etc/prometheus/rules.yml`.
         ``` yaml
         groups:
         -  name: Down
            rules:
            - alert: InstanceDown
               expr: up == 0
               for: 3m
               labels:
                  severity: 'critical'
               annotations:
                  summary: "Instance  is down"
                  description: " of job  has been down for more than 3 minutes."
         ```
   1. Configure Systemd
      ``` bash
      sudo systemctl daemon-reload
      sudo systemctl enable alertmanager
      sudo systemctl start alertmanager
      ```
   2. Restart Prometheus service: `sudo systemctl restart prometheus`