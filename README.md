# hw14
Docker

***Задание***

1. Создайте свой кастомный образ nginx на базе alpine. 
2. После запуска nginx должен отдавать кастомную страницу (достаточно изменить дефолтную страницу nginx).
3. Определите разницу между контейнером и образом, вывод опишите в домашнем задании.
4. Ответьте на вопрос: Можно ли в контейнере собрать ядро?
5. Собранный образ необходимо запушить в docker hub и дать ссылку на ваш репозиторий.

***Решение***

*Установка Docker из репозитория Docker'a*
```
igels@LaptopAll:~$ sudo apt update              #Обновляю пакеты
igels@LaptopAll:~$ sudo apt install apt-transport-https ca-certificates curl software-properties-common    #Обновляю программы, которые понядобятся для докера
igels@LaptopAll:~$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gp  #Добавляю ключи официального репозитория докера
igels@LaptopAll:~$ lsb_release -a
No LSB modules are available.
Distributor ID:	Ubuntu
Description:	Ubuntu 22.04.1 LTS
Release:	22.04
Codename:	jammy         #Буквально за пару дней до выполнения ДЗ обновился, поэтому был немного удивлён, увидев вместо фокала 
igels@LaptopAll:~$ echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null #Добавляю репозиторий докера
igels@LaptopAll:~$ sudo apt install docker-ce   #Непосредственно установка докера
igels@LaptopAll:~$ sudo systemctl status docker
[sudo] пароль для igels: 
Попробуйте ещё раз.
[sudo] пароль для igels: 
● docker.service - Docker Application Container Engine
     Loaded: loaded (/lib/systemd/system/docker.service; enabled; vendor preset>
     Active: active (running) since Sun 2022-08-28 19:59:51 +03; 24min ago
TriggeredBy: ● docker.socket
       Docs: https://docs.docker.com
   Main PID: 1728 (dockerd)
      Tasks: 17
     Memory: 110.5M
        CPU: 977ms
     CGroup: /system.slice/docker.service
             └─1728 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/cont>
igels@LaptopAll:~$ sudo systemctl enable docker; systemctl start docker #устанавливаю в автозагрузку и стартую докер
igels@LaptopAll:~$ docker run hello-world

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/
```
*Установка Docker-compose, с [сайта](https://github.com/docker/compose/releases) скачиваю файл 2.10.2 и перекладываю как /usr/local/bin/docker-compose*
```
igels@LaptopAll:~$ sudo chmod +x /usr/local/bin/docker-compose #Делаю файл исполняемым
igels@LaptopAll:~$ docker-compose --version
Docker Compose version v2.10.2      # проверяю установку
```
**1. Кастомный образ nginx на базе alpine.**

*Создаю файл Dockerfile*
```
FROM alpine:3.14                #Собираю на базу alpine 3.14 потому как она ещё поддерживает php7

MAINTAINER Igelslecha             #Подпись автора

RUN apk add --no-cache --update \ #Установка Nginx и php7
    nginx \
    php7 \
    php7-fpm \
    php7-ctype \
    php7-mbstring \
    php7-json \
    php7-opcache \
    curl \                        #Для использования в директиве HEALTHCHECK в Dockerfile для проверки состояния контейнера. 
    tzdata \                      #Для обеспечения поддержки часовых поясов внутри контейнера;
    tini \                        #Инициализатор процессов init, специально разработанный для запуска в контейнере. Позволяет корректно управлять процессами внутри контейнера и убивать зомби-процессы, не позволяя исчерпать пространство PID. 
    supervisor \                  #Так как идеология запуска процессов в линукс-контейнерах подразумевает запуск только одного процесса на контейнер, то в качестве этого одного процесса можно использовать супервизор, задача которого лежит в запуске остальных процессов;
    logrotate \                   #Инструмент для ротации логов, если таковые есть. 
    dcron \                       #Легковесный crontab. Необходим для периодического запуска процессов внутри контейнера. В данном примере он будет запускать logrotate для периодической ротации логов. Также может пригодиться для периодического запуска каки-то фоновых процессов вашего приложения.
    libcap \                      #Инструмент, который используется для запуска cron от имени непривилегированного пользователя. Так как все процессы в целях безопасности в контейнере будут запускаться от имени пользователя nobody, то и cron должен запускаться от имени этого пользователя, что по умолчанию невозможно. Данный инструмент позволяет решить эту задачу.
    && chown nobody:nobody /usr/sbin/crond \   #Назначаю владельца файла crond нашего пользователя nobody.  
    && setcap cap_setgid=ep /usr/sbin/crond \  #Изменяю setuid-бита на capabilities.
    && mkdir -p /app /logs \                   #Создаю все необходимые каталоги. Как минимум каталог /app для PHP-файлов,а также каталог /logs для размещения файлов логов в нем.
    && rm -rf /tmp/* \                         #Удаляю все временные файлы, все логи, кэши, чтобы образ занимал как можно меньше места,а также все базовые кронтабы (конфиги для cron) и удаляю каталог с подключаемыми конфигами ротатора логов logratate. 
    /var/{cache,log}/* \
    /etc/logrotate.d \
    /crontabs/* \
    /etc/periodic/daily/logrotate              #Конец строки RUN, такая длинная строка, состоящая из множества разных команд создает один слой в образе Docker. Чем меньше слоев — тем лучше.
    
COPY rootfs /                                  #Копирую содержимое каталога /rootfs/ в корень Докер образа

RUN chown -R nobody:nobody /app \              #Изменяю владельца каталогов
    && chown -R nobody:nobody /logs \
    && chown -R nobody:nobody /run \
    && chown -R nobody:nobody /var/lib \
    && chown -R nobody:nobody /var/log/nginx \
    && chown -R nobody:nobody /etc/crontabs 

USER nobody                                   #Меняю пользователя на nobody

WORKDIR /app                                  #Делаю каталог /app внутри образа текущим рабочим каталогом. При запуске контейнера из этого образа этот каталог будет каталогом по умолчанию, относительно которого будет выполняться все в контейнере.

COPY --chown=nobody:nobody app /app           #Копирую содержимое каталога app в app, где лежат файлы php

VOLUME "/logs"                                #Внешняя папка для выгрузки логов

EXPOSE 8080                                   #Nginx работает на порту 8080

ENTRYPOINT ["/sbin/tini", "--", "/usr/bin/docker-entrypoint.sh"]       #Обычно точкой входа является скрипт с именем наподобие entrypoint.sh, который инициализирует контейнер при старте.

CMD ["/usr/bin/supervisord", "-c", "/etc/supervisor/conf.d/supervisord.conf"]    #Указываю выполняемую по умолчанию команду.

HEALTHCHECK --timeout=10s CMD curl --silent --fail http://127.0.0.1:8080/fpm-ping  #Благодаря инструкции HEALTHCHECK можно запускать периодическую проверку жизнеспособности контейнера. В данном примере будем обращаться каждые 30 секунд (по умолчанию) к эндпоинту http://127.0.0.1:8080/fpm-ping. Если последние 3 проверки (по умолчанию) будут провалены, контейнер будет считаться нерабочим.
```
*Запускаю построение образа*
```
igels@LaptopAll:~/hw14/nginx-alpine$ docker build --pull --tag=igelslecha\nginxphp .
***
Successfully tagged igelslechanginxphp2:latest
```
*Создаю каталог для логов*
```
igels@LaptopAll:~/hw14/nginx-alpine$ mkdir -p log
igels@LaptopAll:~/hw14/nginx-alpine$ chmod 777 log
```
*Запускаю контейнер*
```
igels@LaptopAll:~/hw14/nginx-alpine$ docker run -it --name=nginxphp4     -p 80:8080     -v $(pwd)/log:/logs     -v /etc/localtime:/etc/localtime:ro     -v /etc/timezone:/etc/timezone:ro     igelslechanginxphp4
2022-09-04 10:50:28,518 INFO supervisord started with pid 7
2022-09-04 10:50:29,522 INFO spawned: 'crond' with pid 10
2022-09-04 10:50:29,523 INFO spawned: 'nginx' with pid 11
2022-09-04 10:50:29,524 INFO spawned: 'php-fpm' with pid 12
[04-Sep-2022 10:50:29] NOTICE: fpm is running, pid 12
[04-Sep-2022 10:50:29] NOTICE: ready to handle connections
2022-09-04 10:50:30,558 INFO success: crond entered RUNNING state, process has stayed up for > than 1 seconds (startsecs)
2022-09-04 10:50:30,558 INFO success: nginx entered RUNNING state, process has stayed up for > than 1 seconds (startsecs)
2022-09-04 10:50:30,558 INFO success: php-fpm entered RUNNING state, process has stayed up for > than 1 seconds (startsecs)
[04/Sep/2022:10:54:51 +0300] NGINX: 172.17.0.1 "GET / HTTP/1.1" 200 21415 0.017 0.020 "-" "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/105.0.0.0 Safari/537.36" "-"
2022/09/04 10:54:51 [error] 13#13: *17 access forbidden by rule, client: 172.17.0.1, server: _, request: "GET /favicon.ico HTTP/1.1", host: "127.0.0.1", referrer: "http://127.0.0.1/"
[04/Sep/2022:10:54:51 +0300] NGINX: 172.17.0.1 "GET /favicon.ico HTTP/1.1" 403 186 0.000 - "http://127.0.0.1/" "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/105.0.0.0 Safari/537.36" "-"
```
**2. ![Скрин с рабочей машины](https://github.com/igelslecha/hw14/blob/main/Docker1.png "Скрин запроса")**

**3. Разница между контейнером и образом**

*Образ Докера это неизменяемый файл, содержащий исходный код, библиотеки, зависимости, инструменты и другие файлы, необходимые для запуска приложения.*

*По сути образ это шаблон, на базе которого, добавлением ещё одного изменяемого уровня создаются контейнеры.*

**4. О сборке ядра в контейнере**

*Согласно [статье](https://russianblogs.com/article/1138863585/) это возможно, но сборка ядра контейнера и сборка ядра линукса это немного разные понятия, так как должны выполнятся некоторые условия:*

*1. Не ограничено Докером*

*2. Элемент конфигурации можно увидеть в контейнере*

*3. Элемент конфигурации можно разместить в пространстве имён*

 *Подытоживая можно сказать, что сборка ядра контейнера возможна, но с намного меньшими параметрами и ограничениями чем для хостовой ОС*
 
 **5. Выкладывание созданного образа в Docker hub**
 
 *Логинюсь к докеру, заранее зарегистрированным аккаунтом и выкладываю*
 ```
 igels@LaptopAll:~/hw14/nginx-alpine$ docker login
Login with your Docker ID to push and pull images from Docker Hub. If you don't have a Docker ID, head over to https://hub.docker.com to create one.
Username: igelslecha
Password: 
WARNING! Your password will be stored unencrypted in /home/igels/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
igels@LaptopAll:~/hw14/nginx-alpine$ docker push igelslecha/nginx:4
The push refers to repository [docker.io/igelslecha/nginx]
1399e1bf4377: Pushed 
7db9ab36c44a: Pushed 
9e49fb641867: Pushed 
06a03dc8649e: Pushed 
63493a9ab2d4: Mounted from library/alpine 
4: digest: sha256:0f0cb030a1f0c44102970d87b6168a0af6acfee05530c1039ef092380ee2ff15 size: 1363
```
*[Ссылка на образ](https://hub.docker.com/repository/docker/igelslecha/nginx)*
 







