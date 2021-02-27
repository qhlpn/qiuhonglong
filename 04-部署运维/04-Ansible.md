### 相关文件
```
配置文件
    /etc/ansible/ansible.cfg  主配置文件,配置ansible工作特性(一般无需修改)
    /etc/ansible/hosts        主机清单(将被管理的主机放到此文件)
    /etc/ansible/roles/       存放角色的目录

程序
    /usr/bin/ansible          主程序，临时命令执行工具
    /usr/bin/ansible-doc      查看配置文档，模块功能查看工具
    /usr/bin/ansible-galaxy   下载/上传优秀代码或Roles模块的官网平台
    /usr/bin/ansible-playbook 定制自动化任务，编排剧本工具
    /usr/bin/ansible-pull     远程执行命令的工具
    /usr/bin/ansible-vault    文件加密工具
    /usr/bin/ansible-console  基于Console界面与用户交互的执行工具
```

### 配置文件
```
Ansible 配置文件/etc/ansible/ansible.cfg 

vim /etc/ansible/ansible.cfg

[defaults]
#inventory     = /etc/ansible/hosts      # 主机列表配置文件
#library       = /usr/share/my_modules/  # 库文件存放目录
#remote_tmp    = $HOME/.ansible/tmp      # 临时py命令文件存放在远程主机目录
#local_tmp     = $HOME/.ansible/tmp      # 本机的临时命令执行目录  
#forks         = 5                       # 默认并发数,同时可以执行5次
#sudo_user     = root                    # 默认sudo 用户
#ask_sudo_pass = True                    # 每次执行ansible命令是否询问ssh密码
#ask_pass      = True                    # 每次执行ansible命令是否询问ssh口令
#remote_port   = 22                      # 远程主机的端口号(默认22)

建议优化项： 
host_key_checking = False               # 检查对应服务器的host_key，建议取消注释
log_path=/var/log/ansible.log           # 日志文件,建议取消注释
module_name   = command                 # 默认模块
```

### 主机清单
```
Inventory 主机清单 
/etc/ansible/hosts文件格式
文件遵循INI文件风格，中括号中的字符为组名。
可以将同一个主机同时归并到多个不同的组中；
此外，当如若目标主机使用了非默认的SSH端口，还可以在主机名称之后使用冒号加端口号来标明
    ntp.magedu.com   不分组,直接加
    
    [webservers]     webservers组
    www1.magedu.com:2222  可以指定端口
    www2.magedu.com
    
    [dbservers]
    db1.magedu.com
    db2.magedu.com
    db3.magedu.com

如果主机名称遵循相似的命名模式，还可以使用列表的方式标识各主机
示例：
    [websrvs]
    www[1:100].example.com   ip: 1-100
    
    [dbsrvs]
    db-[a:f].example.com     dba-dbff
```


### 常用命令
```
🍿: ansible ansible-doc ansible-playbook 
    ansible-vault ansible-console '
    ansible-galaxy ansible-pull 拉取仓库playbook配置文件
```

+ **ansible-doc**：显示模块帮助

  ```
  ansible-doc -l      列出所有模块
  ansible-doc ping    查看指定模块帮助用法
  ansible-doc -s ping 查看指定模块帮助用法
  ```

+ **ansible**：配置管理、应用部署、任务执行

  ```
  ansible <host-pattern> [-m module] [-a args]
      --version              显示版本
      -m module              指定模块，默认为command
      -v                     详细过程 –vv -vvv更详细
      --list-hosts           显示主机列表，可简写 --list
      -k, --ask-pass         提示输入ssh连接密码,默认Key验证
      -C, --check            检查，并不执行
      -T, --timeout=TIMEOUT  执行命令的超时时间,默认10s
      -u, --user=REMOTE_USER 执行远程执行的用户
      -b, --become           代替旧版的sudo切换
          --become-user=USERNAME 指定sudo的runas用户,默认为root
      -K, --ask-become-pass  提示输入sudo时的口令
  eg: ansible all -m ping
      ansible srvs -m command -a 'service vsftpd start'
  
  ansible命令执行过程
      1. 加载自己的配置文件 默认/etc/ansible/ansible.cfg
      2. 加载自己对应的模块文件，如command
      3. 通过ansible将模块或命令生成对应的临时py文件，
         并将该文件传输至远程服务器的对应执行用户$HOME/.ansible/tmp/ansible-tmp-数字/XXX.PY文件
      4. 给文件+x执行
      5. 执行并返回结果
      6. 删除临时py文件，sleep 0退出
  
  执行状态：
      绿色：执行成功并且不需要做改变的操作
      黄色：执行成功并且对目标主机做变更
      红色：执行失败
  ```

