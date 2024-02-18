#Тестовое задание Sipuni#

*****На всех серверах*****

1. Прописать верное имя сервера.

Прописываем новые имена серверов командой ```hostnamectl set-hostname new_hostname``` проверяем командой ```hostname```. Так же добавляем имена в /etc/hosts чтобы они правильно резольвились командой ```echo "10.30.30.141 srv1" | sudo tee -a /etc/hosts && echo "10.30.30.142 srv2" | sudo tee -a /etc/hosts```.

2. Отключить использование ipv6.

Проверяем наличие IPv6-интерфейсов командой ```ip addr | grep inet6```, добавляем строчку net.ipv6.conf.all.disable_ipv6 = 1 в конец файла /etc/sysctl.conf командой ```echo "net.ipv6.conf.all.disable_ipv6 = 1" >> /etc/sysctl.conf && sysctl -p``` снова проверяем наличие IPv6-интерфейсов командой ```ip addr | grep inet6```.

3. Добавить для себя пользователя и прописать ssh-ключ для доступа.

Предварительно необходимо прописать DNS-сервер для полноценной работы интернета командой ```echo "nameserver 8.8.8.8" >> /etc/resolv.conf```. Далее устанавливаем пакет sudo командой ```apt update && apt install sudo -y```. После добавляем пользователя evgeniy командой ```adduser evgeniy``` и даём ему права sudo командой ```usermod -a -G sudo evgeniy```. 

На srv1 логинимся под новым пользователем и генерируем пару ssh-ключей командой ```ssh-keygen -t rsa```. Выполняем команду cat ~/.ssh/id_rsa и копируем приватный ключ в свой ssh-клиент. Далее командой ```cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys``` копируем приватный ключ в список авторизованых ключей. Не забываем скопировать публичный ключ на srv2.Для этого сначала создаём каталог и файл командой ```ssh evgeniy@10.30.30.142 'mkdir -p ~/.ssh && touch ~/.ssh/authorized_keys'``` и потом добавляем в файл authorized_keys открытый ключ командой ```cat ~/.ssh/id_rsa.pub | ssh evgeniy@10.30.30.142 'cat >> ~/.ssh/authorized_keys'```. Пробуем залогиниться на srv1 и srv2 с помощью ключа без использования пароля.

4. Отключить вход по паролю через ssh.

Для отключения входа по паролю через ssh необходимо отредактировать файл /etc/ssh/sshd_config в нём нужно менять строчку PasswordAuthentication yes на PasswordAuthentication no и перезапустить службу ssh. Сделать это можно с помощью команды ```sed -i 's/PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config && systemctl restart ssh```.

5. Настроить для своего пользователя переключение в режим root через sudo -i без ввода пароля.

Для настройки своего пользователя на ввод sudo -i без пароля нужно осуществить вход на сервер под этим пользователем и отредактировать файл /etc/sudoers добавив в него строчку evgeniy ALL=(ALL) NOPASSWD: ALL. Эта строчка позволит пользователю запускать все команды используя sudo без ввода пароля. Добавить эту строчку в файл можно командой ```echo "evgeniy ALL=(ALL) NOPASSWD: ALL" | sudo tee -a /etc/sudoers```.

6. Настроить мастер-мастер репликацию между srv1 и srv2 (если необходимо доустановить необходимое ПО).
7. Написать роль ансибла по конфигурированию инстанса mysql на srv1 и srv2 (конфигурационные файлы, создание пользователей) - выслать архив с ролью.

По согласованию данные два пункта преобразованы в один с формулировкой "Написать роль ансибла, которая настроит репликацию MySQL master-master на srv1 и srv2. Предоставить файлы роли".

На клиентском хосте с Ansible копируем каталог ansible из репозитория. Копируем приватный ключ на сервер с Ansible. После подготовки всех файлов запускаем плейбук командой ```ansible-playbook playbook.yaml -i inventory.yaml```. По завершению работы Ansible получаем 2 инстанса MySQL (MariaDB) с настроенной репликацией master-master.

*****На srv1*****

1. Установить nginx, php-fpm (версия 7.0), percona-server (версия 5.7).

Предварительно поставим нужные пакеты командой ```apt-get install curl wget gnupg2 ca-certificates lsb-release apt-transport-https -y```.
Устанавливаем Nginx командой ```apt install nginx -y```. 
Т.к. в официальных репозиториях php-fpm версии 7.0 уже не поддерживается добавляем  репозиторий sury php и ключ GPG для репозитория командой ```wget https://packages.sury.org/php/apt.gpg && apt-key add apt.gpg && echo "deb https://packages.sury.org/php/ $(lsb_release -sc) main" | sudo tee /etc/apt/sources.list.d/php7.list```. Обновляем список пакетов и устанавливаем php-fpm 7.0 командой ```apt update && apt install php7.0 php7.0-fpm php7.0-mysql libapache2-mod-php7.0 libapache2-mod-fcgid -y```. Узнаем рабочую директорию для Nginx командой ```cat /etc/nginx/sites-available/default | grep root```, в данном случае директория /var/www/html. Добавляем в неё проверочный файл командой ```echo "<?php" > /var/www/html/phpinfo.php && echo "phpinfo();" >> /var/www/html/phpinfo.php && echo "?>" >> /var/www/html/phpinfo.php```. Включаем php-fpm набором команд ```systemctl start php7.0-fpm && systemctl enable php7.0-fpm```. В конфигурационный файл nginx /etc/nginx/sites-available/default добавляем блок с директивой php-fpm:

