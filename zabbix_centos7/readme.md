一、安装zabbix_agentd
1、创建目录
```markdown
1）在下面显示的目录结构中，包含了zabbix安装、卸载和配置三个角色(roles)，以及每一个roles的所有tasks列表、vars变量和zabbix配置文件。其中安装和卸载分别通过zabbix_install.yml、zabbix_delete.yml主任务程序的入口文件调用
2）在/etc/ansible 目录下创建 所需目录和文件
mkdir zabbix_centos7
mkdir zabbix_centos7/zabbix_agent
mkdir zabbix_centos7/zabbix_agent/roles
mkdir zabbix_centos7/zabbix_agent/roles/{common,install,uninstall,configure}/{handlers,files,meta,tasks,templates,vars} -p
3）https://www.cnblogs.com/wanstack/p/8649495.html
3) 主机列表
[root@ansible-server ansible]# cat hosts | egrep -v "^$|^#"
[zabbix_agent]
172.26.4.203
172.26.4.204
172.26.4.205
[zabbix_agent:vars]
ansible_ssh_user=root
ansible_ssh_pass=root1234
ansible_ssh_port=22
```
2、安装程序的tasks任务列表
```markdown
1)定义安装程序入口文件 zabbix_install.yml
[root@ansible /etc/ansible/zabbix_centos7/zabbix_agent ]# vim zabbix_install.yml 
---
- hosts: testhosts
  remote_user: root
  gather_facts: True
  roles:
    - common
    - install

2) 定义安装程序-创建用户任务01-create-user.yml
[root@ansible /etc/ansible/zabbix_rhel/zabbix_agent ]# vim roles/install/tasks/01-create-user.yml 
---
- name: Create zabbix user
  user: name={{ zabbix_user }} state=present createhome=no shell=/sbin/nologin
  
3) 定义安装程序-拷贝安装文件任务02-copy-code.yml
[root@ansible /etc/ansible/zabbix_rhel/zabbix_agent ]# vim roles/install/tasks/02-copy-code.yml 
---
- name: Copy zabbix agentd code file to clients
  copy: src=ansible-zabbix-{{ zabbix_version }}.tar.gz dest=/usr/local/src/ansible-zabbix-{{ zabbix_version }}.tar.gz
        owner=root group=root
- name: Uncompression ansible-zabbix-{{ zabbix_version }}.tar.gz
  shell: tar zxf /usr/local/src/ansible-zabbix-{{ zabbix_version }}.tar.gz -C /usr/local
- name: Copy zabbix start script
  template: src=zabbix_agentd dest=/etc/init.d/zabbix_agentd owner=root group=root mode=0755
- name: Copy zabbix config file
  template: src=zabbix_agentd.conf dest={{ zabbix_dir }}/etc/zabbix_agentd.conf
            owner={{ zabbix_user }} group={{ zabbix_user }} mode=0644
- name: Modify zabbix basedir permisson
  file: path={{ zabbix_dir }} owner={{ zabbix_user }} group={{ zabbix_user }} mode=0755 recurse=yes
- name: Link zabbix_agentd command
  shell: ln -s {{ zabbix_dir }}/sbin/zabbix_agentd /usr/local/sbin/zabbix_agentd
- name: Delete ansible-zabbix-{{ zabbix_version }}.tar.gz source file
  shell: rm -rf /usr/local/src/ansible-zabbix-{{ zabbix_version }}.tar.gz
 
4) 定义安装程序-启动zabbix_agentd服务任务03-start-zabbix.yml 
[root@ansible /etc/ansible/zabbix_rhel/zabbix_agent ]# vim roles/install/tasks/03-start-zabbix.yml 
---
- name: Start zabbix service
  shell: /etc/init.d/zabbix_agentd start
- name: Add boot start zabbix service
  shell: chkconfig --level 345 zabbix_agentd on

5) 定义安装程序-添加iptable规则04-add-iptables.yml 
[root@ansible /etc/ansible/zabbix_rhel/zabbix_agent ]# vim roles/install/tasks/04-add-iptables.yml 
---
- name: insert iptables rule for zabbix
  lineinfile: dest=/etc/sysconfig/iptables create=yes state=present regexp="{{ zabbix_agentd_port }}"
              insertafter="^:OUTPUT "
              line="-A INPUT -p tcp --dport {{ zabbix_agentd_port }} -s {{ zabbix_server_ip }} -j ACCEPT"
  notify: restart iptables
  
6) 定义安装程序-tasks任务列表的主调用接口文件main.yml
[root@ansible /etc/ansible/zabbix_rhel/zabbix_agent ]# vim roles/install/tasks/main.yml 
---
- include: 01-create-user.yml
- include: 02-copy-code.yml
- include: 03-start-zabbix.yml
- include: 04-add-iptables.yml

总结: tasks任务列表说明:
Playbook允许用户将tasks任务细分为多个任务列表，通过一个main任务来调用。
当然你也可以将涉及的所有任务全部写到main.yml文件中。

7) 定义common任务
该任务为安装一些依赖软件包

[root@ansible /etc/ansible/zabbix_rhel/zabbix_agent ]# vim roles/common/tasks/main.yml 
- name: Install initializtion require software
  yum: name={{ item }} state=latest
  with_items:
    - libselinux-python
    - libcurl-devel


8) 帮助
# 关于lineinfile 参考 https://www.cnblogs.com/jackchen001/p/6710233.html
# 关于roles 的 参考文档 https://www.cnblogs.com/wanstack/p/8650434.html
这个 playbook 为一个角色 ‘x’ 指定了如下的行为

    如果 roles/x/tasks/main.yml 存在, 其中列出的 tasks 将被添加到 play 中
    如果 roles/x/handlers/main.yml 存在, 其中列出的 handlers 将被添加到 play 中
    如果 roles/x/vars/main.yml 存在, 其中列出的 variables 将被添加到 play 中
    如果 roles/x/defaults /main.yml存在，其中列出的变量将被添加到play 中
    如果 roles/x/meta/main.yml 存在, 其中列出的 “角色依赖” 将被添加到 roles 列表中
    所有 copy tasks 可以引用 roles/x/files/ 中的文件，不需要指明文件的路径。
    所有 script tasks 可以引用 roles/x/files/ 中的脚本，不需要指明文件的路径。
    所有 template tasks 可以引用 roles/x/templates/ 中的文件，不需要指明文件的路径。
    所有 include tasks 可以引用 roles/x/tasks/ 中的文件，不需要指明文件的路径
加载顺序: 
    meta/main.yml
    tasks/main.yml
    handlers/main.yml
    vars/main.yml
    defaults/main.yml

```

