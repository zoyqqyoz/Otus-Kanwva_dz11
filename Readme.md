Домашнее задание

Подготовить стенд на Vagrant как минимум с одним сервером. На этом сервере, используя Ansible, необходимо развернуть nginx со следующими условиями:
- необходимо использовать модуль yum/apt
- конфигурационные файлы должны быть взяты из шаблона jinja2 с переменными
- после установки nginx должен быть в режиме enabled в systemd
- должен быть использован notify длā старта nginx после установки
- сайт должен слушать на нестандартном порту - 8080, для этого использовать переменные в Ansible
* Сделать все это с использованием Ansible роли

1. Проверяем, что у нас установлены Ansible и python:

```
neva@Uneva:~/Ansible$ ansible --version
ansible [core 2.12.10]
  config file = /etc/ansible/ansible.cfg
  configured module search path = ['/home/neva/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/lib/python3/dist-packages/ansible
  ansible collection location = /home/neva/.ansible/collections:/usr/share/ansible/collections
  executable location = /usr/bin/ansible
  python version = 3.10.6 (main, May 29 2023, 11:10:38) [GCC 11.3.0]
  jinja version = 3.0.3
  libyaml = True
```

2. Создаём директорию Ansible и складываем в неё предложенный в методичке Vagrantfile. Стартуем ВМ, проверяем, что мы можем к ней подключиться:

```
neva@Uneva:~/Ansible$ vagrant up
Bringing machine 'nginx' up with 'virtualbox' provider...
==> nginx: Checking if box 'centos/7' version '2004.01' is up to date...
==> nginx: Clearing any previously set network interfaces...
==> nginx: Preparing network interfaces based on configuration...
    nginx: Adapter 1: nat
    nginx: Adapter 2: hostonly
==> nginx: Forwarding ports...
    nginx: 22 (guest) => 2222 (host) (adapter 1)
==> nginx: Running 'pre-boot' VM customizations...
==> nginx: Booting VM...
==> nginx: Waiting for machine to boot. This may take a few minutes...
    nginx: SSH address: 127.0.0.1:2222
    nginx: SSH username: vagrant
    nginx: SSH auth method: private key
    nginx:
    nginx: Vagrant insecure key detected. Vagrant will automatically replace
    nginx: this with a newly generated keypair for better security.
    nginx:
    nginx: Inserting generated public key within guest...
    nginx: Removing insecure key from the guest if it's present...
    nginx: Key inserted! Disconnecting and reconnecting using new SSH key...
==> nginx: Machine booted and ready!
==> nginx: Checking for guest additions in VM...
    nginx: No guest additions were detected on the base box for this VM! Guest
    nginx: additions are required for forwarded ports, shared folders, host only
    nginx: networking, and more. If SSH fails on this machine, please install
    nginx: the guest additions and repackage the box to continue.
    nginx:
    nginx: This is not an error message; everything may continue to work properly,
    nginx: in which case you may ignore this message.
==> nginx: Setting hostname...
==> nginx: Configuring and enabling network interfaces...
==> nginx: Running provisioner: shell...
    nginx: Running: inline script
neva@Uneva:~/Ansible$ vagrant ssh
[vagrant@nginx ~]$
```

3. Для подключения к хосту nginx нам необходимо будет передать множество параметров - это особенность Vagrant. Узнать эти параметры можно с помощью команды vagrant ssh-config:

```
neva@Uneva:~/Ansible$ vagrant ssh-config
Host nginx
  HostName 127.0.0.1
  User vagrant
  Port 2222
  UserKnownHostsFile /dev/null
  StrictHostKeyChecking no
  PasswordAuthentication no
  IdentityFile /home/neva/Ansible/.vagrant/machines/nginx/virtualbox/private_key
  IdentitiesOnly yes
  LogLevel FATAL
```

