# 6. 更大的舞台(1)：组织多个文件以及 Role

- [6. 更大的舞台(1)：组织多个文件以及 Role](#6-更大的舞台1组织多个文件以及-role)
  - [6.1 使用 include 还是 import？](#61-使用-include-还是-import)
  - [6.2 组织 task](#62-组织-task)
    - [6.2.1 在循环中 include 文件](#621-在循环中-include-文件)
  - [6.3 组织 handler](#63-组织-handler)
  - [6.4 组织变量](#64-组织变量)
    - [6.4.1 vars_files](#641-vars_files)
    - [6.4.2 include_vars](#642-include_vars)
    - [6.4.3 –extra-vars 选项](#643-extra-vars-选项)
  - [6.5 组织 playbook 文件](#65-组织-playbook-文件)
  - [6.6 更为规范的组织方式：Role](#66-更为规范的组织方式role)
    - [6.6.1 Role 文件结构一览](#661-role-文件结构一览)
    - [6.6.2 定义 Role 的 task](#662-定义-role-的-task)
    - [6.6.3 定义 Role 的 handler](#663-定义-role-的-handler)
    - [6.6.4 定义 Role 的变量](#664-定义-role-的变量)
    - [6.6.5 Role 用到的外部文件和模板文件](#665-role-用到的外部文件和模板文件)
    - [6.6.6 Role 中定义的模块和插件](#666-role-中定义的模块和插件)
    - [6.6.7 定义 Role 的依赖关系](#667-定义-role-的依赖关系)
    - [6.6.8 动手写一个 Role](#668-动手写一个-role)
  - [6.7 使用 Role：roles、include_role 和 import_role](#67-使用-rolerolesinclude_role-和-import_role)
  - [6.8 查看任务和打标签 tags](#68-查看任务和打标签-tags)
  - [6.9 Ansible Galaxy 和 Collection](#69-ansible-galaxy-和-collection)
    - [Ansible Collection](#ansible-collection)
  - [6.10 playbook 的执行顺序](#610-playbook-的执行顺序)
    - [6.10.1 playbook 解析、动态加载和静态加载](#6101-playbook-解析动态加载和静态加载)

在上一篇文章的最后，我将初始化配置服务器的多个任务组合到了单个 playbook 中，这种组织方式的可读性和可维护性都很差，整个 playbook 看上去也非常凌乱。如图：

<img src="images/Chapter06/1577701433835.png" alt="img"  />

所以，我又将各类任务分类后单独存放在各自的 playbook 中，然后在入口 playbook 文件中使用 import_playbook 指令来组织这些 playbook，如此一来，各类任务分门别类且实现了自治，维护起来更为清晰、方便。如图：

![img](images/Chapter06/1577701544968.png)

Ansible 中除了可以将 play 进行分类自治，还提供了其它几种内容的组织方式，可组织的内容包括：

- playbook(或 play)

- task
- variable
- handler(实际上 handler 也是 task，只不过编写在 handlers 指令内部)

此外，Ansible 还提供了更为规范的组织方式：Role 以及 Ansible 2.8 才加入的新功能 Collection。本文将对各种组织文件的方式和 Role 逐一进行探索，并简单介绍 Collection。

## 6.1 使用 include 还是 import？

将各类文件分类存放后，最终需要在某个入口文件去汇集引入这些外部文件。加载这些外部文件通常可以使用 include 指令、include_xxx 指令和 import_xxx 指令，其中 xxx 表示内容类型。

在早期 Ansible 版本，组织文件的方式均使用 include 指令，但随着版本的更迭，Ansible 对这方面做了更为细致的区分。虽然目前仍然支持 include，但早已纳入废弃的计划，所以现在不要再使用 include 指令，在后文中我也不会使用 include 指令。

对于 playbook(或 play)或 task，可以使用 include_xxx 或 import_xxx 指令：

- include_tasks 和 import_tasks 用于引入外部任务文件；

- import_playbook 用于引入 playbook 文件；
- include 可用于引入几乎所有内容文件，但建议不要使用它；

对于`handler`，因为它本身也是`task`，所以它也能使用`include_tasks`、`import_tasks`来引入，但是这并不是想象中那么简单，后文再细说。

对于`variable`，使用`include_vars`(这是核心模块提供的功能)或其它组织方式(如`vars_files`)，没有对应的`import_vars`。

对于后文要介绍的`Role`，使用`include_role`或`import_role`或`roles`指令。

既然某类内容文件既可以使用`include_xxx`引入，也可以使用`import_xxx`引入，那么就有必要去搞清楚它们有什么区别。本文最后我会详细解释它们，现在我先把结论写在这：

- `include_xxx`指令是在遇到它的时候才加载文件并解析执行，所以它是动态解析的；
- `import_xxx`是在解析`playbook`的时候解析的，也就是说在执行`playbook`之前就已经解析好了，所以它也称为静态加载。

## 6.2 组织 task

在此前的所有示例中，一直都是将所有任务编写在单个 playbook 文件中。但 Ansible 允许将任务分离到不同的文件中，然后去引入外部任务文件。

用示例来解释会非常简单。假设，两个 playbook 文件 pb1.yml 和 pb2.yml。

pb1.yml 文件内容如下：

```yaml
---
- name: play1
  hosts: localhost
  gather_facts: false
  tasks:
    - name: task1 in play1
      debug:
        msg: "task1 in play1"

    # - include_tasks: pb2.yml
    - import_tasks: pb2.yml
```

pb2.yml 文件内容如下：

```yaml
- name: task2 in play1
  debug:
    msg: "task2 in play1"

- name: task3 in play1
  debug:
    msg: "task3 in play1"
```

执行 pb1.yml：

```shell
$ ansible-playbook pb1.yml
```

上面是在 pb1.yml 文件中通过 import_tasks 引入了额外的任务文件 pb2.yml，对于此处来说，将 import_tasks 替换成 include_tasks 也能正确工作，不会有任何影响。

但如果是在循环中(比如 loop)，则只能使用`include_tasks`而不能再使用`import_tasks`。

### 6.2.1 在循环中 include 文件

修改 pb1.yml 和 pb2.yml 文件内容：

pb1.yml 内容如下，注意该文件中的 include_tasks 指令：

```yaml
---
- name: play1
  hosts: localhost
  gather_facts: false
  tasks:
    - name: task1 in play1
      debug:
        msg: "task1 in play1"

    - name: include two times
      include_tasks: pb2.yml
      loop:
        - ONE
        - TWO
```

pb2.yml 内容如下，注意该文件中的变量引用：

```yaml
- name: task2 in play1
  debug:
    msg: "task2 in {{item}}"
```

执行 pb1.yml 文件，观察执行结果：

```shell
$ ansible-playbook pb1.yml

TASK [task1 in play1] ************************
ok: [localhost] => {
    "msg": "task1 in play1"
}

TASK [include two times] *********************
included: /root/ansible/pb2.yml for localhost
included: /root/ansible/pb2.yml for localhost

TASK [task2 in play1] ************************
ok: [localhost] => {
    "msg": "task2 in ONE"
}

TASK [task2 in play1] ************************
ok: [localhost] => {
    "msg": "task2 in TWO"
}
```

上面是在 loop 循环中加载两次 pb2.yml 文件，该文件中的任务被执行了两次，并且在 pb2.yml 中能够引用外部文件(pb1.yml)中定义的变量。

分析一下上面的执行流程：

1. 解析 playbook 文件 pb1.yml
2. 执行第一个 play
3. 当执行到 pb1.yml 中的第二个任务时，该任务在循环中， 且其作用是加载外部任务文件 pb2.yml
4. 开始循环，每轮循环都加载、解析并执行 pb2.yml 文件中的所有任务
5. 退出

正是因为`include_tasks`指令是在遇到它的时候才进行加载解析以及执行，所以在 pb2.yml 中才能使用变量。

如果将上面 loop 循环中的`include_tasks`换成`import_tasks`呢？语法会报错，后面我会详细解释。

## 6.3 组织 handler

handler 其本质也是 task，所以也可以使用 include_tasks 或 import_tasks 来加载外部任务文件。但是它们引入 handler 任务文件的方式有很大的差别。

先看 include_tasks 引入 handler 任务文件的示例：

pb1.yml 的内容：

```yml
---
- name: play1
  hosts: localhost
  gather_facts: false
  handlers:
    - name: h1
      include_tasks: handler1.yml

  tasks:
    - name: task1 in play1
      debug:
        msg: "task1 in play1"
      changed_when: true
      notify:
        - h1
```

注意在 tasks 的任务中加了一个指令 changed_when: true，它用来强制指定它所在任务的 changed 状态，如果条件为真，则 changed=1，否则 changed=0。使用这个指令是因为 debug 模块默认不会引起 changed=1 行为，所以只能使用该指令来强制其状态为 changed=1。

当 Ansible 监控到了 changed=1，notify 指令会生效，它会去触发对应的 handler，它触发的 handler 的名称是 handler1，其作用是使用 include_tasks 指令引入 handler1.yml 文件。

下面是 handler1.yml 文件的内容：

```yaml
---
- name: h11
  debug:
    msg: "task h11"
```

注意两个名称，一个是 notify 触发 handler 的任务名称(“h1”)，一个是引入文件中任务的名称(“h11”)，它们是两个任务。

再来看 import_tasks 引入 handler 文件的示例，注意观察名称的不同点。

如下是 pb1.yml 文件的内容：

```yaml
---
- name: play1
  hosts: localhost
  gather_facts: false
  handlers:
    - name: h2
      import_tasks: handler2.yml

  tasks:
    - name: task1 in play1
      debug:
        msg: "task1 in play1"
      changed_when: true
      notify:
        - h22
```

下面是使用 import_tasks 引入的 handler2.yml 文件的内容：

```yaml
---
- name: h22
  debug:
    msg: "task h22"
```

在引入 handler 任务文件的时候，include_tasks 和 import_tasks 的区别表现在：

- 使用 include_tasks 时，notify 指令触发的 handler 名称是 include_tasks 任务本身的名称

- 使用 import_tasks 时，notify 指令触发的 handler 名称是 import_tasks 所引入文件内的任务名称

将上面的两个示例合在一起，或许要更清晰一点：

```yaml
---
- name: play1
  hosts: localhost
  gather_facts: false
  handlers:
    - name: h1
      include_tasks: handler1.yml
    - name: h2
      import_tasks: handler2.yml

  tasks:
    - name: task1 in play1
      debug:
        msg: "task1 in play1"
      changed_when: true
      notify:
        - h1 # 注意h1和h22名称的不同
        - h22
```

其实分析一下就很容易理解为什么 notify 触发的名称要不同：

- include_tasks 是在遇到这个指令的时候才引入文件的，所以 notify 不可能去触发外部 handler 文件里的名称(h11)，外部 handler 文件中的名称在其引入之前根本就不存在

- import_tasks 是在解析 playbook 的时候引入的，换句话说，在执行 play 之前就已经把外部 handler 文件的内容引入并替换在 handler 的位置处，而原来的名称(h2)则被覆盖了

最后，不要忘了 import_tasks 或 include_tasks 自身也是任务，既然是任务，就能使用 task 层次的指令。例如下面的示例：

```yaml
handlers:
  - name: h1
    include_tasks: handler.yml
    vars:
      my_var: my_value
    when: my_var == "my_value"
```

但这两个指令对 task 层次指令的处理方式不同，相关细节仍然保留到后文统一解释。

## 6.4 组织变量

在 Ansible 中有很多种定义变量的方式，想要搞清楚所有这些散布各个角落的知识，是一个很大的难点。好在，没必要去过多关注，只需要掌握几个常用的变量定义和应用的方式即可。此处我要介绍的是将变量定义在外部文件中，然后去引入这些外部文件中的变量。

引入保存了变量的文件有两种方式：include_vars 和 vars_files。此外，还可以在命令行中使用-e 或--extra-vars 选项来引入。

### 6.4.1 vars_files

先介绍 vars_files，它是一个 play 级别的指令，可用于在解析 playbook 的阶段引入一个或多个保存了变量的外部文件。

例如，`pb.yml` 文件如下：

```yaml
---
- name: play1
  hosts: localhost
  gather_facts: false
  vars_files:
    - varfile1.yml
    - varfile2.yml
  tasks:
    - debug:
        msg: "var in varfile1: {{var1}}"
    - debug:
        msg: "var in varfile2: {{var2}}"
```

pb.yml 文件通过 vars_files 引入了两个变量文件，变量文件的写法要求遵守 YAML 或 JSON 格式。下面是这两个文件的内容：

```yaml
# 下面是varfile1.yml文件的内容
---
var1: "value1"
var11: "value11"

# 下面是varfile2.yml文件的内容
---
var2: "value2"
var22: "value22"
```

需要说明的是，vars_files 指令是 play 级别的指令，且是在解析 playbook 的时候加载并解析的，所以所引入变量的变量是 play 范围内可用的，其它 play 不可使用这些变量。

### 6.4.2 include_vars

include_vars 指令也可用于引入外部变量文件，它和 vars_files 不同。一方面，include_vars 是模块提供的功能，它是一个实实在在的任务，所以在这个任务执行之后才会创建变量。另一方面，既然 include_vars 是一个任务，它就可以被一些 task 级别的指令控制，如 when 指令。

例如：

```yaml
---
- name: play1
  hosts: localhost
  gather_facts: false
  tasks:
    - name: include vars from files
      include_vars: varfile1.yml
      when: 3 > 2
    - debug:
        msg: "var in varfile1: {{var1}}"
```

上面示例中引入变量文件的方式是直接指定文件名 include_vars: varfile1.yml，也可以明确使用 file 参数来指定路径。

```yaml
- name: include vars from files
  include_vars:
    file: varfile1.yml
```

如果想要引入多个文件，可以使用循环的方式。例如：

```yaml
- name: include two var files
  include_vars:
    file: "{{item}}"
  loop:
    - varfile1.yml
    - varfile2.yml
```

需要注意， include*vars 在引入文件的时候要求文件已经存在，如果有多个可能的文件但不确定文件是否已经存在，可以使用 with* first* found 指令或 lookup 的 first* found 插供，它们作用相同，都用从文件列表中找出存在
的文件，找到后立即停止，之前就曾提到过 with_xxx 的本质是用 lookup 对应的插件。

例如：

```yaml
tasks:
  - name: include vars from files
    include_vars:
      file: "{{item}}"
    with_first_found:
      - varfile1.yml
      - varfile2.yml
      - default.yml

# 等价于：
tasks:
  - name: include vars from files
    include_vars:
      file: "{{ lookup('first_found',any_files) }}"
    vars:
      any_files:
        - varfile1.yml
        - varfile2.yml
        - default.yml
```

此外，include_vars 还能从目录中导入多个文件，默认会递归到子目录中。例如：

```yaml
- name: Include all files in vars/all
  include_vars:
    dir: vars/all
```

### 6.4.3 –extra-vars 选项

ansible-playbook 命令的-e 选项或--extra-vars 选项也可以用来定义变量或引入变量文件。

```shell
# 定义单个变量
$ ansible-playbook -e 'var1="value1"' xxx.yml

# 定义多个变量
$ ansible-playbook -e 'var1="value1" var2="value2"' xxx.yml

# 引入单个变量文件
$ ansible-playbook -e '@varfile1.yml' xxx.yml

# 引入多个变量文件
$ ansible-playbook -e '@varfile1.yml' -e '@varfile2.yml' xxx.yml
```

因为是通过选项的方式来定义变量的，所以它所定义的变量是全局的，对所有 play 都有效。

通常来说不建议使用-e 选项，因为这对用户来说是不透明也不友好的，要求用户记住要定义哪些变量。

## 6.5 组织 playbook 文件

当单个 playbook 文件中的任务过多时，或许就是将任务划分到多个文件中的时刻。我想各位在经过上一篇文章的”洗礼”后，应该能体会这需求是多么的迫切。

import_playbook 指令可用于引入 playbook 文件，它是一个 play 级别的指令，其本质是引入外部文件中的一个或多个 play。

例如，pb.yml 是入口 playbook 文件，此文件中引入了其它 playbook 文件，其内容如下：

```yaml
---
# 引入其它playbook文件
- import_playbook: pb1.yml
- import_playbook: pb2.yml

# 文件本身的play
- name: play in self
  hosts: localhost
  gather_facts: false
  tasks:
    - debug: 'msg="file pb.yml"'
```

pb1.yml 文件是一个完整的 playbook，它可以包含一个或多个 play，其内容如下：

```yaml
---
- name: play in pb1.yml
  hosts: localhost
  gather_facts: false
  tasks:
    - debug: 'msg="imported file: pb1.yml"'
```

pb2.yml 文件也是一个完整的 playbook，其内容如下：

```yaml
---
- name: play in pb2.yml
  hosts: localhost
  gather_facts: false
  tasks:
    - debug: 'msg="imported file: pb2.yml"'
```

## 6.6 更为规范的组织方式：Role

前面介绍了组织各种文件的方式，它们都非常实用，但是各种 yml 文件多了，特别是多个 playbook 任务混在一起时，很容易混乱。

例如：

```shell
.
├── default.yml
├── handler_restart_mysql.yml
├── handler_restart_nginx.yml
├── main.yml
├── mysql.yml
├── nginx.yml
├── var_mysql.yml
└── var_nginx.yml
```

或许我们可以按照自己的文件组织方式，将各文件进行分类，比如`nginx`任务相关的放在`nginx`目录下，mysql 相关的放在`mysql`目录下，`nginx`相关的变量放在`nginx/vars`目录中，`mysql`相关的`handler`放在`mysql/handlers`目录中，等等。

其实 Ansible 已经设计好了各类文件的组织方式：Role。Role 并不算新知识点，它仅仅只是提供了一种更为通用、更为规范的文件组织方式，当然看上去也更为"专业”一点，只要按照它的规则去存放各类文件，它就会自动去引入对应文件中的内容，不需要再手动 include 或 import。

当然，使用 Role 和手动使用 include_xxx、import_xxx 并不冲突，有时候也确实需要手动去引入其它文件。

所以关于 Role，需要学习的就是它的文件组织方式，我将会一一介绍。不过在此之前，先简单看看整个 Role 的结构。

### 6.6.1 Role 文件结构一览

Role 可以组织任务、变量、handler 以及其它一些内容，所以一个完整的 Role 里包含的目录和文件可能较多，手动去创建所有这些目录和文件是一件比较烦人的事，好在可以使用`ansible-galaxy init ROLE_NAME`命令来快速创建一个符合 Role 文件组织规范的框架。关于 ansible galaxy，我稍后会简单介绍一下它。

例如，下面创建了一个名为 first_role 的 Role：

```shell
$ ansible-galaxy init first_role
$ tree
.
└── first_role
    ├── defaults
    │   └── main.yml
    ├── files
    ├── handlers
    │   └── main.yml
    ├── meta
    │   └── main.yml
    ├── README.md
    ├── tasks
    │   └── main.yml
    ├── templates
    ├── tests
    │   ├── inventory
    │   └── test.yml
    └── vars
        └── main.yml
```

可以使用`ansible-galaxy init --help`查看更多选项。比如，使用`--init-path`选项指定创建的 Role 路径：

```shell
$ ansible-galaxy init --init-path /etc/ansible/roles first_role
```

可以看到，这里面已经包含了不少目录和文件，这些目录的含义稍后我会一一解释，不过从部分文件名中，大概能看出一个 Role 包含了任务、变量、handler 等。这些目录或目录里的文件允许不存在，在没有使用到相关文件的时候并不强制这些文件或目录存在。

此外，不难发现文件大多命名为`main.yml`， 这是 Role 在使用到它们的时候默认加载的文件名，如果换成其它名称、需要手动使用 inc1ude_xx 或 Import_xxx 去加载。另一方面，这些目录下可能还包含它 yml 文件，比如 tasks 目录下有多个任务文件，那么需要在这些 main.ym1 文件中使用 include_xxx 或 Import_xxx 去加载其它外部文件。

初次理解这些可能有些费劲，没关系，稍后会一一解释清楚。不过在此之前，我还要把 Role 目录结构相关的内容介绍完。

因为有可能同时会有多个 Role，比如创建一个 Nginx 的 Role，再创建一个 MySQL 的 Role，还创建一个`Haproxy`的 Role，所以为了组织多个 Role，通常会将每个 Role 放在一个称为 roles 的目录下。即：

```shell
$ tree -L 2
.
└── roles
    ├── first_role
    └── second_role
```

有了 Role 之后，就可以将 Role 当作一个不可分割的任务整体来对待，一个 Role 相当于是一个完整的功能。但在此需要明确一个层次上的概念，Role 只是用于组织一个或多个任务，原来在 play 级别中使用 tasks 指令来定义任务，现在使用 roles 指令来引入 Role 中定义的任务。当然，roles 指令和 tasks 指令并不冲突，它们可以共存。

通过下面的图，应能帮助理解 Role 的角色。

![img](images/Chapter06/1577350407650.png)

既然 Role 是一个完整的任务体系，拥有 Role 之后就可以去使用它，或者也可以分发给别人使用，但是一个 Role 仅仅只是目录而已，如何去使用这个 Role 呢？

所以，还需要提供一个被 ansible-playbook 执行的入口 playbook 文件(就像 main()函数一样)，在这个入口文件中引入一个或多个 roles 目录下的 Role。入口文件的名称可以随意，比如`www.yml`、`site.yml`、`main.yml`等，但注意它们和*roles*目录在同一个目录下。

例如：

```shell
.
├── enter.yml
└── roles
    ├── first_role/
    └── second_role/
```

上面和 roles 同目录的`enter.yml`文件内容如下，此文件中使用 roles 指令引入了 roles 目录内的两个 Role。

```yaml
---
- name: play with role
  hosts: nginx
  gather_facts: false
  roles:
    - first_role
    - second_role
```

如果遵循了 Role 规范，入口文件中可以直接使用 Role 名称来引入 roles 目录下的 Role(正如上面的示例)，也可以指定 Role 的路径来引入。

下面再一一介绍 Role 详细的内容。

### 6.6.2 定义 Role 的 task

Role 的任务定义在`roles/xxx/tasks/main.yml`文件中，`main.yml`是该 Role 任务的入口，在执行 Role 的时候会自动执行`main.yml`中的任务。可以直接将所有任务定义在此文件中，也可以定义在其它文件中，然后在`main.yml`文件中去引入。

以 first_role 这个 Role 为例，例如，直接将任务定义在`main.yml`文件中：

```yaml
---
- name: task in main.yml
  debug:
    msg: "task in main.yml"
```

或者，将任务定义在 roles/xxx/tasks/目录下的其它文件中，如`mytask.yml`：

```yaml
---
- name: task in mymain.yml
  debug:
    msg: "task in mymain.yml"
```

然后在`roles/xxx/tasks/main.yml`中通过`include_tasks`或`import_tasks`引入它：

```yaml
---
- include_tasks: mytask.yml
# 或者
#- import_tasks: mytask.yml
```

前面已经提到过 include_xxx 和 import_xxx 的区别，这里不对其展开描述，后面还会详细解释。

Role 的任务文件定义好后，然后在 Role 的入口文件(即 roles 同目录下的 playbook 文件)`enter.yml`中引入这个 Role：

```yaml
---
- name: play1
  hosts: localhost
  gather_facts: false
  roles:
    - first_role
```

执行它：

```shell
$ ansible-playbook enter.yml
```

### 6.6.3 定义 Role 的 handler

handler 和 task 类似，它定义在`roles/xxx/handlers/main.yml`中，当 Role 的 task 触发了对应的 handler，会自动来此文件中寻找。

仍然要说的是，可以将 handler 定义在其它文件中，然后在 roles/xxx/handlers/main.yml 使用 include_tasks 或 import_tasks 指令来引入，而且前面也提到过这两者在 handler 上的区别和注意事项。

例如，`roles/first_role/handlers/main.yml`中定义了如下简单的 handler：

```yaml
---
- name: handler for test
  debug:
    msg: "a simple handler for test"
```

在`roles/first_role/tasks/main.yml`中通过 notify 触发该 Handler：

```shell
---
- name: task in main.yml
  debug:
    msg: "task in main.yml"
  changed_when: true
  notify: handler for test

```

然后执行：

```shell
$ ansible-playbook enter.yml
```

### 6.6.4 定义 Role 的变量

Role 中有两个地方可以定义变量：

- `roles/xxx/vars/main.yml`
- `roles/xxx/defaults/main.yml`

这两个文件之间的区别在于，`defaults/main.yml`中定义的变量优先级低于`vars/main.yml`中定义的变量。事实上，`defaults/main.yml`中的变量优先级几乎是最低的，基本上其它任何地方定义的变量都可以覆盖它。

### 6.6.5 Role 用到的外部文件和模板文件

有时候需要将 Ansible 端的文件拷贝到远程节点上，比如拷贝本地已经写好的 MySQL 配置文件`my.cnf`到多个远程节点上，拷贝本地写好的脚本文件到多个远程节点执行，等等。

这时候在进行拷贝的模块中可以指定这些文件的绝对路径。但在 Role 中，可以将这些文件放在`roles/xxx/files/`或`roles/xxx/templates/`目录下，遵守了这个 Role 规范，就可以在模块中直接指定文件名称，而不用加上路径前缀(当然，加上也不会错)。

例如，Role 中有一个 copy 模块的任务，想要拷贝`roles/first_role/files/my.cnf`到目标节点的`/etc/my.cnf`，则：

```yaml
- name: copy file
  copy:
    src: my.cnf # 直接指定文件名my.cnf即可
    dest: /etc/my.cnf
```

这些模块知道去`roles/xxx/files/`目录或`roles/xxx/templates/`下搜索对应文件的原因，在于这些模块的代码内部定义了文件搜索路径，不同的模块搜索路径不同，且可能不止一个搜索路径。

例如对于 Role 中的 template 模块任务(template 模块目前尚未介绍，之后遇到的时候再解释，或者各位可自搜其用法，现在将其当作 copy 模块即可)，如果其参数`src=my.cnf`，则依次搜索如下路径：

```shell
roles/first_role/templates/my.cnf
roles/first_role/my.cnf
roles/first_role/tasks/templates/my.cnf
roles/first_role/tasks/my.cnf
templates/my.cnf
my.cnf
```

一般来说，需要考虑源文件存放位置的模块包括`copy`、`script`、`template`模块，前两个模块以及其它可能的模块，一般会先搜索`roles/xxx/files/`目录，但不会搜索`templates`目录，而`template`模块则会先搜索`templates`目录而不会搜索 files 目录。

换句话说，除了`template`模块外，其它模块使用到的文件很可能都应该存放于`roles/xxx/files/`目录。如果不确定某个模块的搜索路径，测试一番即可，或者直接看报错信息中给出的路径搜索过程。

### 6.6.6 Role 中定义的模块和插件

对于绝大多数需求，使用 Ansible 已经提供的模块和插件就能解决问题，但有时候确实有些需求需要自己去写模块或插件，Ansible 也支持用户自定义的模块和插件。

对于 Role 来说，如果这个 Role 需要额外使用自己编写的模块或插件，则模块放在 roles/xxx/librarys/目录下，而插件放在各自对应类型的目录下：

```shell
roles/xxx/action_plugins/
roles/xxx/lookup_plugins/
roles/xxx/callback_plugins/
roles/xxx/connection_plugins/
roles/xxx/filter_plugins/
roles/xxx/strategy_plugins/
roles/xxx/cache_plugins/
roles/xxx/test_plugins/
roles/xxx/shell_plugins/
```

一般情况用不上自定义模块或插件，目前各位了解即可。

### 6.6.7 定义 Role 的依赖关系

很多时候，一个系统化配置管理的需求中并不仅仅只有一个 Role。一个典型的案例是部署 LAMP,这里面涉及到了部署 `Apache`、 `MYSQL`、`PHP`,但通常不会将它们全部定义到单个`Role`中，而是按照一些分类逻辑进行`Role`的划分，比如 MySQL 作为 DB,单独定义成一个`Role`， `apache `和`php`作为一个 webserver 定义成一个`Role`。如何对功能进行划分由我们自己决定，正如上面的`apache`和`php`完全可以定义成两个`Role`。

有了多个`Role`, 这些`Role`可能会存在依赖关系。例如，在新结点上部署集群服务的时候，通常都要对这些新结点进行一些初始化配置，比如设置时间同步，只有满足了这个条件，才会去启动集群服务

换句话说，有些任务必须先行，这些先行任务就是被依赖的任务。

按照 Role 规范，被依赖的先行任务都定义在`roles/xxx/meta/main.yml`文件中。

例如：

```yaml
---
dependencies:
  - second_role
  - third_role
```

注意，Role 的 dependencies 指令只能指定被依赖的 Role，不能直接指定被依赖的任务。例如，下面是**错误**的依赖定义：

```yaml
---
dependencies:
  - debug: msg="check it"
```

当真正开始执行 Role 的时候，会先检查是否有依赖任务，如果有，则先执行依赖任务，依赖任务执行完后再开始执行普通任务。

### 6.6.8 动手写一个 Role

了解完 Role 各个目录和文件的意义后，可以开始动手写一个 Role 来体验一番。

就以 first_role 为例，这个 Role 没有具体的功能，全部都是 debug 模块的调试信息，所以这个 Role 非常简单，这个 Role 唯一的意义是：学会写最简单的 Role 并看懂执行流程。

首先在`defaults/main.yml`中定义一个变量`default_var`。

```yaml
---
default_var: default_value
```

然后在`vars/main.yml`中定义两个变量`my_var`和`default_var`：

```yaml
---
my_var: my_value
default_var: overrided_default_value
```

显然`vars/main.yml`中的`default_var`会覆盖`defaults/main.yml`中的`default_var`。

定义完变量之后，就可以在 task、handler 甚至 template 模板文件中使用这些变量。当然，在实际编写 Role 的时候，一般不可能预先知道要定义哪些变量，通常都是在编写 task 的过程中来变量文件中添加变量的。

然后是`tasks/main.yml`文件，在此文件中定义了一个使用变量的任务，并引入了一个外部 task 文件`t.yml`。内容如下：

```yaml
---
- name: task1
  debug:
    msg: "task in my_var: {{my_var}}"

- name: include t.yml
  import_tasks: t.yml
```

在`t.yml`中定义了一个任务，且通过 notify 触发一个 handler，其内容为：

```yaml
---
- name: task in t.yml
  debug:
    msg: "default_var: {{default_var}}"
  changed_when: true
  notify: "go to handler"
```

然后去`handlers/main.yml`中定义对应的 handler 即可，其内容为：

```yaml
---
- name: go to handler
  debug:
    msg: "new_var: {{new_var}}"
```

这个 Role 就这么简单，因为没有定义依赖关系，也没有拷贝文件，所以`roles/first_role/{meta,files,templates}`这三个目录都可以删掉。

写好 Role 后，再提供一个被 ansible-playbook 命令执行的入口 playbook 文件，然后在此 playbook 文件中去加载对应的 Role 并执行。例如，这个入口文件名为`enter.yml`，其内容如下：

```yaml
---
- name: play1
  hosts: localhost
  gather_facts: false
  roles:
    - role: first_role
      vars:
        new_var: new_value
```

最后执行该入口文件：

```shell
$ ansible-playbook enter.yml

PLAY [play1] ****************************************
TASK [first_role : task1] ***************************
ok: [localhost] => {
    "msg": "task in my_var: my_value"
}

TASK [first_role : task in t.yml] *******************
changed: [localhost] => {
    "msg": "default_var: overrided_default_value"
}

RUNNING HANDLER [first_role : go to handler] ********
ok: [localhost] => {
    "msg": "new_var: new_value"
}
```

## 6.7 使用 Role：roles、include_role 和 import_role

写好 Role 之后就是使用 Role，即在一个入口 playbook 文件中去加载 Role。

加载 Role 的方式有多种：

- roles 指令：play 级別的指令，在 playbook 解析阶段加载对应文件，这是传统的引入 Role 的方式
- import_role 指令：task 级別的指令，在 playbook 解析阶段加载对应文件
- include_role 指令：task 级别的指令，在遇到该指令的时候才加载 Role 对应文件

例如前面使用的是 roles，如下：

```yaml
---
- name: play1
  hosts: localhost
  gather_facts: false
  roles:
    - first_role
```

上面通过 roles 指令来定义要解析和执行的 Role，可以同时指定多个 Role，且也可以加上 role:参数，例如：

```yaml
roles:
  - first_role
  - role: seconde_role
  - role: third_role
```

也可以使用 include_role 和 import_role 来引入 Role，但需注意，这两个指令是 tasks 级别的，也正因为它们是 task 级别，使得它们可以和其它 task 共存。

例如：

```yaml
---
- hosts: localhost
  gather_facts: false
  tasks:
    - debug:
        msg: "before first role"
    - import_role:
        name: first_role
    - include_role:
        name: second_role
    - debug:
        msg: "after second role"
```

这三种引入 Role 的方式都可以为对应的 Role 传递参数，例如：

```yaml
---
- hosts: localhost
  gather_facts: false
  roles:
    - role: first_role
      varvar: "valuevalue"
      vars:
        var1: value1

  tasks:
    - import_role:
        name: second_role
      vars:
        var1: value1
    - include_role:
        name: third_role
      vars:
        var1: value1
```

有时候需要让某个 Role 按需执行，比如对于目标节点是 CentOS 7 时执行 Role7 而不执行 Role6，目标节点是 CentOS 6 时执行 Role6 而不是 Role7，这可以使用 when 指令来控制。

例如：

```yaml
---
- hosts: localhost
  gather_facts: false
  roles:
    # 下面是等价的，分别采用YAML和Json语法书写
    - role: first_role
      when: xxx
    - { role: ffirst_role, when: xxx }
  tasks:
    - import_role:
        name: second_role
      when: xxx
    - include_role:
        name: third_role
      when: xxx
```

注意，在 roles、import_role 和 include_role 三种方式中，when 指令的层次。

通常来说，无论使用哪种方式来引入 Role 都可以，只是某些场景下需要小心一些陷阱。

## 6.8 查看任务和打标签 tags

每个 Role 的任务有可能分布在多个文件中，还有可能会在入口文件中加载多个 Role,这时想要知道执行 p1 abook 时具体会执行哪些任务，可以使用 ansible- playbook 命令的`--list-tasks`选项，它会列出某个 playbook 中的所有任务

```shell
$ ansible-playbook--list-tasks enter.yml
playbook: enter yml
play #1 (localhost): play1 		TAGS: []
tasks:
	first role task1			TAGS: []
	first role: task in t.ymi 	TAGS: []
```

上面的结果表示，`enter.yml` 这个 playbook 文件中只有一个 play,这个 play 中包含了两个任务.

从结果中还看到 play 和 task 的后面都带有`TAGS: []`，它是标签。当在 play 或 task 级别使用 tags 指令后就表示为此 play 或 task 打了标签。

1. 可以在 task 级别为单个任务打一个或多个标签，多个任务可以打同一个标签名。

例如：

```yaml
- name: yum install ntp
  yum:
    name: ntp
    state: present
  tags:
    - initialize
    - pkginstall
    - ntp

- name: started ntpd
  service:
    name: ntpd
    state: started
  tags:
    - ntp
    - initialize
```

当任务具有了标签之后，就可以在 ansible-playbook 命令行中使用`--tags`来指定只有带有某标记的任务才执行，也可以使用`--skip-tags`选项明确指定不要执行某个任务。

```shell
# 只执行第一个任务
$ ansible-playbook test.yml --tags "pkginstall"

# 两个任务都执行
$ ansible-playbook test.yml --tags "ntp,initialize"

# 第一个任务不执行
$ ansible-playbook test.yml --skip-tags "pkginstall"
```

如果想要确定 tag 筛选之后会执行哪些任务，加上--list-tasks 即可：

```shell
$ ansible-playbook test.yml --tags "ntp" --list-tasks
```

2. 可以在 play 级别打标签，这等价于对 play 中的所有任务都打上标签。

例如：

```yaml
- name: play1
  hosts: localhost
  gather_facts: false
  tags:
    - tag1
    - tag2
  pre_tasks:
    - debug: "msg='pre_task1'"
    - debug: "msg='pre_task2'"
  tasks:
    - debug: "msg='task1'"
    - debug: "msg='task2'"
```

这会为 4 个任务都打 tag1 和 tag2 标签。

```shell
$ ansible-playbook a.yml --list-tasks

playbook: a.yml

  play #1 (localhost): play1    TAGS: [tag1,tag2]
    tasks:
      debug     TAGS: [tag1, tag2]
      debug     TAGS: [tag1, tag2]
      debug     TAGS: [tag1, tag2]
      debug     TAGS: [tag1, tag2]
```

3. 在静态加载文件的指令上打标签，等价于为所加载文件中所有子任务打标签。在动态加载文件的指令上打标签，不会为子任务打标签，而是为父任务自身打标签。

关于静态、动态加载，本文最后会详细说明。现在说结论：

- 静态加载的指令有：roles、include、import_tasks、import_role

- 动态加载的指令只有 include_xxx，包括 include_tasks、include_role
- import_playbook 和 include_playbook 因为本身就是 play 级别或高于 play 级别，所以不能为这两个指令打标签。

例如，在 b.yml 文件中有两个任务：

```yaml
---
- debug: "msg='task1 in b.yml'"
- debug: "msg='task2 in b.yml'"
```

在 c.yml 中也有两个任务：

```yaml
---
- debug: "msg='task1 in c.yml'"
- debug: "msg='task2 in c.yml'"
```

然后在 a.yml 中分别使用 import_tasks 指令引入 b.yml，使用 include_tasks 指令引入 c.yml，同时为这两个指令打标签：

```yaml
- name: play1
  hosts: localhost
  gather_facts: false

  tasks:
    - import_tasks: b.yml
      tags: [tag1, tag2]

    - include_tasks: c.yml
      tags: [tag3, tag4]
```

这会为 b.yml 中的两个任务打上 tag1 和 tag2 标签，还会为 a.yml 中的 include_tasks 任务自身打上标签 tag3 和 tag4。

```shell
$ ansible-playbook a.yml --list-tasks

playbook: a.yml

  play #1 (localhost): play1    TAGS: []
    tasks:
      debug     TAGS: [tag1, tag2]
      debug     TAGS: [tag1, tag2]
      include_tasks     TAGS: [tag3, tag4]
```

关于是否要打标签，众说纷纭。我个人的看法是不要单独为任务打标签，要么为整个 Role 打标签，要么为静态加载进来的整个文件打标签，如果手动在任务级别上打标签，标签数量一多，playbook 会显得非常混乱。

## 6.9 Ansible Galaxy 和 Collection

很多时候我们想要实现的 Ansible 部署需求其实别人已经写好了，所以我们自己不用再动手写(甚至不应该自己写)，直接去网上找别人已经写好的轮子即可。

[Ansible Galaxy](https://galaxy.ansible.com/)是一个 Ansible 官方的 Role 仓库，世界各地的人都在里面分享自己写好的 Role，我们可以直接去 Galaxy 上搜索是否有自己想要的 Role，如果有符合自己心意的，直接安装便可。当然，我们也可以将写好的 Role 分享出去给别人使用。

Ansible 提供了一个 ansible-galaxy 命令行工具，可以快速创建、安装、管理由该工具维护的 Role。它常用的命令有：

```yaml
# 安装Role:
ansible-galaxy install username.role_name

# 移除Role:
ansible-galaxy remove username.role_name

# 列出已安装的Role:
ansible-galaxy list

# 查看Role信息:
ansible-galaxy info username.role_name

# 搜索Role:
ansible-galaxy search role_name

# 创建Role
ansible-galaxy init role_name

# 此外还有：'delete','import', 'setup', 'login'
# 它们都用于管理galaxy.ansible.com个人账户或里面的Role
# 无视它们
```

例如，前面已经用该命令快速创建过一个 Role，免去了手动创建 Role 的一堆目录和文件。

```shell
$ ansible-galaxy init --init-path /etc/ansible/roles first_role
```

当从 Galaxy 中搜索到了 Role 之后，可以直接使用 ansible-galaxy install author.rolename 来安装，之所以要加上作者名 author，是因为不同的人可能会上传名称相同的 Role。

例如，我搜索到了一个[`helloworld`的测试 Role](https://galaxy.ansible.com/chusiang/helloworld)

点进去后，就能看到安装方式。比如：

```shell
$ ansible-galaxy install chusiang.helloworld

- downloading role 'helloworld', owned by chusiang
- downloading role from ......
- extracting chusiang.helloworld to /root/.ansible/roles/chusiang.helloworld
- chusiang.helloworld (master) was installed successfully
```

默认情况下，ansible-galaxy install 安装 Role 的位置顺序是：

- ~/.ansible/roles
- /usr/share/ansible/roles
- /etc/ansible/roles

可以使用-p 或--roles-path 选项指定安装路径。

```shell
$ ansible-galaxy install -p roles/ chusiang.helloworld
```

安装完成后，就可以直接使用这个 Role。例如，创建一个 enter.yml 文件，并在此文件中引入该 Role，其内容如下：

```yaml
---
- name: role from galaxy
  hosts: localhost
  gather_facts: false
  roles:
    - role: chusiang.helloworld
```

然后执行：

```shell
$ ansible-playbook enter.yml
```

虽然 Ansible Galaxy 中有大量的 Role，但有时候我们也会在`Github`上搜索 Role，而且 Galaxy 仓库上的 Role 大多也都在`Github`上。ansible-galaxy install 也可以直接从 git 上下载安装 Role。

例如，上面”helloworld” Role 存放在https://github.com/chusiang/helloworld.ansible.role，直接从github上安装它：

```shell
$ ansible-galaxy install -p roles/ git+https://github.com/chusiang/helloworld.ansible.role.git
```

注意，从 git 安装和从 Galaxy 上安装的 Role 名称可能不一样。例如，下面 roles/目录下有两个”helloworld” Role，但名称不同：

```shell
$ ansible-galaxy list -p roles
# /root/ansible/role_test/roles
- first_role, (unknown version)
- chusiang.helloworld, master
- helloworld.ansible.role, (unknown version)
# /root/.ansible/roles
- chusiang.helloworld, master
# /usr/share/ansible/roles
# /etc/ansible/roles
```

### Ansible Collection

对于文件组织结构，在 Ansible 2.8 以前只支持 Role 的概念，但 Ansible 2.8 中添加了一项目前仍处于实验性的功能 Collection，它以包的管理模式来结构化管理 Ansible playbook 涉及到的各个文件。

比如，我们可以将整个写好的功能构建、打包，然后分发出去，别人就可以使用 ansible-galaxy(要求 Ansible 2.9)去安装这个打包好的文件，这为自动化构建和部署带来了很大的便利。

如下，是一个 collection 的目录组织结构示例：

```shell
long/           # author name
└── testing     # collection name
    ├── docs/
    ├── galaxy.yml
    ├── plugins/
    │ ├──  modules/
    │ │ └──  module1.py
    │ ├──  inventory/
    │ └──  .../
    ├── README.md
    ├── roles/
    │ ├──  role1/
    │ ├──  role2/
    │ └──  .../
    ├── playbooks/
    │ ├──  files/
    │ ├──  vars/
    │ ├──  templates/
    │ └──  tasks/
    └──  tests/
```

目前 Ansible Galaxy 上的 Collection 还非常少，在我写这篇文章的时候，Ansible Galaxy 上目前只提交了 11 个 collection。

关于 Collection 更详细的内容，我不多作介绍，目前它还处于试验阶段，各位如有兴趣可自行参考官方手册的说明：https://docs.ansible.com/ansible/latest/galaxy/user_guide.html。

## 6.10 playbook 的执行顺序

最后，再解释一下 Ansible 从开始执行 playbook 到执行结束中间经历的大致过程，让各位对 Ansible 的工作流程有一个全局的认识。

这里所介绍的不涉及执行策略，比如一次性选中几个节点执行、执行完后是否立即切入下一个任务、下一个节点执行等等，这里所说的流程，是对每个节点而言，Ansible 将以何种顺序去执行。

当 Ansible 解析完 inventory 之后，就进入解析 playbook 的阶段，解析完 playbook 之后，才开始执行第一个 play。

每个 play 中可能有多种、多项任务，它们的执行顺序依次为：

- gather_facts 任务
- pre_tasks 指令中的任务
- pre_tasks 中触发的所有 handler
- roles 指令加载的 Role
- tasks 指令中的认为有
- roles 和 tasks 中触发的所有 handler
- post_tasks 指令中的任务
- post_tasks 中触发的所有 handler

上面的逻辑应该非常容易理解，但几个事项需说明：

- roles 指令加载的 Role 比 tasks 中的任务先执行

- 每个阶段的 handler 默认都在当前阶段所有任务完成之后才开始执行，且重复触发的 handler 将只执行一次

例如下面的 playbook：

```yaml
---
- name: play1
  hosts: localhost
  gather_facts: false
  pre_tasks:
    - name: pre_task1
      debug:
        msg: "hello pretask"
      changed_when: true
      notify: "notify me"

  roles:
    - role: first_role
    - role: second_role

  tasks:
    - name: task1
      debug:
        msg: "hello task"
      changed_when: true
      notify: "notify me"

  post_tasks:
    - name: post_task1
      debug:
        msg: "hello posttask"
      changed_when: true
      notify: "notify me"

  handlers:
    - name: notify me
      debug:
        msg: "I am handler"
```

整个 play 的执行流程似乎非常简单，但是 playbook 或 play 中可能不是直接定义内容，而是通过 include_xxx 或 import_xxx 从其它文件中加载，它们之间有区别。本文从头到尾，对于它们的区别，我都只是简单重复一个结论：import_xxx 是 playbook 解析的阶段加载，include_xxx 是遇到指令的时候加载。现在，我要花点篇幅去解释解释 include、roles、include_xxx、import_xxx 之间的区别。

### 6.10.1 playbook 解析、动态加载和静态加载

还是这个结论：

- import_xxx 是在 playbook 的解析阶段加载文件

- include_xxx 是遇到指令的时候加载文件

只要理解了这两个结论，所有相关的现象都能理解。

那么 playbook 的解析是什么意思，它做了什么事呢？

第一个要明确的是 playbook 解析处于哪个阶段执行：inventory 解析完成后、play 开始执行前的阶段。

第二个要明确的是 playbook 解析做了哪些哪些事。一个简单又直观的描述是，playbook 解析过程中，会扫描 playbook 文件中的内容，然后检查语法并转换成 Ansible 认识的内部格式，以便让 Ansible 去执行。

更具体一点，在解析 playbook 期间：

1. 当在 playbook 文件中遇到了 roles、 include、 import_xx 指令，则会将它们指定的文件内容“插入到”指令位置处，也即原文替换，这个过程对用户来说是适明的。实际上并非真的会插入替换，稍后我会再补充，但这样理解会更容易些;
   - roles、 include、 import_xx 同属一类，它们都是静态加載，都在 playbook 解析阶段加载文件，而 include_xxx 属于另一类，是动态加载，遇到指令的时候临时去加载文件；
   - 之所以有这么多看似功能重复的指令，这和 Ansible 版本的发展有关，不同的版本可能会小有区别；
   - 早期版本只有 include 指令，所以它的行为有些混乱，建议不要对其做太多考究，也尽量不要使用该指令；
2. 解析每个 play 级别的指令，比如:
   - 解析 play 的 name 指令，确定 play 的名称
   - 解析 hosts 指令，确定要执行该 play 的远程节点有哪些
   - 确定 play 的执行策略，比如是等特所有远程节点完成一个任务后再进入下个任务，还是节点只要完成任务就立即进入下一个任务
   - 确定远程连接相关行为，比如 port、 become、 become user 等等
     - 虽然在 inventory 中已经解析过远程连接的行为，但在开始执行任务前，在 play 级別中仍然能够指定一些远程连能项，它会指盖 inventory 中的同名选项，因为 play 级别的指令是在解所 playbook 的阶段解析的
   - 解析 gather\_ facts 指令，确定该 paly 中是否要收集远程节点的信息
   - 解析 play 级别的 vars、vars_fi1es 指令中定义的变量

根据这些描述，再试着来理解一下下面这些现象或结论(有些在前文已经出现过)，应该不会难理解了。

(1).在循环中，使用 include_xxx 而不能使用 import_xxx。

例如，某个等待被加载的文件 b.yml 内容如下：

```yaml
---
- name: task1
  debug: "msg='hello'"
- name: task2
  debug: "msg='world'"
```

然后 a.yml 中使用循环通过 include_tasks 去加载 b.yml：

```yaml
tasks:
  - name: loop task
    include_tasks: b.yml
    loop: [1, 2]
```

这并没有什么问题，当开始执行到这个父级别循环任务的时候，每循环一轮去加载一次这个文件然后执行这个文件中的所有子任务。

但如果 a.yml 中使用 import_tasks 去加载 b.yml，在解析 playbook 的时候，就已经将 b.yml 中的任务替换到这个指令的位置处，假设这里不会报错，那么在执行到这个循环任务的时候，这个任务的内容大概变成了这样：

```yaml
tasks:
  - name: loop task
    - name: task1
      debug: "msg='hello'"
    - name: task2
      debug: "msg='world'"
    loop: [1,2]
```

这看上去不伦不类，显然会出现语法错误。事实上也确实如此，各位可以测试一下然后观察报错的阶段正是语法检查阶段，并不是在执行任务的阶段报错的。

但是要给各位提个醒，对于 task 级别的 import_xxx 或 include 指令，比如 import_tasks、import_role，它并非直接原文插入，而是先解析父级别任务的指令，并将这些指令复制到所加载文件中每个子任务上，然后再原文替换(前面已经接触过一个 tags 指令，它会将标签复制到所有子任务上)。

所以，在不报错的假设下，上面示例在解析后应该是类似这样的：

```yaml
tasks:
  - name: task1
    debug: "msg='hello'"
    loop: [1, 2]
  - name: task2
    debug: "msg='world'"
    loop: [1, 2]
```

这看上去没有语法错误，但明显已经违背了我们的期望，所以 Ansible 在解析阶段就检测这种不合理行为。

(2).使用 include_tasks 时，其加载文件内定义的变量不能在调用它的外部(即父级任务)使用。

例如下面 when 指令结合 include_xxx 的示例。

在 b.yml 中有两个任务，都定义了 num 变量。

```yaml
---
- name: task1
  debug:
    msg: "{{num}}"
  vars:
    num: 4

- name: task2
  debug:
    msg: "{{num}}"
  vars:
    num: 2
```

在 a.yml 中使用 include_tasks 去加载 b.yml，并加上 when 或其它 task 级别的指令来使用变量 num：

```yaml
---
- name: play1
  hosts: localhost
  gather_facts: false

  tasks:
    - name: task1
      include_tasks: b.yml
      when: num > 3
```

这会报错，提示使用了未定义变量，各位可自行测试并观察一下。注意这不是语法错误(即 playbook 的解析阶段是成功的)，而是执行这个任务时的运行时错误。报错的原因在于执行该任务时，会先解析 task 级别的指令 when，然后再执行模块任务，所以解析 when 条件的时候，b.yml 文件尚未加载。

如果将 include_tasks 替换成 import_tasks 则不会出错。因为在使用 import_tasks 时是将 when 指令复制到 b.yml 中的所有任务上，所以 playbook 解析完后等价于：

```yaml
---
- name: play1
  hosts: localhost
  gather_facts: false

  tasks:
    - name: task1
      debug:
        msg: "{{num}}"
      vars:
        num: 4
      when: num > 3

    - name: task2
      debug:
        msg: "{{num}}"
      vars:
        num: 2
      when: num > 3
```

(3).当在 handlers 指令中通过 include_tasks 和 import_tasks 加载任务文件时，在 notify 指令中指定 handler 名称的方式不同。

例如，h.yml 文件中定义了两个 handler 等待被 notify，其内容如下：

```yaml
---
- name: handler1
  debug: 'msg="hello handler"'
- name: handler2
  debug: 'msg="world handler"'
```

在 a.yml 中定义了两个任务，都触发刚才定义的两个 handler，但因为使用不同指令来加载 h.yml，使得 notify 指令中的名称也不一样。

```yaml
tasks:
  - name: task1
    debug: 'msg="hello task"'
    changed_when: true
    notify: "notify me"

  - name: task2
    debug: 'msg="world task"'
    changed_when: true
    notify:
      - "handler1"
      - "handler2"

handlers:
  - name: notify me
    include_tasks: h.yml
  - name: dont notify me
    import_tasks: h.yml
```

如果按照前面所描述的，将 import_tasks 加载的文件内容替换到 a.yml 中，再去理解为何 notify 的名称不同就很容易了。如下是替换后的内容：

```yaml
handlers:
  - name: notify me
    include_tasks: h.yml
  - name: handler1
    debug: 'msg="hello handler"'
  - name: handler2
    debug: 'msg="world handler"'
```

经过这几个示例，我想各位已经意识到了，使用 include_tasks 时，这个指令自身占用一个任务，使用 import_tasks 的时候，这个指令自身没有任务，它所在的任务会在解析 playbook 的时候被其加载的子任务覆盖。

(4).play 的 name 指令中如果使用了变量，则这个变量必须是在解析 playbook 时就已经定义好的。换句话说，play 名称中的变量不能是 task 中定义的变量。这应该不难理解。

(5).无法使用--list-tags 列出 include_xxx 中的 tags，无法使用--list-tasks 列出 include_xxx 中的任务，因为它们都是临时动态加载的。

还有其它一些需要小心的陷阱，不过在知道它们的行为之后，便能够分析使用哪个指令以及在遇到错误的时候知道如何排查。
