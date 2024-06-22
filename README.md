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
1. Исходя из условия задачи логичным выбром PAM-аутентификации выглядит использования модуля pam_time. Однако данный модуль не работает с локальными группами пользователей. Для предлагаемой задачи более подходящем решением является подготовка небольшого скрипта и использование модуля pam_exec.<br/>
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
