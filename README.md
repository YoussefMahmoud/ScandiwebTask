# ScandiwebTask
This repository provides comprehensive documentation and configuration files for how i set up Magento 2 Open Source on a server using nginx, PHP-FPM, MySQL, Varnish, Redis, and Elasticsearch. It serves as a detailed guide to say how i did it.

### Technologies Used:

nginx: A high-performance web server and reverse proxy.
PHP-FPM: PHP FastCGI Process Manager for handling PHP requests.
MySQL: A relational database management system used to store Magento 2 data.
Varnish: An HTTP accelerator and reverse proxy for caching content.
Redis: An in-memory data structure store used for caching and session storage in Magento 2.
Elasticsearch: A search engine used for advanced search capabilities in Magento 2.
AWS: an online platform that provides scalable and cost-effective cloud computing solutions.


## Step 1: Create an AWS instance:
Platform Choice
I chose to complete this task on AWS due to its robust infrastructure and scalable services, which are ideal for hosting production-grade applications like Magento 2. AWS provides a wide range of services that integrate seamlessly, offering high availability, scalability, and security.

Choice of Ubuntu Instance
For the server operating system, I selected Ubuntu due to its popularity, strong community support, and compatibility with a wide range of software packages and tools required for Magento 2. Ubuntu's LTS (Long Term Support) releases provide stability and security updates, making it suitable for long-term server deployments.

Selection of t2.medium Instance
I opted for a t2.medium instance type because it strikes a balance between performance and cost-effectiveness for running Elasticsearch and OpenSearch. The t2.large instance offers moderate CPU performance and a sufficient amount of memory, which is crucial for handling the indexing and search capabilities required by Elasticsearch and OpenSearch in a Magento 2 environment.


## Step 2: Server Setup Commands

Installing nginx:
```bash
sudo apt update
sudo apt install nginx
```
PHP 8.1-FPM:
```bash
sudo apt install software-properties-common
sudo add-apt-repository ppa:ondrej/php
sudo apt update
sudo apt install php8.1-fpm php8.1-mysql php8.1-xml php8.1-mbstring php8.1-curl php8.1-intl php8.1-gd php8.1-zip php8.1-bcmath php8.1-redis
```
Mysql:
```bash
sudo apt install mysql-server
sudo mysql_secure_installation
```
Varnish:
```bash
sudo apt install varnish
```
Redis:
```bash
sudo apt install redis-server
```
OpenSearch:
```bash
wget https://download.opensuse.org/repositories/devel:/languages:/go/Ubuntu_20.04/amd64/opensearch_1.0.0_amd64.deb
sudo dpkg -i opensearch_1.0.0_amd64.deb
sudo apt-get update
sudo apt-get install opensearch
```
ElasticSearch:
```bash
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
sudo apt-get install apt-transport-https
echo "deb https://artifacts.elastic.co/packages/7.x/apt stable main" | sudo tee /etc/apt/sources.list.d/elastic-7.x.list
sudo apt-get update
sudo apt-get install elasticsearch
```
Composer:
```bash
curl -sS https://getcomposer.org/installer | php
mv composer.phar  /usr/local/bin/composer
chmod +x   /usr/local/bin/composer
```
Magento2:
```bash
mkdir ~/magento2
cd ~/magento2
composer create-project --repository-url=https://repo.magento.com/ magento/project-community-edition .
```
Magento file premessions:
```bash
sudo chown -R www-data:www-data .
sudo chmod -R 755 .
```
Magento Setup:
```bash
php bin/magento setup:install \
    --base-url=ip \
    --db-host=localhost \
    --db-name=magento \
    --db-user=magentouser \
    --db-password=y${HWH1(& \
    --admin-firstname=Admin \
    --admin-lastname=User \
    --admin-email=admin@example.com \
    --admin-user=admin \
    --admin-password=admin123 \
    --language=en_US \
    --currency=USD \
    --timezone=UTC \
    --use-rewrites=1
```
## Step 3: Configurations
### nginx
Access conf file for nginx
```bash
sudo nano /etc/nginx/sites-available/magento2
```
Configured nginx to listen to port 80 for http
listen to port 443 for ssl, added the self signed ssl certificate
added the path to php8.1-fpm and port 8080 for it.
```bash
server {
    listen 80;
    server_name ip;

    # Redirect HTTP to HTTPS
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name ip;

    ssl_certificate /etc/ssl/mycerts/mycert.crt;
    ssl_certificate_key /etc/ssl/mycerts/mycert.key;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;
    ssl_ciphers "ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA384";

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Websockets
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.1-fpm.sock;
}

}

# Varnish server configuration
server {
    listen 8080;
    server_name ip;

    location / {
        proxy_pass http://127.0.0.1:8081;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```