3、安装程序的vars变量文件
```markdown
在roles/install/vars/目录中创建main.yml文件，并定义安装过程中使用到变量

[root@ansible /etc/ansible/zabbix_rhel/zabbix_agent ]# vim roles/install/vars/main.yml
zabbix_dir: /usr/local/zabbix
zabbix_version: 4.0.0
zabbix_user: zabbix
zabbix_server_port: 10051
zabbix_agentd_port: 10050
zabbix_server_ip: 172.26.4.202

```

4、安装程序的handlers任务定义
```markdown
handlers目录中为tasks任务中notify调用的动作（当文件发生改变时，通过notify执行相关的操作，比如修改了配置文件后，需要重启相应的服务）

[root@ansible /etc/ansible/zabbix_rhel/zabbix_agent ]# vim roles/install/handlers/main.yml
---
- name: restart iptables
  service: name=iptables state=restarted
```

5、安装程序的zabbix安装文件
```markdown
将预先编译好的zabbix打包后，放到files/目录中，Playbook在执行操作过程中会根据tasks任务将文件发送到目标主机上

[root@ansible /etc/ansible/zabbix_rhel/zabbix_agent ]# ls roles/install/files/
ansible-zabbix-4.0.0.tar.gz
```

6、安装程序的zabbix配置文件
```markdown
使用Playbook安装zabbix涉及的所有配置文件都通过templates模块去同步

[root@ansible /etc/ansible/zabbix_rhel/zabbix_agent ]# ls roles/install/templates/
zabbix_agentd  zabbix_agentd.conf

在安装任务中，zabbix_agentd.conf配置文件主要涉及zabbix服务端IP地址的修改，可以通过预定义的变量来替换
zabbix_agentd.conf文件如下
LogFile=/tmp/zabbix_agentd.log
Server={{ zabbix_server_ip }}
ServerActive={{ zabbix_server_ip }}:10051
Hostname={{ ansible_default_ipv4.address }}

# 启动脚本文件
#!/bin/sh
#chkconfig: 345 95 95

# Zabbix
# Copyright (C) 2001-2018 Zabbix SIA
#
# This program is free software; you can redistribute it and/or modify
# it under the terms of the GNU General Public License as published by
# the Free Software Foundation; either version 2 of the License, or
# (at your option) any later version.
#
# This program is distributed in the hope that it will be useful,
# but WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the
# GNU General Public License for more details.
#
# You should have received a copy of the GNU General Public License
# along with this program; if not, write to the Free Software
# Foundation, Inc., 51 Franklin Street, Fifth Floor, Boston, MA  02110-1301, USA.

# Start/Stop the Zabbix agent daemon.
# Place a startup script in /sbin/init.d, and link to it from /sbin/rc[023].d

SERVICE="Zabbix agent"
DAEMON=/opt/zabbix-agent/sbin/zabbix_agentd
PIDFILE=/tmp/zabbix_agentd.pid

case $1 in
  'start')
    if [ -x ${DAEMON} ]
    then
      $DAEMON
      # Error checking here would be good...
      echo "${SERVICE} started."
    else
      echo "Can't find file ${DAEMON}."
      echo "${SERVICE} NOT started."
    fi
  ;;
  'stop')
    if [ -s ${PIDFILE} ]
    then
      if kill `cat ${PIDFILE}` >/dev/null 2>&1
      then
        echo "${SERVICE} terminated."
        rm -f ${PIDFILE}
      fi
    fi
  ;;
  'restart')
    $0 stop
    sleep 10
    $0 start
  ;;
  *)
    echo "Usage: $0 start|stop|restart"
    ;;
esac

```
7、执行安装zabbix_agentd
```markdown
ansible-playbook zabbix_install.yml
```

