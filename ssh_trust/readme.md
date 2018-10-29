1、配置/etc/ansible/hosts 文件
```markdown
[zabbix_agent]
172.26.4.203
172.26.4.204
172.26.4.205
[zabbix_agent:vars]
ansible_ssh_user=root
ansible_ssh_pass=root1234
ansible_ssh_port=22
```
2、在ansible-server端生成秘钥对
```markdown
ssh-keygen -t rsa
```
3、创建playbook
```markdown
1) 第一种方式: 
# 新增本地(ansible-server)公钥内容到远端客户端.ssh目录中authorized_keys文件，没有则创建authorized_keys文件
# state: 1) present 添加，2) absent 删除
---
- hosts: zabbix_agent
  gather_facts: false

  tasks:
    - name: deliver authorized_keys
      authorized_key:
        user: root
        key: "{{ lookup('file', '/root/.ssh/id_rsa.pub) }}"

# 解释：
添加或移除authorized keys为特定用户：比如上面的是添加读取本地的id_rsa.pub文件到远端主机的authorized_keys文件中
      
      
# 第二种方式:
- hosts: zabbix_agent
  remote_user: root

  tasks:
  - name: mkdir /root/.ssh
    command: mkdir -p /root/.ssh

  - name: copy ssh key
    copy: src=/root/.ssh/id_rsa.pub dest=/root/.ssh owner=root group=root mode=0644
```
4、运行playbook
cd /etc/ansible/playbook/ssh_trust
ansible-playbook rsync_key.yml 



5、可能出现的错误以及解决方法：
```markdown
[root@ansible-server ssh_trust]# ansible-playbook rsync_key.yml 

PLAY [zabbix_agent] ************************************************************************************************************************************************

TASK [deliver authorized_keys] *************************************************************************************************************************************
fatal: [172.26.4.204]: FAILED! => {"msg": "Using a SSH password instead of a key is not possible because Host Key checking is enabled and sshpass does not support this.  Please add this host's fingerprint to your known_hosts file to manage this host."}
fatal: [172.26.4.203]: FAILED! => {"msg": "Using a SSH password instead of a key is not possible because Host Key checking is enabled and sshpass does not support this.  Please add this host's fingerprint to your known_hosts file to manage this host."}
fatal: [172.26.4.205]: FAILED! => {"msg": "Using a SSH password instead of a key is not possible because Host Key checking is enabled and sshpass does not support this.  Please add this host's fingerprint to your known_hosts file to manage this host."}
	to retry, use: --limit @/etc/ansible/playbook/ssh_trust/rsync_key.retry

PLAY RECAP *********************************************************************************************************************************************************
172.26.4.203               : ok=0    changed=0    unreachable=0    failed=1   
172.26.4.204               : ok=0    changed=0    unreachable=0    failed=1   
172.26.4.205               : ok=0    changed=0    unreachable=0    failed=1 


2)原因和解决办法：
ssh第一次连接的时候一般会提示输入yes 进行确认为将key字符串加入到 ~/.ssh/known_hosts 文件中。而本机的~/.ssh/known_hosts文件中并有fingerprint key串
解决方法：在ansible.cfg文件中更改下面的参数：
#host_key_checking = False 将#号去掉即可
```