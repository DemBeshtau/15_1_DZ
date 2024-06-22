# Практика с Pluggable Authentication Modules (PAM)
1. Запретить всем пользователям кроме группы admin логин в выходные (суббота и воскресенье) без учёта праздников.<br/>
### Исходные данные ###
&ensp;&ensp;ПК на Linux c 8 ГБ ОЗУ или виртуальная машина (ВМ) с включенной Nested Virtualization.<br/>
&ensp;&ensp;Предварительно установленное и настроенное ПО:<br/>
&ensp;&ensp;&ensp;Hashicorp Vagrant (https://www.vagrantup.com/downloads);<br/>
&ensp;&ensp;&ensp;Oracle VirtualBox (https://www.virtualbox.org/wiki/Linux_Downloads).<br/>
&ensp;&ensp;&ensp;Ansible (https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html).<br/>
&ensp;&ensp;Все действия проводились с использованием Vagrant 2.4.0, VirtualBox 7.0.14, Ansible 9.3.0 и образа ubuntu/jammy64 версии 20240301.0.0.<br/> 
### Ход решения ###
1. Исходя из условия задачи логичным выбром PAM-аутентификации выглядит использование модуля pam_time. Однако данный модуль не работает с локальными группами пользователей. Для предлагаемой задачи более подходящим решением является подготовка небольшого скрипта и использование модуля pam_exec.<br/>
&ensp;&ensp;Скрипт должен работать по следующему принципу: если сегодня суббота или воскресенье, то нужно проверить, входит ли пользователь в группу admin, если не входит, то подключение запрещено. При любых других вариантах подключение разрешено. <br/>
&ensp;&ensp;Скрипт располагается в /usr/local/bin.
```shell
#!/bin/bash
# Проверка наличия выходного дня
if [[ ($(date +%a) = "Sat") || ($(date +%a) = "Sun") ]]
then
        # Проверка вхождения пользователя в группу admin
        if getent group admin | grep -qw "$PAM_USER" 
        then
                # При соответствии условию, подключение возможно
                exit 0
        
        else
                # Иначе, подключение не возможно
                exit 1
        fi
# Если день не выходной, то подключиться может любой пользователь
else
        exit 0
fi
```
2. Создание пользователей otusadm и otus и установка им паролей:
```shell
root@pam:~# useradd otusadm && sudo useradd otus
root@pam:~# echo 'otusadm:Otus2022!' | sudo chpasswd && echo 'otus:Otus2022!' | sudo chpasswd
```
3. Создание группы admin и добавление в неё пользователей vagrant, root, otusadm:
```shell
root@pam:~# groupadd -f admin
root@pam:~# usermod otusadm -aG admin && usermod root -aG admin && usermod vagrant -aG admin
```
4. Проверка наличия пользователей vagrant, root, otusadm в группе admin:
```shel
root@pam:~# cat /etc/group | grep admin
admin:x:118:otusadm,root,vagrant
```
5. Добавление модуля pam_exec и скрипта login.sh в PAM конфигурацию сервиса SSH:
```shell
root@pam:~# nano /etc/pam.d/sshd
...
# Disallow non-root logins when /etc/nologin exists.
account    required     pam_nologin.so
auth       required     pam_exec.so /usr/local/bin/login.sh
...
```
6. Проверка работы полученной конфигурации PAM (в субботу):
```shell
root@pam:~# date
Sat Jun 22 12:13:02 UTC 2024

dem@calculate ~ $ ssh otus@192.168.57.10
The authenticity of host '192.168.57.10 (192.168.57.10)' can't be established.
ED25519 key fingerprint is SHA256:qP41PPCytFAwCZ/hhE8bK61PqmdZ5/MaEFVLb2rrF9Q.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.57.10' (ED25519) to the list of known hosts.
otus@192.168.57.10's password: 
Permission denied, please try again.

dem@calculate ~ $ ssh otusadm@192.168.57.10
otusadm@192.168.57.10's password: 
Welcome to Ubuntu 22.04.4 LTS (GNU/Linux 5.15.0-97-generic x86_64)
... 
otusadm@pam:~$ 

dem@calculate ~ $ ssh vagrant@192.168.57.10
vagrant@192.168.57.10's password: 
Welcome to Ubuntu 22.04.4 LTS (GNU/Linux 5.15.0-97-generic x86_64)
...
vagrant@pam:~$ 
```
&ensp;&ensp;Окончательная конфигурация системы, в соответствии с условиями задания, производится с помощью Ansible. Плейбук и необходимые файлы находятся в директории provisioning.
