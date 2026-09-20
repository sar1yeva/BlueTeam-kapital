
# ELK Stack Deployment on Ubuntu 22.04: Elasticsearch, Logstash & Kibana

<img width="667" height="325" alt="image" src="https://github.com/user-attachments/assets/f43643c3-ad49-43e6-8edf-e0f8579c3d4d" />


## Introduction

The **ELK Stack** is a popular platform for centralized log collection, processing, storage, and visualization. It consists of three main components:

* **Elasticsearch** — stores and indexes collected data.
* **Logstash** — collects and processes log events.
* **Kibana** — provides a web interface for searching, analyzing, and visualizing the data.

In this project, I deployed the ELK Stack on **Ubuntu 22.04.5 LTS** inside a virtual machine. The final setup was used to collect Ubuntu system logs, send them through Logstash to Elasticsearch, and analyze them through Kibana.

The final workflow was:

**Ubuntu Logs → Logstash → Elasticsearch → Kibana**

---

# 1. ELK Stack Overview

The first image provides an overview of the three main Elastic components: **Elasticsearch, Logstash, and Kibana**.

Elasticsearch serves as the central search and storage engine, Logstash is responsible for collecting and processing events, and Kibana provides the interface used to explore and visualize the collected data.

The purpose of this project was not only to install these components but also to connect them into a working logging pipeline.

---

# 2. Preparing Ubuntu 22.04

Before installing the ELK components, I prepared an Ubuntu virtual machine.

The environment used **Ubuntu 22.04.5 LTS (Jammy Jellyfish)**. 

Using a stable LTS release provides a reliable base for running the ELK environment.

<img width="667" height="429" alt="image" src="https://github.com/user-attachments/assets/c4c39b76-bb58-4703-bbcd-015169ded5e3" />


---

# 3. Configuring the Virtual Machine

I allocated **5 GB of RAM** and **2 virtual processors** to the Ubuntu virtual machine.

The virtual disk was configured with approximately **50 GB of storage**, which provides enough space for the operating system, ELK components, configuration files, and the log data generated during the lab.

Because Elasticsearch and Logstash can consume significant system resources, allocating sufficient memory is important for a smooth lab environment.

<img width="667" height="524" alt="image" src="https://github.com/user-attachments/assets/f2e83688-708e-45b8-b6fd-a389400e8d97" />


---

# 4. Installing Ubuntu

During the Ubuntu installation process, I selected the **English (US)** keyboard layout.

The installer then provided the available disk installation options. Since this was a dedicated virtual machine, I selected the option to install Ubuntu on the virtual disk.

The installation process then continued with the system configuration.

<img width="667" height="529" alt="image" src="https://github.com/user-attachments/assets/64763e62-1384-4f7e-a227-d2ef6be6c335" />

<img width="667" height="525" alt="image" src="https://github.com/user-attachments/assets/d68fb92f-cc0e-428e-9d91-3a831fa5839c" />



---

# 5. Selecting the System Location

Ubuntu was configured with **Baku** as the selected location.

The location is important because it determines regional settings such as the system timezone and related localization settings.

After completing the installation, Ubuntu booted into the configured user environment.

<img width="667" height="529" alt="image" src="https://github.com/user-attachments/assets/011375f7-45af-43ed-b47f-1deccf95d48a" />

<img width="667" height="428" alt="image" src="https://github.com/user-attachments/assets/b46fb2c5-94dd-4cae-aeaa-04960f5b752a" />



---

# 6. Verifying the Ubuntu Environment

After logging into the newly installed system, I verified the environment before beginning the ELK installation.

I used:

```bash
hostname -I
```

This command displays the IP address assigned to the system.

I also checked the Ubuntu release:

```bash
lsb_release -a
```

The output confirmed:

```text
Ubuntu 22.04.5 LTS
Codename: jammy
```

To check available disk space, I used:

```bash
df -h
```

The `df -h` command displays filesystem capacity and usage in a human-readable format.

Finally, I checked memory availability:

```bash
free -h
```

The system had approximately **4.8 GiB of RAM**, with around **2.5 GiB free** at the time of the check.

These checks confirmed that the Ubuntu environment was ready for the ELK deployment.

<img width="667" height="446" alt="image" src="https://github.com/user-attachments/assets/4fe0692b-1ab4-4924-904f-e6dfef1ce065" />


