
# DVWA Logs to ELK — Walkthrough

## 1. Verify the ELK Environment

Before deploying DVWA, I first verified that the existing ELK components were running correctly. Since Elasticsearch will store the processed DVWA events and Kibana will be used for log analysis, both services must be operational before configuring the log pipeline.

I checked the Elasticsearch service with:

```bash
sudo systemctl status elasticsearch
```

The `systemctl status` command displays the current state of the Elasticsearch service, including whether the service is running and whether it has encountered any startup errors.

The service was confirmed to be active and running. This verified that Elasticsearch was ready to receive and index events from Logstash.

I then verified Kibana:

```bash
sudo systemctl status kibana
```

Kibana was also confirmed to be running. Kibana will later provide the interface for searching, filtering, and analyzing the DVWA events stored in Elasticsearch.

<img width="900" height="448" alt="image" src="https://github.com/user-attachments/assets/f0b55ba4-d0f4-4fb1-98da-5e6f5b2969d6" />


<img width="900" height="470" alt="image" src="https://github.com/user-attachments/assets/a38820fe-094d-4797-abbe-46f67e76777e" />


---

## 2. Install the DVWA Dependencies

The required packages for DVWA were installed using:

```bash
sudo apt install -y apache2 mariadb-server mariadb-client php php-mysqli php-gd libapache2-mod-php git
```

These packages provide the components required to run DVWA:

- `apache2` — web server responsible for serving the DVWA application.
- `mariadb-server` — database server used by DVWA.
- `mariadb-client` — command-line client for managing the MariaDB database.
- `php` — runtime required by the PHP-based DVWA application.
- `php-mysqli` — PHP extension required for MySQL/MariaDB database communication.
- `php-gd` — PHP graphics extension required by some DVWA functionality.
- `libapache2-mod-php` — allows Apache to execute PHP files.
- `git` — used to download the DVWA source code from its repository.

After installing the dependencies, Apache and MariaDB were enabled and started:

```bash
sudo systemctl enable --now apache2
sudo systemctl enable --now mariadb
```

The `enable` option configures both services to start automatically during system boot, while `--now` starts them immediately without requiring a separate `systemctl start` command.

At this stage, the web server and database services required by DVWA were running.

<img width="900" height="224" alt="image" src="https://github.com/user-attachments/assets/747764a5-ab79-4a8b-ab99-fbf6ea629cd2" />


<img width="900" height="121" alt="image" src="https://github.com/user-attachments/assets/56467a67-7a86-4a33-8f5f-33704ed44b77" />


---

## 3. Download and Deploy DVWA

I moved to Apache's default web directory:

```bash
cd /var/www/html
```

This directory is the default document root used by Apache on Ubuntu.

I then cloned the DVWA repository:

```bash
sudo git clone https://github.com/digininja/DVWA.git
```

This downloaded the DVWA source code into:

```text
/var/www/html/DVWA
```

Because the application was placed inside Apache's document root, Apache can serve DVWA through the local web server.

The application can therefore be accessed using the `/DVWA` path.

<img width="900" height="211" alt="image" src="https://github.com/user-attachments/assets/b74f9d9a-1875-4922-b155-13485f5edeac" />


---

## 4. Create the DVWA Configuration File

I entered the DVWA directory:

```bash
cd /var/www/html/DVWA
```

DVWA provides a default configuration template. I copied this template to create the active configuration file:

```bash
sudo cp config/config.inc.php.dist config/config.inc.php
```

The new `config.inc.php` file contains the database connection settings used by the application.

I opened the configuration file with:

```bash
sudo nano config/config.inc.php
```

The database parameters were configured as follows:

```php
$_DVWA[ 'db_server' ]   = '127.0.0.1';
$_DVWA[ 'db_database' ] = 'dvwa';
$_DVWA[ 'db_user' ]     = 'dvwa';
$_DVWA[ 'db_password' ] = 'dvwa';
$_DVWA[ 'db_port' ]     = '3306';
```

These parameters configure DVWA to connect to the local MariaDB instance through TCP port `3306`.

The application uses the dedicated `dvwa` database and database account instead of connecting directly with the MariaDB root account.\

