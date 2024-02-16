# Настраиваем центральный сервер сбор логов

## Цели домашнего задания

1. Научится проектировать централизованный сбор логов.
2. Рассмотреть особенности разных платформ для сбора логов.

## Описание домашнего задания

1. В Vagrant разворачиваем 2 виртуальные машины web и log
2. на web настраиваем nginx
3. на log настраиваем центральный лог сервер на любой системе на выбор: journald, rsyslog, elk.
4. настраиваем аудит, следящий за изменением конфигов nginx

Все критичные логи с web должны собираться и локально и удаленно. Все логи с nginx должны уходить на удаленный сервер (локально только критичные).
Логи аудита должны также уходить на удаленную систему.

Формат сдачи ДЗ - vagrant + ansible

## Пошаговая инструкция выполнения домашнего задания

### Создаём виртуальные машины

- Создаём каталог, в котором будут храниться настройки виртуальной машины. В каталоге создаём файл с именем [Vagrantfile](./Vagrantfile)
- Результатом выполнения команды vagrant up станут 2 созданные виртуальные машины
- Заходим на web-сервер: vagrant ssh web

> Дальнейшие действия выполняются от пользователя root

- Переходим в root пользователя

```bash
sudo -i
```

- Для правильной работы c логами, нужно, чтобы на всех хостах было настроено одинаковое время. Укажем часовой пояс (Московское время)

```bash
\cp /usr/share/zoneinfo/Europe/Moscow /etc/localtime
```

- Перезупустим службу NTP Chrony:

```bash
systemctl restart chronyd
```

- Проверим, что служба работает корректно:

```bash
systemctl status chronyd
```

