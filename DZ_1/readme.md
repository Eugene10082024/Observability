### Prometheus - Exporters, Service Discovery
### Установка и настройка Prometheus, использование exporters
#### Цель:
Установить и настроить Prometheus.
Результатом выполнения данного ДЗ будет являться публичный репозиторий, в системе контроля версий (Github, Gitlab, etc.), в котором будет находиться Readme с описанием выполненых действий.

#### Описание ДЗ:
На виртуальной машине установите любую open source CMS, которая включает в себя следующие компоненты: nginx, php-fpm, database (MySQL or Postgresql)
На этой же виртуальной машине установите Prometheus exporters для сбора метрик со всех компонентов системы (начиная с VM и заканчивая DB, не забудьте про blackbox exporter, который будет проверять доступность вашей CMS)
На этой же или дополнительной виртуальной машине установите Prometheus, задачей которого будет раз в 5 секунд собирать метрики с экспортеров.

#### Задание со звездочкой (повышенная сложность)
на VM с установленной CMS слишком много “портов экспортеров торчит наружу” и они доступны всем, попробуйте настроить доступ только по одному и добавить авторизацию.
Если вы выполнили задание со звездочкой номер 1, то - добавить SSL =)

### Выполнение.
Для выполнения ДЗ было создано 2 ВМ.

    ВМ1 - ОС - RED ОS 7.3.4, IP 192.168.122.200. На данной ВМ развернут CMS - WordPress на базе NGINX,php-fpm 8.1, postgresql 14.11

    ВМ2 - ОС - RED ОS 8.1, IP 192.168.122.253. На данной ВМ развернут Prometheus server и Grafana.

#### 1. Установка и настройка exporters на ВМ1.
На ВМ1 были разверуты следущие exporters:

1.1. Node-exporter

less /etc/systemd/system/node_exporter.service 
    
            [Unit]
            Description=Prometheus Node Exporter
            Documentation=https://github.com/prometheus/node_exporter
            After=network-online.target
            
            [Service]
            User=node_exporter
            Group=node_exporter
            EnvironmentFile=/etc/default/node_exporter
            ExecStart=/usr/bin/node_exporter $OPTIONS
            Restart=on-failure
            RestartSec=5
            
            [Install]
            WantedBy=multi-user.target
    
less /etc/default/node_exporter
            
            OPTIONS=''

1.2. Postgres-exporter
   
less /etc/systemd/system/postgres-exporter.service

            [Unit]
            Description=Prometheus PostgreSQL Exporter
            After=network.target
            
            [Service]
            Type=simple
            Restart=always
            User=postgres
            Group=postgres
            Environment=DATA_SOURCE_NAME="postgresql://postgresql_exporter:Qwerty12@localhost/postgres?sslmode=disable"
            ExecStart=/usr/bin/postgres_exporter --web.listen-address="localhost:9187"
            
            [Install]
            WantedBy=multi-user.target

1.3. Nginx-exporter

less /etc/systemd/system/nginx-prometheus-exporter.service
   
            [Unit]
            Description=NGINX Prometheus Exporter
            Documentation=https://github.com/nginxinc/nginx-prometheus-exporter
            After=network-online.target nginx.service
            Wants=network-online.target
            
            [Service]
            #Restart=always
            User=prometheus
            EnvironmentFile=/etc/default/nginx-prometheus-exporter
            ExecStart=/usr/bin/nginx-prometheus-exporter $ARGS
            ExecReload=/bin/kill -HUP $MAINPID
            
            #TimeoutStopSec=20s
            #SendSIGKILL=no
            
            [Install]
            WantedBy=multi-user.target

less /etc/default/nginx-prometheus-exporter

            ARGS=""

Для получения метрик с nginx в конфигурационный файл /etc/nginx/nginx/conf добавил:

            server {
            listen 8080;
            server_name _;
            location /stub_status {
                stub_status on;
                allow 127.0.0.1;
                deny all;
            }
         }

1.4. php-fpm_exporter

