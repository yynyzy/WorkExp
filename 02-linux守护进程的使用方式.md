# 一、问题的由来
Web应用写好后，下一件事就是启动，让它一直在后台运行。
```js
var http = require('http');

http.createServer(function(req, res) {
  res.writeHead(200, {'Content-Type': 'text/plain'});
  res.end('Hello World');
}).listen(5000);
```

你在命令行下启动它。
```
$ node server.js
```
一旦你退出命令行窗口，这个应用就一起退出了，无法访问了。  

怎么才能让它变成系统的守护进程（daemon），成为一种服务（service），一直在那里运行呢？

# 二、前台任务与后台任务
上面这样启动的脚本，称为"前台任务"。它会独占命令行窗口，只有运行完了或者手动中止，才能执行其他命令。    

变成守护进程的第一步，就是把它改成"后台任务"（background job）。  
```
$ node server.js &
```
只要在命令的尾部加上符号&，启动的进程就会成为"后台任务"。如果要让正在运行的"前台任务"变为"后台任务"，可以先按ctrl + z，然后执行bg命令（让最近一个暂停的"后台任务"继续执行）。

## "后台任务"有两个特点
继承当前 session（对话）的标准输出（stdout）和标准错误（stderr）。因此，后台任务的所有输出依然会同步地在命令行下显示。  
不再继承当前 session 的标准输入（stdin）。你无法向这个任务输入指令了。如果它试图读取标准输入，就会暂停执行。  

"后台任务"与"前台任务"的本质区别只有一个：是否继承标准输入。所以，执行后台任务的同时，用户还可以输入其他命令。  

# 三、SIGHUP信号
变为"后台任务"后，一个进程是否就成为了守护进程呢？或者说，用户退出 session 以后，"后台任务"是否还会继续执行？  
Linux系统是这样设计的。
```
1.用户准备退出 session
2.系统向该 session 发出SIGHUP信号
3.session 将SIGHUP信号发给所有子进程
4.子进程收到SIGHUP信号后，自动退出
```
上面的流程解释了，为什么"前台任务"会随着 session 的退出而退出：因为它收到了SIGHUP信号。  

那么，"后台任务"是否也会收到SIGHUP信号？  
这由 Shell 的huponexit参数决定的。
```
$ shopt | grep huponexit
```
执行上面的命令，就会看到huponexit参数的值。  

大多数Linux系统，这个参数默认关闭（off）。因此，session 退出的时候，不会把SIGHUP信号发给"后台任务"。所以，一般来说，"后台任务"不会随着 session 一起退出。  

# 四、disown 命令
通过"后台任务"启动"守护进程"并不保险，因为有的系统的huponexit参数可能是打开的（on）。  

更保险的方法是使用disown命令。它可以将指定任务从"后台任务"列表（jobs命令的返回结果）之中移除。一个"后台任务"只要不在这个列表之中，session 就肯定不会向它发出SIGHUP信号。  

```
$ node server.js &
$ disown
```
执行上面的命令以后，server.js进程就被移出了"后台任务"列表。你可以执行jobs命令验证，输出结果里面，不会有这个进程。  

## disown的用法如下
```
# 移出最近一个正在执行的后台任务
$ disown

# 移出所有正在执行的后台任务
$ disown -r

# 移出所有后台任务
$ disown -a

# 不移出后台任务，但是让它们不会收到SIGHUP信号
$ disown -h

# 根据jobId，移出指定的后台任务
$ disown %2
$ disown -h %2
```

# 五、标准 I/O
使用disown命令之后，还有一个问题。那就是，退出 session 以后，如果后台进程与标准I/O有交互，它还是会挂掉。
```js
var http = require('http');

http.createServer(function(req, res) {
  console.log('server starts...'); // 加入此行
  res.writeHead(200, {'Content-Type': 'text/plain'});
  res.end('Hello World');
}).listen(5000);
```

启动上面的脚本，然后再执行disown命令。
```
$ node server.js &
$ disown
```
接着，你退出 session，访问5000端口，就会发现连不上。  

这是因为"后台任务"的标准 I/O 继承自当前 session，disown命令并没有改变这一点。一旦"后台任务"读写标准 I/O，就会发现它已经不存在了，所以就报错终止执行。  

