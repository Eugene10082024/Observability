### Отказоустойчивость Prometheus, хранилища метрик для Prometheus (Thanos, VictoriaMetrics, Mimir)
### Цель: Научиться устанавливать и работать с хранилищем метрик.

### Установка и настройка хранилища метрик VictoriaMetrics
1. загружаем архив, ссылку на который мы скопировали
	
 	sudo mkdir /tmp/install_victoriametrics && cd /tmp/install_victoriametrics

	sudo wget https://github.com/VictoriaMetrics/VictoriaMetrics/releases/download/v1.99.0/victoria-metrics-linux-amd64-v1.99.0.tar.gz

2. Распаковываем архив в /usr/bin

	sudo tar zxf victoria-metrics-linux-amd64-*.tar.gz -C /usr/bin/
