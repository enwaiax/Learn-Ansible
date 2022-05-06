# Learn-Ansible

本仓库仅用于个人学习 ansible 的记录，不做任何商业用途

所有内容来源于博客 [骏马金龙](https://www.junmajinlong.com/ansible/index/)，有细微改动

## 开始阅读

- [Github](Chapters/SUMMARY.md)

## 目录

1. [学习不迷茫：Ansible 要如何学至精通](Chapters/Chapter01.md)
2. [初入 Ansible 世界：用法概览和初体验](Chapters/Chapter02.md)
3. [制定演员表：inventory](Chapters/Chapter03.md)
4. [嘿，瞧瞧 Ansible 的灵魂：playbook](Chapters/Chapter04.md)
5. [Ansible 力量初显：批量初始化服务器](Chapters/Chapter05.md)
6. [更大的舞台(1)：组织多个文件以及 Role](Chapters/Chapter06.md)
7. [更大的舞台(2)：利用 Role 部署 LNMP 案例](Chapters/Chapter07.md)
8. [回归 Ansible 并进阶：变量、条件、循环、异常处理及其它](Chapters/Chapter08.md)
9. [如虎添翼的力量：解锁强大的 Jinja2 模板](Chapters/Chapter09.md)
10. [服务 0 downtime 的追求：Haproxy+Nginx 集群服务的滚动发布和节点伸缩](Chapters/Chapter10.md)
11. [Ansible 你快点：Ansible 执行过程分析、异步、效率优化](Chapters/Chapter11.md)
12. [让 Ansible 更安全：使用 Vault 进行加密](Chapters/Chapter12.md)
13. [蚂蚁多了也咬不死 Ansible：Ansible Tower](Chapters/Chapter13.md)
14. [Ansible 管理 docker 和 openstack](Chapters/Chapter14.md)
15. [意外之喜：Ansible 管理 Windows 主机](Chapters/Chapter15.md)
16. [成就感源于创造：自己动手写 Ansible 模块](Chapters/Chapter16.md)

## 完整内容列表

```
1.学习不迷茫：Ansible要如何学至精通
  1.1 三分钟内我要Ansible的所有资料
  1.2 如何学习并学好Ansible
2.初入Ansible世界：用法概览和初体验
  2.1 测试环境并配置ssh主机互信
    2.1.1 安装最新版的Ansible
    2.1.2 Ansible参数补全功能
    2.1.3 配置主机互信
  2.2 Ansible初体验
  2.3 Ansible配置文件
3.制定演员表：inventory
  3.1 inventory文件路径
  3.2 配置inventory
    3.2.1 一行一主机的定义方式
    3.2.2 inventory中的普通变量
    3.2.3 主机分组
    3.2.4 主机组变量
    3.2.5 组嵌套
    3.2.6 多个inventory文件
  3.3 动态inventory和临时添加主机
  3.4 本文使用的inventory文件
4.嘿，瞧瞧Ansible的灵魂：playbook
  4.1 playbook、play和task的关系
  4.2 playbook的语法：YAML
    4.2.1 对象
    4.2.2 数组
    4.2.3 字典
    4.2.4 复合结构
    4.2.5 字符串续行
    4.2.6 空值
    4.2.7 YAML中的单双引号和转义
  4.3 playbook的写法
  4.4 playbook模块参数的传递方式
  4.5 指定执行play的目标主机
  4.6 默认的任务执行策略
5.Ansible力量初显：批量初始化服务器
  5.1 批量初始化服务器案例
  5.2 Ansible配置SSH密钥认证
  5.2.1 方案一：命令行改写为playbook
    command、shell、raw和script模块
    connection和delegate_to指令
  5.2.2 方案二：使用authorized_key模块
    lookup()或query()读取外部数据
  5.3 配置主机名
    5.3.1 vars设置变量
    5.3.2 when条件判断
    5.3.3 loop循环
  5.4 互相添加DNS解析记录
    5.4.1 lineinfile模块
    5.4.2 play_hosts和hostvars变量
  5.5 配置yum镜像源并安装软件
  5.6 时间同步
  5.7 关闭Selinux
  5.8 配置防火墙
  5.9 远程修改sshd配置文件并重启
    5.9.1 notify和handlers
  5.10 整合所有任务到单个playbook中
6.更大的舞台(1)：组织多个文件以及Role
  6.1 使用include还是import？
  6.2 组织task
  6.2.1 在循环中include文件
  6.3 组织handler
  6.4 组织变量
    6.4.1 vars_files
    6.4.2 include_vars
    6.4.3 --extra-vars选项
  6.5 组织playbook文件
  6.6 更为规范的组织方式：Role
    6.6.1 Role文件结构一览
    6.6.2 定义Role的task
    6.6.3 定义Role的handler
    6.6.4 定义Role的变量
    6.6.5 Role用到的外部文件和模板文件
    6.6.6 Role中定义的模块和插件
    6.6.7 定义Role的依赖关系
    6.6.8 动手写一个Role
  6.7 使用Role：roles、include_role和import_role
  6.8 查看任务和打标签tags
  6.9 Ansible Galaxy和Collection
    Ansible Collection
  6.10 playbook的执行顺序
    6.10.1 playbook解析、动态加载和静态加载
7.更大的舞台(2)：利用Role部署LNMP案例
  7.1 实验环境说明
  7.2 创建Role并提供inventory
  7.3 配置yum镜像源
  7.4 安装并配置Nginx
  7.5 整理nginx的众多任务
  7.6 安装并配置PHP
  7.7 安装并配置MySQL
  7.8 任务分离到多个Role中
    7.8.1 common Role
    7.8.2 Nginx Role
    7.8.3 PHP Role
    7.8.4 MySQL Role
  7.9 为每个Role提供一个playbook
8.回归Ansible并进阶：变量、条件、循环、异常处理及其它
  8.1 inventory的进阶
    8.1.1 inventory解析
    8.1.2 inventory变量文件：host_vars和group_vars
    8.1.3 动态inventory
    8.1.4 临时添加节点：add_host模块
    8.1.5 group_by运行时临时设置主机组
    8.1.6 --limit再次限制目标主机
  8.2 收集目标节点的信息：Facts
    8.2.1 如何收集Facts信息？
    8.2.2 如何访问Facts信息？
    8.2.3 local Facts
    8.2.4 委托Facts
    8.2.5 set_fact模块
    8.2.6 Facts缓存
  8.3 Ansible变量的进阶
    8.3.1 访问列表、字典变量的两种方式
    8.3.2 --extra-vars选项定义额外变量
    8.3.3 inventory变量
    8.3.4 Role变量
    8.3.5 play变量
    8.3.6 task变量
    8.3.7 block变量
    8.3.8 Facts信息变量
    8.3.9 预定义特殊变量
    8.3.10 变量作用域
  8.4 YAML纯文本裸字符串
  8.5 handler的进阶
  8.6 when条件判断
    8.6.1 同时满足多个条件
    8.6.2 按条件导入文件
    8.6.3 when和循环
  8.7 循环迭代的进阶
    8.7.1 with_list
    8.7.2 with_items和with_flattened
    8.7.3 with_indexed_items
    8.7.4 with_dict
    8.7.5 with_together
    8.7.6 with_sequence
    8.7.7 with_subelements
    8.7.8 with_nested
    8.7.9 with_random_choice
    8.7.10 with_fileglob
    8.7.11 with_lines
    8.7.12 循环和when
    8.7.13 循环和register
    8.7.14 循环的控制：loop_control
      1.label参数
      2.pause参数
      3.index_var参数
      4.loop_var参数
      5.extended参数
  8.8 异常和错误处理
    8.8.1 人为制造失败：fail模块
    8.8.2 断言：assert模块
    8.8.3 ignore_errors
    8.8.4 failed_when
    8.8.5 rescue和always
    8.8.6 any_errors_fatal
    8.8.7 max_fail_percentage
    8.8.8 处理连接失败(unreachable)的异常
    8.8.9 任务失败导致handler未执行
  8.9 其它Ansible流程控制逻辑的进阶
    8.9.1 until和retry
    8.9.2 pause模块暂停、休眠
    8.9.3 wait_for模块和wait_for_connection模块
9.如虎添翼的力量：解锁强大的Jinja2模板
  9.1 Jinja2是什么？模板是什么？
  9.2 Ansible哪里使用了Jinja2
  9.3 Jinja2访问元素的两种方式
  9.4 Jinja2条件判断
    9.4.1 if语句块
    9.4.2 行内if表达式
  9.5 for循环
    9.5.1 for迭代列表
    9.5.2 for迭代字典
    9.5.3 for的特殊控制变量
  9.6 Macro
  9.7 block
  9.8 变量赋值和作用域
  9.8.1 如何跨作用域
  9.9 Jinja2的空白处理
  9.10 真实案例：完全自定义的nginx虚拟主机配置
  9.11 基本运算符
  9.12 Jinja2内置的is测试
  9.13 Ansible扩展的测试函数
    9.13.1 测试字符串
    9.13.2 版本号大小比较
    9.13.3 子集、父集测试
    9.13.4 成员测试
    9.13.5 测试文件
    9.13.6 测试任务的执行状态
  9.14 Jinja2内置Filter
  9.15 Ansible扩展的Filter
    9.15.1 类型转换类筛选器
    9.15.2 获取当前时间点
    9.15.3 YAML、JSON格式化
    9.15.4 参数忽略
    9.15.5 列表元素连接
    9.15.6 列表压平
    9.15.7 并集、交集、差集
    9.15.8 dict和list转换
    9.15.9 zip和zip_longest
    9.15.10 子元素subelements
    9.15.11 random生成随机数
    9.15.12 shuffle打乱顺序
    9.15.13 json_query
    9.15.14 ip地址筛选
    9.15.15 正则表达式筛选器
    9.15.16 URL处理筛选器
    9.15.17 编写注释的筛选器
    9.15.18 extract提取元素
    9.15.19 dict合并
    9.15.20 hash值计算
    9.15.21 base64编解码筛选器
    9.15.22 文件名处理
    9.15.23 日期时间类处理
    9.15.24 human_to_bytes和human_readable
    9.15.25 其它筛选器
  9.16 Python对象自身的方法
    9.16.1 Python字符串对象方法
    9.16.2 list对象方法
10.服务0 downtime的追求：Haproxy+Nginx集群服务的滚动发布和节点伸缩
  10.1 环境说明
  10.2 安装配置HAProxy Role
  10.3 安装配置Nginx Role
  10.4 滚动发布：Load Balance后端节点升降级
  10.5 理解Ansible执行策略
    10.5.1 Ansible的默认执行策略
    10.5.2 serial将节点分批执行play
    10.5.3 strategy指定执行策略
    10.5.4 throttle限制最大并发任务数
  10.6 Ansible完成滚动发布
  10.7 集群后端节点的伸缩
  10.8 处理Ansible部署、维护过程中的异常
11.Ansible你快点：Ansible执行过程分析、异步、效率优化
  11.1 测量任务执行速度：profile_tasks插件
  11.2 Ansible执行流程分析
  11.3 回顾Ansible的执行策略
  11.4 加大forks的值
  11.5 修改执行策略
  11.6 使Ansible异步执行任务
    11.6.1 async和poll指令
    11.6.2 等待异步任务
    11.6.3 何时使用异步任务
  11.7 开启ssh长连接
    11.7.1 开启ssh长连接后的注意事项
  11.8 开启Pipelining
    11.8.1 开启Pipelining后的注意事项
    11.8.2 开启Pipelining后的执行流程
    11.8.3 开启和不开启Pipelining的效率比较
  11.9 修改facts收集行为
  11.10 Shell层次上的优化：将任务分开执行
  11.11 第三方策略插件：Mitogen for Ansible
12.让Ansible更安全：使用Vault进行加密
  12.1 一个入门示例：创建一个加密文件
  12.2 Vault ID和凭据密码提供方式
  12.3 加密已存在的文件
  12.4 加密协议头的含义
  12.5 解密已加密的文件
  12.6 修改Vault ID和凭据密码
  12.7 编辑已加密文件的内容
  12.8 ansible-playbook任务中使用Vault加密文件
  12.9 加密字符串并嵌入YAML文件
  12.10 加速加解密的过程
  12.11 Vault加密最佳实践
13.蚂蚁多了也咬不死Ansible：Ansible Tower
  13.1 使用Ansible之痛
  13.2 Ansible Tower简介
  13.3 安装Tower
    13.3.1 安装环境配置
    13.3.2 安装Tower
    13.3.3 登录tower
  13.4 让Ansible Tower执行playbook
    13.4.1 配置Tower Credential
    13.4.2 配置inventory
    13.4.3 创建Project
    13.4.4 创建Template并运行任务
  13.5 Tower的用户和权限控制
14.Ansible管理docker和openstack
  14.1 Ansible管理docker
    14.1.1 Ansible构建并运行Docker镜像
    14.1.2 无Dockerfile启动镜像并连接容器
    14.1.3 docker inventory
    14.1.4 其它Ansible容器管理工具
  14.2 Ansible管理OpenStack
    14.2.1 创建虚拟机
    14.2.2 将新虚拟机动态添加到inventory
    14.2.3 收集OpenStack虚拟机的动态inventory
15.意外之喜：Ansible管理Windows主机
  15.1 Ansible如何管理Windows
  15.2 Ansible管理Windows前的设置
  15.3 测试能否成功管理Windows
  15.4 Ansible执行PowerShell、CMD命令
  15.5 创建域控制器
  15.6 将主机加入域
16.成就感源于创造：自己动手写Ansible模块
    16.1 自定义模块简介
    16.1.1 自定义模块前须知：模块的存放路径
    16.1.2 自定义模块前须知：模块的要求
  16.2 Shell脚本自定义模块(一)：Hello World
  16.3 Shell脚本自定义模块(二)：简版file模块
  16.4 Python自定义模块
```