为了解决这个问题，需要对"后台任务"的标准 I/O 进行重定向。
```
$ node server.js > stdout.txt 2> stderr.txt < /dev/null &
$ disown
```

# 六、nohup 命令
还有比disown更方便的命令，就是nohup。
```
$ nohup node server.js &
```
nohup命令对server.js进程做了三件事。  
```
  1.阻止SIGHUP信号发到这个进程。  
  2.关闭标准输入。该进程不再能够接收任何输入，即使运行在前台。  
  3.重定向标准输出和标准错误到文件nohup.out。  
```
也就是说，nohup命令实际上将子进程与它所在的 session 分离了。  
注意，nohup命令不会自动把进程变为"后台任务"，所以必须加上&符号。

# 七、Screen 命令与 Tmux 命令
另一种思路是使用 terminal multiplexer （终端复用器：在同一个终端里面，管理多个session），典型的就是 Screen 命令和 Tmux 命令。  

它们可以在当前 session 里面，新建另一个 session。这样的话，当前 session 一旦结束，不影响其他 session。而且，以后重新登录，还可以再连上早先新建的 session。  

## Screen 的用法如下
```
# 新建一个 session
$ screen
$ node server.js
```
然后，按下ctrl + A和ctrl + D，回到原来的 session，从那里退出登录。下次登录时，再切回去。

```
$ screen -r
```
如果新建多个后台 session，就需要为它们指定名字。
```
$ screen -S name

# 切回指定 session
$ screen -r name
$ screen -r pid_number

# 列出所有 session
$ screen -ls
```
如果要停掉某个 session，可以先切回它，然后按下ctrl + c和ctrl + d。  

Tmux 比 Screen 功能更多、更强大，它的基本用法如下。
```
$ tmux
$ node server.js

# 返回原来的session
$ tmux detach
```
除了tmux detach，另一种方法是按下Ctrl + B和d ，也可以回到原来的 session。

```
# 下次登录时，返回后台正在运行服务session
$ tmux attach
```
如果新建多个 session，就需要为每个 session 指定名字。
```
# 新建 session
$ tmux new -s session_name

# 切换到指定 session
$ tmux attach -t session_name

# 列出所有 session
$ tmux list-sessions

# 退出当前 session，返回前一个 session 
$ tmux detach

# 杀死指定 session
$ tmux kill-session -t session-name
```

# 八、Node 工具
对于 Node 应用来说，可以不用上面的方法，有一些专门用来启动的工具：forever，nodemon 和 pm2。  

## 1.forever 的功能很简单，就是保证进程退出时，应用会自动重启。
```
# 作为前台任务启动
$ forever server.js

# 作为服务进程启动 
$ forever start app.js

# 停止服务进程
$ forever stop Id

# 重启服务进程
$ forever restart Id

# 监视当前目录的文件变动，一有变动就重启
$ forever -w server.js

# -m 参数指定最多重启次数
$ forever -m 5 server.js 

# 列出所有进程
$ forever list
```

## 2.nodemon一般只在开发时使用，它最大的长处在于 watch 功能，一旦文件发生变化，就自动重启进程。
```
# 默认监视当前目录的文件变化
$ nodemon server.js

＃ 监视指定文件的变化   
$ nodemon --watch app --watch libs server.js  
```
## 3.pm2 的功能最强大，除了重启进程以外，还能实时收集日志和监控。

```
# 启动应用
$ pm2 start app.js

# 指定同时起多少个进程（由CPU核心数决定），组成一个集群
$ pm2 start app.js -i max

# 列出所有任务
$ pm2 list

# 停止指定任务
$ pm2 stop 0

＃ 重启指定任务
$ pm2 restart 0

# 删除指定任务
$ pm2 delete 0

# 保存当前的所有任务，以后可以恢复
$ pm2 save

# 列出每个进程的统计数据
$ pm2 monit

# 查看所有日志
$ pm2 logs

# 导出数据
$ pm2 dump

# 重启所有进程
$ pm2 kill
$ pm2 resurect

# 启动web界面 http://localhost:9615
$ pm2 web
```

# 九、Systemd
除了专用工具以外，Linux系统有自己的守护进程管理工具 Systemd 。它是操作系统的一部分，直接与内核交互，性能出色，功能极其强大。
## 一、由来
历史上，Linux 的启动一直采用init进程。  

