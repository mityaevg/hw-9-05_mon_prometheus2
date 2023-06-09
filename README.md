# hw-9-05_mon_prometheus2
HW-9-05_Система мониторинга Prometheus. Часть 2

# Домашнее задание к занятию «Система мониторинга Prometheus. Часть 2»

### Задание 1

Создайте файл с правилом оповещения, как в лекции, и добавьте его в конфиг Prometheus.

#### Процесс выполнения

Создадим конфиг-файл **netology-test.yml** с правилом оповещения:
```
nano /etc/prometheus/netology-test.yml
```
```
groups: # Список групп

- name: netology-test # Имя группы
  rules: # Список правил текущей группы
  - alert: InstanceDown # Название текущего правила
    expr: up == 0 # Логическое выражение
    for: 1m # Сколько ждать отбоя сработки перед отправкой оповещения
    labels:
      severity: critical # Критичность события
    annotations: # Описание
      description: '{{ $labels.instance }} of job {{ $labels.job }} has been down for more than 1 minute.' # Полное описание оповещения
      summary: Instance {{ $labels.instance }} down # Краткое описание оповещения
```
Присвоим пользователю **prometheus** права доступа с созданному конфиг-файлу **netology-test.yml**:
```
chown prometheus:prometheus /etc/prometheus/netology-test.yml
```
Подключим правило **InstanceDown** к **Prometheus**:
```
cd /etc/prometheus
nano ./prometheus.yml
```
В конфиг-файле **prometheus.yml** найдем раздел **rule_files:** и пропишем там конфиг-файл **netology-test.yml**
своего правила:
```
rule_files:
  - "netology-test.yml"
```
<kbd>![Раздел rule_files в prometheus.yml](img/prometheus_config_rule_files.png)</kbd>

Остановим сервис **node_exporter.service**:
```
systemctl stop node_exporter.service
systemctl status node_exporter.service
```
Работа сервиса остановлена:

<kbd>![Сервис node_exporter остановлен](img/node_exporter_status_stopped.png)</kbd>

Скриншот раздела оповещений **Alerts** в **Prometheus**:

<kbd>![Prometheus Alerts Pending Status](img/prometheus_alerts_pending.png)</kbd>


---

### Задание 2

Установите Alertmanager и интегрируйте его с Prometheus.

#### Процесс выполнения

Скачаем последнюю версию **Alertmanager 0.25.0** из GitHub:
```
wget https://github.com/prometheus/alertmanager/releases/alertmanager-0.25.0.linux-and64.tar.tz
```
Распакуем скачанный архив:
```
tar xvfz alertmanager-0.25.0.linux-and64.tar.tz
```
Скопируем содержимое получившейся директории в определенные расположения на нашем хосте и предоставим
необходимые права доступа пользователю **prometheus**:
```
cd alertmanager-0.25.0.linux-and64.tar.tz
cp ./alertmanager /usr/local/bin/alertmanager
chown prometheus:prometheus /usr/local/bin/alertmanager
cp ./amtool /usr/local/bin/amtool
chown prometheus:prometheus /usr/local/bin/amtool
cp ./alertmanager.yml /etc/prometheus/alertmanager.yml
chown prometheus:prometheus /etc/prometheus/alertmanager.yml
```
Создадим сервис **prometheus-alertmanager.service**:
```
nano /etc/systemd/system/prometheus-alertmanager.service
```
```
[Unit]
Description=Alertmanager Service
After=network.target
[Service]
EnvironmentFile=-/etc/default/alertmanager
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/alertmanager \
--config.file=/etc/prometheus/alertmanager.yml \
--storage.path=/var/lib/prometheus/alertmanager $ARGS
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure
[Install]
WantedBy=multi-user.target
```
Запустим сервис **prometheus-alertmanager.service**:
```
systemctl enable prometheus-alertmanager.service
systemctl start prometheus-alertmanager.service
systemctl status prometheus-alertmanager.service
```
Проверка статуса работы сервиса **prometheus-alertmanager.service**:

<kbd>![Статус prometheus-alertmanager.service](img/prometheus-alertmanager.service_status.png)</kbd>

Настройка **Prometheus** на работу c **Alertmanager**:
```
nano /etc/prometheus/prometheus.yml
```
Находим раздел **#Alertmanager configuration**, раскомментируем строку ```# - alertmanager:9093``` и
заменим на ```- localhost: 9093```:

```
# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - localhost:9093
```
<kbd>![Alert manager configuration в prometheus.yml](img/prometheus.yml_alertmanager_config.png)</kbd>

Перезапустим **prometheus.service** и проверим статус работы сервиса:
```
systemctl restart prometheus.service
systemctl status prometheus.service
```
<kbd>![Статус работы prometheus.service](img/prometheus_service_status.png)</kbd>

Скриншот из веб-интерфейса **Prometheus**, показывающий состояние правила оповещения **InstanceDown**, 
- **FIRING**:

<kbd>![Prometheus Webinterface Rule Firing](img/prometheus_webinterface_alerts_instancedown_firing.png)</kbd>

Скриншот из веб-интерфейса **Alertmanager**:

<kbd>![Alertmanager Webinterface InstanceDown правило](img/alertmananer_webinterface_instancedown.png)</kbd>

---

### Задание 3

Активируйте экспортёр метрик в Docker и подключите его к Prometheus.

#### Процесс выполнения

1. Установка **docker engine** согласно инструкции приведенной на официальном сайте **Docker**:
```
sudo apt update
sudo apt install ca-certificates curl gnupg

sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
"deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian \
"$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update

sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

docker -v

sudo docker run hello-world
```
<kbd>![Вывод на экран версии докера](img/docker_version.png)</kbd>

<kbd>![Hello world container](img/docker_hello-world.png)</kbd>

Включим автозапуск **docker.service**:
```
systemctl enable docker.service
```
Запустим **docker.service** и проверим его статус:
```
systemctl start docker.service
systemctl status docker.service
```
2. Для того, чтобы активировать экспортер метрик в **Docker** (заранее заложенный туда функционал
мониторига с помощью **Prometheus**), необходимо создать override-файл **daemon.json** для изменения
стандартных параметров, с помощью которых запускается **Docker**:
```
nano /etc/docker/daemon.json
```
```
{
 "metrics-addr": "0.0.0.0:9323",
 "experimental": true
}
```
3. Перезапустим **docker.service** и проверим его статус:
```
systemctl restart docker.service
systemctl status docker.service
```
4. Подключаемся к эндпоинту **http://10.0.2.15:9323/metrics** и видим список различных метрик,
которые выводим сам о себе **Docker**:

<kbd>![Метрики Докера](img/docker_endpoint_metrics.png)</kbd>

5. Добавим эндпоинт в конфиг **Prometheus**:
```
nano /etc/prometheus/prometheus.yml
```
В разделе **static_configs:** добавим новый **target** - **"10.0.2.15:9323"**:

<kbd>![Добавление таргета в конфиг Prometheus](img/adding_target_prometheus_config.png)</kbd>

Перезапустим **prometheus.service**:
```
systemctl restart prometheus.service
```
6. Проверим статус добавленного endpoint в веб-интерфейсе **Prometheus** - **http://10.0.2.15:9090/targets**:

<kbd>![Status -> Targets](img/prometheus_status_targets.png)</kbd>