![2024-02-16_09-44-49](https://github.com/dimkaspaun/log/assets/156161074/4b38059f-52b3-4ec0-821a-a85c0091e872)



- Далее проверим, что время и дата указаны правильно

```bash
date
```

> Настроить NTP нужно на обоих серверах

### Установка nginx на виртуальной машине web

- Для установки nginx сначала нужно установить epel-release

```bash
yum install epel-release -y
```

- Установим nginx

```bash
yum install -y nginx
systemctl enable nginx
systemctl start nginx
```

- Проверим, что nginx работает корректно

```bash
systemctl status nginx
```

![2024-02-16_09-46-27](https://github.com/dimkaspaun/log/assets/156161074/8c12484e-1df8-4ad3-86c4-839a078b6275)

```
ss -tln | grep 80

LISTEN     0      128          *:80                       *:*                  
LISTEN     0      128       [::]:80                    [::]:* 
```

> Также работу nginx можно проверить на хосте. В браузере ввведем в адерсную строку <http://192.168.50.10>

### Настройка центрального сервера сбора логов

- Откроем ещё одно окно терминала и подключимся по ssh к ВМ log: vagrant ssh rsyslog
- Перейдем в пользователя root: sudo -i
- rsyslog должен быть установлен по умолчанию в нашей ОС, проверим это

```bash
yum list rsyslog
...
Installed Packages
rsyslog.x86_64                                   8.24.0-52.el7                                        @anaconda
Available Packages
rsyslog.x86_64                                   8.24.0-57.el7_9.1                                    updates 
```

> Все настройки Rsyslog хранятся в файле /etc/rsyslog.conf

Для того, чтобы наш сервер мог принимать логи, нам необходимо внести следующие изменения в файл:

- Открываем порт 514 (TCP и UDP) и Находим закомментированные строки

```bash
# Provides UDP syslog reception
#$ModLoad imudp
#$UDPServerRun 514

# Provides TCP syslog reception
#$ModLoad imtcp
#$InputTCPServerRun 514
```

- И приводим их к виду

```bash
# Provides UDP syslog reception
$ModLoad imudp
$UDPServerRun 514

# Provides TCP syslog reception
$ModLoad imtcp
$InputTCPServerRun 514
```

- В конец файла /etc/rsyslog.conf добавляем правила приёма сообщений от хостов

```bash
$template RemoteLogs,"/var/log/rsyslog/%HOSTNAME%/%PROGRAMNAME%.log"
*.* ?RemoteLogs
& ~
```

> Данные параметры будут отправлять в папку /var/log/rsyslog логи, которые будут приходить от других серверов. Например, Access-логи nginx от сервера web, будут идти в файл /var/log/rsyslog/web/nginx_access.log

- Далее сохраняем файл и перезапускаем службу rsyslog

```bash
systemctl restart rsyslog
```

- Если ошибок не допущено, то у нас будут видны открытые порты TCP,UDP 514


 ![2024-02-16_09-56-38](https://github.com/dimkaspaun/log/assets/156161074/6148b25c-1daf-4542-9caf-dda5c2f983e5)


### Далее настроим отправку логов с web-сервера

- Заходим на web сервер: vagrant ssh web
- Переходим в root пользователя: sudo -i
- Проверим версию nginx
- Находим в файле /etc/nginx/nginx.conf раздел с логами и приводим их к следующему виду

```bash
error_log /var/log/nginx/error.log;
error_log syslog:server=192.168.50.11:514,tag=nginx_error;

http {
    access_log syslog:server=192.168.50.11:514,tag=nginx_access,severity=info combined;
```

Для Access-логов указываем удаленный сервер и уровень логов, которые нужно отправлять. Для error_log добавляем удаленный сервер. Если требуется чтобы логи хранились локально и отправлялись
на удаленный сервер, требуется указать 2 строки. Tag нужен для того, чтобы логи записывались в разные файлы.

По умолчанию, error-логи отправляют логи, которые имеют severity: error, crit, alert и emerg. Если трубуется хранили или пересылать логи с другим severity, то это также можно указать в
настройках nginx.

- Далее проверяем, что конфигурация nginx указана правильно: nginx -t


![2024-02-16_09-59-04](https://github.com/dimkaspaun/log/assets/156161074/26002988-a58c-46cf-8c1f-4dc12f79d650)



- Далее перезапустим nginx: systemctl restart nginx

- Чтобы проверить, что логи ошибок также улетают на удаленный сервер, можно удалить картинку, к которой будет обращаться nginx во время открытия веб-сраницы

```bash
rm -f /usr/share/nginx/html/img/header-background.png
```

- Попробуем несколько раз зайти по адресу <http://192.168.50.10>

![2024-02-16_10-16-39](https://github.com/dimkaspaun/log/assets/156161074/eb5bcca5-293a-49e4-8a2a-e2ddeb1dd092)

  
- Далее заходим на log-сервер и смотрим информацию об nginx



![2024-02-16_10-23-34](https://github.com/dimkaspaun/log/assets/156161074/571d2332-5c15-43c3-a408-0697c0cc8c57)



> Видим, что логи отправляются корректно.

### Настройка аудита, контролирующего изменения конфигурации nginx

- За аудит отвечает утилита auditd, в RHEL-based системах обычно он уже предустановлен. Проверим это


![2024-02-16_10-24-17](https://github.com/dimkaspaun/log/assets/156161074/00fcb991-69c3-465d-838c-7515adedef2b)


> Настроим аудит изменения конфигурации nginx

- Добавим правило, которое будет отслеживать изменения в конфигруации nginx. Для этого в конец файла /etc/audit/rules.d/audit.rules добавим следующие строки:

```bash
-w /etc/nginx/nginx.conf -p wa -k nginx_conf
-w /etc/nginx/default.d/ -p wa -k nginx_conf
```

Данные правила позволяют контролировать запись (w) и измения атрибутов (a) в:

- /etc/nginx/nginx.conf
- Всех файлов каталога /etc/nginx/default.d/

> Для более удобного поиска к событиям добавляется метка nginx_conf

- Перезапускаем службу auditd: service auditd restart
- После данных изменений у нас начнут локально записываться логи аудита. Чтобы проверить, что логи аудита начали записываться локально, нужно внести изменения в файл /etc/nginx/nginx.conf или поменять его атрибут, потом посмотреть информацию об изменениях

```bash
ausearch -k nginx_conf -f /etc/nginx/nginx.conf
```

![2024-02-16_10-30-10](https://github.com/dimkaspaun/log/assets/156161074/74cba74f-3e5d-4270-8dd8-bb1f145d5d87)




- Также можно воспользоваться поиском по файлу /var/log/audit/audit.log, указав наш тэг

```bash
grep nginx_conf /var/log/audit/audit.log

```

![2024-02-16_10-28-01](https://github.com/dimkaspaun/log/assets/156161074/1aa8d76e-93f5-4206-ab36-030425950d65)


### Далее настроим пересылку логов на удаленный сервер. Auditd по умолчанию не умеет пересылать логи, для пересылки на web-сервере потребуется установить пакет audispd-plugins

```bash
yum -y install audispd-plugins
```

- Найдем и поменяем следующие строки в файле /etc/audit/auditd.conf

```bash
log_format = RAW
name_format = HOSTNAME
```

> В name_format указываем HOSTNAME, чтобы в логах на удаленном сервере отображалось имя хоста.

В файле /etc/audisp/plugins.d/au-remote.conf поменяем параметр active на yes

```bash
active = yes
direction = out
path = /sbin/audisp-remote
type = always
#args =
format = string
```

- В файле /etc/audisp/audisp-remote.conf требуется указать адрес сервера и порт, на который будут отправляться логи

```bash
remote_server = 192.168.50.11
```

- Далее перезапускаем службу auditd: service auditd restart

> На этом настройка web-сервера завершена. Далее настроим Log-сервер.

- Откроем порт TCP 60, для этого уберем значки комментария в файле /etc/audit/auditd.conf

```bash
tcp_listen_port = 60
```

- Перезапускаем службу auditd: service auditd restart

- На этом настройка пересылки логов аудита закончена. Можем попробовать поменять атрибут у файла /etc/nginx/nginx.conf и проверить на log-сервере, что пришла информация об изменении атрибута


![2024-02-16_10-36-51](https://github.com/dimkaspaun/log/assets/156161074/cea997c9-f560-4938-9fb6-104ada812283)



- Видим лог об изменении атрибута файла на web

```bash
grep web /var/log/audit/audit.log
```

![2024-02-16_10-37-32](https://github.com/dimkaspaun/log/assets/156161074/e8212e97-bfd4-40ab-859d-615f0ec3acc4)