---

# 7. Installing Required Packages

Before adding the Elastic repository, I installed the packages required to communicate with HTTPS repositories and manage the repository signing key:

```bash
sudo apt install apt-transport-https wget gnupg -y
```

The command installs:

* `apt-transport-https` — enables APT to work with HTTPS repositories.
* `wget` — downloads files from remote servers.
* `gnupg` — handles GPG keys used for package verification.

The installation completed successfully.

<img width="667" height="367" alt="image" src="https://github.com/user-attachments/assets/501dbc79-663a-4486-b3e9-7575f3ad4a3f" />


---

# 8. Adding the Elastic Repository

Next, I added the official Elastic package repository.

First, I imported the Elastic GPG signing key:

```bash
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | \
sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg
```

The signing key allows APT to verify the authenticity of packages downloaded from the Elastic repository.

I then added the Elastic 9.x repository:

```bash
echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] \
https://artifacts.elastic.co/packages/9.x/apt stable main" | \
sudo tee /etc/apt/sources.list.d/elastic-9.x.list
```

After adding the repository, I updated the package index:

```bash
sudo apt update
```

This allowed Ubuntu to retrieve package information from the Elastic repository.

---

# 9. Installing Elasticsearch

I installed Elasticsearch using:

```bash
sudo apt install elasticsearch -y
```



The package installation also created the required Elasticsearch configuration and system-service files.

After installation, Elasticsearch was configured to run as a systemd service.

<img width="667" height="380" alt="image" src="https://github.com/user-attachments/assets/9e10f958-aa43-48ca-890f-3feed1dc7c34" />



---

# 10. Starting Elasticsearch

The installation output provided the commands required to configure Elasticsearch to start automatically.

I enabled the service:

```bash
sudo systemctl enable elasticsearch
```

Then started it:

```bash
sudo systemctl start elasticsearch
```

I verified the service status:

```bash
sudo systemctl status elasticsearch
```

The output showed:

```text
Active: active (running)
```

This confirmed that Elasticsearch was successfully running.

<img width="667" height="234" alt="image" src="https://github.com/user-attachments/assets/9ae5b686-4f1d-4683-83a7-d2b36ca8531b" />



---

# 11. Setting the Elasticsearch Password

Elastic's recent versions enable security by default, so authentication is required when accessing Elasticsearch.

I used the built-in password reset utility:

```bash
sudo /usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic
```

The tool generated a new password for the built-in `elastic` superuser.

This account is then used to authenticate administrative requests to Elasticsearch.

<img width="667" height="277" alt="image" src="https://github.com/user-attachments/assets/86d38d1e-5868-4e54-9590-cbd4676067b0" />



---

## 12. Testing the Elasticsearch API

After successfully resetting the password for the built-in `elastic` user, I tested the Elasticsearch REST API using `curl`:

```bash
curl -k -u elastic https://localhost:9200
```

The command uses:

* `-k` — allows `curl` to establish the HTTPS connection without verifying the server certificate.
* `-u elastic` — authenticates to Elasticsearch using the built-in `elastic` user.
* `https://localhost:9200` — connects to the Elasticsearch HTTPS endpoint on port `9200`.

After entering the password, Elasticsearch returned information about the node, cluster, and version.

<img width="667" height="368" alt="image" src="https://github.com/user-attachments/assets/67c5b9be-95d4-44d0-a41a-69a48b0726b2" />


---

# 13. Installing Kibana

With Elasticsearch running, I installed Kibana:

```bash
sudo apt install kibana -y
```

After installation, I reloaded systemd:

```bash
sudo systemctl daemon-reload
```

Then I enabled Kibana:

```bash
sudo systemctl enable kibana
```

And started the service:

```bash
sudo systemctl start kibana
```

Finally, I checked its status:

```bash
sudo systemctl status kibana
```

The service was shown as:

```text
Active: active (running)
```

<img width="667" height="419" alt="image" src="https://github.com/user-attachments/assets/f354b348-701e-425a-93ae-4a7523c86a85" />



---

# 14. Verifying Kibana's Listening Port

I checked the listening network sockets using:

```bash
sudo ss -tulpn
```

The output showed Kibana listening on:

