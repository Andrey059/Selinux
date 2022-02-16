# Описание домашнего задания
1. Запустить nginx на нестандартном порту 3-мя разными способами:  
- переключатели setsebool;  
- добавление нестандартного порта в имеющийся тип;  
- формирование и установка модуля SELinux.  
К сдаче:  
README с описанием каждого решения
(скриншоты и демонстрация приветствуются).
2. Обеспечить работоспособность приложения при включенном selinux.
- развернуть приложенный стенд
https://github.com/mbfx/otus-linux-adm/tree/master/selinux_dns_problems;
- выяснить причину неработоспособности механизма обновления зоны
(см. README);
- предложить решение (или решения) для данной проблемы;
- выбрать одно из решений для реализации, предварительно обосновав
выбор;
- реализовать выбранное решение и продемонстрировать его
работоспособность.


### 1. Создаём виртуальную машину
Запускаем Vagrantfile приложенный в методичке к выполнению ДЗ.  
Результатом выполнения команды vagrant up станет созданная виртуальная
машина с установленным nginx, который работает на порту TCP 4881. 
Порт TCP 4881 уже проброшен до хоста. SELinux включен.  
Установим semanage с помощью команды ``` yum install policycoreutils-python -y ```
### 2. Запуск nginx на нестандартном порту 3-мя разными способами
Для начала проверим, что в ОС отключен файервол: ``` systemctl status firewalld ```
```
[root@selinux ~]# systemctl status firewalld
● firewalld.service - firewalld - dynamic firewall daemon
   Loaded: loaded (/usr/lib/systemd/system/firewalld.service; disabled; vendor preset: enabled)
   Active: inactive (dead)
     Docs: man:firewalld(1)
```
Также можно проверить, что конфигурация nginx настроена без ошибок:  
```
[root@selinux ~]# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```
Далее проверим режим работы SELinux: getenforce
```
[root@selinux ~]# getenforce
Enforcing
```
Данный режим означает, что SELinuxбудет блокировать запрещенную активность.  
Разрешим в SELinux работу nginx на порту TCP 4881 c помощью переключателей setsebool  
Находим в логах (/var/log/audit/audit.log) информацию о блокировании порта.  
Копируем время, в которое был записан этот лог, и, с помощью утилиты audit2why смотрим информации о запрете: grep 1645009202.190:819
/var/log/audit/audit.log | audit2why
```
[root@selinux ~]# grep 1645009202.190:819 /var/log/audit/audit.log | audit2why
type=AVC msg=audit(1645009202.190:819): avc:  denied  { name_bind } for  pid=2798 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0

	Was caused by:
	The boolean nis_enabled was set incorrectly. 
	Description:
	Allow nis to enabled

	Allow access by executing:
	# setsebool -P nis_enabled 1
```
Исходя из вывода утилиты, мы видим, что нам нужно поменять параметр nis_enabled.  
Включим параметр nis_enabled и перезапустим nginx: setsebool -P nis_enabled on  
```
[root@selinux ~]# setsebool -P nis_enabled on
[root@selinux ~]# systemctl restart nginx
[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Wed 2022-02-16 11:25:32 UTC; 11s ago
  Process: 21746 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 21744 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 21742 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 21748 (nginx)
   CGroup: /system.slice/nginx.service
           ├─21748 nginx: master process /usr/sbin/nginx
           └─21749 nginx: worker process

Feb 16 11:25:32 selinux systemd[1]: Starting The nginx HTTP and reverse pro.....
Feb 16 11:25:32 selinux nginx[21744]: nginx: the configuration file /etc/ng...ok
Feb 16 11:25:32 selinux nginx[21744]: nginx: configuration file /etc/nginx/...ul
Feb 16 11:25:32 selinux systemd[1]: Started The nginx HTTP and reverse prox...r.
Hint: Some lines were ellipsized, use -l to show in full.
```


Проверить статус параметра можно с помощью команды: getsebool -a | grep nis_enabled
```
[root@selinux ~]# getsebool -a | grep nis_enabled
nis_enabled --> on
```
Вернём запрет работы nginx на порту 4881 обратно. Для этого отключим
```
[root@selinux ~]# setsebool -P nis_enabled off
```
После отключения nis_enabled служба nginx снова не запустится.