<img width="900" height="42" alt="image" src="https://github.com/user-attachments/assets/3441ec9b-ccad-4be4-9155-55173847cbde" />


<img width="900" height="41" alt="image" src="https://github.com/user-attachments/assets/d797a21b-03c9-44b0-b74b-9f93f4c8a3b3" />


<img width="900" height="323" alt="image" src="https://github.com/user-attachments/assets/82b3dd86-12ba-4a11-b43b-0846a3f60b43" />


<img width="900" height="369" alt="image" src="https://github.com/user-attachments/assets/fd7c7790-37cd-4fb8-b08e-308976d199e1" />


---

## 5. Create the DVWA Database

The MariaDB command-line interface was opened with:

```bash
sudo mysql
```

Inside MariaDB, I created the database required by DVWA:

```sql
CREATE DATABASE dvwa;
```

I then created a dedicated database user:

```sql
CREATE USER 'dvwa'@'localhost' IDENTIFIED BY 'dvwa';
```

The required permissions were granted:

```sql
GRANT ALL PRIVILEGES ON dvwa.* TO 'dvwa'@'localhost';
```

The privilege table was then reloaded:

```sql
FLUSH PRIVILEGES;
```

Finally, I exited MariaDB:

```sql
EXIT;
```

This completed the database-side configuration required by DVWA.

The resulting architecture at this point was:

```text
DVWA
  ↓
PHP / Apache
  ↓
MariaDB
```

<img width="900" height="551" alt="image" src="https://github.com/user-attachments/assets/ff8d0fe2-a2ad-4af8-ae79-2fa7e3eed803" />

---

## 6. Configure Apache for DVWA

The ownership of the DVWA directory was changed to the Apache service account:

```bash
sudo chown -R www-data:www-data /var/www/html/DVWA
```

The `www-data` account is used by Apache on Ubuntu. Changing the ownership ensures that the web server can access the DVWA application files correctly.

I then enabled Apache's rewrite module:

```bash
sudo a2enmod rewrite
```

The rewrite module provides URL rewriting functionality that may be required by the application.

After enabling the module, Apache was restarted:

```bash
sudo systemctl restart apache2
```

At this point, Apache was configured to serve the DVWA application.

<img width="900" height="125" alt="image" src="https://github.com/user-attachments/assets/1b35b204-0c99-41fb-acf9-5516622d2342" />


---

## 7. Initialize the DVWA Application

I opened the DVWA setup page in the browser:

```text
http://localhost/DVWA/setup.php
```

The setup page provides an environment check for DVWA. It verifies the availability of required PHP functionality, database connectivity, permissions, and other configuration requirements.

After confirming that the required components were available, the **Create / Reset Database** function was used.

This initializes the DVWA database structure and creates the tables required by the application.

After the database was initialized, I opened the login page:

```text
http://localhost/DVWA/login.php
```

Successful authentication provided access to the main DVWA interface:

```text
http://localhost/DVWA/index.php
```

This confirmed that the DVWA application was successfully deployed and communicating with the MariaDB backend.

<img width="900" height="728" alt="image" src="https://github.com/user-attachments/assets/20ac3941-f0a4-4acf-a608-9c4d09dffb7c" />

<img width="870" height="693" alt="image" src="https://github.com/user-attachments/assets/79a78190-d61c-4190-8730-5c9ed51de325" />

<img width="750" height="475" alt="image" src="https://github.com/user-attachments/assets/b8b19c41-9e5c-45d3-adb2-8db8237b40e0" />

<img width="900" height="707" alt="image" src="https://github.com/user-attachments/assets/2fc1b6c8-52ce-48e0-92ab-ce4f47326403" />

<img width="897" height="516" alt="image" src="https://github.com/user-attachments/assets/6f97fea8-3903-44d0-acec-195cd49116d3" />


---

# 8. Generate DVWA Security-Test Traffic

The next step was to generate application traffic that could later be collected and analyzed by ELK.

I accessed the SQL Injection module:

```text
http://localhost/DVWA/vulnerabilities/sqli/
```

DVWA contains intentionally vulnerable modules designed for controlled security testing. Interacting with these modules generates HTTP requests that are processed by Apache.

Apache automatically records these requests in:

```text
/var/log/apache2/access.log
```

To monitor the log in real time, I used:

```bash
sudo tail -f /var/log/apache2/access.log
```

The `tail -f` command displays the latest entries in the file and continues monitoring it as new events are written.

When DVWA requests were generated, corresponding Apache access-log entries appeared in the terminal.

These entries contain information such as:

- source/client IP address
- timestamp
- HTTP request method
- requested URI
- HTTP protocol version
- HTTP response status
- response size
- referrer
- User-Agent

This step verified the first part of the logging pipeline:

```text
DVWA
  ↓
Apache
  ↓
access.log
```

<img width="900" height="276" alt="image" src="https://github.com/user-attachments/assets/01f47065-c13d-4bb4-8883-677c5d5021db" />


---

# 9. Create the Logstash Pipeline

After confirming that Apache was generating logs, I configured Logstash to collect them.

I created a dedicated Logstash configuration file:

```bash
sudo nano /etc/logstash/conf.d/dvwa-apache.conf
```

The Logstash pipeline was configured to monitor:

```text
/var/log/apache2/access.log
```

The file input allows Logstash to continuously read new Apache log entries.

The Apache logs were then parsed using the built-in `COMBINEDAPACHELOG` Grok pattern:

```ruby
grok {
  match => {
    "message" => "%{COMBINEDAPACHELOG}"
  }
}
```

The purpose of the Grok filter is to transform the raw Apache log line into structured fields.

For example, instead of keeping the entire log entry as one text string, fields such as the client address, HTTP method, request URI, response code, and User-Agent can be extracted separately.

A date filter was also configured to process the Apache timestamp:

```ruby
date {
  match => [ "timestamp", "dd/MMM/yyyy:HH:mm:ss Z" ]
  target => "@timestamp"
}
```

This converts the timestamp from the Apache log format into Logstash's standard `@timestamp` field.

Finally, additional metadata was added to identify the events as DVWA Apache logs:

```ruby
mutate {
  add_field => {
    "application" => "DVWA"
    "log_type" => "apache_access"
  }
}
```

These fields make it easier to identify and filter DVWA events later in Kibana.

The processing stage can therefore be represented as:

```text
Apache Raw Log
      ↓
Logstash File Input
      ↓
Grok Parsing
      ↓
Timestamp Normalization
      ↓
DVWA Metadata
      ↓
Structured Event
```

<img width="900" height="58" alt="image" src="https://github.com/user-attachments/assets/edab1704-fcdd-42c5-92a5-23087a104f75" />


Full code:

```ruby
input {
  file {
    path => "/var/log/apache2/access.log"
    start_position => "beginning"
    sincedb_path => "/dev/null"
  }
}

filter {
  grok {
    match => {
      "message" => "%{COMBINEDAPACHELOG}"
    }
  }

  date {
    match => [ "timestamp", "dd/MMM/yyyy:HH:mm:ss Z" ]
    target => "@timestamp"
  }

  mutate {
    add_field => {
      "application" => "DVWA"
      "log_type" => "apache_access"
    }
  }
}

output {
  elasticsearch {
    hosts => ["https://localhost:9200"]
    index => "dvwa-access-%{+YYYY.MM.dd}"
    ssl_certificate_authorities => ["/etc/elasticsearch/certs/http_ca.crt"]
    user => "elastic"
    password => "YOUR_ELASTIC_PASSWORD"
  }

  stdout {
    codec => rubydebug
  }
}
```

<img width="900" height="501" alt="image" src="https://github.com/user-attachments/assets/bbd22e7d-2450-45aa-8785-701ec94b649f" />


---

# 10. Validate the Logstash Configuration

Before starting Logstash, I validated the configuration file to ensure that there were no syntax errors.

The Logstash configuration was tested using:

```bash
sudo -u logstash /usr/share/logstash/bin/logstash \
--path.settings /etc/logstash \
-f /etc/logstash/conf.d/dvwa-apache.conf \
--config.test_and_exit
```

The `--config.test_and_exit` option instructs Logstash to validate the configuration and exit without starting the full pipeline.

The output indicated:

```text
Configuration OK
```

This confirmed that the Logstash configuration syntax was valid.