+ **ansible-playbook**：执行playbook脚本

  ```
  ansible-playbook hello.yml
  ```

  

### 常用模块

```
Command：在远程主机执行命令
Shell：在远程主机执行shell语句
Script：在远程主机上运行ansible服务器上的脚
Copy：从主控端复制文件到远程主机
Fetch：从远程主机提取文件至主控端，copy相反，目前不支持目录,可以先打包,再提取文件
File：设置文件属性    
Unarchive：解包解压缩，有两种用法：
Archive：打包压缩
Hostname：管理主机名
Cron：远程主机定时任务
Yum：管理包
Service：管理服务
User：管理用户
Group：管理组
Lineinfile | Replace：流式修改文件内容
Setup：收集主机系统信息
```


### Playbook
```
用户通过ansible命令直接调用yml语言写好的playbook，playbook由多条play组成
每条play都有一个任务(task)相对应的操作，然后调用模块modules，应用在主机清单上
通过ssh远程连接，从而控制远程主机或者网络设备
ansibel具有幂等性，多次playbook执行后的主机状态应相同
```
+ **运行指令**

  ```
  ansible-playbook <filename.yml> ... [options]
  
  常见选项
      --check -C       只检测可能会发生的改变，但不真正执行操作 
                       (只检查语法,如果执行过程中出现问题,-C无法检测出来)
                       (执行playbook生成的文件不存在,后面的程序如果依赖这些文件,也会导致检测失败)
      --list-hosts     列出运行任务的主机
      --list-tags      列出tag  (列出标签)
      --list-tasks     列出task (列出任务)
      --limit 		 主机列表 只针对主机列表中的主机执行
      -v -vv -vvv      显示过程
  ```