下面的命令用来启动服务。
```
$ sudo /etc/init.d/apache2 start
# 或者
$ service apache2 start
```
这种方法有两个缺点。
```
一是启动时间长。init进程是串行启动，只有前一个进程启动完，才会启动下一个进程。
二是启动脚本复杂。init进程只是执行启动脚本，不管其他事情。脚本需要自己处理各种情况，这往往使得脚本变得很长。
```

## 二、Systemd 概述
使用了 Systemd，就不需要再用init了。Systemd 取代了initd，成为系统的第一个进程（PID 等于 1），其他进程都是它的子进程。  
通过命令查看 Systemd 的版本。
```
$ systemctl --version
```

## 三、系统管理
Systemd 并不是一个命令，而是一组命令，涉及到系统管理的方方面面。
### 3.1 systemctl
systemctl是 Systemd 的主命令，用于管理系统。
```
# 重启系统
$ sudo systemctl reboot

# 关闭系统，切断电源
$ sudo systemctl poweroff

# CPU停止工作
$ sudo systemctl halt

# 暂停系统
$ sudo systemctl suspend

# 让系统进入冬眠状态
$ sudo systemctl hibernate

# 让系统进入交互式休眠状态
$ sudo systemctl hybrid-sleep

# 启动进入救援状态（单用户状态）
$ sudo systemctl rescue
```

### 3.2 systemd-analyze
systemd-analyze命令用于查看启动耗时。
```
# 查看启动耗时
$ systemd-analyze                                                                                       

# 查看每个服务的启动耗时
$ systemd-analyze blame

# 显示瀑布状的启动过程流
$ systemd-analyze critical-chain

# 显示指定服务的启动流
$ systemd-analyze critical-chain atd.service
```

### 3.3 hostnamectl
hostnamectl命令用于查看当前主机的信息。
```
# 显示当前主机的信息
$ hostnamectl

# 设置主机名。
$ sudo hostnamectl set-hostname rhel7
```

### 3.4 localectl
localectl命令用于查看本地化设置。
```
# 查看本地化设置
$ localectl

# 设置本地化参数。
$ sudo localectl set-locale LANG=en_GB.utf8
$ sudo localectl set-keymap en_GB
```

### 3.5 timedatectl
timedatectl命令用于查看当前时区设置。
```
# 查看当前时区设置
$ timedatectl

# 显示所有可用的时区
$ timedatectl list-timezones                                                                                   

# 设置当前时区
$ sudo timedatectl set-timezone America/New_York
$ sudo timedatectl set-time YYYY-MM-DD
$ sudo timedatectl set-time HH:MM:SS
```

### 3.6 loginctl
loginctl命令用于查看当前登录的用户。
```
# 列出当前session
$ loginctl list-sessions

# 列出当前登录用户
$ loginctl list-users

# 列出显示指定用户的信息
$ loginctl show-user ruanyf
```

## 四、Unit
### 4.1 含义
Systemd 可以管理所有系统资源。不同的资源统称为 Unit（单位）。Unit 一共分成12种。
| 名称 | 作用|
|----|----|
|Service unit：  |系统服务|
|Target unit：   |多个| Unit 构成的一个组
|Device Unit：   |硬件设备|
|Mount Unit：    |文件系统的挂载点|
|Automount Unit：|自动挂载点|
|Path Unit：     |文件或路径|
|Scope Unit：    |不是由| Systemd 启动的外部进程
|Slice Unit：    |进程组|
|Snapshot Unit： |Systemd| 快照，可以切回某个快照
|Socket Unit：   |进程间通信的| socket
|Swap Unit：     |swap| 文件
|Timer Unit：    |定时器|

systemctl list-units命令可以查看当前系统的所有 Unit 。
```
# 列出正在运行的 Unit
$ systemctl list-units

# 列出所有Unit，包括没有找到配置文件的或者启动失败的
$ systemctl list-units --all

# 列出所有没有运行的 Unit
$ systemctl list-units --all --state=inactive

# 列出所有加载失败的 Unit
$ systemctl list-units --failed

# 列出所有正在运行的、类型为 service 的 Unit
$ systemctl list-units --type=service
```

