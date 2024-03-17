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
    ВМ 1 - ОС - RED ОS 7.3.4, IP 192.168.122.200. На данной ВМ развернут CMS - WordPress на базе NGINX,php-fpm 8.1, postgresql 14.11
    ВМ 2 - ОС - RED ОS 8.1, IP 192.168.122.253. На данной ВМ развернут Prometheus server и Grafana.

#### Установка и настройка exporters на ВМ1.
На ВМ1 были разверуты следущие exporters:

    1. Node-exporter
    
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

2. Postgres-exporter
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

3. Nginx-exporter
   
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
        
        #ExecStart=/usr/bin/nginx-prometheus-exporter --nginx.scrape-uri="http://127.0.0.1:80/stub_status" 
        
        ExecReload=/bin/kill -HUP $MAINPID
        
        #TimeoutStopSec=20s
        #SendSIGKILL=no
        
        [Install]
        WantedBy=multi-user.target














        