Restart nginx
```bash
sudo systemctl restart nginx
```
### PHP
Access PHP configuration file
```bash
sudo nano /etc/php/8.1/fpm/php.ini
```
Confiure settings
```bash
memory_limit = 2G
max_execution_time = 1800
zlib.output_compression = On
```
Restart php
```bash
sudo systemctl restart php8.1-fpm
```
### Varnish
Acess Varnish VCL file
```bash
sudo nano /etc/varnish/default.vcl
```
```bash
backend default {
    .host = "127.0.0.1";
    .port = "8081";
}
```
restart Varnish

```bash
sudo systemctl daemon-reload
sudo systemctl restart varnish
```
### Redis
Access `env.php` in Magento2 directory to configure Redis
```bash
sudo nano /var/www/html/magento2/app/etc/env.php
```
Added the following configuration
```bash
'session' => [
    'save' => 'redis',
    'redis' => [
        'host' => '127.0.0.1',
        'port' => '6379',
        'password' => '',
        'timeout' => '2.5',
        'persistent_identifier' => '',
        'database' => '2',
        'compression_threshold' => '2048',
        'compression_library' => 'gzip',
        'log_level' => '1',
        'max_concurrency' => '6',
        'break_after_frontend' => '5',
        'break_after_adminhtml' => '30',
        'first_lifetime' => '600',
        'bot_first_lifetime' => '60',
        'bot_lifetime' => '7200',
        'disable_locking' => '0',
        'min_lifetime' => '60',
        'max_lifetime' => '2592000'
    ],
],
```
Restart redis
```bash
sudo systemctl restart redis-server
```
### OpenSearch
Acess opensearch.yml config file
```bash
sudo nano /etc/opensearch/opensearch.yml
```
Added the following settings
```bash
network.host: 127.0.0.1
http.port: 9200
discovery.type: single-node
```
Start OpenSearch
```bash
sudo systemctl start opensearch
sudo systemctl enable opensearch
```
Configure Magento2 to use OpenSearch
```bash
php bin/magento config:set catalog/search/engine opensearch
php bin/magento config:set catalog/search/opensearch_server_hostname 127.0.0.1
php bin/magento config:set catalog/search/opensearch_server_port 9200
php bin/magento config:set catalog/search/opensearch_index_prefix magento2
php bin/magento config:set catalog/search/opensearch_enable_auth 0
php bin/magento config:set catalog/search/opensearch_server_timeout 15
php bin/magento indexer:reindex
```
## Step 4: Verifying the Setup
Nginx
```bash
sudo systemctl status nginx
```
PHP-FPM
```bash
sudo systemctl status php8.1-fpm
```
MYSQL
```bash
sudo systemctl status mysql
```
Redis
```bash
sudo systemctl status redis-server

```
Varnish
```bash
sudo systemctl status varnish
```
OpenSearch
```bash
sudo systemctl status opensearch
```
Elasticsearch
```bash
sudo systemctl status elasticsearch
```

## Conclusion
This project involved setting up a production-grade instance of Magento 2 Open Source on an AWS Ubuntu server. We installed and configured the following technologies:

nginx: For handling web server requests
php-fpm: For processing PHP code
MySQL: For managing the database
Varnish: For caching HTTP requests
Redis: For session and cache storage
OpenSearch: For search capabilities
Elasticsearch: For search capabilities
Magento 2: The e-commerce platform
