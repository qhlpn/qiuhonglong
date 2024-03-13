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

stdout_callback = yaml                  # 输出日志格式化
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
  
  ansible -i hosts all -m shell -a "uname -r" -b
  ```


### Playbook
+ **运行指令**

  ```shell
  ansible-playbook <filename.yml> ... [options]
  
  常见选项
  
  -u REMOTE_USER, --user=REMOTE_USER  
  ＃ ssh 连接的用户名
  -k, --ask-pass    
  ＃ssh登录认证密码
  -s, --sudo           
  ＃sudo 到root用户，相当于Linux系统下的sudo命令
  -U SUDO_USER, --sudo-user=SUDO_USER    
  ＃sudo 到对应的用户
  -K, --ask-sudo-pass     
  ＃用户的密码（—sudo时使用）
  -T TIMEOUT, --timeout=TIMEOUT 
  ＃ ssh 连接超时，默认 10 秒
  -C, --check      
  ＃ 指定该参数后，执行 playbook 文件不会真正去执行，而是模拟执行一遍，然后输出本次执行会对远程主机造成的修改
  
  -e EXTRA_VARS, --extra-vars=EXTRA_VARS    
  ＃ 设置额外的变量如：key=value 形式 或者 YAML or JSON，以空格分隔变量，或用多个-e
  
  -f FORKS, --forks=FORKS    
  ＃ 进程并发处理，默认 5
  -i INVENTORY, --inventory-file=INVENTORY   
  ＃ 指定 hosts 文件路径，默认 default=/etc/ansible/hosts
  -l SUBSET, --limit=SUBSET    
  ＃ 指定一个 pattern，对- hosts:匹配到的主机再过滤一次
  --list-hosts  
  ＃ 只打印有哪些主机会执行这个 playbook 文件，不是实际执行该 playbook
  --list-tasks   
  ＃ 列出该 playbook 中会被执行的 task
  
  --private-key=PRIVATE_KEY_FILE   
  ＃ 私钥路径
  --step    
  ＃ 同一时间只执行一个 task，每个 task 执行前都会提示确认一遍
  --syntax-check  
  ＃ 只检测 playbook 文件语法是否有问题，不会执行该 playbook 
  -t TAGS, --tags=TAGS   
  ＃当 play 和 task 的 tag 为该参数指定的值时才执行，多个 tag 以逗号分隔
  --skip-tags=SKIP_TAGS   
  ＃ 当 play 和 task 的 tag 不匹配该参数指定的值时，才执行
  -v, --verbose   
  ＃输出更详细的执行过程信息，-vvv可得到所有执行过程信息。
  
  # ansible-playbook playbook.yml -i hosts -v -u eecdn -k
  SSH password:
  ```