4. Используя эти параметры, создадим свой первый inventory файл с именем hosts в директории пользователя /home/neva/Ansible/staging/ со следующими параметрами:
nginx ansible_host=127.0.0.1 ansible_port=2222 ansible_user=vagrant ansible_private_key_file=.vagrant/machines/nginx/virtualbox/private_key
Теперь можно проверить, что Ansible может управлять нашим хостом:

```
neva@Uneva:~/Ansible$ ansible nginx -i staging/hosts -m ping
The authenticity of host '[127.0.0.1]:2222 ([127.0.0.1]:2222)' can't be established.
ED25519 key fingerprint is SHA256:ntf3dM2KPH+90RrJBuEeymgRnoWx/FrAb7NUgrahzGY.
This key is not known by any other names
Are you sure you want to continue connecting (yes/no/[fingerprint])? y
Please type 'yes', 'no' or the fingerprint: yes
nginx | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": false,
    "ping": "pong"
}
```

5.  Как видимо, нам придется каждый раз явно указывать наш инвентори файл и вписывать в него много информации. Это можно обойти, используя ansible.cfg файл - прописав конфигурацию в нем.
 Для этого в текущем каталоге создадим файл ansible.cfg со следующим
содержанием:

```
[defaults]
inventory = staging/hosts
remote_user = vagrant
host_key_checking = False
retry_files_enabled = False
```
Теперь из hosts можно убрать информацию о пользователе: ansible_user=vagrant. Получится так:

```
nginx ansible_host=127.0.0.1 ansible_port=2222 ansible_private_key_file=.vagrant/machines/nginx/virtualbox/private_key
```

Еще раз убедимся, что управляемый хост в доступе, только теперь без явного указания inventory файла:

```
neva@Uneva:~/Ansible$ ansible nginx -m ping
nginx | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": false,
    "ping": "pong"
```

6. Теперь  мы можем конфигурировать наш хост. Для начала воспользуемся Ad-Hoc командами и выполним некоторые удаленные команды на нашем хосте.

Посмотрим, какое ядро установлено на хосте:

```
neva@Uneva:~/Ansible$ ansible nginx -m command -a "uname -r"
nginx | CHANGED | rc=0 >>
3.10.0-1127.el7.x86_64
```

Проверим статус сервиса firewalld:

```
neva@Uneva:~/Ansible$ ansible nginx -m systemd -a name=firewalld
nginx | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": false,
    "name": "firewalld",
    "status": {
        "ActiveEnterTimestampMonotonic": "0",
        "ActiveExitTimestampMonotonic": "0",
        "ActiveState": "inactive",
```

Видим, что неактивен: "ActiveState": "inactive"
7.  Установим пакет epel-release на наш хост:

```
neva@Uneva:~/Ansible$ ansible nginx -m yum -a "name=epel-release state=present" -b
nginx | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": true,
    "changes": {
        "installed": [
            "epel-release"
        ]
    },
    "msg": "warning: /var/cache/yum/x86_64/7/extras/packages/epel-release-7-11.noarch.rpm: Header V3 RSA/SHA256 Signature, key ID f4a80eb5: NOKEY\nImporting GPG key 0xF4A80EB5:\n Userid     : \"CentOS-7 Key (CentOS 7 Official Signing Key) <security@centos.org>\"\n Fingerprint: 6341 ab27 53d7 8a78 a7c2 7bb1 24c6 a8a7 f4a8 0eb5\n Package    : centos-release-7-8.2003.0.el7.centos.x86_64 (@anaconda)\n From       : /etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7\n",
    "rc": 0,
    "results": [
        "Loaded plugins: fastestmirror\nDetermining fastest mirrors\n * base: mirror.fra1.de.leaseweb.net\n * extras: ftp.hosteurope.de\n * updates: mirror.imt-systems.com\nResolving Dependencies\n--> Running transaction check\n---> Package epel-release.noarch 0:7-11 will be installed\n--> Finished Dependency Resolution\n\nDependencies Resolved\n\n================================================================================\n Package                Arch             Version         Repository        Size\n================================================================================\nInstalling:\n epel-release           noarch           7-11            extras            15 k\n\nTransaction Summary\n================================================================================\nInstall  1 Package\n\nTotal download size: 15 k\nInstalled size: 24 k\nDownloading packages:\nPublic key for epel-release-7-11.noarch.rpm is not installed\nRetrieving key from file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7\nRunning transaction check\nRunning transaction test\nTransaction test succeeded\nRunning transaction\n  Installing : epel-release-7-11.noarch                                     1/1 \n  Verifying  : epel-release-7-11.noarch                                     1/1 \n\nInstalled:\n  epel-release.noarch 0:7-11                                                    \n\nComplete!\n"
    ]
}
```