### 4.2 Unit 的状态
systemctl status命令用于查看系统状态和单个 Unit 的状态。
```
# 显示系统状态
$ systemctl status

# 显示单个 Unit 的状态
$ sysystemctl status bluetooth.service

# 显示远程主机的某个 Unit 的状态
$ systemctl -H root@rhel7.example.com status httpd.service
```
除了status命令，systemctl还提供了三个查询状态的简单方法，主要供脚本内部的判断语句使用。
```
# 显示某个 Unit 是否正在运行
$ systemctl is-active application.service

# 显示某个 Unit 是否处于启动失败状态
$ systemctl is-failed application.service

# 显示某个 Unit 服务是否建立了启动链接
$ systemctl is-enabled application.service
```

### 4.3 Unit 管理
对于用户来说，最常用的是下面这些命令，用于启动和停止 Unit（主要是 service）。
```
# 立即启动一个服务
$ sudo systemctl start apache.service

# 立即停止一个服务
$ sudo systemctl stop apache.service

# 重启一个服务
$ sudo systemctl restart apache.service

# 杀死一个服务的所有子进程
$ sudo systemctl kill apache.service

# 重新加载一个服务的配置文件
$ sudo systemctl reload apache.service

# 重载所有修改过的配置文件
$ sudo systemctl daemon-reload

# 显示某个 Unit 的所有底层参数
$ systemctl show httpd.service

# 显示某个 Unit 的指定属性的值
$ systemctl show -p CPUShares httpd.service

# 设置某个 Unit 的指定属性
$ sudo systemctl set-property httpd.service CPUShares=500
```

### 4.4 依赖关系
Unit 之间存在依赖关系：A 依赖于 B，就意味着 Systemd 在启动 A 的时候，同时会去启动 B。  

systemctl list-dependencies命令列出一个 Unit 的所有依赖。
```
$ systemctl list-dependencies nginx.service
```

上面命令的输出结果之中，有些依赖是 Target 类型（详见下文），默认不会展开显示。如果要展开 Target，就需要使用--all参数。
```
$ systemctl list-dependencies --all nginx.service
```

## 五、Unit 的配置文件
### 5.1 概述
每一个 Unit 都有一个配置文件，告诉 Systemd 怎么启动这个 Unit 。  

Systemd 默认从目录/etc/systemd/system/读取配置文件。但是，里面存放的大部分文件都是符号链接，指向目录/usr/lib/systemd/system/，真正的配置文件存放在那个目录。  

systemctl enable命令用于在上面两个目录之间，建立符号链接关系。
```
$ sudo systemctl enable clamd@scan.service
# 等同于
$ sudo ln -s '/usr/lib/systemd/system/clamd@scan.service' '/etc/systemd/system/multi-user.target.wants/clamd@scan.service'
```
如果配置文件里面设置了开机启动，systemctl enable命令相当于激活开机启动。  

与之对应的，systemctl disable命令用于在两个目录之间，撤销符号链接关系，相当于撤销开机启动。
```
$ sudo systemctl disable clamd@scan.service
```
配置文件的后缀名，就是该 Unit 的种类，比如sshd.socket。如果省略，Systemd 默认后缀名为.service，所以sshd会被理解成sshd.service。

### 5.2 配置文件的状态
systemctl list-unit-files命令用于列出所有配置文件。
```
# 列出所有配置文件
$ systemctl list-unit-files

# 列出指定类型的配置文件
$ systemctl list-unit-files --type=service
```

这个命令会输出一个列表。
```
$ systemctl list-unit-files

UNIT FILE              STATE
chronyd.service        enabled
clamd@.service         static
clamd@scan.service     disabled
```
这个列表显示每个配置文件的状态，一共有四种。
```
enabled：已建立启动链接
disabled：没建立启动链接
static：该配置文件没有[Install]部分（无法执行），只能作为其他配置文件的依赖
masked：该配置文件被禁止建立启动链接
```
注意，从配置文件的状态无法看出，该 Unit 是否正在运行。这必须执行前面提到的systemctl status命令。

```
$ systemctl status bluetooth.service
```
一旦修改配置文件，就要让 SystemD 重新加载配置文件，然后重新启动，否则修改不会生效。
```
$ sudo systemctl daemon-reload
$ sudo systemctl restart httpd.service
```

