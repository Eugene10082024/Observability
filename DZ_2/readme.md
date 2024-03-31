### Отказоустойчивость Prometheus, хранилища метрик для Prometheus (Thanos, VictoriaMetrics, Mimir)
### Цель: Научиться устанавливать и работать с хранилищем метрик.

#### Установка и настройка хранилища метрик - VictoriaMetrics

1. загружаем архив, ссылку на который мы скопировали
	
 		sudo mkdir /tmp/install_victoriametrics && cd /tmp/install_victoriametrics

		sudo wget https://github.com/VictoriaMetrics/VictoriaMetrics/releases/download/v1.99.0/victoria-metrics-linux-amd64-v1.99.0.tar.gz

2. Распаковываем архив в /usr/bin

		sudo tar zxf victoria-metrics-linux-amd64-*.tar.gz -C /usr/bin/

3. Создаем пользователя victoriametrics

   		sudo useradd -r  victoriametrics

4. создаем каталоги для хранения данных и идентификатора процесса

   		sudo mkdir -p /var/lib/victoriametrics /run/victoriametrics

5. Зададим в качестве владельца для созданных каталогов

   		sudo chown victoriametrics:victoriametrics /var/lib/victoriametrics /run/victoriametrics

6. Создаем victoriametrics.service:

   		sudo vi /etc/systemd/system/victoriametrics.service

		[Unit]
		Description=VictoriaMetrics
		After=network.target
		
		[Service]
		Type=simple
		User=victoriametrics
		PIDFile=/run/victoriametrics/victoriametrics.pid
		ExecStart=/usr/local/bin/victoria-metrics-prod -storageDataPath /var/lib/victoriametrics -retentionPeriod 12
		ExecStop=/bin/kill -s SIGTERM $MAINPID
		StartLimitBurst=5
		StartLimitInterval=0
		Restart=on-failure
		RestartSec=1
		
		[Install]
		WantedBy=multi-user.target

7. Перечитываем конфигурацию systemd:

		sudo systemctl daemon-reload

8. Разрешаем запуск юнита и стартуем его:

   		sudo systemctl status victoriametrics

9. По умолчанию VictoriaMetrics слушает на порту 8428. Мы можем сделать запрос с помощью curl:

		curl 127.0.0.1:8428
Получаем вывод:

#### Хранение метрик Prometheus в VictoriaMetrics

Добавляем в global:

		global:
		  external_labels:
		    server_name: prometheus01

 опция external_labels позволяет навесить тег, с помощью которого мы будем понимать, что конкретные метрики пришли с конкретного сервера Prometheus.

И добавляем опцию:

		remote_write:
		  - url: http://192.168.0.15:8428/api/v1/write
		    queue_config:
		      max_samples_per_send: 10000
		      capacity: 20000
		      max_shards: 30


И перезапустим прометеус:

systemctl restart prometheus

При желании, мы можем зайти на веб-интерфейс VictoriaMetrics по адресу http://192.168.0.15:8428/vmui/, где 192.168.0.15 — адрес нашего сервера. В поле Query 1 можно ввести любой запрос PromQL, например:

node_memory_MemTotal_bytes{server_name="prometheus01"}



















 
















   
