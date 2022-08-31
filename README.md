<h3>### LOGS ###</h3>

<h4>Описание домашнего задания</h4>

<ol>
<li>в вагранте поднимаем 2 машины web и log</li>
<li>на web поднимаем nginx</li>
<li>на log настраиваем центральный лог сервер на любой системе на выбор</li>
<ul>
<li>journald;</li>
<li>rsyslog;</li>
<li>elk.</li>
</ul>
<li>настраиваем аудит, следящий за изменением конфигов нжинкса</li>
</ol>
<p>Все критичные логи с web должны собираться и локально и удаленно.<br />
Все логи с nginx должны уходить на удаленный сервер (локально только критичные).<br />
Логи аудита должны также уходить на удаленную систему.<br />
Формат сдачи ДЗ - vagrant + ansible</p>

<h4>1. Создаём виртуальные машины web и log</h4>

<p>В домашней директории создадим директорию logs, в котором будут храниться настройки виртуальных машин web и log:</p>

<pre>[user@localhost otus]$ mkdir ./logs
[user@localhost otus]$</pre>

<p>Перейдём в директорию logs:</p>

<pre>[user@localhost otus]$ cd ./logs/
[user@localhost logs]$</pre>

<p>Создадим файл Vagrantfile:</p>

<pre>[user@localhost logs]$ vi ./Vagrantfile</pre>

<p>Заполним следующим содержимым:</p>

<pre># -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  # Base VM OS configuration.
  config.vm.box = "centos/7"
  config.vm.box_version = "2004.01"
  
  config.vm.provider :virtualbox do |v|
    v.memory = 512
    v.cpus = 1
  end
  
  # Define two VMs with static private IP addresses.
  boxes = [
    { :name => "web",
      :ip => "192.168.50.10",
    },
    { :name => "log",
      :ip => "192.168.50.15",
    }
  ]
  # Provision each of the VMs.
  boxes.each do |opts|
    config.vm.define opts[:name] do |config|
      config.vm.hostname = opts[:name]
      config.vm.network "private_network", ip: opts[:ip]
	  
    end
  end
end
</pre>

<p>Запустим эти виртуальные машины:</p>

<pre>[user@localhost logs]$ vagrant up</pre>

<p>Проверим состояние созданных и запущенных машин:</p>

<pre>[user@localhost logs]$ vagrant status
Current machine states:

web                       running (virtualbox)
log                       running (virtualbox)

This environment represents multiple VMs. The VMs are all listed
above with their current state. For more information about a specific
VM, run `vagrant status NAME`.
[user@localhost logs]$</pre>

<p>Заходим на сервер web:</p>

<pre>[user@localhost logs]$ vagrant ssh web
[vagrant@web ~]$</pre>

<p>Заходим под правами root:</p>

<pre>[vagrant@web ~]$ sudo -i
[root@web ~]#</pre>

<p>Для правильной работы c логами, нужно, чтобы на всех хостах было настроено одинаковое время.<br />
Укажем часовой пояс (Московское время):</p>

<pre>[root@web ~]# cp /usr/share/zoneinfo/Europe/Moscow /etc/localtime
cp: overwrite ‘/etc/localtime’? y
[root@web ~]#</pre>

<p>Перезупустим службу NTP Chrony:</p>

<pre>[root@web ~]# systemctl restart chronyd
[root@web ~]#</pre>

<p>Проверим, что служба работает корректно:</p>

<pre>[root@web ~]# systemctl status chronyd
● chronyd.service - NTP client/server
   Loaded: loaded (/usr/lib/systemd/system/chronyd.service; enabled; vendor preset: enabled)
   Active: active (running) since Wed 2022-08-31 10:22:56 MSK; 1min 3s ago
     Docs: man:chronyd(8)
           man:chrony.conf(5)
  Process: 3244 ExecStartPost=/usr/libexec/chrony-helper update-daemon (code=exited, status=0/SUCCESS)
  Process: 3240 ExecStart=/usr/sbin/chronyd $OPTIONS (code=exited, status=0/SUCCESS)
 Main PID: 3242 (chronyd)
   CGroup: /system.slice/chronyd.service
           └─3242 /usr/sbin/chronyd

