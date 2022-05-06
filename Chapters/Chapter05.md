# 5. Ansible 力量初显：批量初始化服务器

- [5. Ansible 力量初显：批量初始化服务器](#5-ansible-力量初显批量初始化服务器)
  - [5.1 批量初始化服务器案例](#51-批量初始化服务器案例)
  - [5.2 Ansible 配置 SSH 密钥认证](#52-ansible-配置-ssh-密钥认证)
    - [5.2.1 方案一：命令行改写为 playbook](#521-方案一命令行改写为-playbook)
      - [command、shell、raw 和 script 模块](#commandshellraw-和-script-模块)
      - [connection 和 delegate_to 指令](#connection-和-delegate_to-指令)
    - [5.2.2 方案二：使用 authorized_key 模块](#522-方案二使用-authorized_key-模块)
      - [lookup()或 query()读取外部数据](#lookup或-query读取外部数据)
  - [5.3 配置主机名](#53-配置主机名)
    - [5.3.1 vars 设置变量](#531-vars-设置变量)
    - [5.3.2 when 条件判断](#532-when-条件判断)
    - [5.3.3 loop 循环](#533-loop-循环)
  - [5.4 互相添加 DNS 解析记录](#54-互相添加-dns-解析记录)
    - [5.4.1 lineinfile 模块](#541-lineinfile-模块)
    - [5.4.2 play_hosts 和 hostvars 变量](#542-play_hosts-和-hostvars-变量)
  - [5.5 配置 yum 镜像源并安装软件](#55-配置-yum-镜像源并安装软件)
  - [5.6 时间同步](#56-时间同步)
  - [5.7 关闭 Selinux](#57-关闭-selinux)
  - [5.8 配置防火墙](#58-配置防火墙)
  - [5.9 远程修改 sshd 配置文件并重启](#59-远程修改-sshd-配置文件并重启)
    - [5.9.1 notify 和 handlers](#591-notify-和-handlers)
  - [5.10 整合所有任务到单个 playbook 中](#510-整合所有任务到单个-playbook-中)

## 5.1 批量初始化服务器案例

既然前面已经学完了必要理论基础知识：inventory 和 playbook，现在或许是自己上手来试试 Ansible 如何进行批量配置管理的好机会。

在本文中，将借助批量初始化服务器的案例来演示如何使用 Ansible playbook，以便让各位熟悉 playbook 的编写。

本文要实现的初始化配置目标包括：

1. Ansible 配置 SSH 密钥认证
2. Ansible 远程配置主机名
3. Ansible 控制远程主机互相添加 DNS 解析记录
4. Ansible 配置远程主机上的 yum 镜像源以及安装一些软件
5. Ansible 配置远程主机上的时间同步
6. Ansible 关闭远程主机上 selinux
7. Ansible 配置远程主机上的防火墙
8. Ansible 远程修改 sshd 配置文件并重启 sshd，使其更安全

在本文最后，我会将所有这些案例的 playbook 集合到单个 playbook 文件中，同时还会藉此引入 Ansible 组织多个任务文件的概念，为下一篇文章做个简单的铺垫。

本文会介绍一些涉及到的模块用法，但模块用法不是本文的重点，也不是学习 Ansible 的重点，因为学习模块的用法很简单，甚至我觉得学会如何去找模块比学会模块的用法还重要，所以各位在学习 Ansible 的时候千万不要迷失在大量学习 Ansible 模块中。本文的重点是借助这些案例来解释 Ansible 提供的一些重要机制，从而学会如何解决一类问题，这才是真正精通 Ansible 必不可少的能力。

因为本文不仅仅只是写这几个案例就完事(其实这些案例的 playbook 内容都很短)，期间会介绍很多相关的重要知识点，所以篇幅较大，希望各位记好并初步整理好自己的笔记。之所以说是初步整理，是因为这里涉及的知识点都来自于案例，这对理解它们的用法很有帮助，但缺点是出现的比较突兀、比较零散，不方便系统性整理，难免会觉得东一榔头，西一棒子，很凌乱。不过不用担心，在后面我会专门写一篇进阶型的文章，到时候会系统性地介绍它们。

废话说了一大堆，进入正题吧。

下面是在/etc/ansible/hosts 文件中新添加的两个节点，主机组名为 new，它们将作为本文的测试节点：

```yaml
[new]
192.168.200.34
192.168.200.35
```

## 5.2 Ansible 配置 SSH 密钥认证

当要使用 Ansible 去控制管理一个新节点时，第一步肯定是配置控制端(即 Ansible 端)和这个被控制节点之间的 SSH 密钥认证，让它们之间互信，不用在建立 ssh 连接的时候还需要交互输入或手动去指定连接密码。

在之前的 Ansible 初体验文章中已经给出了一种比较简单的方案：

- 使用 ssh-keyscan 命令解决主机认证的交互问题(即不用输入 yes/no)
- 使用 sshpass -pPASSWORD 命令解决 ssh-copy-id 分发公钥时要求输入密码的问题

下面是整个过程：

```shell
# 在control_node节点上生成密钥对
ssh-keygen -t rsa -f ~/.ssh/id_rsa -N ''

for host in 192.168.200.{27..33};do
  # 将各节点的主机信息(host key)写入control_node的
  # `~/.ssh/known_hosts`文件
  ssh-keyscan $host >>~/.ssh/known_hosts 2>/dev/null

  # 将control_node上的ssh公钥分发给各节点：
  sshpass -p'123456' ssh-copy-id root@$host &>/dev/null
done
```

也可以使用 Ansible 去实现这样的功能。

### 5.2.1 方案一：命令行改写为 playbook

上面使用的是 shell 命令方案，也可以直接使用 Ansible playbook 来实现完全相同的步骤，而且写进 playbook 对统一初始化配置的操作是很值得的。

playbook 内容如下：

```yaml
---
- name: configure ssh connection
  hosts: new
  gather_facts: false
  connection: local
  tasks:
    - name: configure ssh connection
      shell: |
        ssh-keyscan {{inventory_hostname}} >>~/.ssh/known_hosts
        sshpass -p'123456' ssh-copy-id root@{{inventory_hostname}}
```

执行该 playbook，然后测试。

```shell
$ ansible-playbook sshkey.yaml
$ ssh 192.168.200.35
```

这个方案是我临时加进来的，目的是为了给各位介绍 connection: local 指令的作用，顺带着介绍一下涉及到的 shell 模块。所以，请各位开始记笔记。

首先要解释的是 {{inventory_hostname}} ，其中 {{}} 在前面的文章中已经解释过了，它可以用来包围变量，在解析时会进行变量替换。而这里引用的变量为 inventory_hostname，该变量表示的是当前正在执行任务的目标节点在 inventory 中定义的主机名。例如，对于本示例中两个执行任务的节点来说，该变量的值分别是 192.168.200.34 和 192.168.200.35。

然后再分别介绍相关模块和 connection: local。

#### command、shell、raw 和 script 模块

Ansible 用于配置管理，但是它最基本的功能得提供：在目标主机上执行命令。

command、shell、raw 和 script 这四个模块的作用和用法都类似，都用于远程执行命令或脚本：

- command 模块： 执行简单的远程 shell 命令，但不支持解析特殊符号`< > | ; &`等，比如需要重定向时不能使用 command 模块，而应该使用 shell 模块
- shell 模块：和 command 相同，但是支持解析特殊 shell 符号
- raw 模块：执行底层 shell 命令，command 和 shell 模块都是通过目标主机上的 python 代码启动`/bin/sh`来执行命令的，但是如果目标主机上没有安装 python，这时只能使用 raw 模块
- script 模块： 在远程主机上执行脚本文件，默认就是将 Ansible 控制端的脚本传到目标主机去执行的

例如：

```yaml
---
- name: use some module
  hosts: new
  gather_facts: false
  tasks:
    - name: use command module
      command: date +"%F %T"

    - name: use shell module
      shell: date +"%F %T" | cat >/tmp/date.log

    - name: use raw module
      raw: date +"%F %T"
```

如果想要看到命令的输出结果，可在执行 playbook 的时候加上一个-v 选项

如果要执行的命令有多行，根据之前文章中介绍的 YAML 语法，可以换行书写。例如：

```yaml
---
- name: use some module
  hosts: new
  gather_facts: false
  tasks:
    - name: use shell module
      shell: |
        date +"%F %T" | cat >/tmp/date.log
        date +"%F %T" | cat >>/tmp/date.log
```

如果要执行的是一个远程主机上已经存在的脚本文件，可以使用 shell、command 或 raw 模块，但有时候脚本是写在 Ansible 控制端的，可以先将它拷贝到目标主机上再使用 shell 模块去执行这个脚本，但更佳的方式是使用 script 模块，script 模块默认就是将 Ansible 控制端的脚本传到目标主机去执行的。此外，script 模块和 raw 模块一样，不要求目标主机上已经装好 python。

例如，Ansible 端有一个 shell 脚本文件/tmp/hello.sh，它还接收一个参数，内容如下：

```shell
#!/bin/bash

echo hello world: $1
```

playbook 内容如下：

```shell
---
- name: use some module
  hosts: new
  gather_facts: false
  tasks:
    - name: use script module
      script: /tmp/hello.sh HELLOWORLD
```

Ansible 中绝大多数的模块都具有幂等特性，意味着执行一次或多次不会产生副作用。但是 shell、command、raw、script 这四个模块默认不满足幂等性，所以操作会重复执行，但有些操作是不允许重复执行的。例如 mysql 的初始化命令 mysql_install_db，逻辑上它应该只在第一次配置的过程中初始化一次，其他任何时候都不应该再执行。所以，每当使用这 4 个模块的时候都要在心中想一想，重复执行这个命令会不会产生负面影响。

当然，除了 raw 模块外，其它三个模块也提供了实现幂等性的参数，即 creates 和 removes：

- creates 参数: 当指定的文件或目录存在时，则不执行命令
- removes 参数: 当指定的文件或目录不存在时，则不执行命令

例如：

```yaml
---
- name: use some module
  hosts: new
  gather_facts: false
  tasks:
    # 网卡配置文件不存在时不执行
    - name: use command module
      command: ifup eth0
      args:
        removes: /etc/sysconfig/network-scripts/ifcfg-eth0

    # mysql配置文件已存在时不执行，避免覆盖
    - name: use shell module
      shell: cp /tmp/my.cnf /etc/my.cnf
      args:
        creates: /etc/my.cnf
```

显然，使用 removes 或 creates 参数之后，就可以实现幂等性，保证命令不会重复执行。

最后，这四个模块都不限于执行 shell 命令或 shell 脚本，可以通过 executable 参数指定其它解释器，比如 expect 执行 expect 脚本、perl 解释器执行 perl 脚本等等：

```yaml
- name: Run a perl script
  script: /some/local/script.pl
  args:
    executable: perl
```

#### connection 和 delegate_to 指令

默认情况下，Ansible 使用 ssh 去连接远程主机，但实际上它提供了多种插件来丰富连接方式：smart、ssh、paramiko、local、docker、winrm，默认为 smart。

smart 表示智能选择 ssh 和 paramiko(paramiko 是 Python 的一个 ssh 协议模块)，当 Ansible 端安装的 ssh 支持 ControlPersist(即持久连接)时自动使用 ssh，否则使用 paramiko。local 和 docker 是非基于 ssh 连接的方式，winrm 是连接 Windows 的插件。

可以在命令行选项中使用-c 或--connection 选项来指定连接方式：

```shell
$ ansible          nginx -c smart -m XXX -a 'YYY'
$ ansible-playbook nginx -c smart -m XXX -a 'YYY'
```

或者在 playbook 的 play 级别或 task 级别上指定连接方式，定义在 task 上时表示该 task 使用所指定的连接方式：

```yaml
---
- hosts: nginx
  connection: smart
  ...
  tasks:
    - copy: src=/etc/passwd dest=/tmp
      connection: local
```

此外，inventory 中也可以通过连接的行为变量 ansible_connection 指定连接类型：

```yaml
192.168.200.29 ansible_connection="smart"
```

通常不需要关注连接类型，但是一种特殊的连接方式是 local，它表示在 Ansible 端本地执行任务。例如：

```yaml
---
- name: exec task at local
  hosts: new
  gather_facts: false
  connection: local
  tasks:
    - name: task1
      debug:
        msg: "{{inventory_hostname}} is executing task"
    - name: task2
      shell: touch /tmp/task1.txt
      args:
        creates: /tmp/task1.txt
```

执行该 playbook，注意观察输出结果：

```shell
$ ansible-playbook conn.yaml
......
ok: [192.168.200.34] => {
    "msg": "192.168.200.34 is executing task"
}
ok: [192.168.200.35] => {
    "msg": "192.168.200.35 is executing task"
}
......
```

不难发现/tmp/task1.txt 文件是创建在 Ansible 端本地，而不是在目标主机上创建。

或许有些人容易将`host:localhost`和`connection:local`搞混淆。虽然他们的效果都是在本地执行任务，但是`host:localhost`是从 inventory 中筛选了目标节点 localhost 来执行任务，而`connection:local`则筛选了其他目标主机，比如上面的示例中同时指定了`host:new`， 这表示筛选出来执行任务的目标主机是 new 组中的节点，但因为指定了 localhost 连接类型，使得 new 组中有多少个节点，就在 Ansible 本地执行多少次该 play。

其实很简单，本该在`host:new`筛选出来的主机上执行任务，但`connection:local`表示每个目标节点都将任务委托(delegate)给了 Ansible 端本地去执行。

事实上，Ansible 提供了另外一个执行：`delegate_to`，它表示将任务委托给谁去执行。显然`connection：local`和`delegate_to：localhost`在功能上是等价的。当热，connection 可以定义在 play 级别或者 task 级别上。

例如：

```yaml
---
- name: play1
  hosts: new
  gather_facts: false
  tasks:
    - name: task1
      debug:
        msg: "{{inventory_hostname}} is executing task"
      delegate_to: 192.168.200.35
```

也许这里会有一个更深的疑惑，为什么要使用 connection: local 或 delegate_to，而不直接在被委托节点上执行任务？

主要原因在于委托执行时获取了目标节点的上下文环境。正如委托给 Ansible 端本地执行时，它是可以获取到目标节点信息的，而如果直接通过 hosts: localhost 指定在本地执行，则除了 localhost 外没有任何其它目标节点，也就无法获取到这些节点的信息，比如无法获取它们的 IP 地址。

花了这么大的篇幅，我想告诉各位的是，delegate_to 是一个很重要的指令，在后面的文章中将会看到它在不少场景下都大放异彩，比如负载均衡(比如 haproxy)管理后端节点的场景。

现在，再回头来看看配置 SSH 主机互信的 playbook，我想应该很容易理解了。

```yaml
---
- name: configure ssh connection
  hosts: new
  gather_facts: false
  connection: local
  tasks:
    - name: configure ssh connection
      shell: |
        ssh-keyscan {{inventory_hostname}} >>~/.ssh/known_hosts
        sshpass -p'123456' ssh-copy-id root@{{inventory_hostname}}
```

### 5.2.2 方案二：使用 authorized_key 模块

Ansible 提供了一个 authorized_key 模块，它可以用来分发 SSH 公钥。

但需注意，authorized_key 模块并不负责主机认证的阶段，所以需要额外去处理主机认证阶段，有三种处理方式：

1. 使用 ssh-keyscan 命令将主机信息添加到 Ansible 端的~/.ssh/known_hosts 中
2. 使用 Ansible 提供的 known_hosts 模块添加主机信息
3. 禁止主机认证的阶段

禁止主机认证阶段的方式有多种，比如去 ssh 或 Ansible 的配置文件中禁止主机认证(两种配置都可以，因为 Ansible 默认是基于 ssh 进行连接的)，以 Ansible 配置文件为例，设置如下项即可：

```shell
host_key_checking = False
```

再比如下面三种单次禁用的方式：

```shell
# 1.在ansible或ansible-playbook命令行中指定跳过主机认证阶段
ansible --ssh-extra-args '-o StrictHostKeyChecking=no' XXX
ansible-playbook --ssh-extra-args '-o StrictHostKeyChecking=no' XXX

# 2.在inventory中通过ansible_ssh_extra_args设置ssh连接选项
192.168.200.34 ansible_ssh_extra_args="-o StrictHostKeyChecking=no"

# 3.使用环境变量的方式指定跳过主机认证
ANSIBLE_HOST_KEY_CHECKING=False ansible XXX
ANSIBLE_HOST_KEY_CHECKING=False ansible-playbook XXX

```

最方便的当然是配置 host_key_checking = False 或使用 ssh-keyscan 命令，这样的操作是一劳永逸的。使用 Ansible 的 known_hosts 模块也可以，但我个人不习惯它，各位如果有 Ansible 基础，可以去了解下如何使用该模块。

解决了主机认证的问题，现在使用 authorized_key 分发公钥到目标主机上。由于这一次分发的任务仍然是要建立 SSH 连接的，所以需要提供 SSH 连接的密码，当这次分发完成之后，以后 Ansible 就不再需要连接密码了。

ssh 连接密码可以指定在 inventory 文件中，比如定义在 all 组中：

```shell
[all:vars]
ansible_password="123456"
```

这样所有目标主机都使用该密码去连接，当分发完成后，再手动将 inventory 中的该变量删除。

不过，由于密码仅在此处分发密钥时使用一次，如果所有分发目标的连接密码都相同，我想更方便的是在执行 playbook 时使用-k 选项要求单次交互式输入密码，而不是将其定义在 inventory 中并在分发完成后删除。稍后将看到相关用法。

准备工作都完成后，开始写使用 authorized_key 模块分发公钥的 playbook，内容如下：

```yaml
---
- name: configure ssh connection
  hosts: new
  gather_facts: false
  tasks:
    - authorized_key:
      key: "{{lookup('file','~/.ssh/id_rsa.pub')}}"
      state: present
      user: root
```

执行该 playbook，主机加上了-k 选项，它会提示用户输入 ssh 连接密码。如果所有目标主机的密码都相同，则只需输入一次即可：

```shell
$ ansible-playbook -k anth_key.yml
SSH password:

PLAY [configure ssh connection] ***********

TASK [authorized_key] *********************
changed: [192.168.200.34]
changed: [192.168.200.35]
......
```

然后验证：

```shell
$ ansible new -a 'echo haha'
```

对于上面的 playbook，需要说明的是 authorized_key 模块的参数。

user 参数和 key 参数是必须的，key 指定公钥字符串。user 参数表示将密钥分发给目标主机上的哪个用户，默认会将公钥写入目标主机的/home/USERNAME/.ssh/authorized_keys 文件中，默认情况下会创建缺失的目录(比如.ssh 目录)并设置好权限。

这里还使用了 state 参数，state 参数值为 present 时，表示如果对方文件中已有完全相同的公钥信息，则不写入，否则写入。如果值为 absent，则表示删除目标节点上与本次待分发公钥完全相同的数据。总结起来就是：

- present：保证目标节点上会保存 Ansible 端本次分发的公钥

- absent：保证目标节点上没有 Ansible 端本次分发的公钥

最后需要说明的是在 key 参数中使用的 lookup()。

#### lookup()或 query()读取外部数据

lookup()在 Ansible 中使用频率非常高，几乎稍微复杂一点的 playbook 都可能会用上它。

lookup()是 Ansible 的一个插件，可用于从外部读取数据，这里的”外部”含义非常广泛，比如：

- 从磁盘文件读取(file 插件)

- 从 redis 中读取(redis 插件)
- 从 etcd 中读取(etcd 插件)
- 从命令执行结果读取(pipe 插件)
- 从 Ansible 变量中读取(vars 插件)
- 从 Ansible 列表中读取(list 插件)
- 从 Ansible 字典中读取(dict 插件)
- …

具体可以从哪些”外部”读取以及如何读取，取决于 Ansible 是否提供了相关的读取插件。官方手册：https://docs.ansible.com/ansible/latest/plugins/lookup.html#plugin-list中列出了所有支持的插件，在我写本文的时刻支持的插件数量有63个。

要学完所有这些插件显然是不可能的，但只需掌握常见的插件即可。在这里我准备介绍 file 和 fileglob、pipe 这三个插件，其它插件在需要时去手册上查找即可，它们的用法也都非常之简单。

下面是 lookup()的语法：

```shell
lookup('<plugin_name>', 'plugin_argument')
```

插件相关的参数可查询官方手册。

首先介绍 fileglob 插件，它使用通配符来通配 Ansible 本地端的文件名。

```yaml
---
- name: play1
  hosts: new
  gather_facts: false
  tasks:
    - name: task1
      debug:
        msg: "filenames: {{lookup('fileglob','/etc/*.conf')}}"
```

需注意的是，fileglob 查询的是 Ansible 端文件，且只能通配文件而不能通配目录，且不会递归通配。如果想要查询目标主机上的文件，可以使用 find 模块。

再介绍 file 插件，这个插件用的应该是最多的，它用来读取 Ansible 本地端文件。例如：

```yaml
---
- name: play1
  hosts: new
  gather_facts: false
  tasks:
    - name: task1
      debug:
        msg: "file content: {{lookup('file','/etc/hosts')}}"
```

最后介绍 pipe 插件，它从 Ansible 端的一个命令执行结果中读取数据：

```yaml
---
- name: play1
  hosts: new
  gather_facts: false
  tasks:
    - name: task1
      debug:
        msg: "command res: {{lookup('pipe','cat /etc/hosts')}}"
```

再者需要说明的是，如果 lookup()查询出来的结果包含多项，则默认以逗号分隔各项的字符串方式返回，如果想要以列表方式返回，则传递一个 lookup 的参数 wantlist=True。例如，fileglob 通配出来的文件如果有多个，加上 wantlist=True：

```yaml
---
- name: play1
  hosts: new
  gather_facts: false
  tasks:
    - name: task1
      debug:
        msg: "filenames: {{lookup('fileglob','/etc/*.conf',wantlist=True)}}"
```

在 Ansible 2.5 中添加了一个新的功能 query()或 q()，后者是前者的等价缩写形式。query()在写法和功能上和 lookup 一致，其实它会自动调用 lookup 插件，并且总是以列表方式返回，而不需要手动加上 wantlist=True 参数。例如：

```yaml
- name: task1
  debug:
    msg: "{{q('fileglob','/etc/*.conf')}}"
```

## 5.3 配置主机名

拿到新的主机之后，很可能需要配置这些主机的主机名，特别是现在云主机刚创建出来时的主机名通常都是一大串乱七八糟的字符，所以配置主机名是一件迫不及待的事情。

配置主机名非常简单，可以使用 shell 模块在远程执行相关命令，也可以使用 Ansible 提供的 hostname 模块。建议使用 hostname 模块，它支持多种操作系统。

当然，要使用 Ansible 去设置多个主机名，要求目标主机和目标名称已经关联好，否则多个主机和多个主机名之间无法对应去设置。

例如，在 new 主机组中有两个节点：192.168.200.34 和 192.168.200.35，分别设置它们的主机名为 new1 和 new2。

playbook 内容如下：

```yaml
---
- name: set hostname
  hosts: new
  gather_facts: false
  vars:
    hostnames:
      - host: 192.168.200.34
        name: new1
      - host: 192.168.200.35
        name: new2
  tasks:
    - name: set hostname
      hostname:
        name: "{{item.name}}"
      when: item.host == inventory_hostname
      loop: "{{hostnames}}"
```

执行：

```shell
$ ansible-playbook hostname.yaml
```

在上面的 playbook 中，除了 hostname 模块中涉及到的 when 和 loop 指令需要介绍，还需要介绍 vars 指令，这些都是非常重要且常见的指令，playbook 离不开它们。

### 5.3.1 vars 设置变量

vars 指令可用于设置变量，可以设置一个或多个变量。下面的设置方式都是合理的：

```yaml
# 设置单个变量
vars:
  var1: value1

vars:
  - var1: value1

# 设置多个变量：
vars:
  var1: value1
  var2: value2

vars:
  - var1: value1
  - var2: value2
```

vars 可以设置在 play 级别，也可以设置在 task 级别：

- 设置在 play 级别，该 play 范围内的 task 可以访问这些变量，其他 play 范围内则无法访问
- 设置在 task 级别，只有该 task 能访问这些变量，其他 task 和其他 play 无法访问

例如：

```yaml
---
- name: play1
  hosts: localhost
  gather_facts: False
  vars:
    - var1: "value1"
  tasks:
    - name: access var1
      debug:
        msg: "var1's value: {{var1}}"

- name: play2
  hosts: localhost
  gather_facts: false
  tasks:
    - name: can't access vars from play1
      debug:
        var: var1

    - name: set and access var2 in this task
      debug:
        var: var2
      vars:
        var2: "value2"

    - name: can't access var2
      debug:
        var: var2
```

执行结果：

```shell
PLAY [play1] *****************************

TASK [access var1] ***********************
ok: [localhost] => {
    "msg": "var1's value: value1"
}

PLAY [play2] *****************************

TASK [can't access vars from play1] ******
ok: [localhost] => {
    "var1": "VARIABLE IS NOT DEFINED!"
}

TASK [set and access var2 in this task] **
ok: [localhost] => {
    "var2": "value2"
}

TASK [can't access var2] *****************
ok: [localhost] => {
    "var2": "VARIABLE IS NOT DEFINED!"
}
```

回到设置主机名实力中设置 vars 指令部分：

```yaml
vars:
  hostnames:
    - host: 192.168.200.34
      name: new1
    - host: 192.168.200.35
      name: new
```

这里只设置了一个变量 hostnames，但这个变量的值是一个数组结构，数组的两个元素又都是对象(字典/hash)结构。

所以想要访问主机名 new1 和它的 IP 地址 192.168.200.34，可以：

```yaml
tasks:
  - debug:
      var: hostnames[0].name
  - debug:
      var: hostnames[0].host
```

在 Ansible 中，变量既是重点，也是难点，目前先了解 vars 可以设置变量即可，后面的文章中会统一解释。

### 5.3.2 when 条件判断

Ansible 作为一个编排、协调任务的配置管理工具，它必不可少的功能是提供流程控制功能，比如条件判断、循环、退出等。

在 Ansible 中，提供的唯一一个通用的条件判断是 when 指令，当 when 指令的值为 true 时，则该任务执行，否则不执行该任务。

例如：

```yaml
---
- name: play1
  hosts: localhost
  gather_facts: false
  vars:
    - myname: "junmajinlong"
  tasks:
    - name: task will skip
      debug:
        msg: "myname is: {{myname}}"
      when: myname == "junma"

    - name: task will execute
      debug:
        msg: "myname is: {{myname}}"
      when: myname == "junmajinlong"
```

该 playbook 中设置了变量 myname 的值为 junmajinlong。第一个任务因为 when 的判断条件是 myname == "junma"，所以判断结果为 false，该任务不执行，即被跳过。同理可知，第二个任务因为 when 的值为 true，所以执行了。

该 playbook 的执行结果：

```shell
TASK [task will skip] **************
skipping: [localhost]

TASK [task will execute] ***********
ok: [localhost] => {
    "msg": "myname is: junmajinlong"
}
```

**需要注意的是**，when 指令因为已经明确是做条件判断，所以它的值必定是一个表达式，它会自动隐式地包围一层 {{}} ，所以在写 when 指令的条件判断时，不要再手动加上 {{}} 。正如上面示例中的写法 when: myname == "junma"，如果改写成 when: {{myname == "junma"}} ，这表示表达式的嵌套，通常来说不会出错，但在某些场景下是错误的，而且 Ansible 会给出警告。

```shell
TASK [set hostname] *********************
[WARNING]: conditional statements should not include jinja2 templating delimiters such as {{ }} or {% %}. Found: {{item.host == inventory_hostname}}

[WARNING]: conditional statements should not include jinja2 templating delimiters such as {{ }} or {% %}. Found: {{item.host == inventory_hostname}}
```

when 一个比较常见的应用场景是实现跳过某个主机不执行任务或者只有满足条件的主机执行任务。例如：

```shell
when: inventory_hostname == "web1"
```

这表示只有 inventory 中设置的主机 web1 才执行任务，其它主机都不执行。

在上面设置主机名的示例中也用到了这一功能，稍后介绍 loop 循环的时候还会再稍作解释。

最后，虽然 when 指令的逻辑很简单：值为 true 则执行任务，否则不执行任务。但是，它的用法并不简单，概因 when 指令的值可以是 Jinja2 的表达式，很多内置在 Jinja2 中的 Python 的语法都可以用在 when 指令中，而这需要掌握 Python 的基本语法。如果不具备这些知识，那么想要实现某种判断功能可能会感觉到较大的局限性，而且别人写的脚本可能看不懂。但是我的建议是，别强求，掌握常用的条件判断方式，以后有需求再去网上搜索对应用法。

### 5.3.3 loop 循环

除条件判断外，另一种分支控制结构是循环结构。

Ansible 提供了很多种循环结构，一般都命名为 with*xxx，例如 with_items、with_list、with_file 等，使用最多的是 with_items。事实上 with*<lookup>结构是对应 lookup 插件的，关于 with_xxx 这些循环结构在以后的文章中再统一介绍。

在这里仅介绍 loop 循环，它是在 Ansible 2.5 版本中新添加的循环结构，等价于 with_list。大多数时候，with_xxx 的循环都可以通过一定的手段转换成 loop 循环，所以从 Ansible 2.5 版本之后，原来经常使用的 with_items 循环都可以尝试转换成 loop。

有了循环结构，生活就美妙多了。例如，在 localhost 上创建两个目录/tmp/test{1,2}，不用循环结构的 playbook 写法：

```yaml
---
- name: play1
  hosts: localhost
  gather_facts: false
  tasks:
    - name: create /tmp/test1
      file:
        path: /tmp/test1
        state: directory

    - name: create /tmp/test2
      file:
        path: /tmp/test2
        state: directory
```

使用 loop 循环的写法：

```yaml
---
- name: play1
  hosts: localhost
  gather_facts: false
  tasks:
    - name: create directories
      file:
        path: "{{item}}"
        state: directory
      loop:
        - /tmp/test1
        - /tmp/test2
```

显然，循环可以将多个任务组织成单个任务并实现相同的功能。

解释下上面的 loop 和 {{item}} 。

loop 等价于 with_list，从名字上可以知道它是遍历数组(列表)的，所以在 loop 指令中，每个元素都以列表的方式去定义。列表有多少个元素，就循环执行 file 模块多少次，每轮循环中，都会将本次迭代的列表元素保存在控制变量 item 中。

或许，使用伪代码去解释这个 loop 结构会更容易理解，这里我使用 shell 伪代码来演示：

```shell
for item in /tmp/test1 /tmp/test2;do
  mkdir ${item}
done
```

回头再来看设置主机名案例的 playbook：

```yaml
---
- name: set hostname
  hosts: new
  gather_facts: false
  vars:
    hostnames:
      - host: 192.168.200.34
        name: new1
      - host: 192.168.200.35
        name: new2
  tasks:
    - name: set hostname
      hostname:
        name: "{{item.name}}"
      when: item.host == inventory_hostname
      loop: "{{hostnames}}"
```

在这个示例中，是对 {{hostnames} }进行循环遍历，hostnames 中包含两个元素，每个元素都是一个 key/value 的对象结构。所以，第一次迭代时，item 变量的值是：

```yaml
{ host: "192.168.200.34", name: "new1" }
```

所以 item.host 和 item.name 分别对应于 192.168.200.34 和 new1。

这里需要注意两点：

1. loop 和 when 结合使用时，when 的条件判断实在循环内部执行的。也就是说循环执行(如`loop`)的解析顺序早于 when 指令

   - 如果想要通过条件判断来决定是否要执行循环呢？等待后面的文章，这里不展开

2. 上面 new 主机组中的两个节点都要执行这个循环，所以总共 4 轮循环，只不过因为 when 条件判断的存在，每个节点都跳过了一轮循环

## 5.4 互相添加 DNS 解析记录

如果新增的机器之间需要通过主机名进行通信，且没有 DNS 服务器，那么可能需要为每个节点都添加其它节点的 DNS 解析记录。

例如，nginx 主机组中有 3 个新节点 A、B、C，A 需要添加 B 和 C 的 DNS 记录，B 需要添加 A 和 C 的，C 需要添加 A 和 B 的。

下面是 playbook 的内容：

```yaml
---
- name: add DNS for each
  hosts: nginx
  gather_facts: true
  tasks:
    - name: add DNS
      lineinfile:
        path: "/etc/hosts"
        line: "{{item}} {{hostvars[item].ansible_hostname}}"
      when: item != inventory_hostname
      loop: "{{ play_hosts }}"
```

执行结果：

```shell
$ ansible-playbook add_dns.yaml

PLAY [add DNS for each] ****************************

TASK [Gathering Facts] *****************************
ok: [192.168.200.27]
ok: [192.168.200.28]
ok: [192.168.200.29]

TASK [add DNS] *************************************
skipping: [192.168.200.27] => (item=192.168.200.27)
changed:  [192.168.200.27] => (item=192.168.200.28)
changed:  [192.168.200.27] => (item=192.168.200.29)
changed:  [192.168.200.28] => (item=192.168.200.27)
skipping: [192.168.200.28] => (item=192.168.200.28)
changed:  [192.168.200.28] => (item=192.168.200.29)
changed:  [192.168.200.29] => (item=192.168.200.27)
changed:  [192.168.200.29] => (item=192.168.200.28)
skipping: [192.168.200.29] => (item=192.168.200.29)
```

在这个 playbook 中使用了一个模块 lineinfile，同时使用了 when 和 loop 以及几个特殊的变量。我一一介绍。

### 5.4.1 lineinfile 模块

lineinfile 模块用于在源文件中插入、删除、替换行，和 sed 命令的功能类似，也支持正则表达式匹配和替换。

要完整介绍其用法，会花掉大量篇幅，所以这里介绍一些典型的需求，之后再去查看官方手册便更容易理解。

测试文件 a.txt 的内容为：

```shell
paragraph 1
first line in paragraph 1
second line in paragraph 1
paragraph 2
first line in paragraph 2
second line in paragraph 2
注意在每次测试后，都将该测试文件还原回原始数据状态。
```

第一个用法是保证某行的存在：

```yaml
---
- name: test inlinefile
  hosts: localhost
  gather_facts: false
  tasks:
    - lineinfile:
        path: "a.txt"
        line: "this line must in"
```

执行：

```
$  ansible-playbook lineinfile.yaml
```

执行完后，”this line must in”将被追加在 a.txt 的最后一行。

如果再次执行，则不会再次追加此行。因为 lineinfile 模块的 state 参数默认值为 present，它能保证幂等性，当要插入的行已经存在时则不会再插入。

如果要移除某行，则设置 state 参数值为 absent 即可。下面的任务会移除 a.txt 中所有的”this line must not in”行(如果多行则全部移除)。

```yaml
- lineinfile:
    path: "a.txt"
    line: "this line must not in"
    state: absent
```

如果想要在某行前、某行后插入指定数据行，则结合 insertbefore 和 insertafter。例如：

```yaml
- lineinfile:
    path: "a.txt"
    line: "LINE1"
    insertbefore: "^para.* 2"

- lineinfile:
    path: "a.txt"
    line: "LINE2"
    insertafter: "^para.* 2"
```

这会将 LINE1 和 LINE2 分别插入在 paragraph 2 行的前后。结果如下：

```shell
paragraph 1
first line in paragraph 1
second line in paragraph 1
LINE1
paragraph 2
LINE2
first line in paragraph 2
second line in paragraph 2
```

注意，insertbefore 和 insertafter 指定的正则表达式如果匹配了多行，则默认选中最后一个匹配行，然后在被选中的行前、行后插入。如果明确要指定选中第一次匹配的行，则指定参数 firstmatch=yes：

```yaml
- lineinfile:
    path: "a.txt"
    line: "LINE1"
    insertbefore: "^para.* 2"
    firstmatch: yes
```

此外，对于 insertbefore，如果它的正则表达式匹配失败，则会插入到文件的尾部。

lineinfile 还可以替换行，使用 regexp 参数指定要匹配并被替换的行即可，默认替换最后一个匹配成功的行。

```yaml
- lineinfile:
    path: "a.txt"
    line: "LINE1"
    regexp: "^para.* 2"
```

结果：

```shell
paragraph 1
first line in paragraph 1
second line in paragraph 1
LINE1
first line in paragraph 2
second line in paragraph 2
```

如果 regexp 指定的正则表达式匹配失败，则行将插入在文件尾部。

regexp 结合 state=absent 时，表示删除所有匹配的行。

```yaml
- lineinfile:
    path: "a.txt"
    regexp: "^para.* 2"
    state: absent
```

lineinfile 最后一个比较常用的功能是 regexp 结合 insertbefore 或结合 insertafter。这时候的行将根据 insertXXX 的位置来插入，而 regexp 参数则充当幂等性判断参数：只有 regexp 匹配失败时，insertXXX 才会插入行。

例如：

```yaml
- lineinfile:
    path: "a.txt"
    line: "hello line"
    regexp: "^hello"
    insertbefore: "^para.* 2"
```

这表示将”hello line”插入在 paragraph 2 行的前面，但如果再次执行，则不会再次插入，因为 regexp 参数指定的正则表达式已经能够已经存在的”hello line”行。

所以，当 regexp 结合 insertXXX 使用时，regexp 的参数通常都会设置为能够匹配插入之后的行的正则表达式，以便实现幂等性。

### 5.4.2 play_hosts 和 hostvars 变量

inventory_hostname 变量已经使用过几次了，它表示当前正在执行任务的主机在 inventory 中定义的名称。在此示例中，inventory 中的主机名都是 IP 地址。

play_hosts 和 hostvars 是 Ansible 的预定义变量，执行任务时可以直接拿来使用，不过在 Ansible 中预定义变量有专门的称呼：魔法变量(magic variables)。Ansible 支持不少魔法变量，详细信息参见官方手册：magic variables，这里只介绍这两个，在后面的一篇进阶型文章里会统一介绍这些变量，所以各位不必急着打开官方手册去了解各个魔法变量的含义。

首先是 play_hosts 变量，它存储当前 play 所涉及的所有主机列表，但连接失败或执行任务失败的节点不会留在此变量中。

例如，某 play 指定了 hosts: nginx，那么执行这个 play 时，play_hosts 中就以列表的方式存储了 nginx 组中的所有主机名。

```shell
play_hosts: [
  "192.168.200.27",
  "192.168.200.28",
  "192.168.200.29"
]
```

有时候会借助这个变量来做判断。比如只让目标主机中的其中一个节点执行任务，其它节点不执行，那么可以：

```shell
tasks:
  - name: xxx
    ping:
    when: inventory_hostname == play_hosts[0]
```

hostvars 变量用于保存所有和主机相关的变量，通常包括 inventory 中定义的主机变量和 gather_facts 收集到的主机信息变量。hostvars 是一个 key/value 格式的字典(即 hash 结构、对象)，key 是每个节点的主机名，value 是该主机对应的变量数据。

例如，在 inventory 中有一行：

```shell
192.168.200.27 myvar="hello world"
```

如果要通过 hostvars 来访问该主机变量，则使用 hostvars['192.168.200.27'].myvar。

因为 gather_facts 收集的主机信息也会保存在 hostvars 变量中，所以也可以通过 hostvars 去访问收集到的信息。gather_facts 中收集了非常多的信息，目前只需记住此处所涉及的`ansible_hostname`即可，它保存的是收集目标主机信息而来的主机名，不是定义在 Ansible 端 inventory 中的主机名(因为可能是 IP 地址)。

> 注意：ansible_hostname 和 ansible_fqdn
> 如果目标节点是 fqdn 格式的主机名，比如www.junmajinlong.com，ansible_fqdn保存的是www.junmajinlong.com，而ansible_hostname保存的是www。

再者，由于 hostvars 中保存了所有目标主机的主机变量，所以任何一个节点都可以通过它去访问其它节点的主机变量。比如示例中`hostvars[item].ansible_hostname`访问的是某个主机的`ansible_hostname`变量。

再来看互相添加 DNS 解析记录的示例：

```yaml
- name: add DNS
  lineinfile:
    path: "/etc/hosts"
    line: "{{item}} {{hostvars[item].ansible_hostname}}"
  when: item != inventory_hostname
  loop: "{{ play_hosts }}"
```

在这里循环迭代 nginx 组中的三个节点(假设简称为 A、B、C)，当 A 节点执行这个任务的时候，如果迭代到 play_hosts 中的 A 节点，则本轮循环跳过，当迭代到 B 节点或 C 节点时，则向/etc/hosts 文件中添加它们的 IP 地址和主机名，由于这时候是 A 节点在执行任务，所以要访问 B 的主机名变量，只能通过 hostvars[B].ansible_hostname 来访问，同理迭代到 C 节点也是如此。

## 5.5 配置 yum 镜像源并安装软件

配置 yum 源并安装一些常用软件，也是初始化配置服务器应有之义。

现在的需求是：

- 备份原有 yum 镜像源文件，并配置清华大学的 yum 镜像源：OS 源和 epel 源

- 安装常用软件，包括 lrzsz、dos2unix、wget、curl、vim 等

playbook 如下：

```yaml
---
- name: config yum repo and install software
  hosts: new
  gather_facts: false
  tasks:
    - name: backup origin yum repos
      shell:
        cmd: "mkdir bak; mv *.repo bak"
        chdir: /etc/yum.repos.d
        creates: /etc/yum.repos.d/bak

    - name: add os repo and epel repo
      yum_repository:
        name: "{{item.name}}"
        description: "{{item.name}} repo"
        baseurl: "{{item.baseurl}}"
        file: "{{item.name}}"
        enabled: 1
        gpgcheck: 0
        reposdir: /etc/yum.repos.d
      loop:
        - name: os
          baseurl: "https://mirrors.tuna.tsinghua.edu.cn/centos/$releasever/os/$basearch"
        - name: epel
          baseurl: "https://mirrors.tuna.tsinghua.edu.cn/epel/$releasever/$basearch"

    - name: install pkgs
      yum:
        name: lrzsz,vim,dos2unix,wget,curl
        state: present
```

第一个任务是将/etc/yum.repos.d 下的所有 repo 文件备份到 bak 目录中，使用了两个参数，chdir 参数表示执行 shell 模块之前先切换到/etc/yum.repos.d 目录下，creates 参数表示 bak 目录存在时则不执行 shell 模块。

第二个任务是使用 yum_repository 模块配置 yum 源，该模块可添加或移除 yum 源。相关常用参数如下：

- name: 指定 repo 的名称，对应于 repo 文件中的[name]

- description: repo 的描述信息，对应于 repo 文件中的 name: xxx
- baseurl: 指定该 repo 的路径
- file: 指定 repo 的文件名，不需要加上.repo 后缀，会自动加上
- reposdir: repo 文件所在的目录，默认为/etc/yum.repos.d 目录
- enabled: 是否启用该 repo，对应于 repo 文件中的 enabled
- gpgcheck: 该 repo 是否启用 gpgcheck，对应于 repo 文件中的 gpgcheck
- state: present 表示保证该 repo 存在，absent 表示移除该 repo

在示例中使用了一个 loop 循环来添加两个 repo：os 和 epel。

第三个任务是使用 yum 模块安装一些 rpm 包，yum 模块可以更新、安装、移除、下载包。常用参数说明：

- name: 指定要操作的包名

  - 可以带版本号
  - 可以是单个包名，也可以是包名列表，或者逗号分隔的多个包名
  - 可以是 url
  - 可以是本地 rpm 包

- state:
  - present 和 installed: 保证包已安装，它们是等价的别名
  - latest: 保证包已安装了最新版本，如果不是则更新
  - absent 和 removed: 移除包，它们是等价的别名
- download_only: 仅下载不安装包(Ansible 2.7 才支持)
- download_dir: 下载包存放在哪个目录下(Ansible 2.8 才支持)

`yum`模块是 RHEL 系列的包管理器，如果是`Ubuntu`,则无法使用。其实，还有一个更为通用的包管理器模块 `package`,它可以自动探测目标节点的包管理器类型并使用它们去管理软件。大多数时候使用`package`来替代`yum`或替代`apt- insta11`等不会有什么题，但有些包名在不同操作系统上是不一样的(如`libama-dev`、`libyam-deve1`),所以还是小心使用 package 模块。
到现在，也许你已经发现了，任条在逻辑上是有关联关系的，比如上面示例中备份 repo 文件和添加 repo,我觉得它们应该作为个整体，但它们毕竟是两个任务，只能分开书写。
其实， Ansible 提供了一个 block 指令，允许将多个任务组织在一起，从而对 block 整体进行一些控制。此处我就不去修改当前案例，在下面的案例中将给各位介绍使用 block 来组织任务的方式。

## 5.6 时间同步

保证时间同步可以避免很多玄学性问题，特别是对集群中的节点。

通常会使用 ntpd 时间服务器来保证时间的同步，但在这里简单点，直接使用 ntpdate 命令初步保证时间同步，并将同步后的时间同步到硬件。

所以，整个 playbook 非常的简单：

```yaml
---
- name: sync time
  hosts: new
  gather_facts: false
  tasks:
    - name: install and sync time
      block:
        - name: install ntpdate
          yum:
            name: ntpdate
            state: present

        - name: ntpdate to sync time
          shell: |
            ntpdate ntp1.aliyun.com
            hwclock -w
```

上面使用了一个 block 指令来组织了两个有关联性的任务，将它们作为了一个整体。由此也可以看到，block 的用法非常简单。其实 block 更多的用于多个关联性任务之间的异常处理，下面的案例中我将介绍一个比较常见也很经典的错误处理方式。

## 5.7 关闭 Selinux

本案例用于关闭 selinux。playbook 如下：

```yaml
---
- name: disable selinux
  hosts: new
  gather_facts: false
  tasks:
    - name: disable on the fly
      shell: setenforce 0

    - name: disable forever in config
      lineinfile:
        path: /etc/selinux/config
        line: "SELINUX=disabled"
        regexp: "^SELINUX="
```

咋一看这个 playbook 没有任何问题，先是使用命令 setenforce 0 来临时禁用 selinux，然后修改/etc/selinux/config 文件来永久禁用。但是一运行，发现失败，错误信息如下(内容太长，我调整了下)：

```shell
TASK [disable on the fly] ***************************
fatal: [192.168.200.34]: FAILED! => {
  ...,
  "changed": true,
  "cmd": "setenforce 0",
  ...,
  "msg": "non-zero return code",
  "rc": 1,
  ...,
  "stderr": "setenforce: SELinux is disabled",
  "stderr_lines": ["setenforce: SELinux is disabled"],
  "stdout": "",
  "stdout_lines": []
}
```

从错误信息中不难看出，在使用 shell 模块执行 setenforce 0 命令的时候，发现该命令的退出状态码不是 0，所以 Ansible 认为这是个失败的命令，于是终止了 play，后面的任务也不再执行。但其实我们自己明确地知道，这个命令是没错的，而且从上面的 stderr 的内容中也可以看出，setenforce 命令给了我们正确的反馈。

使用 Ansible 去执行命令，可能经常会遇到类似问题，Ansible 并没有那么聪明，它默认只认退出状态码 0，其它退出状态码全认为是失败的。

所以，需要让 Ansible 去处理这种异常。处理异常的方式有很多种，这里只介绍最简单的一种：ignore_errors，后面的文章中再统一介绍。

ignore_errors 指令正如其名，表示忽略失败的任务，直接将值设置为 true 即可。

```yaml
---
- name: disable selinux
  hosts: new
  gather_facts: false
  tasks:
    - name: disable on the fly
      shell: setenforce 0
      ignore_errors: true

    - name: disable forever in config
      lineinfile:
        path: /etc/selinux/config
        line: "SELINUX=disabled"
        regexp: "^SELINUX="
```

使用 ignore_errors 虽然简单，但不是很友好。各位去执行一下上面的 playbook，会发现它仍然会将报错信息输出在终端或指定的日志文件中，只不过它不会因为任务失败而导致整个 play 的中止。但无论如何，它能达到目标：在遇到错误的时候继续执行下去。

异常处理(比如此处的 ignore_errors)也经常会跟 block 结合使用，因为在 block 级别上设置异常处理，可以处理 block 内部的所有错误。例如：

```yaml
---
- name: disable selinux
  hosts: new
  gather_facts: false
  tasks:
    - block:
        - name: disable on the fly
          shell: setenforce 0

        - name: disable forever in config
          lineinfile:
            path: /etc/selinux/config
            line: "SELINUX=disabled"
            regexp: "^SELINUX="
      ignore_errors: true
```

实际上，几乎所有任务级别的指令(除循环类指令外，比如 loop)都可以写在 block 级别上，它们会拷贝到 Block 内部的所有任务上(也可也看作是 block 内部的任务继承 block 级别上的指令)。例如：

```shell
---
- name: disable selinux
  hosts: new
  gather_facts: false
  tasks:
    - block:
        - shell: ls /tmp/a.log
        - shell: ls /tmp/b.log
      ignore_errors: true
```

上面 block 层次的 ignore_errors 指令，其实拷贝到了两个 shell 任务上，等价于：

```shell
---
- name: disable selinux
  hosts: new
  gather_facts: false
  tasks:
    - block:
        - shell: ls /tmp/a.log
          ignore_errors: true
        - shell: ls /tmp/b.log
          ignore_errors: true
```

## 5.8 配置防火墙

对于一个新的主机，有可能需要配置防火墙规则。

如果使用 Ansible 去远程配置防火墙规则，方式有好几种，比如：

- 可以通过 shell 模块远程执行 iptables 命令；

- 可以在 Ansible 本地端写好 iptables 规则，然后传送到目标节点上通过 iptables-restore 来恢复规则；
- 可以使用 Ansible 提供的 iptables 模块来远程设置防火墙规则。

下面是使用 shell 模块远程执行 iptables 命令来定义基本的防火墙规则。

```yaml
---
- name: set firewall
  hosts: new
  gather_facts: false
  tasks:
    - name: set iptables rule
      shell: |
        # 备份已有规则
        iptables-save > /tmp/iptables.bak$(date +"%F-%T")
        # 给它三板斧
        iptables -X
        iptables -F
        iptables -Z

        # 放行lo网卡和允许ping
        iptables -A INPUT -i lo -j ACCEPT
        iptables -A INPUT -p icmp -j ACCEPT

        # 放行关联和已建立连接的包，放行22、443、80端口
        iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
        iptables -A INPUT -p tcp -m tcp --dport 22 -j ACCEPT
        iptables -A INPUT -p tcp -m tcp --dport 443 -j ACCEPT
        iptables -A INPUT -p tcp -m tcp --dport 80 -j ACCEPT

        # 配置filter表的三链默认规则，INPUT链丢弃所有包
        iptables -P INPUT DROP
        iptables -P FORWARD DROP
        iptables -P OUTPUT ACCEPT
```

也可以使用 Ansible 提供的 iptables 模块来配置，这里不多介绍，如果会配置 iptables，那么这个模块没有任何难度。

上面的示例非常简单，但是可以将其改造得更为完美一点。比如，可以指定要禁用或放行的端口列表，然后通过循环的方式去配置相关规则，还可以指定协议，指定链的默认规则等等。

此处，我做个简单演示，只支持三种操作：

1. 允许用户指定 filter 表中三链的默认规则

2. 允许用户指定 INPUT 链放行的 tcp 端口号列表
3. 允许执行用户指定的 iptables

修改后的 playbook 内容如下：

```yaml
---
- name: set firewall
  hosts: new
  gather_facts: false
  vars:
    allowed_tcp_ports: [22, 80, 443]
    default_policies:
      INPUT: DROP
      FORWARD: DROP
      OUTPUT: ACCEPT
    user_iptables_rule:
      - iptables -A INPUT -p tcp -m tcp --dport 8000 -j ACCEPT
      - iptables -A INPUT -p tcp -m tcp --dport 8080 -j ACCEPT

  tasks:
    - block:
        - name: backup and empty rules
          shell: |
            # 备份已有规则，并清空规则等
            iptables-save > /tmp/iptables.bak$(date +"%F-%T")
            iptables -X
            iptables -F
            iptables -Z

        - name: green light for lo interface and icmp protocol
          shell: |
            # 放行lo接口、ping和已建立连接的包
            iptables -A INPUT -i lo -j ACCEPT
            iptables -A INPUT -p icmp -j ACCEPT
            iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

        # 放行用户指定的tcp端口列表
        - name: allow for given tcp port
          shell: iptables -A INPUT -p tcp -m tcp --dport {{item}} -j ACCEPT
          loop: "{{ allowed_tcp_ports | default([]) }}"

        # 执行用户自定义的iptables命令
        - name: execute user iptables command
          shell: "{{item}}"
          loop: "{{user_iptables_rule | default([]) }}"

        # 设置filter表三链的默认规则
        - name: default policies for filter table
          shell: iptables -P {{item.key}} {{item.value}}
          loop: "{{ query('dict', default_policies | default({})) }}"
```

上面示例中，定义了三个变量，分别用于存放用户指定要放行的 tcp 端口、filter 表中三链的默认规则以及用户自定义的 iptables 命令。这没什么可解释的，但需要注意的是有些 task 中使用了类似 {{ allowed_tcp_ports | default([]) }} 的代码，这表示当变量 allowed_tcp_ports 未定义时，则将该变量设置为空列表作为其默认值，以免因变量未定义而 loop 循环出错。对于本示例来说，直接使用 {{ allowed_tcp_ports }} 也是健壮的代码。

## 5.9 远程修改 sshd 配置文件并重启

有时候为了服务器的安全，可能会去修改目标节点上 sshd 服务的默认配置，比如禁止 Root 用户登录、禁止密码认证登录而只允许使用 ssh 密码认证等。

在修改服务的配置文件时，一般有几种方法：

1. 通过远程执行 sed 等命令进行修改配置文件

2. 通过 lineinfile 模块去修改配置文件
3. 在 Ansible 本地端写好配置文件，然后使用 copy 模块或 template 模块传输到目标节点上

相对来说，第三种方案是最统一、最易维护的方案。

此外，对于服务进程来说，修改了配置文件往往意味着要重启服务，使其加载新的配置文件。对于 sshd 也一样如此，但是 sshd 要比其它服务特殊一点，因为 Ansible 默认基于 ssh 连接，重启 sshd 服务会使得 Ansible 连接断开，好在 Ansible 默认会重试建立连接，无非是多等待几秒钟。但重建连接有可能会失败，比如修改了配置文件不允许重试、修改了 sshd 的监听端口等，这可能会使得 Ansible 因连接失败而无法再继续执行后续任务。

所以，在修改 sshd 配置文件时，个人建议：

1. 将此任务作为初始化服务器的最后一个任务，即使连接失败也无所谓。这也是我将这个配置任务放在本文最后介绍的原因所在

2. 在 playbook 中加入连接失败的异常处理
3. 如果目标节点修改了 sshd 端口号，建议通过 Ansible 自动或我们自己手动去修改 inventory 文件中的 SSH 连接端口号

这里为了简单，我准备采用 lineinfile 模块去修改配置文件，要修改的内容只有两项：

1. 将 PermitRootLogin 指令设置为 no，禁止 root 用户直接登录

2. 将 PasswordAuthentication 指令设置为 no，不允许使用密码认证的方式登录

playbook 内容如下：

```yaml
---
- name: modify sshd_config
  hosts: new
  gather_facts: false
  tasks:
    # 1. 备份/etc/ssh/sshd_config文件
    - name: backup sshd config
      shell: /usr/bin/cp -f {{path}} {{path}}.bak
      vars:
        - path: /etc/ssh/sshd_config

    # 2. 设置PermitRootLogin no
    - name: disable root login
      lineinfile:
        path: "/etc/ssh/sshd_config"
        line: "PermitRootLogin no"
        insertafter: "^#PermitRootLogin"
        regexp: "^PermitRootLogin"
      notify: "restart sshd"

    # 3. 设置PasswordAuthentication no
    - name: disable password auth
      lineinfile:
        path: "/etc/ssh/sshd_config"
        line: "PasswordAuthentication no"
        regexp: "^PasswordAuthentication yes"
      notify: "restart sshd"

  handlers:
    - name: "restart sshd"
      service:
        name: sshd
        state: restarted
```

执行：

```shell
$ ansible-playbook sshd_config.yaml
```

上面的示例中，唯一需要解释的是两个 lineinfile 任务中的 notify 指令以及最后面的 handlers。

### 5.9.1 notify 和 handlers

此前曾提到过，Ansible 绝大多数的模块都具备幂等性，它能保证多次执行不会改变结果，这使得 Ansible 执行任务时更为安全。

Ansible 的模块如何去保证幂等性呢？以 shell 模块为例：

```yaml
shell:
  cmd: echo haha >> /tmp/only_once.txt
  creates: /tmp/only_once.txt
```

这里使用了 shell 模块中的 creates 参数，它表示如果其指定的文件/tmp/only_once.txt 存在，则不执行 shell 命令，只有该文件不存在时才执行。这就是保证幂等性的一种体现：既然不能保证多次执行 shell 命令的结果不变，那就保证只执行一次。

在此有一个重要的概念，叫做 changed，其实在 Ansible 的执行结果中经常会看到这个字眼。从广义的角度上理解，changed 用于表示目标状态是否发生了改变，通俗而不严谨地讲，就是本次任务是否执行了或执行了是否影响结果。

例如，第一次执行上面的 shell 任务时，由于/tmp/only_once.txt 文件不存在，所以会执行 shell 命令，执行完之后创建了这个文件，于是它关注的目标文件存在性状态发生了改变，这时会设置 changed=1。如果再次执行该任务，因其关注的文件已存在，所以它不会执行 shell 命令，其关注的目标存在性状态未改变，这时会设置 changed=0。

再比如，从 Ansible 端拷贝 nginx 的配置文件到目标主机上，如果所拷贝配置文件的内容和原有的配置文件内容不同，则表示其关注的状态(即文件内容，其实是比较 md5 值)发生了改变，如果内容相同，则表示其关注的状态未发生改变。而配置文件内容发生改变，往往伴随着重启 nginx 服务的操作。

Ansible 会监控 changed 的状态，如果 changed=1,则表示关注的状态发生了改变，即本次任务的执行不具备幕等性，如果 changed=0,则表示本次任务要么没执行，要么执行了也没有影响，即本次任务具备幕等性。

Ansible 提供了 notify 指令和 handlers 功能。如果在某个 task 中定义了， notify 指令，当 Ansible 在监控到该任务 changed=1 时，会触发该 notify 指令所定义的 handler,然后去执行 handler(不是触发后立即执行，稍后会解释)。所谓 handler,其实就是 task,无论在写法上还是作用上它和 task 都没有区別，唯一的区別在于 handler 是被触发而被动执行的，不像普通 task 一样会按流程正常执行。

回头看看修改 sshd 配置文件的示例，就很容易理解：

```yaml
...
  tasks:
    ...
    - name: disable root login
      lineinfile: ...
      notify: "restart sshd"  ----------┐
                                        |
    - name: disable password auth       |
      lineinfile: ...                   |
      notify: "restart sshd"   ---┐     |
                                  |     |
  handlers:                       |     |
    - name: "restart sshd"  <-----┘-----┘
      service:
        name: sshd
        state: restarted
```

当 lineinfile 任务确实修改了/etc/ssh/sshd_config 文件中的内容时，Ansible 会监控到它的 changed=1，于是会根据该任务中 notify 指令指定的名称”restart sshd”去寻找 handlers 中对应该名称”restart sshd”的 handler，然后去执行找到的 handler。

不难看出，handlers 中定义的就是任务，此处 handlers 中的任务使用的是 service 模块，它用于管理服务(sysV 或 systemd 服务都能管理)，比如设置服务是否开机自启，启动、停止、重启服务等。

唯一需要注意的是，notify 和 handlers 中任务的名称必须一致。比如 notify: "restart nginx"，那么 handlers 中必须得有一个任务设置了 name: restart nginx。

此外，在上面的示例中，两个 lineinfile 任务都设置了相同的 notify，但 Ansible 不会多次去重启 sshd，而是在最后重启一次。实际上，Ansible 在执行完某个任务之后并不会立即去执行对应的 handler，而是在当前 play 中所有普通任务都执行完后再去执行 handler，这样的好处是可以多次触发 notify，但最后只执行一次对应的 handler，从而避免多次重启。

关于 handler，暂时只介绍这么多，在后面的文章中再对其进行扩充。

## 5.10 整合所有任务到单个 playbook 中

这里将前面所有案例的 playbook 集合到单个 playbook 文件中去，这样就可以一次性执行所有任务。

```yaml
---
- name: Configure ssh Connection
  hosts: new
  gather_facts: false
  connection: local
  tasks:
    - name: configure ssh connection
      shell: |
        ssh-keyscan {{inventory_hostname}} >>~/.ssh/known_hosts
        sshpass -p'123456' ssh-copy-id root@{{inventory_hostname}}

- name: Set Hostname
  hosts: new
  gather_facts: false
  vars:
    hostnames:
      - host: 192.168.200.34
        name: new1
      - host: 192.168.200.35
        name: new2
  tasks:
    - name: set hostname
      hostname:
        name: "{{item.name}}"
      when: item.host == inventory_hostname
      loop: "{{hostnames}}"

- name: Add DNS For Each
  hosts: new
  gather_facts: true
  tasks:
    - name: add DNS
      lineinfile:
        path: "/etc/hosts"
        line: "{{item}} {{hostvars[item].ansible_hostname}}"
      when: item != inventory_hostname
      loop: "{{ play_hosts }}"

- name: Config Yum Repo And Install Software
  hosts: new
  gather_facts: false
  tasks:
    - name: backup origin yum repos
      shell:
        cmd: "mkdir bak; mv *.repo bak"
        chdir: /etc/yum.repos.d
        creates: /etc/yum.repos.d/bak

    - name: add os repo and epel repo
      yum_repository:
        name: "{{item.name}}"
        description: "{{item.name}} repo"
        baseurl: "{{item.baseurl}}"
        file: "{{item.name}}"
        enabled: 1
        gpgcheck: 0
        reposdir: /etc/yum.repos.d
      loop:
        - name: os
          baseurl: "https://mirrors.tuna.tsinghua.edu.cn/centos/$releasever/os/$basearch"
        - name: epel
          baseurl: "https://mirrors.tuna.tsinghua.edu.cn/epel/$releasever/$basearch"

    - name: install pkgs
      yum:
        name: lrzsz,vim,dos2unix,wget,curl
        state: present

- name: Sync Time
  hosts: new
  gather_facts: false
  tasks:
    - name: install and sync time
      block:
        - name: install ntpdate
          yum:
            name: ntpdate
            state: present

        - name: ntpdate to sync time
          shell: |
            ntpdate ntp1.aliyun.com
            hwclock -w

- name: Disable Selinux
  hosts: new
  gather_facts: false
  tasks:
    - block:
        - name: disable on the fly
          shell: setenforce 0

        - name: disable forever in config
          lineinfile:
            path: /etc/selinux/config
            line: "SELINUX=disabled"
            regexp: "^SELINUX="
      ignore_errors: true

- name: Set Firewall
  hosts: new
  gather_facts: false
  tasks:
    - name: set iptables rule
      shell: |
        # 备份已有规则
        iptables-save > /tmp/iptables.bak$(date +"%F-%T")
        # 给它三板斧
        iptables -X
        iptables -F
        iptables -Z

        # 放行lo网卡和允许ping
        iptables -A INPUT -i lo -j ACCEPT
        iptables -A INPUT -p icmp -j ACCEPT

        # 放行关联和已建立连接的包，放行22、443、80端口
        iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
        iptables -A INPUT -p tcp -m tcp --dport 22 -j ACCEPT
        iptables -A INPUT -p tcp -m tcp --dport 443 -j ACCEPT
        iptables -A INPUT -p tcp -m tcp --dport 80 -j ACCEPT

        # 配置filter表的三链默认规则，INPUT链丢弃所有包
        iptables -P INPUT DROP
        iptables -P FORWARD DROP
        iptables -P OUTPUT ACCEPT

- name: Modify sshd_config
  hosts: new
  gather_facts: false
  tasks:
    - name: backup sshd config
      shell: /usr/bin/cp -f {{path}} {{path}}.bak
      vars:
        - path: /etc/ssh/sshd_config

    - name: disable root login
      lineinfile:
        path: "/etc/ssh/sshd_config"
        line: "PermitRootLogin no"
        insertafter: "^#PermitRootLogin"
        regexp: "^PermitRootLogin"
      notify: "restart sshd"

    - name: disable password auth
      lineinfile:
        path: "/etc/ssh/sshd_config"
        line: "PasswordAuthentication no"
        regexp: "^PasswordAuthentication yes"
      notify: "restart sshd"

  handlers:
    - name: "restart sshd"
      service:
        name: sshd
        state: restarted
```

内容很长，可能你也感受到了，可读性很差，维护也很不方便。

更友好的一种组织方式是将各个任务分类，各自存放在不同的 playbook 文件中(就像未整合那样)，然后使用一个入口 playbook 文件引入所有任务文件。

例如：

- 配置主机 SSH 连接互信的任务放在 sshkey.yaml

- 设置主机名的任务放在 hostname.yaml
- 互相添加 DNS 解析记录的任务放在 add_dns.yaml
- 配置 yum 源并安装常用软件包的任务放在 add_repos.yaml
- 时间同步的任务放在 synctime.yaml
- 禁用 selinux 的任务放在 disable_selinux.yaml
- 配置防火墙的任务放在 iptables.yaml
- 修改 sshd 配置文件的任务放在 sshd_config.yaml

因为这些任务都是初始化服务器的任务，所以将这些任务文件共同存在一个单独的目录中(比如 init_server 目录)，则效果更佳。

最后，通过一个入口文件引入所有这些任务文件将它们组织起来。假设入口文件名为 main.yaml，其内容为：

```yaml
---
- import_playbook: "init_server/sshkey.yaml"
- import_playbook: "init_server/hostname.yaml"
- import_playbook: "init_server/add_dns.yaml"
- import_playbook: "init_server/add_repos.yaml"
- import_playbook: "init_server/synctime.yaml"
- import_playbook: "init_server/disable_selinux.yaml"
- import_playbook: "init_server/iptables.yaml"
- import_playbook: "init_server/sshd_config.yaml"
```

然后使用 ansible-playbook 去执行这个入口 playbook 文件即可：

```shell
$ ansible-playbook main.yaml
```

如此一来，各个任务自治，维护起来更为容易。当然，这只是小菜，敬请期待后面的文章，届时我将专门介绍组织文件的方式以及 Ansible Role。