二、卸载zabbix_agentd

```markdown
1) 定义删除程序入口文件zabbix_delete.yml
[root@ansible /etc/ansible/zabbix_rhel/zabbix_agent ]# vim zabbix_delete.yml 
---
- hosts: testhosts
  remote_user: root
  gather_facts: True
  roles:
    - uninstall
2) 定义删除zabbix文件任务uninstall_zabbix.yml
# 这里执行最好不要使用shell ，因为经过测试如果使用shell 执行ansible-playbook时不具备【幂等性】
[root@ansible /etc/ansible/zabbix_rhel/zabbix_agent ]# vim roles/uninstall/tasks/uninstall_zabbix.yml 
---
- name: stop zabbix agentd service
  shell: /etc/init.d/zabbix_agentd stop
- name: delete zabbix start script
  shell: rm -rf /etc/init.d/zabbix_agentd
- name: delete zabbix_agentd script
  shell: rm -rf /usr/local/sbin/zabbix_agentd
- name: delete zabbix agentd basedir
  shell: rm -rf {{ zabbix_basedir }}
- name: delete zabbix agent logfile
  shell: rm -rf /tmp/zabbix_agentd.log
- name: delete zabbix user
  user: name=zabbix state=absent remove=yes
  
3、定义删除iptables防火墙规则任务del_iptables.yml

[root@ansible /etc/ansible/zabbix_rhel/zabbix_agent ]# vim roles/uninstall/tasks/del_iptables.yml 
---
- name: delete iptables rule for zabbix
  lineinfile: dest=/etc/sysconfig/iptables state=absent
              line="-A INPUT -p tcp --dport {{ zabbix_agentd_port }} -s {{ zabbix_server_ip }} -j ACCEPT"
  notify: restart iptables


4、定义删除主task任务调用文件main.yml   

[root@ansible /etc/ansible/zabbix_rhel/zabbix_agent ]# vim roles/uninstall/tasks/main.yml 
---
- include: uninstall_zabbix.yml
- include: del_iptables.yml


5、定义删除程序的handlers任务

[root@ansible /etc/ansible/zabbix_rhel/zabbix_agent ]# vim roles/uninstall/handlers/main.yml 
---
- name: restart iptables
  service: name=iptables state=restarted

6、定义删除程序的vars变量文件

[root@ansible /etc/ansible/zabbix_rhel/zabbix_agent ]# vim roles/uninstall/vars/main.yml 
zabbix_basedir: /usr/local/zabbix
zabbix_agentd_port: 10050
zabbix_server_ip: 10.17.81.120

7、执行删除输出信息

[root@ansible /etc/ansible/zabbix_rhel/zabbix_agent ]# ansible-playbook zabbix_delete.yml 

```


三、通过ansible-playbook给客户端主机批量增加zabbix监控项目配置
（创建监控项目示例：自动发现远程主机监听的TCP端口、监控远程主机的TCP连接数状态）。