```text
127.0.0.1:5601
```

Port **5601** is the default port used by Kibana's web interface.

This confirmed that the Kibana service was listening successfully.

---

# 15. Opening Kibana for the First Time

I opened Kibana in Firefox using:

```text
http://localhost:5601
```

The initial Kibana page displayed:

**Configure Elastic to get started**

Kibana requested an enrollment token to establish its connection with the Elastic environment.

This is part of the security and enrollment process introduced in modern Elastic Stack versions.

---

# 16. Generating the Kibana Enrollment Token

From the terminal, I generated a Kibana enrollment token using:

```bash
sudo /usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token -s kibana
```

The command generated a temporary enrollment token that could be entered into the Kibana setup page.

The token allows the Kibana instance to securely enroll with Elasticsearch.

<img width="667" height="50" alt="image" src="https://github.com/user-attachments/assets/fbfc2402-3c67-45b1-a294-bef07ade8f82" />

<img width="667" height="373" alt="image" src="https://github.com/user-attachments/assets/f6ca0ad4-6ebe-48a0-9279-e96688d8d38f" />


---

# 17. Kibana Verification

After entering the enrollment token, Kibana requested an additional verification code.

I generated this code using:

```bash
sudo /usr/share/kibana/bin/kibana-verification-code
```

The terminal returned a six-digit verification code.

I entered this code into the Kibana browser interface to continue the setup process.

<img width="667" height="384" alt="image" src="https://github.com/user-attachments/assets/3b697ff7-d28c-4570-b8be-eb16bed9892b" />

<img width="626" height="82" alt="image" src="https://github.com/user-attachments/assets/5522085b-ac45-462a-9535-c97c53e77a85" />



---

# 18. Logging into Kibana

After the initial configuration was completed, Kibana displayed its login page.

I authenticated using the Elasticsearch `elastic` account and the password configured earlier.

After successful authentication, Kibana opened the main Elastic interface.

---

# 19. Kibana Home Dashboard

The Kibana home page provides access to several major Elastic capabilities, including:

* Elasticsearch
* Observability
* Security
* Analytics

This confirmed that Kibana was successfully connected to the Elastic environment.

At this stage, the Elasticsearch and Kibana components were operational.

<img width="667" height="388" alt="image" src="https://github.com/user-attachments/assets/1e04d840-54d3-496a-9e34-f7003b06feb9" />

<img width="667" height="367" alt="image" src="https://github.com/user-attachments/assets/0f4535c8-002d-4d3c-b712-27733b235920" />


---

# 20. Installing Logstash

The next step was to configure Logstash as the log collection and processing component.

I installed it using:

```bash
sudo apt install logstash -y
```

I then enabled and started the service:

```bash
sudo systemctl enable logstash
sudo systemctl start logstash
```

Finally, I verified its status:

```bash
sudo systemctl status logstash
```

The service was running successfully.

<img width="667" height="400" alt="image" src="https://github.com/user-attachments/assets/ba6540f2-fe18-47f6-828b-caab09da5417" />



---

### 21. Preparing the Ubuntu Logs

For this project, I used Ubuntu’s existing system logs as the primary data source for the Logstash pipeline.

I first checked the Elasticsearch certificate directory:

```bash
sudo ls -l /etc/elasticsearch/certs/
```

I then verified that the required Ubuntu log files were available:

```bash
sudo ls -l /var/log/auth.log /var/log/syslog
```

The two main log sources used in the pipeline were:

```text
/var/log/auth.log
/var/log/syslog
```

`auth.log` contains authentication-related events, while `syslog` contains a broader range of system and service events.

After verifying the log files and certificate directory, I created the Logstash pipeline configuration:

```bash
sudo nano /etc/logstash/conf.d/ubuntu-logs.conf
```

The configuration reads events from the two Ubuntu log files, filters out Elasticsearch, Logstash, and Kibana service messages, and forwards the remaining events to Elasticsearch.