Aug 31 10:22:56 web systemd[1]: Stopped NTP client/server.
Aug 31 10:22:56 web systemd[1]: Starting NTP client/server...
Aug 31 10:22:56 web chronyd[3242]: chronyd version 3.4 starting (+CMDMON +NTP +R...G)
Aug 31 10:22:56 web chronyd[3242]: Frequency 0.225 +/- 0.096 ppm read from /var/...ft
Aug 31 10:22:56 web systemd[1]: Started NTP client/server.
Aug 31 10:23:01 web chronyd[3242]: Selected source 129.70.132.36
Hint: Some lines were ellipsized, use -l to show in full.
[root@web ~]#</pre>

<p>Далее проверим, что время и дата указаны правильно:</p>

<pre>[root@web ~]# date
Wed Aug 31 10:25:30 MSK 2022
[root@web ~]#</pre>

<p>Аналогичным образом настрогим на сервере log.</p>

<p>Откроем ещё одно окно терминала и подключимся по ssh к серверу log:</p>

<pre>[user@localhost logs]$ vagrant ssh log
[vagrant@log ~]$</pre>

<p>Заходим под правами root:</p>

<pre>[vagrant@log ~]$ sudo -i
[root@log ~]#</pre>

<p>Для правильной работы c логами, нужно, чтобы на всех хостах было настроено одинаковое время.<br />
Укажем часовой пояс (Московское время):</p>

<pre>[root@log ~]# cp /usr/share/zoneinfo/Europe/Moscow /etc/localtime
cp: overwrite ‘/etc/localtime’? y
[root@log ~]#</pre>

<p>Перезупустим службу NTP Chrony:</p>

<pre>[root@log ~]# systemctl restart chronyd
[root@log ~]#</pre>

<p>Проверим, что служба работает корректно:</p>

<pre>[root@log ~]# systemctl status chronyd
● chronyd.service - NTP client/server
   Loaded: loaded (/usr/lib/systemd/system/chronyd.service; enabled; vendor preset: enabled)
   Active: active (running) since Wed 2022-08-31 10:30:54 MSK; 11s ago
     Docs: man:chronyd(8)
           man:chrony.conf(5)
  Process: 21869 ExecStartPost=/usr/libexec/chrony-helper update-daemon (code=exited, status=0/SUCCESS)
  Process: 21865 ExecStart=/usr/sbin/chronyd $OPTIONS (code=exited, status=0/SUCCESS)
 Main PID: 21867 (chronyd)
   CGroup: /system.slice/chronyd.service
           └─21867 /usr/sbin/chronyd

Aug 31 10:30:54 log systemd[1]: Starting NTP client/server...
Aug 31 10:30:54 log chronyd[21867]: chronyd version 3.4 starting (+CMDMON +NTP +...G)
Aug 31 10:30:54 log chronyd[21867]: Frequency 0.366 +/- 0.148 ppm read from /var...ft
Aug 31 10:30:54 log systemd[1]: Started NTP client/server.
Aug 31 10:30:59 log chronyd[21867]: Selected source 157.90.24.29
Hint: Some lines were ellipsized, use -l to show in full.
[root@log ~]#</pre>

<p>Далее проверим, что время и дата указаны правильно:</p>

<pre>[root@log ~]# date
Wed Aug 31 10:31:46 MSK 2022
[root@log ~]#</pre>

<h4>2. Установка nginx на виртуальной машине web</h4>

<p>Для установки nginx сначала нужно установить epel-release:</p>

<pre>[root@web ~]# yum install -y epel-release
...
Installed:
  epel-release.noarch 0:7-11

Complete!
[root@web ~]#
</pre>

<p>Установим nginx:</p>

