### Prometheus - Exporters, Service Discovery
### Установка и настройка Prometheus, использование exporters
#### Цель:
Установить и настроить Prometheus.

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

#### настройка получения metric через nginx по порту 80. Nginx будем использовать в качестве реверс-прокси.

   Для авторизации используем следующие УЗ:
   
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

В Файл /etc/nginx/.htpasswd добавляем hash пароля от УЗ.

less .htpasswd

        auth_user_postgres:$apr1$RRkJIvRV$aNtSTBy6XtKPcFMZ8yphg/
        auth_user:$apr1$Mm8Q.Qh7$WVNzo/.IIt6UKar0tzzvv1
        auth_user_blackbox:$apr1$R7mLlTS9$cBqN.0GWaRCw9N2cEMpaZ.
        auth_user_nginx:$apr1$4WbWczya$cwhGXI9L9amqJy3zDAHN01
        auth_user_php:$apr1$88R8d1YJ$PPb.IIQaqJEqCP3wladXM0

#### Настройка exporters для передачи метрик.

##### NODE_EXPORTER:

vi /etc/systemd/system/node_exporter.service

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

vi /etc/default/node_exporter

        OPTIONS = --web.listen-address=localhost:9100  --web.config.file=/etc/node_exporter/config.yml

##### POSTGRES-EXPORTER

vi /etc/systemd/system/postgres-exporter.service
        [Unit]
        Description=Prometheus PostgreSQL Exporter
        After=network.target
        
        [Service]
        Type=simple
        Restart=always
        User=postgres
        Group=postgres
        EnvironmentFile=/etc/default/postgres_exporter
        Environment=DATA_SOURCE_NAME="postgresql://postgresql_exporter:Qwerty12@localhost/postgres?sslmode=disable"
        ExecStart=/usr/bin/postgres_exporter $OPTIONS
        
        [Install]
        WantedBy=multi-user.target

vi /etc/default/postgres_exporter

        OPTIONS = --web.listen-address=localhost:9187  --web.config.file=/etc/postgres_exporter/config.yml
        
#### NGINX-PROMETHEUS-EXPORTER

vi /etc/systemd/system/nginx-prometheus-exporter.service

        [Unit]
        Description=NGINX Prometheus Exporter
        Documentation=https://github.com/nginxinc/nginx-prometheus-exporter
        After=network-online.target nginx.service
        Wants=network-online.target
        
        [Service]
        #Restart=always
        User=prometheus
        EnvironmentFile=/etc/default/nginx_exporter
        ExecStart=/usr/bin/nginx-prometheus-exporter $OPTIONS
        #ExecStart=/usr/bin/nginx-prometheus-exporter --nginx.scrape-uri="http://127.0.0.1:80/stub_status" 
        ExecReload=/bin/kill -HUP $MAINPID
        #TimeoutStopSec=20s
        #SendSIGKILL=no
        
        [Install]
        WantedBy=multi-user.target

    
vi /etc/default/nginx_exporter    

        OPTIONS = --web.listen-address="localhost:9113" --web.config.file=/etc/nginx_exporter/config.yml

#### PHP-FPM_EXPORTER

vi /etc/systemd/system/php-exporter.service

        [Unit]
        Description=PHP Exporter
        After=network.target
        [Service]
        User=php_exporter
        Group=php_exporter
        Type=simple
        Restart=always
        RestartSec=500ms
        
        EnvironmentFile=/etc/default/php_exporter
        ExecStart=/usr/bin/php-fpm_exporter server –phpfpm.scrape-uri «unix:///var/run/php-fpm/www.sock:/status-php-back» --web.listen-address="localhost:9253"
        SyslogIdentifier=php-exporter
        
        StandardOutput=syslog
        StandardError=syslog
        SyslogIdentifier=php-exporter
        
        [Install]
        WantedBy=multi-user.target

#### BLACKBOX-EXPORTER

vi /etc/blackbox/blackbox.yml

        modules:
          http_2xx:
            prober: http
            http:
              preferred_ip_protocol: "ip4"
          http_post_2xx:
            prober: http
            http:
              method: POST
          tcp_connect:
            prober: tcp
          pop3s_banner:
            prober: tcp
            tcp:
              query_response:
              - expect: "^+OK"
              tls: true
              tls_config:
                insecure_skip_verify: false
          grpc:
            prober: grpc
            grpc:
              tls: true
              preferred_ip_protocol: "ip4"
          grpc_plain:
            prober: grpc
            grpc:
              tls: false
              service: "service1"
          ssh_banner:
            prober: tcp
            tcp:
              query_response:
              - expect: "^SSH-2.0-"
              - send: "SSH-2.0-blackbox-ssh-check"
          irc_banner:
            prober: tcp
            tcp:
              query_response:
              - send: "NICK prober"
              - send: "USER prober prober prober :prober"
              - expect: "PING :([^ ]+)"
                send: "PONG ${1}"
              - expect: "^:[^ ]+ 001"
          icmp:
            prober: icmp
          icmp_ttl5:
            prober: icmp
            timeout: 5s
            icmp:
              ttl: 5
        
#### Настройка базовой авторизации в prometheus.conf
        - job_name: "postgres_exporter"
        scrape_interval: 5s
        basic_auth:
          username: auth_user_postgres
          password: "87654321"
        metrics_path: /postgres-exporter
        static_configs:
          - targets: ["192.168.122.200:80"]
    
      - job_name: "node_exporter"
        scrape_interval: 5s
        basic_auth:
          username: auth_user
          password: "12345678"
        metrics_path: /node-exporter
        static_configs:
          - targets: ["192.168.122.200:80"]
    
      - job_name: "nginx_exporter"
        scrape_interval: 5s
        basic_auth:
          username: auth_user_nginx
          password: "Xswqaz@12"
        metrics_path: /nginx-exporter
        static_configs:
          - targets: ["192.168.122.200:80"]
    
      - job_name: "php-fpm_exporter"
        scrape_interval: 5s
        basic_auth:
          username: auth_user_php
          password: "Qwerty@12"
        metrics_path: /php-fpm-exporter
        static_configs:
          - targets: ["192.168.122.200:80"]
       
      - job_name: 'blackbox'
        scrape_interval: 5s
        basic_auth:
          username: auth_user_blackbox
          password: "Zaqwsx@12"
        metrics_path: /blackbox-exporter
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
          - target_label: __address__
            replacement: 192.168.122.200:80

        