```conf
input {
  file {
    path => ["/var/log/auth.log", "/var/log/syslog"]
    start_position => "beginning"
    sincedb_path => "/dev/null"
    stat_interval => 1
  }
}

filter {
  if [message] =~ /(kibana|logstash|elasticsearch)/ or [program] in ["kibana", "logstash", "elasticsearch"] {
    drop { }
  }
}

output {
  elasticsearch {
    hosts => ["https://localhost:9200"]
    user => "elastic"
    password => "<ELASTIC_PASSWORD>"
    ssl_verification_mode => "none"
    ssl_certificate_authorities => ["/etc/elasticsearch/certs/http_ca.crt"]
    index => "ubuntu-logs-%{+YYYY.MM.dd}"
  }

  stdout {
    codec => rubydebug
  }
}
```

The `file` input monitors `/var/log/auth.log` and `/var/log/syslog`. The `start_position => "beginning"` option allows Logstash to read existing entries from the beginning of the files when processing starts.

The `sincedb_path => "/dev/null"` option is useful during testing because Logstash does not preserve the previous file-reading position. This allows the same log files to be read again when the pipeline is restarted.

The filter removes messages associated with Elasticsearch, Logstash, and Kibana. This prevents the pipeline from continuously collecting and processing its own service logs.

In the output section, Logstash connects to the local Elasticsearch instance over HTTPS and stores the processed events using a daily index pattern:

```text
ubuntu-logs-%{+YYYY.MM.dd}
```

For example:

```text
ubuntu-logs-2026.09.20
```

The `stdout` output with the `rubydebug` codec is also enabled. This displays processed events directly in the terminal and makes it easier to verify that Logstash is successfully reading and processing the Ubuntu logs.


<img width="667" height="160" alt="image" src="https://github.com/user-attachments/assets/5e3dc354-ad85-4091-9958-215bfa256604" />\


<img width="667" height="348" alt="image" src="https://github.com/user-attachments/assets/982f49a2-bdb2-42eb-8de0-bb98c23b77fe" />


---

# 22. Troubleshooting the Logstash Configuration

During the initial configuration test, Logstash returned an error related to the Elasticsearch output plugin.

The important part of the error was:

```text
Invalid setting for elasticsearch output plugin
```

The error indicated that the configured certificate path could not be found:

```text
File does not exist or cannot be opened
```

This was caused by a mismatch between the certificate path configured in the Logstash pipeline and the actual certificate location.

Instead of treating the error as a failure of the entire ELK deployment, I used the error message to identify the incorrect configuration and corrected the Logstash settings.

This troubleshooting step was important because it demonstrated how service logs and configuration validation can be used to diagnose ELK deployment problems.

<img width="667" height="301" alt="image" src="https://github.com/user-attachments/assets/db48466e-999f-436c-ba6c-d4f9edc403eb" />


---

# 23. Verifying Logstash Processing

After correcting the configuration, Logstash successfully processed Ubuntu log events.

The terminal output displayed structured events containing fields such as:

```text
@timestamp
host
message
log.file.path
event.original
```

For example, the log source was identified as:

```text
/var/log/syslog
```

The `event.original` field contained the original log message, while fields such as `@timestamp` and `host.name` provided additional metadata.

This confirmed that Logstash was successfully reading and processing Ubuntu system logs.

<img width="667" height="207" alt="image" src="https://github.com/user-attachments/assets/bedb1d1c-bfe2-457e-a72e-90d708d4f677" />


---

# 24. Verifying Logstash and Elasticsearch

I checked the Logstash service again:

```bash
sudo systemctl status logstash
```

The service was shown as active and running.

I then queried Elasticsearch for its available indices:

```bash
curl --cacert /etc/elasticsearch/certs/http_ca.crt \
-u elastic https://localhost:9200/_cat/indices?v
```

The first attempt demonstrated that authentication was required, resulting in an authentication error.

After providing valid credentials, Elasticsearch returned the available indices.

Among them was the Ubuntu log index:

```text
ubuntu-logs-2026.09.20
```

This was a strong confirmation that the complete data path was working:

**Ubuntu logs → Logstash → Elasticsearch**

<img width="1435" height="518" alt="image" src="https://github.com/user-attachments/assets/2469d564-1426-4a73-b047-ba0709103613" />


---

# 25. Exploring Analytics in Kibana

Once the logs were indexed in Elasticsearch, I opened the **Analytics** section in Kibana.

Kibana provides two particularly useful areas for log analysis:

* **Discover** — interactive exploration and filtering of individual events.
* **Dashboard** — visualization and aggregation of data.