### 5.3 配置文件的格式
systemctl cat命令可以查看配置文件的内容。
```
$ systemctl cat atd.service

[Unit]
Description=ATD daemon

[Service]
Type=forking
ExecStart=/usr/bin/atd

[Install]
WantedBy=multi-user.target
```
从上面的输出可以看到，配置文件分成几个区块。每个区块的第一行，是用方括号表示的区别名，比如[Unit]。注意，配置文件的区块名和字段名，都是大小写敏感的。  

每个区块内部是一些等号连接的键值对。注意，键值对的等号两侧不能有空格。

### 5.4 配置文件的区块
[Unit]区块通常是配置文件的第一个区块，用来定义 Unit 的元数据，以及配置与其他 Unit 的关系。它的主要字段如下。
| 字段 | 作用|
|----|----|
|Description   | 简短描述|
|Documentation | 文档地址|
|Requires      | 当前 Unit 依赖的其他 Unit，如果它们没有运行，当前 Unit 会启动失败|
|Wants         | 与当前 Unit 配合的其他 Unit，如果它们没有运行，当前 Unit 不会启动失败|
|BindsTo       | 与Requires类似，它指定的 Unit 如果退出，会导致当前 Unit 停止运行|
|Before        | 如果该字段指定的 Unit 也要启动，那么必须在当前 Unit 之后启动|
|After         | 如果该字段指定的 Unit 也要启动，那么必须在当前 Unit 之前启动|
|Conflicts     | 这里指定的 Unit 不能与当前 Unit 同时运行|
|Condition...  | 当前 Unit 运行必须满足的条件，否则不会运行|
|Assert...     | 当前 Unit 运行必须满足的条件，否则会报启动失败|

[Install]通常是配置文件的最后一个区块，用来定义如何启动，以及是否开机启动。它的主要字段如下。
| 字段 | 作用|
|----|----|
|WantedBy |  它的值是一个或多个 Target，当前 Unit 激活时（enable）符号链接会放入/etc/systemd/system目录下面以 Target 名 + .wants后缀构成的子目录中|
|RequiredBy |它的值是一个或多个 Target，当前 Unit 激活时，符号链接会放入/etc/systemd/system目录下面以 Target 名 + .required后缀构成的子目录中|
|Alias |     当前 Unit 可用于启动的别名|
|Also |      当前 Unit 激活（enable）时，会被同时激活的其他 Unit|

[Service]区块用来 Service 的配置，只有 Service 类型的 Unit 才有这个区块。它的主要字段如下。
| 字段 | 作用|
|----|----|
|Type          |定义启动时的进程行为。它有以下几种值。|
|Type=simple   |默认值，执行ExecStart指定的命令，启动主进程|
|Type=forking  |以fork 方式从父进程创建子进程，创建后父进程会立即退出| 
|Type=oneshot  |一次性进程，Systemd| 会等当前服务退出，再继续往下执行
|Type=dbus    |当前服务通过D|-Bus启动
|Type=notify   |当前服务启动完毕，会通知Systemd，再继续往下执行|
|Type=idle     |若有其他任务执行完毕，当前服务才会运行|
|ExecStart     |启动当前服务的命令|
|ExecStartPre  |启动当前服务之前执行的命令|
|ExecStartPost |启动当前服务之后执行的命令|
|ExecReload    |重启当前服务时执行的命令|
|ExecStop      |停止当前服务时执行的命令|
|ExecStopPost  |停止当其服务之后执行的命令|
|RestartSec    |自动重启当前服务间隔的秒数|
|Restart      |定义何种情况 Systemd 会自动重启当前服务，可能的值包括always（总是重启）、on-success、on-failure、on-abnormal、on-abort、on-watchdog|
|TimeoutSec    |定义 Systemd 停止当前服务之前等待的秒数|
|Environment   |指定环境变量|

## 六、Target
启动计算机的时候，需要启动大量的 Unit。如果每一次启动，都要一一写明本次启动需要哪些 Unit，显然非常不方便。Systemd 的解决方案就是 Target。    

简单说，Target 就是一个 Unit 组，包含许多相关的 Unit 。启动某个 Target 的时候，Systemd 就会启动里面所有的 Unit。从这个意义上说，Target 这个概念类似于"状态点"，启动某个 Target 就好比启动到某种状态。   