Before running the service, the Logstash log directory permissions were also adjusted where required:

```bash
sudo chown -R logstash:logstash /var/log/logstash
```

This ensures that the Logstash service account has the required access to its logging directory.

I then restarted Logstash:

```bash
sudo systemctl restart logstash
```

and verified the service:

```bash
sudo systemctl status logstash
```

The service was expected to show:

```text
Active: active (running)
```

This confirmed that the DVWA Logstash pipeline was running.

<img width="900" height="325" alt="image" src="https://github.com/user-attachments/assets/64617266-9936-4599-8f5f-a70efb27f54d" />


<img width="900" height="271" alt="image" src="https://github.com/user-attachments/assets/3239d367-1438-4c5e-a79d-de83a1fe8489" />


---

# 11. Generate Additional Events

With Logstash running, I generated additional requests against DVWA.

At the same time, Apache continued writing the requests to:

```text
/var/log/apache2/access.log
```

The Logstash file input detected the new entries and processed them according to the configured pipeline.

The complete collection path was now:

```text
DVWA
   ↓
Apache
   ↓
access.log
   ↓
Logstash
   ↓
Grok + Date + Mutate
```

This verified that the log collection stage was operational.

<img width="900" height="378" alt="image" src="https://github.com/user-attachments/assets/b844d769-f92a-4771-a3b7-f4f7ba37aad1" />


---

# 12. Verify Elasticsearch Ingestion

After Logstash processed the Apache events, I verified that Elasticsearch had received the data.

The Elasticsearch index list was checked using:

```bash
sudo curl --cacert /etc/elasticsearch/certs/http_ca.crt \
-u elastic \
"https://localhost:9200/_cat/indices?v"
```

The `/_cat/indices` endpoint provides a human-readable list of Elasticsearch indices and their document statistics.

The output contained a DVWA-specific index following the naming pattern:

```text
dvwa-access-YYYY.MM.DD
```

For example:

```text
dvwa-access-2026.10.01
```

The presence of this index and its documents confirms that Logstash successfully forwarded the processed Apache events to Elasticsearch.

The pipeline had therefore reached the Elasticsearch stage:

```text
DVWA
   ↓
Apache access.log
   ↓
Logstash
   ↓
Elasticsearch
   ↓
dvwa-access-YYYY.MM.DD
```

<img width="900" height="206" alt="image" src="https://github.com/user-attachments/assets/7e88c6a3-231d-43c9-9ebd-dc9965be2e22" />


---

# 13. Open the DVWA Data in Kibana

After confirming that Elasticsearch contained the events, I opened Kibana and navigated to **Discover**.

The `DVWA` data view was selected to access the events stored in the `dvwa-access-*` indices.

The events were displayed as structured documents rather than only raw log lines.

Fields such as:

```text
@timestamp
application
http.request.method
http.response.status_code
http.version
log_type
message
source.address
```

can now be used to search and investigate the collected activity.

For example, `http.response.status_code` can be used to identify HTTP errors, while `http.request.method` can be used to distinguish GET and POST requests.

The `@timestamp` field also allows events to be analyzed chronologically through the Kibana timeline.

This confirms that the final visualization and analysis layer is working.

<img width="900" height="533" alt="image" src="https://github.com/user-attachments/assets/029d4761-1fcc-40b1-9e78-945b871670f2" />


<img width="900" height="379" alt="image" src="https://github.com/user-attachments/assets/0360db1d-3200-4b26-86dc-c618ca4b3411" />


---

# 14. Result of the Complete Pipeline

At this stage, the complete DVWA logging architecture was operational:

```text
                 DVWA
                   │
                   ▼
            Apache HTTP Server
                   │
                   ▼
       /var/log/apache2/access.log
                   │
                   ▼
               Logstash
                   │
          ┌────────┼─────────┐
          │        │         │
        Grok      Date     Mutate
          │        │         │
          └────────┼─────────┘
                   │
                   ▼
             Elasticsearch
                   │
                   ▼
        dvwa-access-YYYY.MM.DD
                   │
                   ▼
                Kibana
                   │
                   ▼
          Security Analysis
```

The pipeline successfully converts DVWA web activity into structured security-relevant events that can be searched and analyzed centrally.

---