At this point, the raw Ubuntu logs were available for analysis through Kibana.

<img width="667" height="377" alt="image" src="https://github.com/user-attachments/assets/4ec0a8d6-fafd-4b0c-ba5f-b95519ac980e" />


---

# 26. Exploring Elastic Security

I also opened the **Security** section of Kibana.

Elastic Security provides functionality for security monitoring, detection, dashboards, rules, and investigations.

Although this project primarily focused on establishing the ELK pipeline, this section demonstrates how the same Elastic environment can later be extended into a broader security monitoring and SIEM-style environment.

<img width="667" height="382" alt="image" src="https://github.com/user-attachments/assets/8df3c232-cf5b-4b61-bb8a-e9c13b3e7cc2" />


---

# 27. Creating a Kibana Data View

To make the Elasticsearch logs available in Kibana's Discover interface, I created a **Data View**.

I navigated to:

**Stack Management → Data Views**

The Data View allows Kibana to identify which Elasticsearch indices should be queried.

I selected:

```text
ubuntu-logs-*
```

This pattern matches indices beginning with:

```text
ubuntu-logs-
```

For example:

```text
ubuntu-logs-2026.09.20
```

I also selected:

```text
@timestamp
```

as the timestamp field.

This allows Kibana to organize and filter events according to their event time.

<img width="667" height="371" alt="image" src="https://github.com/user-attachments/assets/c51d9622-80d3-4ae8-85ac-62cbd1c0063e" />


---

# 28. Saving the Data View

The Data View configuration showed that the pattern successfully matched the Elasticsearch source.

The interface displayed:

```text
Your index pattern matches 1 source.
```

I then saved the Data View to Kibana.

This made the Ubuntu logs available for exploration through Kibana Discover.

<img width="667" height="344" alt="image" src="https://github.com/user-attachments/assets/9b3e6e18-b20e-4c8e-8ad0-e22b84a59a7f" />


---

# 29. Exploring Ubuntu Logs with Discover

I opened Kibana's **Discover** interface and selected the newly created:

```text
ubuntu-logs-*
```

Data View.

Kibana displayed the collected Ubuntu events along with their timestamps and fields.

The available fields included:

* `@timestamp`
* `@version`
* `event.original`
* `host.name`
* `log.file.path`
* `message`

The Discover interface also displayed the number of matching documents and provided a timeline of events.

This demonstrates that the logs successfully traveled through the complete pipeline and became searchable in Kibana.

<img width="667" height="328" alt="image" src="https://github.com/user-attachments/assets/f08a4401-cb34-4cc3-af7c-f46deaf54efc" />


---

# 30. Creating a Visualization

Finally, I created a visualization using the indexed Ubuntu log data.

The visualization displayed the **count of records over time**, providing a simple graphical representation of the collected events.

The timestamp field was used as the horizontal axis, while the number of records was used as the vertical metric.

This demonstrates the final stage of the pipeline:

**Log Collection → Processing → Indexing → Search → Visualization**

<img width="667" height="353" alt="image" src="https://github.com/user-attachments/assets/fa0d2710-f08c-453c-8835-ab83bbf54cb6" />


---

# Conclusion

In this project, I deployed and integrated the three main components of the **ELK Stack** on Ubuntu 22.04:

**Elasticsearch 9.5.4** was used to store and index the collected events.

**Logstash 9.5.4** was configured to collect Ubuntu system logs, process them, and forward them to Elasticsearch.

**Kibana 9.5.4** was then connected to the Elasticsearch data and used to create a Data View, explore individual log events, and build a basic visualization.

The final working architecture was:

```text
Ubuntu System Logs
        |
        v
   +-----------+
   | Logstash  |
   +-----------+
        |
        v
+------------------+
| Elasticsearch    |
| ubuntu-logs-*    |
+------------------+
        |
        v
   +-----------+
   |  Kibana   |
   +-----------+
        |
        v
Discover / Visualizations
```

The project also included real troubleshooting during the Logstash configuration stage. An Elasticsearch output configuration initially referenced an incorrect certificate path, which caused Logstash validation to fail. After correcting the configuration, Logstash successfully processed the Ubuntu logs and Elasticsearch indexed them.

The final Kibana **Discover** view and visualization confirmed that the complete logging pipeline was operational.