+ **核心元素**

  ```
  Hosts         	 执行的远程主机列表(应用在哪些主机上)
  Tasks         	 任务集
  Variables     	 内置变量或自定义变量在playbook中调用
  Templates  	     可替换模板文件中的变量并实现一些简单逻辑的文件
  Handlers Notify  结合使用，由特定条件触发的操作，满足条件方才执行，否则不执行
  Tags     	     指定某条任务执行，用于选择运行playbook中的部分代码。
  ```

  + **hosts**

    ```
    - hosts: websrvs
        remote_user: root   (可省略,默认为root)  以root身份连接
        gather_facts: no   # 不收集主机信息，提高速度
    ```

  + **tasks**

    ```
    两种格式：
        (1) action: module arguments
        (2) module: arguments 建议使用  模块: 参数
    tasks:
      - name: disable selinux   描述
        command: /sbin/setenforce 0   模块名: 模块对应的参数
    ```
    
  + **handlers**

    ```
    触发器 handlers 也是一个是 task 列表
    这些 task 与前述的 task 并没有本质上的不同
    与 notify 配合，可用于在每个 play 的最后被触发 
    这样可避免多次有改变发生时每次都执行指定的操作，仅在所有的变化发生完成后一次性地执行指定操作。
    
    - hosts: websrvs
      remote_user: root
      
      tasks:
        - name: add group nginx
          tags: user
          user: name=nginx state=present
        - name: add user nginx
          user: name=nginx state=present group=nginx
        - name: Install Nginx
          yum: name=nginx state=present
        - name: config
          copy: src=/root/config.txt dest=/etc/nginx/nginx.conf
          notify:
            - Restart Nginx
            - Check Nginx Process
      
      handlers:
        - name: Restart Nginx
          service: name=nginx state=restarted enabled=yes
        - name: Check Nginx process
          shell: killall -0 nginx > /tmp/nginx.log
    ```
    
  + **tags**
    
    ```
    可以指定某一个任务添加一个标签，添加标签以后，想执行某个动作可以做出挑选来执行

    - hosts: websrvs
      remote_user: root
      
      tasks:
        - name: Install httpd
          yum: name=httpd state=present
          tage: install 
        - name: Install configure file
          copy: src=files/httpd.conf dest=/etc/httpd/conf/
          tags: conf
        - name: start httpd service
          tags: service
          service: name=httpd state=started enabled=yes

    ansible-playbook –t install,conf httpd.yml   指定执行install,conf 两个标签
    ```

  + **variables**

    ```
    变量名：仅能由字母、数字和下划线组成，且只能以字母开头
    变量来源：
        1 ansible setup facts 系统变量
           setup 模块可以实现系统信息的显示
                 可以返回每个主机的系统信息包括:版本、主机名、cpu、内存
           ansible all -m setup -a 'filter="ansible_nodename"'     查询主机名    
        2 /etc/ansible/hosts 定义
            普通变量：主机组中主机单独定义，优先级高于公共变量（单个主机）
            公共（组）变量：针对主机组中所有主机定义统一变量（一组主机的同一类别）
        3 命令行 定义，优先级最高
           ansible-playbook –e varname=value  
        4 vars 定义
           vars:
            - var1: value1
            - var2: value2    
        6 role 定义：defaults/main.yml
        7 set_fact 定义
        	set_fact:
        	  key: value
    
    变量定义：key=value
    
    变量调用方式：
        1 通过 {{ variable_name }} 调用变量，且变量名前后必须有空格
        2 ansible-playbook –e 直接定义并调用
          ansible-playbook test.yml -e "hosts=www user=magedu"
          
      
    1. 在主机清单中定义变量,在ansible中使用变量
      vim /etc/ansible/hosts
      [appsrvs]
      192.168.38.17 http_port=817 name=www
      192.168.38.27 http_port=827 name=web
    
    调用变量
    ansible appsrvs -m hostname -a'name={{name}}'  更改主机名为各自被定义的变量 
    
    2. 使用 setup 变量
    - hosts: websrvs
      remote_user: root
      tasks:
        - name: create log file
          file: name=/var/log/ {{ ansible_fqdn }} state=touch
          
    3. 使用文件变量
      cat vars.yml
      var1: httpd
      var2: nginx
    
    cat var.yml
    - hosts: web
      remote_user: root
      vars_files:      # 🍟
        - vars.yml
      tasks:
        - name: create httpd log
          file: name=/app/{{ var1 }}.log state=touch
        - name: create nginx log
        file: name=/app/{{ var2 }}.log state=touch
          
  优先级：
    https://www.cnblogs.com/mauricewei/p/10054300.html
    ```
  
  + **templates**
  
    ```
    文本文件，嵌套有脚本（使用模板编程语言编写） 借助模板生成真正的文件
    Jinja2语言，使用字面量，有下面形式
        字符串：使用单引号或双引号
        数字：整数，浮点数
        列表：[item1, item2, ...]
        元组：(item1, item2, ...)
        字典：{key1:value1, key2:value2, ...}
        布尔型：true/false
  算术运算：+, -, *, /, //, %, **
    比较操作：==, !=, >, >=, <, <=
  逻辑运算：and，or，not
    流表达式：For，If，When
    ```
  
  + **when**
  
    ```
    条件测试:如果需要根据变量、facts或此前任务的执行结果来做为某task执行与否的前提时要用到条件测试,
    通过when语句实现，在task中使用，jinja2的语法格式
    
    when语句
        在task后添加when子句即可使用条件测试；when语句支持Jinja2表达式语法
    示例：
    tasks:
      - name: "shutdown RedHat flavored systems"
        command: /sbin/shutdown -h now
        when: ansible_os_family == "RedHat"  当系统属于红帽系列,执行command模块 
     
    when语句中还可以使用Jinja2的大多"filter"，
    例如要忽略此前某语句的错误并基于其结果(failed或者success)运行后面指定的语句，
    可使用类似如下形式：
    tasks:
      - command: /bin/false
        register: result
        ignore_errors: True
      - command: /bin/something
        when: result|failed
      - command: /bin/something_else
        when: result|success
    - command: /bin/still/something_else
        when: result|skipped
  
    此外，when语句中还可以使用facts或playbook中定义的变量
    ```
  
  + **with_items**
  
    ```
    循环：当有需要重复性执行的任务时，可以使用迭代机制
    对迭代项的引用，固定变量名为 item
    列表格式可以是字符串或者是字典
    
    示例： 创建用户
    - name: add several users
      user: name={{ item }} state=present groups=wheel   #{{ item }} 系统自定义变量
      with_items:       # 定义{{ item }} 的值和个数
        - testuser1
        - testuser2
    
    上面语句的功能等同于下面的语句：
  - name: add user testuser1
      user: name=testuser1 state=present groups=wheel
  - name: add user testuser2
      user: name=testuser2 state=present groups=wheel
    ```
  
  + **roles**
  
    ```
    roles 通过分别将变量、文件、任务、模板及处理器放置于单独的目录，用于层次性、结构化地组织 playbook，通过 include 引入
    https://www.wumingx.com/linux/ansible-roles.html
    ```