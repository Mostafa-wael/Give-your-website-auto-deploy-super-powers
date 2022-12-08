1. Configure the RDS
   1. From AWS side:
      1. Assign a DB name, master name, and password.
      2. Create a new RDS instance.
      3. Make it publicly accessible.
      4. Add the default port `5432` to the security group inbound and outbound.
   2. From CircleCI side:
      1. Add the default port `5432` to the environment variables.
      2. Add the DB name to the environment variables.
      3. Add the password to the environment variables.
2. Configure Prometheus
   1. Create a new instance in EC2.
   2. Install Prometheus on AWS EC2, [check this](https://codewizardly.com/prometheus-on-aws-ec2-part1/).
   3. Prometheus Service Discovery on AWS EC2, [check this](https://codewizardly.com/prometheus-on-aws-ec2-part3/).
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
      3. Restart Prometheus service: `sudo systemctl restart prometheus`