传统的init启动模式里面，有 RunLevel 的概念，跟 Target 的作用很类似。不同的是，RunLevel 是互斥的，不可能多个 RunLevel 同时启动，但是多个 Target 可以同时启动。  
```
# 查看当前系统的所有 Target
$ systemctl list-unit-files --type=target

# 查看一个 Target 包含的所有 Unit
$ systemctl list-dependencies multi-user.target

# 查看启动时的默认 Target
$ systemctl get-default

# 设置启动时的默认 Target
$ sudo systemctl set-default multi-user.target

# 切换 Target 时，默认不关闭前一个 Target 启动的进程，
# systemctl isolate 命令改变这种行为，
# 关闭前一个 Target 里面所有不属于后一个 Target 的进程
$ sudo systemctl isolate multi-user.target
```

它与init进程的主要差别如下。
```
（1）默认的 RunLevel（在/etc/inittab文件设置）现在被默认的 Target 取代，位置是/etc/systemd/system/default.target，通常符号链接到graphical.target（图形界面）或者multi-user.target（多用户命令行）。

（2）启动脚本的位置，以前是/etc/init.d目录，符号链接到不同的 RunLevel 目录 （比如/etc/rc3.d、/etc/rc5.d等），现在则存放在/lib/systemd/system和/etc/systemd/system目录。

（3）配置文件的位置，以前init进程的配置文件是/etc/inittab，各种服务的配置文件存放在/etc/sysconfig目录。现在的配置文件主要存放在/lib/systemd目录，在/etc/systemd目录里面的修改可以覆盖原始设置。
```

## 七、日志管理
Systemd 统一管理所有 Unit 的启动日志。带来的好处就是，可以只用journalctl一个命令，查看所有日志（内核日志和应用日志）。日志的配置文件是/etc/systemd/journald.conf。  
```
# 查看所有日志（默认情况下 ，只保存本次启动的日志）
$ sudo journalctl

# 查看内核日志（不显示应用日志）
$ sudo journalctl -k

# 查看系统本次启动的日志
$ sudo journalctl -b
$ sudo journalctl -b -0

# 查看上一次启动的日志（需更改设置）
$ sudo journalctl -b -1

# 查看指定时间的日志
$ sudo journalctl --since="2012-10-30 18:17:16"
$ sudo journalctl --since "20 min ago"
$ sudo journalctl --since yesterday
$ sudo journalctl --since "2015-01-10" --until "2015-01-11 03:00"
$ sudo journalctl --since 09:00 --until "1 hour ago"

# 显示尾部的最新10行日志
$ sudo journalctl -n

# 显示尾部指定行数的日志
$ sudo journalctl -n 20

# 实时滚动显示最新日志
$ sudo journalctl -f

# 查看指定服务的日志
$ sudo journalctl /usr/lib/systemd/systemd

# 查看指定进程的日志
$ sudo journalctl _PID=1

# 查看某个路径的脚本的日志
$ sudo journalctl /usr/bin/bash

# 查看指定用户的日志
$ sudo journalctl _UID=33 --since today

# 查看某个 Unit 的日志
$ sudo journalctl -u nginx.service
$ sudo journalctl -u nginx.service --since today

# 实时滚动显示某个 Unit 的最新日志
$ sudo journalctl -u nginx.service -f

# 合并显示多个 Unit 的日志
$ journalctl -u nginx.service -u php-fpm.service --since today

# 查看指定优先级（及其以上级别）的日志，共有8级
# 0: emerg
# 1: alert
# 2: crit
# 3: err
# 4: warning
# 5: notice
# 6: info
# 7: debug
$ sudo journalctl -p err -b

# 日志默认分页输出，--no-pager 改为正常的标准输出
$ sudo journalctl --no-pager

# 以 JSON 格式（单行）输出
$ sudo journalctl -b -u nginx.service -o json

# 以 JSON 格式（多行）输出，可读性更好
$ sudo journalctl -b -u nginx.serviceqq
 -o json-pretty

# 显示日志占据的硬盘空间
$ sudo journalctl --disk-usage

# 指定日志文件占据的最大空间
$ sudo journalctl --vacuum-size=1G

$ sudo journalctl --vacuum-time=1years
# 指定日志文件保存多久
```