# 2. 初入 Ansible 世界：用法概览和初体验

- [2. 初入 Ansible 世界：用法概览和初体验](#2-初入-ansible-世界用法概览和初体验)
  - [2.1 测试环境并配置 ssh 主机互信](#21-测试环境并配置-ssh-主机互信)
    - [2.1.1 安装最新版的 Ansible](#211-安装最新版的-ansible)
    - [2.1.2 Ansible 参数补全功能](#212-ansible-参数补全功能)
    - [2.1.3 配置主机互信](#213-配置主机互信)
  - [2.2 Ansible 初体验](#22-ansible-初体验)
  - [2.3 Ansible 配置文件](#23-ansible-配置文件)

Ansible 是一个自动化管理工具，它的用法可以非常简单，只需学几个基本的模块就能完成一些简单的自动化任务，它的用法也可以非常难，不仅需要学习大量 Ansible 知识，还需要大量的实际应用去熟悉最优化、最完美的自动化管理逻辑。比较悲催的是 Ansible 的知识体系比较庞大，它的知识板块也比较零散，想要构建一个比较完善的 Ansible 知识体系确实稍有难度。

但无论如何，千里之行始于足下，学习任何一个新知识，总归要从最基本的用法循序渐进地深入，并辅以逐渐展开的宏观结构，加上学习过程中的不断练习，最终构建出完善的知识体系。

因 Ansible 的基础知识内容较多，本文先介绍最基本的概念和最基本的用法，让大家对 Ansible 的用法以及功能有一个基本且系统性的认识。之后的文章再逐步探讨 Ansible 如何应用在配置管理上。

## 2.1 测试环境并配置 ssh 主机互信

Ansible 的作用是批量控制其它远程主机，并指挥远程主机节点做一些操作、完成一些任务。

所以在这个结构中，分为控制节点和被控制节点。Ansible 是 Agentless 的软件，只需在控制节点安装 Ansible，被控制节点一般不需额外安装任何程序，就像一个普通的命令一样，随装随用。

> Ansible 的模块是用 Python 来执行的，且默认远程连接的方式是 ssh，所以控制端和被控制端都需要有 Python 环境，并且被控制端需要启动 sshd 服务，但通常这两个条件在安装 Linux 系统时就已经具备了。所以使用 Ansible 的安装过程只有一个：在控制端安装 Ansible。

在本文中，将配置如下测试环境：包括一个 Ansible 控制节点和 7 个被控制节点。后面的文章中如果没有特别说明，也都使用此处的主机环境。

| 主机描述     | IP 地址        | 主机名       | 操作系统   | Ansible 版本 |
| ------------ | -------------- | ------------ | ---------- | ------------ |
| control_node | 192.168.200.26 | control_node | CentOS 7.2 | Ansible 2.9  |
| node1        | 192.168.200.27 | node1        | CentOS 7.2 | -            |
| node2        | 192.168.200.28 | node2        | CentOS 7.2 | -            |
| node3        | 192.168.200.29 | node3        | CentOS 7.2 | -            |
| node4        | 192.168.200.30 | node4        | CentOS 7.2 | -            |
| node5        | 192.168.200.31 | node5        | CentOS 7.2 | -            |
| node6        | 192.168.200.32 | node6        | CentOS 7.2 | -            |
| node7        | 192.168.200.33 | node7        | CentOS 7.2 | -            |

所有主机上都已启动 sshd 服务并监听在 22 端口上。

因为有时候也会使用主机名去控制目标节点，所以这里也在 control_node 节点上配置了其余 7 个节点的 DNS 解析，可在 control_node 节点的/etc/hosts 文件中加入如下内容：

```shell
$ cat >>/etc/hosts<<EOF
192.168.200.27 node1
192.168.200.28 node2
192.168.200.29 node3
192.168.200.30 node4
192.168.200.31 node5
192.168.200.32 node6
192.168.200.33 node7
EOF
```

### 2.1.1 安装最新版的 Ansible

Ansible 依赖于 SSH 协议(默认)，只需要在控制节点(即 control_node 主机上)安装一次 Ansible 即可。

各种系统下安装最新版 Ansible 的方式参见官方手册：安装手册。

对于 RHEL 系列的系统来说，配置好 epel 镜像即可安装最新版的 Ansible。在我当前写文章的时候，最新版是 Ansible 2.9 版本。

```shell
$ cat >>/etc/yum.repos.d/epel.repo<<'EOF'
[epel]
name=epel repo
baseurl=https://mirrors.tuna.tsinghua.edu.cn/epel/7/$basearch
enabled=1
gpgcheck=0
EOF
```

然后安装即可：

```shell
$ yum install ansible
```

Ansible 每个版本释放出来之后，都首先提交到 Pypi，所以任何操作系统，都可以使用 pip 工具来安装最新版的 Ansible。

```shell
$ pip3 install ansible
```

但要注意，使用各系统的包管理工具(如 yum)安装 Ansible 时自动会提供一些配置文件，如`/etc/ansible/ansible.cfg`。而使用 pip 安装的 Ansible 默认不提供配置文件。

### 2.1.2 Ansible 参数补全功能

从 Ansible 2.9 版本开始，它支持命令的选项补全功能，它依赖于 python 的 argcomplete 插件。

安装 argcomplete：

```shell
# CentOS/RHEL
yum -y install python-argcomplete

# 任何系统都可以使用pip工具安装argcomplete
pip3 install argcomplete
```

安装完成后，还需激活该插件：

```shell
# 要求bash版本大于等于4.2
sudo activate-global-python-argcomplete

# 如果bash版本低于4.2，则单独为每个ansible命令注册补全功能
eval $(register-python-argcomplete ansible)
eval $(register-python-argcomplete ansible-config)
eval $(register-python-argcomplete ansible-console)
eval $(register-python-argcomplete ansible-doc)
eval $(register-python-argcomplete ansible-galaxy)
eval $(register-python-argcomplete ansible-inventory)
eval $(register-python-argcomplete ansible-playbook)
eval $(register-python-argcomplete ansible-pull)
eval $(register-python-argcomplete ansible-vault)
```

最后，退出当前 Shell 重新进入，或者简单的直接执行如下命令即可：

```shell
exec $SHELL
```

然后就可以按 tab 一次或两次补全参数或提示参数。例如，下面选项输入到一半的时候，按一下 tab 键就会补全得到`ansible --syntax-check`。

### 2.1.3 配置主机互信

因为 Ansible 默认是基于 ssh 连接的，所以要控制其它节点首先需要建立好 ssh 连接，而建立 ssh 连接要么需要提供密码，要么需要配置好认证方式。为了方便后文的测试，这里先配置好 control_node 和其它被控节点之间的主机互信。

为了避免配置主机互信过程中的交互式询问，这里使用 ssh-keyscan 工具添加主机认证信息以及 sshpass 工具(安装 Ansible 时会自动安装 sshpass，也可以 yum -y install sshpass 安装)直接指定 ssh 连接密码。

1. 在 control_node 节点上生成密钥对：

```shell
$ ssh-keygen -t rsa -f ~/.ssh/id_rsa -N ''
```

将各节点的主机信息(host key)写入 control_node 的~/.ssh/known_hosts 文件：

```shell
for host in 192.168.200.{27..33} node{1..7};do
  ssh-keyscan $host >>~/.ssh/known_hosts 2>/dev/null
done
```

将 control_node 上的 ssh 公钥分发给各节点：

```shell
# sshpass -p选项指定的是密码
for host in 192.168.200.{27..33} node{1..7};do
  sshpass -p'123456' ssh-copy-id root@$host &>/dev/null
done
```

配置好 ssh 的主机互信之后，就可以开始体验 Ansible 了。

## 2.2 Ansible 初体验

Ansible 有点类似于 shell 的命令， Shell 下ー个命令实现一个功能，而 Ansible 的作用也是执行任务，只不过它执行的不是命令，而是 Ansible 中提供的模块(Module), 每个模块对应个功能。通常来说，执行一个任务的本质就是执行个模块

Ansible 提供了很多模块（几干个），其中 100 多个核心模块是 Ansible 团队自己维护的，剩余的模块都是 Ansible 社区维护的。有了这几千个模块，基本上想要实现的功能都能够找到对应的模块。即使有些任务比较特殊，实在找不到对应模块， Ansible 也允许用户自定义模块。

好了，第一个关于模块的概念就先介绍这么多，该是体验一下 Ansible 是如何执行任务的时候了。

在控制节点上执行：

```shell
$ ansible localhost -m copy -a 'src=/etc/passwd dest=/tmp'
localhost | CHANGED => {
    "changed": true,
    "checksum": "41a051362a32be1ec4266cc64de2c6e4ad06bc73",
    "dest": "/tmp/passwd",
    "gid": 0,
    "group": "root",
    "md5sum": "f042f8f7c120afd8c54f437944db1108",
    "mode": "0644",
    "owner": "root",
    "size": 1656,
    "src": "/root/.ansible/tmp/ansible-tmp-1575571772.47-60786643831265/source",
    "state": "file",
    "uid": 0
}
```

该命令的作用是拷贝本机的/etc/passwd 文件到本机的/tmp 目录下。

其中的 ansible 是一个命令，这个自不需要多做解释。除 ansible 命令外，后面还会用到 ansible-playbook 命令。

localhost 参数表示 ansible 要控制的节点，即 ansible 将指挥本机执行任务。

执行任务主要是执行模块，模块的执行可以还依赖一些模块参数。在 ansible 命令行中，使用-m Module 来指定要执行哪个模块，即执行什么任务，使用-a ARGS 来指定模块运行时的参数。

本示例中的模块为 copy 模块，传递给 copy 模块的参数包含两项：

- src=/etc/passwd 指定源文件

- dest=/tmp 指定拷贝的目标路径

初学 Ansible，可能会有两个疑惑：

1. Ansible 的模块那么多，我如何知道某个功能要找哪个模块

2. 如何知道某个模块的用法，比如参数有哪些

初学之时，只需学习一些常用的模块，在本文以及后面的文章中会介绍一些常见模块的功能以及用法。熟悉了 Ansible 之后，再按需求到[官方手册](https://docs.ansible.com/ansible/latest/modules/modules_by_category.html)中去寻找。

有时候为了方便快速寻找模块，可以使用`ansible-doc -l | grep 'xxx'`命令来筛选模块。例如，想要筛选具有拷贝功能的模块：

```shell
$ ansible-doc -l | grep 'copy'
vsphere_copy          Copy a file to a VMware datastore
win_copy              Copies files to remote locations on windows hosts
bigip_file_copy       Manage files in datastores on a BIG-IP
ec2_ami_copy          copies AMI between AWS regions, return new image id
win_robocopy          Synchronizes the contents of two directories using Robocopy
copy                  Copy files to remote locations
na_ontap_lun_copy     NetApp ONTAP copy LUNs
icx_copy              Transfer files from or to remote Ruckus ICX 7000 series switches
unarchive             Unpacks an archive after (optionally) copying it from the local machine
postgresql_copy       Copy data between a file/program and a PostgreSQL table
ec2_snapshot_copy     copies an EC2 snapshot and returns the new Snapshot ID
nxos_file_copy        Copy a file to a remote NXOS device
netapp_e_volume_copy  NetApp E-Series create volume copy pairs
```

根据描述，大概找出是否有想要的模块。

找到模块后，想要看看它的功能描述以及用法，可以继续使用 ansible-doc 命令。

```shell
# 详细的模块描述手册
$ ansible-doc copy

# 只包含模块参数用法的模块描述手册
$ ansible-doc -s copy
```

再来一个示例，通过 Ansible 删除本地文件/tmp/passwd。需要使用的模块是 file 模块，file 模块的主要作用是创建或删除文件/目录。

```shell
$ ansible localhost -m file -a 'path=/tmp/passwd state=absent'
```

参数 path=指定要操作的文件路径，state=参数指定执行何种操作，此处指定为 absent 表示删除操作。

Ansible 的很多模块都提供了一个 state 参数，它是一个非常重要的参数。它的值一般都会包含 present 和 absent 两种状态值(并非一定)，不同模块的 present 和 absent 状态表示的含义不同，但通常来说，present 状态表示肯定、存在、会、成功等含义，absent 则相反，表示否定、不存在、不会、失败等含义。

例如这里的 file 模块，absent 状态表示递归删除文件/目录，类似于 rm -r 命令，touch 状态和 touch 命令的功能一样，directory 状态表示递归创建目录，类似于 mkdir -p 命令。

所以，在本地创建文件、创建目录的命令如下：

```shell
# 创建文件
$ ansible localhost -m file -a 'path=/tmp/a.log state=touch'

# 创建目录
$ ansible localhost -m file -a 'path=/tmp/dir1/dir2 state=directory'
```

最后再介绍一个 debug 模块，它用于输出或调试一些数据。正如学习任何一门编程语言都最先执行的代码应该是输出" hello world"。 Ansible 不是编程语言，但也具备语言的部分特性，在学习 Ansible 的过程中，必不可少的需求就是输出调试一些数据。在 Ansible 中，使用 debug 模块完成这项任务。

debug 模块的用法非常简单，就两个常用的参数：msg 参数和 var 参数。这两个参数是互斥的，所以只能使用其中一个。msg 参数可以输出字符串，也可以输出变量的值，var 参数只能输出变量的值。

例如，输出”hello world”，需要使用 msg 参数：

```shell
$ ansible localhost -m debug -a 'msg="hello world"'
localhost | SUCCESS => {
    "msg": "hello world"
}
```

Ansible 中也支持使用变量，这里仅演示最简单的设置变量和引用变量的方式。ansible 命令的`-e`选项或`--extra-vars`选项可以设置变量，设置的方式为`-e 'var1="aaa" var2="bbb"'`

例如，设置变量后使用 debug 的 msg 参数输出：

```
$ ansible localhost -e 'str=world' -m debug -a 'msg="hello {{str}}"'
localhost | SUCCESS => {
    "msg": "hello world"
}
```

注意上面示例中的 msg="hello {{str}}" ，Ansible 的字符串是可以不用引号去包围的，例如 msg=hello 是允许的，但如果字符串中包含了特殊符号，则可能需要使用引号去包围，例如此处的示例出现了会产生歧义的空格。此外，要区分变量名和普通的字符串，需要在变量名上加一点标注：用 {{}} 包围 Ansible 的变量，这其实是 Jinja2 模板(如果不知道，先别管这是什么)的语法。其实不难理解，它的用法和 Shell 下引用变量使用$符号或${}是一样的，例如 echo "hello ${var}"。

debug 模块除了 msg 参数，还有一个 var 参数，它只能用来输出变量(还包括以后要介绍的 Jinja2 表达式)，而且 var 参数引用变量的时候，不能使用 {{}} 包围，因为 var 参数已经隐式地包围了一层 {{}} 。例如：

```
$ ansible localhost -e 'str="hello world"' -m debug -a 'var=str'
localhost | SUCCESS => {
    "str": "hello world"
}
```

体验完 Ansible 模块的基本用法后，下面将简单说说 Ansible 中的配置文件。

## 2.3 Ansible 配置文件

通过操作系统自带的包管理器(比如 yum、dnf、apt)安装的 Ansible 一般都会提供好 Ansible 的配置文件/etc/ansible/ansible.cfg。

这是 Ansible 默认的全局配置文件。实际上，Ansible 支持 4 种方式指定配置文件，它们的解析顺序从上到下：

- ANSIBLE_CFG: 环境变量指定的配置文件

- ansible.cfg: 当前目录下的 ansible.cfg
- ~/.ansible.cfg: 家目录下的 ansible.cfg
- /etc/ansible/ansible.cfg：默认的全局配置文件

Ansible 配置文件采用 ini 风格进行配置，每一项配置都使用 key=value 的方式进行配置。

例如，下面是我从默认的/etc/ansible/ansible.cfg 中截取的[defaults]段落和[inventory]段落的部分配置信息。

```ini
[defaults]
# some basic default values...
#inventory      = /etc/ansible/hosts
#library        = /usr/share/my_modules/
#module_utils   = /usr/share/my_module_utils/
#remote_tmp     = ~/.ansible/tmp
#local_tmp      = ~/.ansible/tmp
#plugin_filters_cfg = /etc/ansible/plugin_filters.yml
#forks          = 5

[inventory]
#enable_plugins = host_list, virtualbox, yaml, constructed
#ignore_extensions = .pyc, .pyo, .swp, .bak, ~, .rpm, .md, .txt, ~, .orig, .ini, .cfg, .retry
#ignore_patterns=
#unparsed_is_failed=False
```

目前没必要去了解配置文件里的每项指令的含义，在后面需要的时候，我会介绍涉及到的相关配置项的含义。