```
location ~ \.php$ {
    include snippets/fastcgi-php.conf;
    fastcgi_pass unix:/var/run/php/php7.0-fpm.sock;
}
```
Проверяем конфигурацию nginx командой ```nginx -t``` и если всё ок, то перезапускаем nginx командой ```nginx -s reload```. Проверяем работу php-fpm командой ```curl localhost/phpinfo.php | grep "Server API"```. Если всё настроено правильно то Server API будет FPM/FastCGI.

Т.к. на srv1 уже установлена MariaDB поставить рядом percona-server проблематично, поэтому поставим её в контейнере. Для этого установим Docker командой ```curl -fsSL https://download.docker.com/linux/debian/gpg | sudo apt-key add - && echo "deb [arch=amd64] https://download.docker.com/linux/debian $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list && apt update && apt-cache policy docker-ce && apt install docker-ce -y``` и поставим Docker-compose командой ```sudo curl -L https://github.com/docker/compose/releases/download/1.25.3/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose && sudo chmod +x /usr/local/bin/docker-compose```. В каталог /srv кладём каталог percona (в нем хранится docker-compose для деплоя Percona) из репозитория и выполняем ```cd /srv/percona && docker-compose up -d```. Проверяем что контейнер поднялся командой ```docker ps``` и что порт сервиса проброшен на целевую машину командой ```lsof -i :13306```. 

2. Создать бд srv1 и пользователя для коннекта к этой базе dbuser c произвольным паролем.

Создаём БД srv1 в percona командой ```mysql -u root -p'SiPuNi' -P 13306 -h 127.0.0.1 -e "CREATE DATABASE srv1;"```. Создаём пользователя dbuser командой ```mysql -u root -p'SiPuNi' -P 13306 -h 127.0.0.1 -e "CREATE USER 'dbuser'@'localhost' IDENTIFIED BY 'SiPuNi';"```. Выдаём права пользователю dbuser на БД srv1 командой ```mysql -u root -p'SiPuNi' -P 13306 -h 127.0.0.1 -e "GRANT ALL PRIVILEGES ON srv1.* TO 'dbuser'@'localhost';"```.

3. Настройка iptables.

Копируем из репозитория файл с правилами для iptables iptables.rules в папку /srv и выполняем команду ```cat /srv/iptables.rules | iptables-restore```. Проверяем примененные правила командой ```iptables-save```. Устанавливаем пакет iptables-persistent ```apt install iptables-persistent -y```. Во время установки соглашаемся сохранить текущие правила. Пакет нужен для восстановление правил iptables после перезагрузки. Дополнительно после установки лучше выполнить вручную команду ```iptables-save > /etc/iptables/rules.v4``` для сохранения текущей конфигурации правил и применения её после перезагрузки. После перезагрузки можем проверить работающие правила командой ```iptables-save```.

*****На srv2*****

1. Создать пользователя builder.

Создаем пользователя на srv2 командой ```adduser builder```. Так же создаем пользователя builder на srv1 ```ssh root@10.30.30.141 adduser builder```. 

2. Настроить возможность коннекта пользователя builder к серверу srv1 через ssh-ключ.

Генерируем пару ssh-ключей на srv2 для пользователя builder ```sudo -u builder ssh-keygen -t rsa```. Добавим публичный ключ на srv1 для пользователя builder 
```
builderopenkey=$(cat /home/builder/.ssh/id_rsa.pub) && \
ssh root@10.30.30.141 "\
sudo -u builder bash -c '
mkdir -p /home/builder/.ssh &&
echo \"$builderopenkey\" >> /home/builder/.ssh/authorized_keys &&
chmod 700 /home/builder/.ssh &&
chmod 600 /home/builder/.ssh/authorized_keys
'"
```
После этого пользователь builder сможет подключаться с srv2 на srv1 по ssh ключу с помощью команды ```ssh builder@10.30.30.141```

3. Установить mysql-клиент.

Mysql клиент уже был установлен на шаге установки кластера Mysql и настройки репликации master-master.

4. Настроить коннект пользователя builder к бд на srv1 при запуске в терминале команды mysql (без запроса пароля).

Создадим пользователя builder на srv1 в СУБД Percona ```mysql -u root -p'SiPuNi' -P 13306 -h 10.30.30.141 -e "CREATE USER 'builder'@'%' IDENTIFIED BY 'SiPuNi';```.
В домашний каталог пользователя builder на srv2 /home/builder скопируем файл .my.cnf из репозитория. Теперь команда ```mysql``` будет автоматиески подключаться к Percona на сервере srv1.

5. Настроить выполнение от пользователя builder по крону добавление в конец файла /tmp/crontest произвольной строки каждый час.

```echo "0 * * * * echo \"\$(date): \$(shuf -i 10000-99999 -n 1)\" >> /tmp/crontest" | crontab -u builder -``` эта команда добавляет задачу в крон для пользователя builder, чтобы выполнить команду в начале каждого часа и добавить случайный набор цифр в файл /tmp/crontest.