По строке  "changed": true видим, что пакет установился.

8. Напишем простой Playbook, который будет установку пакета epel-release. Создадим файл epel.yml со следующим:

```
---
- name: Install EPEL Repo
  hosts: nginx
  become: true
  tasks:
    - name: Install EPEL Repo package from standart repo
      yum:
        name: epel-release
        state: present
```

9. Запустим выполнение Playbook:

```
neva@Uneva:~/Ansible$ ansible-playbook epel.yml

PLAY [Install EPEL Repo] ********************************************************************************************************************************************************************************************************************

TASK [Gathering Facts] **********************************************************************************************************************************************************************************************************************
ok: [nginx]

TASK [Install EPEL Repo package from standart repo] *****************************************************************************************************************************************************************************************
ok: [nginx]

PLAY RECAP **********************************************************************************************************************************************************************************************************************************
nginx                      : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

10. Выполним команду: ansible nginx -m yum -a "name=epel-release state=absent" -b и увидим, что пакет epel-release был удалён:

```
nginx | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": true,
    "changes": {
        "removed": [
            "epel-release"
        ]
    },
    "msg": "",
    "rc": 0,
    "results": [
        "Loaded plugins: fastestmirror\nResolving Dependencies\n--> Running transaction check\n---> Package epel-release.noarch 0:7-11 will be erased\n--> Finished Dependency Resolution\n\nDependencies Resolved\n\n================================================================================\n Package                Arch             Version        Repository         Size\n================================================================================\nRemoving:\n epel-release           noarch           7-11           @extras            24 k\n\nTransaction Summary\n================================================================================\nRemove  1 Package\n\nInstalled size: 24 k\nDownloading packages:\nRunning transaction check\nRunning transaction test\nTransaction test succeeded\nRunning transaction\n  Erasing    : epel-release-7-11.noarch                                     1/1 \n  Verifying  : epel-release-7-11.noarch                                     1/1 \n\nRemoved:\n  epel-release.noarch 0:7-11                                                    \n\nComplete!\n"
    ]
}
```

11. Теперь приступим к написанию Playbook-а длā установки NGINX. Будем писать его постепенно, шаг за шагом. И в итоге трансформируем его в роль.

Переименуем epel.yml в nginx.yml. Добавим в него установку пакета:

```
---
- name: NGINX | Install and configure NGINX
  hosts: nginx
  become: true
  
  tasks:
    - name: NGINX | Install EPEL Repo package from standart repo
      yum:
        name: epel-release
        state: present
      tags:
        - epel-package
        - packages

    - name: NGINX | Install NGINX package from EPEL Repo
      yum:
        name: nginx
        state: latest
      tags:
        - nginx-package
        - packages
```

С помощью tags можно вывести в консоль список тегов и выполнить, например, только установку NGINX. В нашем случае так, например, можно осуществлять его обновление.
Выведем в консоль все теги:

```
neva@Uneva:~/Ansible$ ansible-playbook nginx.yml --list-tags

playbook: nginx.yml

  play #1 (nginx): NGINX | Install and configure NGINX  TAGS: []
      TASK TAGS: [epel-package, nginx-package, packages]