<pre>[root@web ~]# yum install -y nginx
...
Installed:
  nginx.x86_64 1:1.20.1-9.el7

Dependency Installed:
  centos-indexhtml.noarch 0:7-9.el7.centos centos-logos.noarch 0:70.0.6-3.el7.centos
  gperftools-libs.x86_64 0:2.6.1-1.el7     nginx-filesystem.noarch 1:1.20.1-9.el7
  openssl11-libs.x86_64 1:1.1.1k-4.el7

Complete!
[root@web ~]#</pre>

<p>Запустим сервис nginx и включим его в автозапуск:</p>

<pre>[root@web ~]# systemctl enable nginx --now
Created symlink from /etc/systemd/system/multi-user.target.wants/nginx.service to /usr/lib/systemd/system/nginx.service.
[root@web ~]#</pre>

<p>Проверим, что nginx работает корректно:</p>

<pre>[root@web ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; vendor preset: disabled)
   Active: active (running) since Wed 2022-08-31 10:47:35 MSK; 1min 1s ago
  Process: 3497 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 3495 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 3494 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 3499 (nginx)
   CGroup: /system.slice/nginx.service
           ├─3499 nginx: master process /usr/sbin/nginx
           └─3500 nginx: worker process

Aug 31 10:47:35 web systemd[1]: Starting The nginx HTTP and reverse proxy server...
Aug 31 10:47:35 web nginx[3495]: nginx: the configuration file /etc/nginx/nginx... ok
Aug 31 10:47:35 web nginx[3495]: nginx: configuration file /etc/nginx/nginx.con...ful
Aug 31 10:47:35 web systemd[1]: Started The nginx HTTP and reverse proxy server.
Hint: Some lines were ellipsized, use -l to show in full.
[root@web ~]#</pre>

<p>Также работу nginx можно проверить на хосте с помощью команды curl:</p>