less /etc/systemd/system/php-exporter.service

            [Unit]
            Description=PHP Exporter
            After=network.target
            [Service]
            User=php_exporter
            Group=php_exporter
            Type=simple
            Restart=always
            RestartSec=500ms
            
            ExecStart=/usr/bin/php-fpm_exporter server –phpfpm.scrape-uri «unix:///var/run/php-fpm/www.sock:/status-php-back» --web.listen-address="localhost:9253"
            SyslogIdentifier=php-exporter
            
            StandardOutput=syslog
            StandardError=syslog
            SyslogIdentifier=php-exporter
            
            [Install]
            WantedBy=multi-user.target

1.5. blackbox-exporter

less /etc/systemd/system/blackbox-exporter.service

            [Unit]
            Description=Blackbox Exporter Service
            Wants=network-online.target
            After=network-online.target
            
            [Service]
            Type=simple
            User=user_blackbox
            Group=user_blackbox
            ExecStart=/usr/bin/blackbox_exporter \
              --config.file=/etc/blackbox/blackbox.yml \
              --web.listen-address="0.0.0.0:9115"
            Restart=always
            
            [Install]
            WantedBy=multi-user.target


#### 2. Настройка Prometheus server на ВМ2 

less /etc/prometheus/prometheus.yml

            global:
              scrape_interval: 15s 
              evaluation_interval: 15s 
            alerting:
              alertmanagers:
                - static_configs:
                    - targets:
            rule_files:
            scrape_configs:
              - job_name: "prometheus"
                static_configs:
                  - targets: ["localhost:9090"]
                  
              - job_name: "postgres_exporter"
                scrape_interval: 5s
                static_configs:
                  - targets: ["192.168.122.200:9187"]
                  
              - job_name: "node_exporter"
                scrape_interval: 5s
                static_configs:
                  - targets: ["192.168.122.200:9100"]
                  
              - job_name: "nginx_exporter"
                scrape_interval: 5s
                static_configs:
                  - targets: ["192.168.122.200:9113"]
                  
              - job_name: "php-fpm_exporter"
                scrape_interval: 5s
                static_configs:
                  - targets: ["192.168.122.200:9253"]
                  
              - job_name: 'blackbox'
                scrape_interval: 5s
                metrics_path: /probe
                params:
                  module: [http_2xx]
                static_configs:
                  - targets:
                    - http://redos_wordpress
                relabel_configs:
                  - source_labels: [__address__]
                    target_label: __param_target
                  - source_labels: [__param_target]
                    target_label: instance

### Настройка доступа только по одному одному порту  и добавление авторизации.

1. Выполним настройку поллучения metric через nginx по порту 80. Nginx буду использовать в качестве реверс-прокси.

2. Для авторизации используем следующие УЗ:
   
        node_exporter:
              username: auth_user
              password: "12345678"
        
        postgres_exporter
              username: auth_user_postgres
              password: "87654321"
        blackbox
            username: auth_user_blackbox
            password: Zaqwsx@12
        
        nginx_exporter
            username: auth_user_nginx
            password: Xswqaz@12
        
        php-fpm_exporter
            username: auth_user_php
            password: Qwerty@12


Конфигурационный файл nginx как реверс проки + включение авторизации.

/etc/nginx/nginx.conf 

       server {
                listen       80 default_server;
                listen       [::]:80 default_server;
                server_name  _;
                root         /usr/share/nginx/html;

                # Load configuration files for the default server block.
                include /etc/nginx/default.d/*.conf;

                error_page 404 /404.html;
                location = /404.html {
                }

                error_page 500 502 503 504 /50x.html;
                location = /50x.html {
                }
                
                auth_basic      "Exporters area";
                auth_basic_user_file /etc/nginx/.htpasswd;



                location /node-exporter {
                proxy_pass http://localhost:9100/metrics;
            }

            location /postgres-exporter {
                proxy_pass http://localhost:9187/metrics;
            }

            location /nginx-exporter {
                proxy_pass http://localhost:9113/metrics;
            }

            location /php-fpm-exporter {
                proxy_pass http://localhost:9253/metrics;
            }

            location /blackbox-exporter {
                proxy_pass http://localhost:9115/metrics;
            }
    }




        