```

Запустим только установку NGINX, но предварительно нам надо обратано установить epel-release, который мы удалили в предыдущих шагах:


```
ansible nginx -m yum -a "name=epel-release state=present" -b
neva@Uneva:~/Ansible$ ansible-playbook nginx.yml -t nginx-package

PLAY [NGINX | Install and configure NGINX] **************************************************************************************************************************************************************************************************

TASK [Gathering Facts] **********************************************************************************************************************************************************************************************************************
ok: [nginx]

TASK [NGINX | Install NGINX package from EPEL Repo] *****************************************************************************************************************************************************************************************
changed: [nginx]

PLAY RECAP **********************************************************************************************************************************************************************************************************************************
nginx                      : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

Далее добавим шаблон для конфига NGINX и модуль, который будет копировать этот шаблон на хост:

```
- name: NGINX | Create NGINX config file from template
template:
src: templates/nginx.conf.j2
dest: /tmp/nginx.conf
tags:
- nginx-configuration
```

и укажем порт, на котором будет слушать nginx:

```
vars:
    nginx_listen_port: 8080
```

Создаём шаблон в директории templates и называем его nginx.conf.j2:


```
# {{ ansible_managed }}
events {
    worker_connections 1024;
}

http {
    server {
        listen       {{ nginx_listen_port }} default_server;
        server_name  default_server;
        root         /usr/share/nginx/html;

        location / {
        }
    }
}
```
Теперь создадим handler и добавим notify к копированию шаблона. Теперь каждый раз, когда конфиг будет изменяться - сервис перезагрузится.
Секция с handlers будет выглядеть следующим образом:

```
handlers:
    - name: restart nginx
      systemd:
        name: nginx
        state: restarted
        enabled: yes
    
    - name: reload nginx
      systemd:
        name: nginx
        state: reloaded
```

Notify будут выглядеть так:

```
- name: NGINX | Install NGINX package from EPEL Repo
yum:
name: nginx
state: latest
notify:
- restart nginx
tags:
- nginx-package
- packages

- name: NGINX | Create NGINX config file from template
template:
src: templates/nginx.conf.j2
dest: /etc/nginx/nginx.conf
notify:
- reload nginx
tags:
- nginx-configuration

```

Итого получаем:

```
---
- name: NGINX | Install and configure NGINX
  hosts: nginx
  become: true
  vars:
    nginx_listen_port: 8080

  tasks:
    - name: NGINX | Install EPEL Repo package from standart repo
      yum:
        name: epel-release
        state: present
      tags:
        - epel-package
        - packages

    - name: NGINX | Install NGINX package from EPEL Repo
      yum:
        name: nginx
        state: latest
      notify:
        - restart nginx
      tags:
        - nginx-package
        - packages

    - name: NGINX | Create NGINX config file from template
      template:
        src: templates/nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      notify:
        - reload nginx
      tags:
        - nginx-configuration

  handlers:
    - name: restart nginx
      systemd:
        name: nginx
        state: restarted
        enabled: yes
    
    - name: reload nginx
      systemd:
        name: nginx
        state: reloaded
```

Запустим:

```
neva@Uneva:~/Ansible$ ansible-playbook nginx.yml

PLAY [NGINX | Install and configure NGINX] **************************************************************************************************************************************************************************************************

TASK [Gathering Facts] **********************************************************************************************************************************************************************************************************************
ok: [nginx]

TASK [NGINX | Install EPEL Repo package from standart repo] *********************************************************************************************************************************************************************************
ok: [nginx]

TASK [NGINX | Install NGINX package from EPEL Repo] *****************************************************************************************************************************************************************************************
ok: [nginx]

TASK [NGINX | Create NGINX config file from template] ***************************************************************************************************************************************************************************************
changed: [nginx]

RUNNING HANDLER [reload nginx] **************************************************************************************************************************************************************************************************************
changed: [nginx]

PLAY RECAP **********************************************************************************************************************************************************************************************************************************
nginx                      : ok=5    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

Проверим из консоли доступность:

```
 curl http://192.168.56.12:8080
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


```