Теперь разрешим в SELinux работу nginx на порту TCP 4881 c помощью добавления нестандартного порта в имеющийся тип:
Поиск имеющегося типа, для http трафика: 
```
[root@selinux ~]# semanage port -l | grep http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989

```
Добавим порт в тип http_port_t: 
```
[root@selinux ~]# semanage port -a -t http_port_t -p tcp 4881
[root@selinux ~]# semanage port -l | grep http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      4881, 80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989

```

Теперь перезапустим службу nginx и проверим её работу: 
```
[root@selinux ~]# systemctl restart nginx
[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Wed 2022-02-16 11:34:16 UTC; 3s ago
  Process: 21794 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 21792 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 21790 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 21796 (nginx)
   CGroup: /system.slice/nginx.service
           ├─21796 nginx: master process /usr/sbin/nginx
           └─21797 nginx: worker process

Feb 16 11:34:16 selinux systemd[1]: Starting The nginx HTTP and reverse pro.....
Feb 16 11:34:16 selinux nginx[21792]: nginx: the configuration file /etc/ng...ok
Feb 16 11:34:16 selinux nginx[21792]: nginx: configuration file /etc/nginx/...ul
Feb 16 11:34:16 selinux systemd[1]: Started The nginx HTTP and reverse prox...r.
Hint: Some lines were ellipsized, use -l to show in full.

```
Удалить нестандартный порт из имеющегося типа можно с помощью команды: 
```
[root@selinux ~]# semanage port -d -t http_port_t -p tcp 4881
[root@selinux ~]# systemctl restart nginx
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
```
Разрешим в SELinux работу nginx на порту TCP 4881 c помощью формирования
и установки модуля SELinux:  
Попробуем снова запустить nginx: systemctl start nginx  
```
[root@selinux ~]# systemctl start nginx
Job for nginx.service failed because the control process exited with
error code. See "systemctl status nginx.service" and "journalctl -xe"
for details.
```
Nginx не запуститься, так как SELinux продолжает его блокировать.  
Посмотрим логи SELinux, которые относятся к nginx:
```
[root@selinux ~]# grep nginx /var/log/audit/audit.log
type=SOFTWARE_UPDATE msg=audit(1645009198.872:817): pid=2682 uid=0 auid=1000 ses=2 subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 msg='sw="nginx-filesystem-1:1.20.1-9.el7.noarch" sw_type=rpm key_enforce=0 gpg_res=1 root_dir="/" comm="yum" exe="/usr/bin/python2.7" hostname=? addr=? terminal=? res=success'
type=SOFTWARE_UPDATE msg=audit(1645009200.307:818): pid=2682 uid=0 auid=1000 ses=2 subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 msg='sw="nginx-1:1.20.1-9.el7.x86_64" sw_type=rpm key_enforce=0 gpg_res=1 root_dir="/" comm="yum" exe="/usr/bin/python2.7" hostname=? addr=? terminal=? res=success'
type=AVC msg=audit(1645009202.190:819): avc:  denied  { name_bind } for  pid=2798 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
type=SYSCALL msg=audit(1645009202.190:819): arch=c000003e syscall=49 success=no exit=-13 a0=7 a1=55c4e85767d8 a2=1c a3=7ffcd6faf134 items=0 ppid=1 pid=2798 auid=4294967295 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=4294967295 comm="nginx" exe="/usr/sbin/nginx" subj=system_u:system_r:httpd_t:s0 key=(null)
type=SERVICE_START msg=audit(1645009202.200:820): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=nginx comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=failed'
type=AVC msg=audit(1645009712.021:864): avc:  denied  { name_bind } for  pid=2955 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
type=SYSCALL msg=audit(1645009712.021:864): arch=c000003e syscall=49 success=no exit=-13 a0=7 a1=55fc126c57d8 a2=1c a3=7ffd02da3da4 items=0 ppid=1 pid=2955 auid=4294967295 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=4294967295 comm="nginx" exe="/usr/sbin/nginx" subj=system_u:system_r:httpd_t:s0 key=(null)
type=SERVICE_START msg=audit(1645009712.031:865): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=nginx comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=failed'
type=SERVICE_START msg=audit(1645010732.271:881): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=nginx comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=success'
type=SERVICE_STOP msg=audit(1645011112.601:886): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=nginx comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=success'
type=AVC msg=audit(1645011112.682:887): avc:  denied  { name_bind } for  pid=21769 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
type=SYSCALL msg=audit(1645011112.682:887): arch=c000003e syscall=49 success=no exit=-13 a0=7 a1=55cdca1bd7d8 a2=1c a3=7ffe304a5d34 items=0 ppid=1 pid=21769 auid=4294967295 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=4294967295 comm="nginx" exe="/usr/sbin/nginx" subj=system_u:system_r:httpd_t:s0 key=(null)
type=SERVICE_START msg=audit(1645011112.682:888): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=nginx comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=failed'
type=SERVICE_START msg=audit(1645011256.722:892): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=nginx comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=success'
type=SERVICE_STOP msg=audit(1645011300.292:896): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=nginx comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=success'
type=AVC msg=audit(1645011300.362:897): avc:  denied  { name_bind } for  pid=21816 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
type=SYSCALL msg=audit(1645011300.362:897): arch=c000003e syscall=49 success=no exit=-13 a0=7 a1=55f5092137d8 a2=1c a3=7ffcedd02cd4 items=0 ppid=1 pid=21816 auid=4294967295 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=4294967295 comm="nginx" exe="/usr/sbin/nginx" subj=system_u:system_r:httpd_t:s0 key=(null)
type=SERVICE_START msg=audit(1645011300.362:898): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=nginx comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=failed'
type=AVC msg=audit(1645011330.972:899): avc:  denied  { name_bind } for  pid=21827 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
type=SYSCALL msg=audit(1645011330.972:899): arch=c000003e syscall=49 success=no exit=-13 a0=7 a1=5626830667d8 a2=1c a3=7fffbed33e44 items=0 ppid=1 pid=21827 auid=4294967295 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=4294967295 comm="nginx" exe="/usr/sbin/nginx" subj=system_u:system_r:httpd_t:s0 key=(null)
type=SERVICE_START msg=audit(1645011330.982:900): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=nginx comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=failed'
```
Воспользуемся утилитой audit2allow для того, чтобы на основе логов
SELinux сделать модуль, разрешающий работу nginx на нестандартном
порту:
```
[root@selinux ~]# grep nginx /var/log/audit/audit.log | audit2allow -M nginx
******************** IMPORTANT ***********************
To make this policy package active, execute:

semodule -i nginx.pp
```
Audit2allow сформировал модуль, и сообщил нам команду, с помощью
которой можно применить данный модуль: semodule -i nginx.pp
```
[root@selinux ~]# semodule -i nginx.pp
[root@selinux ~]# systemctl start nginx
[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Wed 2022-02-16 11:38:39 UTC; 4s ago
  Process: 21855 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 21853 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 21851 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 21857 (nginx)
   CGroup: /system.slice/nginx.service
           ├─21857 nginx: master process /usr/sbin/nginx
           └─21858 nginx: worker process

Feb 16 11:38:39 selinux systemd[1]: Starting The nginx HTTP and reverse pro.....
Feb 16 11:38:39 selinux nginx[21853]: nginx: the configuration file /etc/ng...ok
Feb 16 11:38:39 selinux nginx[21853]: nginx: configuration file /etc/nginx/...ul
Feb 16 11:38:39 selinux systemd[1]: Started The nginx HTTP and reverse prox...r.
Hint: Some lines were ellipsized, use -l to show in full.
```
После добавления модуля nginx запустился без ошибок. При использовании
модуля изменения сохранятся после перезагрузки.  
Просмотр всех установленных модулей: semodule -l  
Для удаления модуля воспользуемся командой: semodule -r nginx
```
[root@selinux ~]# semodule -r nginx
libsemanage.semanage_direct_remove_key: Removing last nginx module (no
other nginx module exists at another priority).
```