<pre>[root@web ~]# curl localhost
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN">
<html>
<head>
  <title>Welcome to CentOS</title>
  <style rel="stylesheet" type="text/css">

        html {
        background-image:url(img/html-background.png);
        background-color: white;
        font-family: "DejaVu Sans", "Liberation Sans", sans-serif;
        font-size: 0.85em;
        line-height: 1.25em;
        margin: 0 4% 0 4%;
        }

        body {
        border: 10px solid #fff;
        margin:0;
        padding:0;
        background: #fff;
        }

        /* Links */

        a:link { border-bottom: 1px dotted #ccc; text-decoration: none; color: #204d92; }
        a:hover { border-bottom:1px dotted #ccc; text-decoration: underline; color: green; }
        a:active {  border-bottom:1px dotted #ccc; text-decoration: underline; color: #204d92; }
        a:visited { border-bottom:1px dotted #ccc; text-decoration: none; color: #204d92; }
        a:visited:hover { border-bottom:1px dotted #ccc; text-decoration: underline; color: green; }

        .logo a:link,
        .logo a:hover,
        .logo a:visited { border-bottom: none; }

        .mainlinks a:link { border-bottom: 1px dotted #ddd; text-decoration: none; color: #eee; }
        .mainlinks a:hover { border-bottom:1px dotted #ddd; text-decoration: underline; color: white; }
        .mainlinks a:active { border-bottom:1px dotted #ddd; text-decoration: underline; color: white; }
        .mainlinks a:visited { border-bottom:1px dotted #ddd; text-decoration: none; color: white; }
        .mainlinks a:visited:hover { border-bottom:1px dotted #ddd; text-decoration: underline; color: white; }

        /* User interface styles */

        #header {
        margin:0;
        padding: 0.5em;
        background: #204D8C url(img/header-background.png);
        text-align: left;
        }

        .logo {
        padding: 0;
        /* For text only logo */
        font-size: 1.4em;
        line-height: 1em;
        font-weight: bold;
        }

        .logo img {
        vertical-align: middle;
        padding-right: 1em;
        }

        .logo a {
        color: #fff;
        text-decoration: none;
        }

        p {
        line-height:1.5em;
        }

        h1 {
                margin-bottom: 0;
                line-height: 1.9em; }
        h2 {
                margin-top: 0;
                line-height: 1.7em; }

        #content {
        clear:both;
        padding-left: 30px;
        padding-right: 30px;
        padding-bottom: 30px;
        border-bottom: 5px solid #eee;
        }

    .mainlinks {
        float: right;
        margin-top: 0.5em;
        text-align: right;
    }

    ul.mainlinks > li {
    border-right: 1px dotted #ddd;
    padding-right: 10px;
    padding-left: 10px;
    display: inline;
    list-style: none;
    }

    ul.mainlinks > li.last,
    ul.mainlinks > li.first {
    border-right: none;
    }

  </style>

</head>

<body>

<div id="header">

    <ul class="mainlinks">
        <li> <a href="http://www.centos.org/">Home</a> </li>
        <li> <a href="http://wiki.centos.org/">Wiki</a> </li>
        <li> <a href="http://wiki.centos.org/GettingHelp/ListInfo">Mailing Lists</a></li>
        <li> <a href="http://www.centos.org/download/mirrors/">Mirror List</a></li>
        <li> <a href="http://wiki.centos.org/irc">IRC</a></li>
        <li> <a href="https://www.centos.org/forums/">Forums</a></li>
        <li> <a href="http://bugs.centos.org/">Bugs</a> </li>
        <li class="last"> <a href="http://wiki.centos.org/Donate">Donate</a></li>
    </ul>

        <div class="logo">
                <a href="http://www.centos.org/"><img src="img/centos-logo.png" border="0"></a>
        </div>

</div>

<div id="content">

        <h1>Welcome to CentOS</h1>

        <h2>The Community ENTerprise Operating System</h2>

        <p><a href="http://www.centos.org/">CentOS</a> is an Enterprise-class Linux Distribution derived from sources freely provided
to the public by Red Hat, Inc. for Red Hat Enterprise Linux.  CentOS conforms fully with the upstream vendors
redistribution policy and aims to be functionally compatible. (CentOS mainly changes packages to remove upstream vendor
branding and artwork.)</p>

        <p>CentOS is developed by a small but growing team of core
developers.&nbsp; In turn the core developers are supported by an active user community
including system administrators, network administrators, enterprise users, managers, core Linux contributors and Linux enthusiasts from around the world.</p>

        <p>CentOS has numerous advantages including: an active and growing user community, quickly rebuilt, tested, and QA'ed errata packages, an extensive <a href="http://www.centos.org/download/mirrors/">mirror network</a>, developers who are contactable and responsive, Special Interest Groups (<a href="http://wiki.centos.org/SpecialInterestGroup/">SIGs</a>) to add functionality to the core CentOS distribution, and multiple community support avenues including a <a href="http://wiki.centos.org/">wiki</a>, <a
href="http://wiki.centos.org/irc">IRC Chat</a>, <a href="http://wiki.centos.org/GettingHelp/ListInfo">Email Lists</a>, <a href="https://www.centos.org/forums/">Forums</a>, <a href="http://bugs.centos.org/">Bugs Database</a>, and an <a
href="http://wiki.centos.org/FAQ/">FAQ</a>.</p>

        </div>

</div>


</body>
</html>
[root@web ~]#</pre>

<p>Наблюдаем, что nginx запустился и работает корректно.</p>

<h4>3. Настройка центрального сервера сбора логов</h4>

<p>rsyslog должен быть установлен по умолчанию в нашёй ОС, проверим это:</p>

<pre>[root@log ~]# yum list rsyslog
Loaded plugins: fastestmirror
Determining fastest mirrors
 * base: mirror.23m.com
 * extras: ftp.hosteurope.de
 * updates: mirror.imt-systems.com
base                                                          | 3.6 kB  00:00:00
extras                                                        | 2.9 kB  00:00:00
updates                                                       | 2.9 kB  00:00:00
(1/4): extras/7/x86_64/primary_db                             | 247 kB  00:00:00
(2/4): base/7/x86_64/group_gz                                 | 153 kB  00:00:00
(3/4): base/7/x86_64/primary_db                               | 6.1 MB  00:00:01
(4/4): updates/7/x86_64/primary_db                            |  17 MB  00:00:02
Installed Packages
rsyslog.x86_64                      8.24.0-52.el7                           @anaconda
Available Packages
rsyslog.x86_64                      8.24.0-57.el7_9.3                       updates
[root@log ~]#</pre>

<p>Все настройки Rsyslog хранятся в файле /etc/rsyslog.conf</p>

<p>Для того, чтобы наш сервер мог принимать логи, нам необходимо внести следующие изменения в файл:</p>

<pre>[root@log ~]# vi /etc/rsyslog.conf</pre>

<p>Открываем порт 514 (TCP и UDP):</p>

<p>Находим закомментированные строки:</p>

<pre>...
# Provides UDP syslog reception
#$ModLoad imudp
#$UDPServerRun 514

# Provides TCP syslog reception
#$ModLoad imtcp
#$InputTCPServerRun 514
...</pre>

<p>Раскомментируем нужные строки и приводим к виду:</p>

<pre>...
# Provides UDP syslog reception
$ModLoad imudp
$UDPServerRun 514

# Provides TCP syslog reception
$ModLoad imtcp
$InputTCPServerRun 514
...</pre>

<p>В конец файла /etc/rsyslog.conf добавляем правила приёма сообщений от хостов:</p>

<pre># Add remote logs
$template RemoteLogs,"/var/log/rsyslog/%HOSTNAME%/%PROGRAMNAME%.log"
*.* ?RemoteLogs
& ~</pre>

<p>Данные параметры будут отправлять в папку /var/log/rsyslog логи, которые будут приходить от других серверов. Например, Access-логи nginx от сервера web, будут идти в файл /var/log/rsyslog/web/nginx_access.log<br />
Далее сохраняем файл и перезапускаем службу rsyslog:</p>

<pre>[root@log ~]# systemctl restart rsyslog
[root@log ~]#</pre>

<p>Если ошибок не допущено, то у нас будут видны открытые порты TCP,UDP 514:</p>

<pre>[root@log ~]# ss -tuln
Netid State      Recv-Q Send-Q Local Address:Port               Peer Address:Port    
udp   UNCONN     0      0            *:111                      *:*
udp   UNCONN     0      0            *:928                      *:*
udp   UNCONN     0      0            *:514                      *:*
udp   UNCONN     0      0      127.0.0.1:323                      *:*                
udp   UNCONN     0      0            *:68                       *:*
udp   UNCONN     0      0         [::]:111                   [::]:*
udp   UNCONN     0      0         [::]:928                   [::]:*
udp   UNCONN     0      0         [::]:514                   [::]:*
udp   UNCONN     0      0        [::1]:323                   [::]:*
tcp   LISTEN     0      25           *:514                      *:*
tcp   LISTEN     0      128          *:111                      *:*
tcp   LISTEN     0      128          *:22                       *:*
tcp   LISTEN     0      100    127.0.0.1:25                       *:*                
tcp   LISTEN     0      25        [::]:514                   [::]:*
tcp   LISTEN     0      128       [::]:111                   [::]:*
tcp   LISTEN     0      128       [::]:22                    [::]:*
tcp   LISTEN     0      100      [::1]:25                    [::]:*
[root@log ~]#</pre>

<h4>Далее настроим отправку логов с web-сервера</h4>

<p>На сервере web проверим версию nginx:</p>

<pre>[root@web ~]# rpm -qa | grep nginx
nginx-filesystem-1.20.1-9.el7.noarch
nginx-1.20.1-9.el7.x86_64
[root@web ~]# nginx -version
nginx version: nginx/1.20.1
[root@web ~]#</pre>

<p>Версия nginx должна быть 1.7 или выше. В нашем примере используется версия nginx 1.20.</p>
<p>Находим в файле /etc/nginx/nginx.conf раздел с логами и приводим их к следующему виду:</p>

<pre>[root@web ~]# vi /etc/nginx/nginx.conf</pre>

<pre>...
error_log /var/log/nginx/error.log;
error_log syslog:server=192.168.50.15:514,tag=nginx_error;
...
    access_log  /var/log/nginx/access.log  main;
    access_log syslog:server=192.168.50.15:514,tag=nginx_access,severity=info combined;
...</pre>

<p>Для Access-логов указыаем удаленный сервер и уровень логов, которые нужно отправлять. Для error_log добавляем удаленный сервер. Если требуется чтобы логи хранились локально и отправлялись на удаленный сервер, требуется указать 2 строки.</p>
<p>Tag нужен для того, чтобы логи записывались в разные файлы.</p>
<p>По умолчанию, error-логи отправляют логи, которые имеют severity: error, crit, alert и emerg. Если трубуется хранили или пересылать логи с другим severity, то это также можно указать в настройках nginx.</p>
<p>Далее проверяем, что конфигурация nginx указана правильно:</p>

<pre>[root@web ~]# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
[root@web ~]#</pre>

<p>Перезапустим nginx сервис: </p>

<pre>[root@web ~]# systemctl restart nginx
[root@web ~]#</pre>

<p>Чтобы проверить, что логи ошибок также улетают на удаленный сервер, можно удалить картинку, к которой будет обращаться nginx во время открытия веб-сраницы:</p>

<pre>[root@web ~]# rm -f /usr/share/nginx/html/img/header-background.png
[root@web ~]#</pre>

<p>Попробуем несколько раз зайти по адресу http://192.168.50.10:</p>

<pre>[root@web ~]# curl 192.168.50.10
...</pre>

<pre>[root@web ~]# curl 192.168.50.10
...</pre>

<pre>[root@web ~]# curl 192.168.50.10
...</pre>

<p>Далее заходим на сервер log и смотрим информацию об nginx:</p>

<pre>[root@log ~]# cat /var/log/rsyslog/web/nginx_access.log
Aug 31 12:11:10 web nginx_access: 192.168.50.10 - - [31/Aug/2022:12:11:10 +0300] "GET / HTTP/1.1" 200 4833 "-" "curl/7.29.0"
Aug 31 12:11:15 web nginx_access: 192.168.50.10 - - [31/Aug/2022:12:11:15 +0300] "GET / HTTP/1.1" 200 4833 "-" "curl/7.29.0"
Aug 31 12:11:19 web nginx_access: 192.168.50.10 - - [31/Aug/2022:12:11:19 +0300] "GET / HTTP/1.1" 200 4833 "-" "curl/7.29.0"

Aug 31 12:23:39 web nginx_access: 192.168.50.10 - - [31/Aug/2022:12:23:39 +0300] "GET / HTTP/1.0" 200 4833 "-" "Lynx/2.8.8dev.15 libwww-FM/2.14 SSL-MM/1.4.1 OpenSSL/1.0.1e-fips"
Aug 31 12:24:09 web nginx_access: 192.168.50.10 - - [31/Aug/2022:12:24:09 +0300] "GET / HTTP/1.0" 200 4833 "-" "Lynx/2.8.8dev.15 libwww-FM/2.14 SSL-MM/1.4.1 OpenSSL/1.0.1e-fips"
Aug 31 12:24:16 web nginx_access: 192.168.50.10 - - [31/Aug/2022:12:24:16 +0300] "GET / HTTP/1.0" 200 4833 "-" "Lynx/2.8.8dev.15 libwww-FM/2.14 SSL-MM/1.4.1 OpenSSL/1.0.1e-fips"

[root@log ~]#</pre>






















