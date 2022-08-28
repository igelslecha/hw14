# hw14
Docker

***Задание***

* Создайте свой кастомный образ nginx на базе alpine. 
* После запуска nginx должен отдавать кастомную страницу (достаточно изменить дефолтную страницу nginx)
* Определите разницу между контейнером и образом, вывод опишите в домашнем задании.
* Ответьте на вопрос: Можно ли в контейнере собрать ядро?
* Собранный образ необходимо запушить в docker hub и дать ссылку на ваш репозиторий.

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
*Установка Docker-compose, с сайта https://github.com/docker/compose/releases скачиваю файл 2.10.2 и перекладываю как /usr/local/bin/docker-compose*
```
igels@LaptopAll:~$ sudo chmod +x /usr/local/bin/docker-compose #Делаю файл исполняемым
igels@LaptopAll:~$ docker-compose --version
Docker Compose version v2.10.2      # проверяю установку
```
**Разница между контейнером и образом**
*Образ Докера это неизменяемый файл, содержащий исходный код, библиотеки, зависимости, инструменты и другие файлы, необходимые для запуска приложения.*

*По сути образ это шаблон, на базе которого, добавлением ещё одного изменяемого уровня создаются контейнеры.*

**О сборке ядра в контейнере**

*Согласно статье это возможно, но сборка ядра контейнера и сборка ядра линукса это немного разные понятия, так как должны выполнятся некоторые условия:*

*1. Не ограничено Докером*

*2. Элемент конфигурации можно увидеть в контейнере*

*3. Элемент конфигурации можно разместить в пространстве имён*







