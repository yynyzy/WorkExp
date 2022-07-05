# 基本知识
# 1.SSH 是什么
历史上，网络主机之间的通信是不加密的，属于明文通信。这使得通信很不安全，一个典型的例子就是服务器登录。登录远程服务器的时候，需要将用户输入的密码传给服务器，如果这个过程是明文通信，就意味着传递过程中，线路经过的中间计算机都能看到密码，这是很可怕的。  

SSH 就是为了解决这个问题而诞生的，它能够加密计算机之间的通信，保证不被窃听或篡改。它还能对操作者进行认证（authentication）和授权（authorization）。明文的网络协议可以套用在它里面，从而实现加密。

# 2.SSH 架构
SSH 的软件架构是服务器-客户端模式（Server - Client）。在这个架构中，SSH 软件分成两个部分：向服务器发出请求的部分，称为客户端（client），OpenSSH 的实现为 ssh；接收客户端发出的请求的部分，称为服务器（server），OpenSSH 的实现为 sshd。

# 客户端
# 1.SSH 客户端
## 1.简介
OpenSSH 的客户端是二进制程序 ssh。它在 Linux/Unix 系统的位置是/usr/local/bin/ssh，Windows 系统的位置是\Program Files\OpenSSH\bin\ssh.exe。

# 2.基本用法
ssh 登录服务器的命令如下。
```
$ ssh hostname
```
上面命令中，hostname是主机名，它可以是域名，也可能是 IP 地址或局域网内部的主机名。不指定用户名的情况下，将使用客户端的当前用户名，作为远程服务器的登录用户名。如果要指定用户名，可以采用下面的语法。
```
$ ssh user@hostname
```
上面的命令中，用户名和主机名写在一起了，之间使用@分隔。  

用户名也可以使用ssh的-l参数指定，这样的话，用户名和主机名就不用写在一起了。
```
$ ssh -l username host
```

ssh 默认连接服务器的22端口，-p参数可以指定其他端口。
```
$ ssh -p 8821 foo.com
```
上面命令连接服务器foo.com的8821端口。

# 3.连接流程
ssh 连接远程服务器后，首先有一个验证过程，验证远程服务器是否为陌生地址。  

如果是第一次连接某一台服务器，命令行会显示一段文字，表示不认识这台机器，提醒用户确认是否需要连接。
```
The authenticity of host 'foo.com (192.168.121.111)' can't be established.
ECDSA key fingerprint is SHA256:Vybt22mVXuNuB5unE++yowF7lgA/9/2bLSiO3qmYWBY.
Are you sure you want to continue connecting (yes/no)?
```
上面这段文字告诉用户，foo.com这台服务器的指纹是陌生的，让用户选择是否要继续连接（输入 yes 或 no）。  

所谓“服务器指纹”，指的是 SSH 服务器公钥的哈希值。每台 SSH 服务器都有唯一一对密钥，用于跟客户端通信，其中公钥的哈希值就可以用来识别服务器。  

下面的命令可以查看某个公钥的指纹。
```
$ ssh-keygen -l -f /etc/ssh/ssh_host_ecdsa_key.pub
256 da:24:43:0b:2e:c1:3f:a1:84:13:92:01:52:b4:84:ff   (ECDSA)
```
上面的例子中，ssh-keygen -l -f命令会输出公钥/etc/ssh/ssh_host_ecdsa_key.pub的指纹。  

ssh 会将本机连接过的所有服务器公钥的指纹，都储存在本机的~/.ssh/known_hosts文件中。每次连接服务器时，通过该文件判断是否为陌生主机（陌生公钥）。在上面这段文字后面，输入yes，就可以将当前服务器的指纹也储存在本机~/.ssh/known_hosts文件中，并显示下面的提示。以后再连接的时候，就不会再出现警告了。

# 4.服务器密钥变更
服务器指纹可以防止有人恶意冒充远程主机。如果服务器的密钥发生变更（比如重装了 SSH 服务器），客户端再次连接时，就会发生公钥指纹不吻合的情况。这时，客户端就会中断连接，并显示一段警告信息。  

这时，你需要确认是什么原因，使得公钥指纹发生变更，到底是恶意劫持，还是管理员变更了 SSH 服务器公钥。如果新的公钥确认可以信任，需要继续执行连接，你可以执行下面的命令，将原来的公钥指纹从~/.ssh/known_hosts文件删除。
```
$ ssh-keygen -R hostname
```
上面命令中，hostname是发生公钥变更的主机名。

除了使用上面的命令，你也可以手工修改known_hosts文件，将公钥指纹删除。  

删除了原来的公钥指纹以后，重新执行 ssh 命令连接远程服务器，将新的指纹加入known_hosts文件，就可以顺利连接了。  

# 5.执行远程命令
可以输入想要在远程主机执行的命令。  

或者另一种执行远程命令的方法，是将命令直接写在ssh命令的后面。
```
$ ssh username@hostname command
```
上面的命令会使得 SSH 在登录成功后，立刻在远程主机上执行命令command。  

采用这种语法执行命令时，ssh 客户端不会提供互动式的 Shell 环境，而是直接将远程命令的执行结果输出在命令行。但是，有些命令需要互动式的 Shell 环境，这时就要使用-t参数。
```
# 报错
$ ssh remote.server.com emacs
emacs: standard input is not a tty
```
```
# 不报错
$ ssh -t server.example.com emacs
```
上面代码中，emacs命令需要一个互动式 Shell，所以报错。只有加上-t参数，ssh 才会分配一个互动式 Shell。

# 6.加密参数
SSH 连接的握手阶段，客户端必须跟服务端约定加密参数集（cipher suite）。  

TLS_RSA_WITH_AES_128_CBC_SHA    它的含义如下。
```
TLS：加密通信协议
RSA：密钥交换算法
AES：加密算法
128：加密算法的强度
CBC：加密算法的模式
SHA：数字签名的 Hash 函数
```
客户端向服务器发出的握手信息,握手信息（ClientHello）之中，Cipher Suites字段就是客户端列出可选的加密参数集，服务器在其中选择一个自己支持的参数集。服务器选择完毕之后，向客户端发出回应。

# 7.ssh 命令行配置项
ssh 命令有很多配置项，修改它的默认行为。

## -c参数指定加密算法。
```
$ ssh -c blowfish,3des server.example.com
# 或者
$ ssh -c blowfish -c 3des server.example.com
```
上面命令指定使用加密算法blowfish或3des。

## -C参数表示压缩数据传输。
```
$ ssh -C server.example.com
```

## -D参数指定本机的 Socks 监听端口
该端口收到的请求，都将转发到远程的 SSH 主机，又称动态端口转发。
```
$ ssh -D 1080 server
```
上面命令将本机 1080 端口收到的请求，都转发到服务器server。

## -f参数表示 SSH 连接在后台运行。

## -F参数指定配置文件。
```
$ ssh -F /usr/local/ssh/other_config
```
上面命令指定使用配置文件other_config。

## -i参数用于指定私钥
意为“identity_file”，默认值为~/.ssh/id_dsa（DSA 算法）和~/.ssh/id_rsa（RSA 算法）。注意，对应的公钥必须存放到服务器。
```
$ ssh -i my-key server.example.com
```

## -l参数指定远程登录的账户名。
```
$ ssh -l sally server.example.com
# 等同于
$ ssh sally@server.example.com
```

## -L参数设置本地端口转发
```
$ ssh  -L 9999:targetServer:80 user@remoteserver
```
上面命令中，所有发向本地9999端口的请求，都会经过remoteserver发往 targetServer 的 80 端口，这就相当于直接连上了 targetServer 的 80 端口。

## -m参数指定校验数据完整性的算法。
```
$ ssh -m hmac-sha1,hmac-md5 server.example.com
```
上面命令指定数据校验算法为hmac-sha1或hmac-md5。

## -N参数用于端口转发
表示建立的 SSH 只用于端口转发，不能执行远程命令，这样可以提供安全性。

## -o参数用来指定一个配置命令
```
$ ssh -o "Keyword Value"
```
举例来说，配置文件里面有如下内容。  
User sally  
Port 220  
通过-o参数，可以把上面两个配置命令从命令行传入。  
```
$ ssh -o "User sally" -o "Port 220" server.example.com
```
使用等号时，配置命令可以不用写在引号里面，但是等号前后不能有空格。
```
$ ssh -o User=sally -o Port=220 server.example.com
```

## -p参数指定 SSH 客户端连接的服务器端口
```
$ ssh -p 2035 server.example.com
```
上面命令连接服务器的2035端口。

# 8.-q 参数表示安静模式（quiet）
不向用户输出任何警告信息。
```
$ ssh –q foo.com
root’s password:
```
上面命令使用-q参数，只输出要求用户输入密码的提示。

## -R参数指定远程端口转发
```
$ ssh -R 9999:targetServer:902 local
```
上面命令需在跳板服务器执行，指定本地计算机local监听自己的 9999 端口，所有发向这个端口的请求，都会转向 targetServer 的 902 端口。

## -t参数在 ssh 直接运行远端命令时，提供一个互动式 Shell
```
$ ssh -t server.example.com emacs
```

## -v参数显示详细信息
```
$ ssh -v server.example.com
```
-v可以重复多次，表示信息的详细程度，比如-vv和-vvv。

## -V参数输出 ssh 客户端的版本
```
$ ssh –V
ssh: SSH Secure Shell 3.2.3 (non-commercial version) on i686-pc-linux-gnu
```
上面命令输出本机 ssh 客户端版本是SSH Secure Shell 3.2.3。

## -X参数表示打开 X 窗口转发
```
$ ssh -X server.example.com
```

## -1，-2
-1参数指定使用 SSH 1 协议。  
-2参数指定使用 SSH 2 协议。  
```
$ ssh -2 server.example.com
```

## -4，-6
-4指定使用 IPv4 协议，这是默认值。
```
$ ssh -4 server.example.com
```
-6指定使用 IPv6 协议。
```
$ ssh -6 server.example.com
```

# 9.客户端配置文件
## 位置
SSH 客户端的全局配置文件是/etc/ssh/ssh_config，用户个人的配置文件在~/.ssh/config，优先级高于全局配置文件。  
除了配置文件，~/.ssh目录还有一些用户个人的密钥文件和其他文件。下面是其中一些常见的文件。  
```
~/.ssh/id_ecdsa：用户的 ECDSA 私钥。
~/.ssh/id_ecdsa.pub：用户的 ECDSA 公钥。
~/.ssh/id_rsa：用于 SSH 协议版本2 的 RSA 私钥。
~/.ssh/id_rsa.pub：用于SSH 协议版本2 的 RSA 公钥。
~/.ssh/identity：用于 SSH 协议版本1 的 RSA 私钥。
~/.ssh/identity.pub：用于 SSH 协议版本1 的 RSA 公钥。
~/.ssh/known_hosts：包含 SSH 服务器的公钥指纹。
```

## 主机设置
用户个人的配置文件~/.ssh/config，可以按照不同服务器，列出各自的连接参数，从而不必每一次登录都输入重复的参数。下面是一个例子。
```
Host *
     Port 2222

Host remoteserver
     HostName remote.example.com
     User neo
     Port 2112
```
上面代码中，Host *表示对所有主机生效，后面的Port 2222表示所有主机的默认连接端口都是2222，这样就不用在登录时特别指定端口了。这里的缩进并不是必需的，只是为了视觉上，易于识别针对不同主机的设置。  

后面的Host remoteserver表示，下面的设置只对主机remoteserver生效。remoteserver只是一个别名，具体的主机由HostName命令指定，User和Port这两项分别表示用户名和端口。这里的Port会覆盖上面Host *部分的Port设置。  

以后，登录remote.example.com时，只要执行ssh remoteserver命令，就会自动套用 config 文件里面指定的参数。
单个主机的配置格式如下。  
```
$ ssh remoteserver
# 等同于
$ ssh -p 2112 neo@remote.example.com
```
Host命令的值可以使用通配符，比如Host *表示对所有主机都有效的设置，Host *.edu表示只对一级域名为.edu的主机有效的设置。它们的设置都可以被单个主机的设置覆盖。

## 配置命令的语法
ssh 客户端配置文件的每一行，就是一个配置命令。配置命令与对应的值之间，可以使用空格，也可以使用等号。
```
Compression yes
# 等同于
Compression = yes
```
#开头的行表示注释，会被忽略。空行等同于注释。

# 密钥登录
# 1.密钥登录的过程
SSH 密钥登录分为以下的步骤。  

预备步骤，客户端通过ssh-keygen生成自己的公钥和私钥。  

第一步，手动将客户端的公钥放入远程服务器的指定位置。  

第二步，客户端向服务器发起 SSH 登录的请求。  

第三步，服务器收到用户 SSH 登录的请求，发送一些随机数据给用户，要求用户证明自己的身份。  

第四步，客户端收到服务器发来的数据，使用私钥对数据进行签名，然后再发还给服务器。  

第五步，服务器收到客户端发来的加密签名后，使用对应的公钥解密，然后跟原始数据比较。如果一致，就允许用户登录。

# 2.ssh-keygen命令：生成密钥 
## 基本用法
密钥登录时，首先需要生成公钥和私钥。OpenSSH 提供了一个工具程序ssh-keygen命令，用来生成密钥。  

直接输入ssh-keygen，程序会询问一系列问题，然后生成密钥。  
```
$ ssh-keygen
```
通常做法是使用-t参数，指定密钥的加密算法。

```
$ ssh-keygen -t dsa
```
上面示例中，-t参数用来指定密钥的加密算法，一般会选择 DSA 算法或 RSA 算法。如果省略该参数，默认使用 RSA 算法。  

公钥文件和私钥文件都是文本文件，可以用文本编辑器看一下它们的内容。公钥文件的内容类似下面这样。  

下面的命令可以列出用户所有的公钥。  
```
$ ls -l ~/.ssh/id_*.pub
```

生成密钥以后，建议修改它们的权限，防止其他人读取。
```
$ chmod 600 ~/.ssh/id_rsa
$ chmod 600 ~/.ssh/id_rsa.pub
```

## 配置项
ssh-keygen的命令行配置项，主要有下面这些。

### -b
-b参数指定密钥的二进制位数。这个参数值越大，密钥就越不容易破解，但是加密解密的计算开销也会加大。  
一般来说，-b至少应该是1024，更安全一些可以设为2048或者更高。

### -C
-C参数可以为密钥文件指定新的注释，格式为username@host。
下面命令生成一个4096位 RSA 加密算法的密钥对，并且给出了用户名和主机名。
```
$ ssh-keygen -t rsa -b 4096 -C "your_email@domain.com"
```
### -f
-f参数指定生成的私钥文件。
```
$ ssh-keygen -t dsa -f mykey
```
上面命令会在当前目录生成私钥文件mykey和公钥文件mykey.pub。

### -F
-F参数检查某个主机名是否在known_hosts文件里面。
```
$ ssh-keygen -F example.com
```

### -N
-N参数用于指定私钥的密码（passphrase）。
```
$ ssh-keygen -t dsa -N secretword
```

### -p
-p参数用于重新指定私钥的密码（passphrase）。它与-N的不同之处在于，新密码不在命令中指定，而是执行后再输入。ssh 先要求输入旧密码，然后要求输入两遍新密码。

### -R
-R参数将指定的主机公钥指纹移出known_hosts文件。
```
$ ssh-keygen -R example.com
```

### -t
-t参数用于指定生成密钥的加密算法，一般为dsa或rsa

# 3.手动上传公钥
生成密钥以后，公钥必须上传到服务器，才能使用公钥登录。  

OpenSSH 规定，用户公钥保存在服务器的~/.ssh/authorized_keys文件。你要以哪个用户的身份登录到服务器，密钥就必须保存在该用户主目录的~/.ssh/authorized_keys文件。只要把公钥添加到这个文件之中，就相当于公钥上传到服务器了。每个公钥占据一行。如果该文件不存在，可以手动创建。  

用户可以手动编辑该文件，把公钥粘贴进去，也可以在本机计算机上，执行下面的命令。
```
$ cat ~/.ssh/id_rsa.pub | ssh user@host "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```
上面示例中，user@host要替换成你所要登录的用户名和主机名。  

注意，authorized_keys文件的权限要设为644，即只有文件所有者才能写。如果权限设置不对，SSH 服务器可能会拒绝读取该文件。
```
$ chmod 644 ~/.ssh/authorized_keys
```
只要公钥上传到服务器，下次登录时，OpenSSH 就会自动采用密钥登录，不再提示输入密码。

# 4.ssh-copy-id 命令：自动上传公钥
OpenSSH 自带一个ssh-copy-id命令，可以自动将公钥拷贝到远程服务器的~/.ssh/authorized_keys文件。如果~/.ssh/authorized_keys文件不存在，ssh-copy-id命令会自动创建该文件。  

用户在本地计算机执行下面的命令，就可以把本地的公钥拷贝到服务器。
```
$ ssh-copy-id -i key_file user@host
```
上面命令中，-i参数用来指定公钥文件，user是所要登录的账户名，host是服务器地址。如果省略用户名，默认为当前的本机用户名。执行完该命令，公钥就会拷贝到服务器。  

注意，公钥文件可以不指定路径和.pub后缀名，ssh-copy-id会自动在~/.ssh目录里面寻找。
```
$ ssh-copy-id -i id_rsa user@host
```
上面命令中，公钥文件会自动匹配到~/.ssh/id_rsa.pub。  

ssh-copy-id会采用密码登录，系统会提示输入远程服务器的密码。  

注意，ssh-copy-id是直接将公钥添加到authorized_keys文件的末尾。如果authorized_keys文件的末尾不是一个换行符，会导致新的公钥添加到前一个公钥的末尾，两个公钥连在一起，使得它们都无法生效。所以，如果authorized_keys文件已经存在，使用ssh-copy-id命令之前，务必保证authorized_keys文件的末尾是换行符（假设该文件已经存在）。

# 5.ssh-agent 命令，ssh-add 命令
## 基本用法
私钥设置了密码以后，每次使用都必须输入密码，有时让人感觉非常麻烦。比如，连续使用scp命令远程拷贝文件时，每次都要求输入密码。  

ssh-agent命令就是为了解决这个问题而设计的，它让用户在整个 Bash 对话（session）之中，只在第一次使用 SSH 命令时输入密码，然后将私钥保存在内存中，后面都不需要再输入私钥的密码了。  

第一步，使用下面的命令新建一次命令行对话。
```
$ ssh-agent bash
```
上面命令中，如果你使用的命令行环境不是 Bash，可以用其他的 Shell 命令代替。比如zsh和fish。  

如果想在当前对话启用ssh-agent，可以使用下面的命令。
```
$ eval `ssh-agent`
```

上面命令中，ssh-agent会先自动在后台运行，并将需要设置的环境变量输出在屏幕上，类似下面这样。
```
$ ssh-agent
SSH_AUTH_SOCK=/tmp/ssh-barrett/ssh-22841-agent; export SSH_AUTH_SOCK;
SSH_AGENT_PID=22842; export SSH_AGENT_PID;
echo Agent pid 22842;
```

eval命令的作用，就是运行上面的ssh-agent命令的输出，设置环境变量。  

第二步，在新建的 Shell 对话里面，使用ssh-add命令添加默认的私钥（比如~/.ssh/id_rsa，或~/.ssh/id_dsa，或~/.ssh/id_ecdsa，或~/.ssh/id_ed25519）。  
```
$ ssh-add
Enter passphrase for /home/you/.ssh/id_dsa: ********
Identity added: /home/you/.ssh/id_dsa (/home/you/.ssh/id_dsa)
```
上面例子中，添加私钥时，会要求输入密码。以后，在这个对话里面再使用密钥时，就不需要输入私钥的密码了，因为私钥已经加载到内存里面了。  

如果添加的不是默认私钥，ssh-add命令需要显式指定私钥文件。
```
$ ssh-add my-other-key-file
```
上面的命令中，my-other-key-file就是用户指定的私钥文件。  

第三步，使用 ssh 命令正常登录远程服务器。
```
$ ssh remoteHost
```
上面命令中，remoteHost是远程服务器的地址，ssh 使用的是默认的私钥。这时如果私钥设有密码，ssh 将不再询问密码，而是直接取出内存里面的私钥。  

如果要使用其他私钥登录服务器，需要使用 ssh 命令的-i参数指定私钥文件。
```
$ ssh –i OpenSSHPrivateKey remoteHost
```

最后，如果要退出ssh-agent，可以直接退出子 Shell（按下 Ctrl + d），也可以使用下面的命令。
```
$ ssh-agent -k
```

## ssh-add命令
ssh-add命令用来将私钥加入ssh-agent，它有如下的参数。

### -d
-d参数从内存中删除指定的私钥。
```
$ ssh-add -d name-of-key-file
```

### -D
-D参数从内存中删除所有已经添加的私钥。
```
$ ssh-add -D
```

-l
-l参数列出所有已经添加的私钥。
```
$ ssh-add -l
```

# 6.关闭密码登录
为了安全性，启用密钥登录之后，最好关闭服务器的密码登录。  

对于 OpenSSH，具体方法就是打开服务器 sshd 的配置文件/etc/ssh/sshd_config，将PasswordAuthentication这一项设为no。  
```
PasswordAuthentication no
```

# SSH 服务器
# 1.简介
SSH 的架构是服务器/客户端模式，两端运行的软件是不一样的。OpenSSH 的客户端软件是 ssh，服务器软件是 sshd。本章介绍 sshd 的各种知识。  

如果没有安装 sshd，可以用下面的命令安装。
```
# Debian
$ sudo aptitude install openssh-server

# Red Hat
$ sudo yum install openssh-server
```

一般来说，sshd 安装后会跟着系统一起启动。如果当前 sshd 没有启动，可以用下面的命令启动。
```
$ sshd
```
上面的命令运行后，如果提示“sshd re-exec requires execution with an absolute path”，就需要使用绝对路径来启动。这是为了防止有人出于各种目的，放置同名软件在$PATH变量指向的目录中，代替真正的 sshd。
```
# Centos、Ubuntu、OS X
$ /usr/sbin/sshd
```

上面的命令运行以后，sshd 自动进入后台，所以命令后面不需要加上&。  
除了直接运行可执行文件，也可以通过 Systemd 启动 sshd。  
```
# 启动
$ sudo systemctl start sshd.service

# 停止
$ sudo systemctl stop sshd.service

# 重启
$ sudo systemctl restart sshd.service
```

下面的命令让 sshd 在计算机下次启动时自动运行。
```
$ sudo systemctl enable sshd.service
```

# 2.sshd 配置文件
sshd 的配置文件在/etc/ssh目录，主配置文件是sshd_config，此外还有一些安装时生成的密钥。

* /etc/ssh/sshd_config：配置文件
* /etc/ssh/ssh_host_ecdsa_key：ECDSA 私钥。
* /etc/ssh/ssh_host_ecdsa_key.pub：ECDSA 公钥。
* /etc/ssh/ssh_host_key：用于 SSH 1 协议版本的 RSA 私钥。
* /etc/ssh/ssh_host_key.pub：用于 SSH 1 协议版本的 RSA 公钥。
* /etc/ssh/ssh_host_rsa_key：用于 SSH 2 协议版本的 RSA 私钥。
* /etc/ssh/ssh_host_rsa_key.pub：用于 SSH 2 协议版本的 RSA 公钥。
* /etc/pam.d/sshd：PAM 配置文件。

注意，如果重装 sshd，上面这些密钥都会重新生成，导致客户端重新连接 ssh 服务器时，会跳出警告，拒绝连接。为了避免这种情况，可以在重装 sshd 时，先备份/etc/ssh目录，重装后再恢复这个目录。  

配置文件sshd_config的格式是，每个命令占据一行。每行都是配置项和对应的值，配置项的大小写不敏感，与值之间使用空格分隔。
```
Port 2034
```
上面的配置命令指定，配置项Port的值是2034。Port写成port也可。  

配置文件还有另一种格式，就是配置项与值之间有一个等号，等号前后的空格可选。
```
Port = 2034
```
配置文件里面，#开头的行表示注释。注意，注释只能放在一行的开头，不能放在一行的结尾。空行等同于注释。  

sshd 启动时会自动读取默认的配置文件。如果希望使用其他的配置文件，可以用 sshd 命令的-f参数指定。
```
$ sshd -f /usr/local/ssh/my_config
```

上面的命令指定 sshd 使用另一个配置文件my_config。  

修改配置文件以后，可以用 sshd 命令的-t（test）检查有没有语法错误。
```
$ sshd -t
```

配置文件修改以后，并不会自动生效，必须重新启动 sshd。
```
$ sudo systemctl restart sshd.service
```

# 3.sshd 密钥
sshd 有自己的一对或多对密钥。它使用密钥向客户端证明自己的身份。所有密钥都是公钥和私钥成对出现，公钥的文件名一般是私钥文件名加上后缀.pub。  

DSA 格式的密钥文件默认为/etc/ssh/ssh_host_dsa_key（公钥为ssh_host_dsa_key.pub），RSA 格式的密钥为/etc/ssh/ssh_host_rsa_key（公钥为ssh_host_rsa_key.pub）。如果需要支持 SSH 1 协议，则必须有密钥/etc/ssh/ssh_host_key。

如果密钥不是默认文件，那么可以通过配置文件sshd_config的HostKey配置项指定。默认密钥的HostKey设置如下。  
```
# HostKey for protocol version 1
# HostKey /etc/ssh/ssh_host_key

# HostKeys for protocol version 2
# HostKey /etc/ssh/ssh_host_rsa_key
# HostKey /etc/ssh/ssh_host_dsa_ke
```
上面命令前面的#表示这些行都是注释，因为这是默认值，有没有这几行都一样。  

如果要修改密钥，就要去掉行首的#，指定其他密钥。
```
HostKey /usr/local/ssh/my_dsa_key
HostKey /usr/local/ssh/my_rsa_key
HostKey /usr/local/ssh/my_old_ssh1_key
```

# 4.sshd 配置项
以下是/etc/ssh/sshd_config文件里面的配置项。

* AcceptEnv  
AcceptEnv指定允许接受客户端通过SendEnv命令发来的哪些环境变量，即允许客户端设置服务器的环境变量清单，变量名之间使用空格分隔（AcceptEnv PATH TERM）。

* AllowGroups  
AllowGroups指定允许登录的用户组（AllowGroups groupName，多个组之间用空格分隔。如果不使用该项，则允许所有用户组登录。

* AllowUsers  
AllowUsers指定允许登录的用户，用户名之间使用空格分隔（AllowUsers user1 user2），也可以使用多行AllowUsers命令指定，用户名支持使用通配符。如果不使用该项，则允许所有用户登录。该项也可以使用用户名@域名的格式（比如AllowUsers jones@example.com）。

* AllowTcpForwarding  
AllowTcpForwarding指定是否允许端口转发，默认值为yes（AllowTcpForwarding yes），local表示只允许本地端口转发，remote表示只允许远程端口转发。

* AuthorizedKeysFile  
AuthorizedKeysFile指定储存用户公钥的目录，默认是用户主目录的ssh/authorized_keys目录（AuthorizedKeysFile .ssh/authorized_keys）。

* Banner  
Banner指定用户登录后，sshd 向其展示的信息文件（Banner /usr/local/etc/warning.txt），默认不展示任何内容。

* ChallengeResponseAuthentication  
ChallengeResponseAuthentication指定是否使用“键盘交互”身份验证方案，默认值为yes。从理论上讲，“键盘交互”身份验证方案可以向用户询问多重问题，但是实践中，通常仅询问用户密码。如果要完全禁用基于密码的身份验证，请将PasswordAuthentication和ChallengeResponseAuthentication都设置为no。

* Ciphers  
Ciphers指定 sshd 可以接受的加密算法（Ciphers 3des-cbc），多个算法之间使用逗号分隔。

* ClientAliveCountMax  
ClientAliveCountMax指定建立连接后，客户端失去响应时，服务器尝试连接的次数（ClientAliveCountMax 8）。

* ClientAliveInterval  
ClientAliveInterval指定允许客户端发呆的时间，单位为秒（ClientAliveInterval 180）。如果这段时间里面，客户端没有发送任何信号，SSH 连接将关闭。

* Compression  
Compression指定客户端与服务器之间的数据传输是否压缩。默认值为yes（Compression yes）

* DenyGroups  
DenyGroups指定不允许登录的用户组（DenyGroups groupName）。

* DenyUsers  
DenyUsers指定不允许登录的用户（DenyUsers user1），用户名之间使用空格分隔，也可以使用多行DenyUsers命令指定。

* FascistLogging  
SSH 1 版本专用，指定日志输出全部 Debug 信息（FascistLogging yes）。

* HostKey  
HostKey指定 sshd 服务器的密钥，详见前文。

* KeyRegenerationInterval  
KeyRegenerationInterval指定 SSH 1 版本的密钥重新生成时间间隔，单位为秒，默认是3600秒（KeyRegenerationInterval 3600）。

* ListenAddress  
ListenAddress指定 sshd 监听的本机 IP 地址，即 sshd 启用的 IP 地址，默认是 0.0.0.0（ListenAddress 0.0.0.0）表示在本机所有网络接口启用。可以改成只在某个网络接口启用（比如ListenAddress 192.168.10.23），也可以指定某个域名启用（比如ListenAddress server.example.com）。  
如果要监听多个指定的 IP 地址，可以使用多行ListenAddress命令。
```
ListenAddress 172.16.1.1
ListenAddress 192.168.0.1
```

* LoginGraceTime  
LoginGraceTime指定允许客户端登录时发呆的最长时间，比如用户迟迟不输入密码，连接就会自动断开，单位为秒（LoginGraceTime 60）。如果设为0，就表示没有限制。

* LogLevel  
LogLevel指定日志的详细程度，可能的值依次为QUIET、FATAL、ERROR、INFO、VERBOSE、DEBUG、DEBUG1、DEBUG2、DEBUG3，默认为INFO（LogLevel INFO）。

* MACs  
MACs指定sshd 可以接受的数据校验算法（MACs hmac-sha1），多个算法之间使用逗号分隔。

* MaxAuthTries  
MaxAuthTries指定允许 SSH 登录的最大尝试次数（MaxAuthTries 3），如果密码输入错误达到指定次数，SSH 连接将关闭。

* MaxStartups  
MaxStartups指定允许同时并发的 SSH 连接数量（MaxStartups）。如果设为0，就表示没有限制。  
这个属性也可以设为A:B:C的形式，比如MaxStartups 10:50:20，表示如果达到10个并发连接，后面的连接将有50%的概率被拒绝；如果达到20个并发连接，则后面的连接将100%被拒绝。  

* PasswordAuthentication  
PasswordAuthentication指定是否允许密码登录，默认值为yes（PasswordAuthentication yes），建议改成no（禁止密码登录，只允许密钥登录）。

* PermitEmptyPasswords  
PermitEmptyPasswords指定是否允许空密码登录，即用户的密码是否可以为空，默认为yes（PermitEmptyPasswords yes），建议改成no（禁止无密码登录）。

* PermitRootLogin  
PermitRootLogin指定是否允许根用户登录，默认为yes（PermitRootLogin yes），建议改成no（禁止根用户登录）。  
还有一种写法是写成prohibit-password，表示 root 用户不能用密码登录，但是可以用密钥登录。
```
PermitRootLogin prohibit-password
```

* PermitUserEnvironment  
PermitUserEnvironment指定是否允许 sshd 加载客户端的~/.ssh/environment文件和~/.ssh/authorized_keys文件里面的environment= options环境变量设置。默认值为no（PermitUserEnvironment no）。

* Port  
Port指定 sshd 监听的端口，即客户端连接的端口，默认是22（Port 22）。出于安全考虑，可以改掉这个端口（比如Port 8822）。  
配置文件可以使用多个Port命令，同时监听多个端口。  
```
Port 22
Port 80
Port 443
Port 8080
```


* PrintMotd  
PrintMotd指定用户登录后，是否向其展示系统的 motd（Message of the day）的信息文件/etc/motd。该文件用于通知所有用户一些重要事项，比如系统维护时间、安全问题等等。默认值为yes（PrintMotd yes），由于 Shell 一般会展示这个信息文件，所以这里可以改为no。

* PrintLastLog  
PrintLastLog指定是否打印上一次用户登录时间，默认值为yes（PrintLastLog yes）。

* Protocol  
Protocol指定 sshd 使用的协议。Protocol 1表示使用 SSH 1 协议，建议改成Protocol 2（使用 SSH 2 协议）。Protocol 2,1表示同时支持两个版本的协议。

* PubKeyAuthentication  
PubKeyAuthentication指定是否允许公钥登录，默认值为yes（PubKeyAuthentication yes）。

* QuietMode  
SSH 1 版本专用，指定日志只输出致命的错误信息（QuietMode yes）。

* RSAAuthentication  
RSAAuthentication指定允许 RSA 认证，默认值为yes（RSAAuthentication yes）。

* ServerKeyBits  
ServerKeyBits指定 SSH 1 版本的密钥重新生成时的位数，默认是768（ServerKeyBits 768）。

* StrictModes  
StrictModes指定 sshd 是否检查用户的一些重要文件和目录的权限。默认为yes（StrictModes yes），即对于用户的 SSH 配置文件、密钥文件和所在目录，SSH 要求拥有者必须是根用户或用户本人，用户组和其他人的写权限必须关闭。

* SyslogFacility  
SyslogFacility指定 Syslog 如何处理 sshd 的日志，默认是 Auth（SyslogFacility AUTH）。

* TCPKeepAlive  
TCPKeepAlive指定打开 sshd 跟客户端 TCP 连接的 keepalive 参数（TCPKeepAlive yes）。

* UseDNS  
UseDNS指定用户 SSH 登录一个域名时，服务器是否使用 DNS，确认该域名对应的 IP 地址包含本机（UseDNS yes）。打开该选项意义不大，而且如果 DNS 更新不及时，还有可能误判，建议关闭。

* UseLogin  
UseLogin指定用户认证内部是否使用/usr/bin/login替代 SSH 工具，默认为no（UseLogin no）。

* UserPrivilegeSeparation  
UserPrivilegeSeparation指定用户认证通过以后，使用另一个子线程处理用户权限相关的操作，这样有利于提高安全性。默认值为yes（UsePrivilegeSeparation yes）。

* VerboseMode  
SSH 2 版本专用，指定日志输出详细的 Debug 信息（VerboseMode yes）。

* X11Forwarding  
X11Forwarding指定是否打开 X window 的转发，默认值为 no（X11Forwarding no）。  
修改配置文件以后，可以使用下面的命令验证，配置文件是否有语法错误。新的配置文件生效，必须重启 sshd。
```
$ sshd -t
```
```
$ sudo systemctl restart sshd
```

# sshd 的命令行配置项
sshd 命令有一些配置项。这些配置项在调用时指定，可以覆盖配置文件的设置。

## -d
-d参数用于显示 debug 信息。

## -D
-D参数指定 sshd 不作为后台守护进程运行。

## -e
-e参数将 sshd 写入系统日志 syslog 的内容导向标准错误（standard error）。

## -f
-f参数指定配置文件的位置。

## -h
-h参数用于指定密钥。
```
$ sshd -h /usr/local/ssh/my_rsa_key
```

## -o
-o参数指定配置文件的一个配置项和对应的值。
```
$ sshd -o "Port 2034"
```
配置项和对应值之间，可以使用等号。
```
$ sshd -o "Port = 2034"
```
如果省略等号前后的空格，也可以不使用引号。
```
$ sshd -o Port=2034
```
-o参数可以多个一起使用，用来指定多个配置关键字。

## -p
-p参数指定 sshd 的服务端口。
```
$ sshd -p 2034
```
上面命令指定 sshd 在2034端口启动。

-p参数可以指定多个端口。
```
$ sshd -p 2222 -p 3333
```

## -t
-t参数检查配置文件的语法是否正确。

# scp 命令
scp是 SSH 提供的一个客户端程序，用来在两台主机之间加密传送文件（即复制文件）。

# 1.简介
scp是 secure copy 的缩写，相当于cp命令 + SSH。它的底层是 SSH 协议，默认端口是22，相当于先使用ssh命令登录远程主机，然后再执行拷贝操作。  

scp主要用于以下三种复制操作。
* 本地复制到远程。
* 远程复制到本地。
* 两个远程系统之间的复制。
使用scp传输数据时，文件和密码都是加密的，不会泄漏敏感信息。

# 2.基本语法
scp的语法类似cp的语法。
```
$ scp source destination
```
上面命令中，source是文件当前的位置，destination是文件所要复制到的位置。它们都可以包含用户名和主机名。

```
$ scp user@host:foo.txt bar.txt
```
上面命令将远程主机（user@host）用户主目录下的foo.txt，复制为本机当前目录的bar.txt。可以看到，主机与文件之间要使用冒号（:）分隔。  

scp会先用 SSH 登录到远程主机，然后在加密连接之中复制文件。客户端发起连接后，会提示用户输入密码，这部分是跟 SSH 的用法一致的。  

用户名和主机名都是可以省略的。用户名的默认值是本机的当前用户名，主机名默认为当前主机。注意，scp会使用 SSH 客户端的配置文件.ssh/config，如果配置文件里面定义了主机的别名，这里也可以使用别名连接。  

scp支持一次复制多个文件。
```
$ scp source1 source2 destination
```
上面命令会将source1和source2两个文件，复制到destination。  

注意，如果所要复制的文件，在目标位置已经存在同名文件，scp会在没有警告的情况下覆盖同名文件。

# 3.用法示例
* （1）本地文件复制到远程

复制本机文件到远程系统的用法如下。
```
# 语法
$ scp SourceFile user@host:directory/TargetFile
# 示例
$ scp file.txt remote_username@10.10.0.2:/remote/directory
```
下面是复制整个目录的例子。
```
# 将本机的 documents 目录拷贝到远程主机，
# 会在远程主机创建 documents 目录
$ scp -r documents username@server_ip:/path_to_remote_directory

# 将本机整个目录拷贝到远程目录下
$ scp -r localmachine/path_to_the_directory username@server_ip:/path_to_remote_directory/

# 将本机目录下的所有内容拷贝到远程目录下
$ scp -r localmachine/path_to_the_directory/* username@server_ip:/path_to_remote_directory/
```
* （2）远程文件复制到本地
从远程主机复制文件到本地的用法如下。
```
# 语法
$ scp user@host:directory/SourceFile TargetFile

# 示例
$ scp remote_username@10.10.0.2:/remote/file.txt /local/directory
```
下面是复制整个目录的例子。
```
# 拷贝一个远程目录到本机目录下
$ scp -r username@server_ip:/path_to_remote_directory local-machine/path_to_the_directory/

# 拷贝远程目录下的所有内容，到本机目录下
$ scp -r username@server_ip:/path_to_remote_directory/* local-machine/path_to_the_directory/
$ scp -r user@host:directory/SourceFolder TargetFolder
```
* （3）两个远程系统之间的复制
本机发出指令，从远程主机 A 拷贝到远程主机 B 的用法如下。
```
# 语法
$ scp user@host1:directory/SourceFile user@host2:directory/SourceFile

# 示例
$ scp user1@host1.com:/files/file.txt user2@host2.com:/files
```
系统将提示你输入两个远程帐户的密码。数据将直接从一个远程主机传输到另一个远程主机。

# 4.配置项
## -c
-c参数用来指定文件拷贝数据传输的加密算法。
```
$ scp -c blowfish some_file your_username@remotehost.edu:~
```
上面代码指定加密算法为blowfish。

## -C
-C参数表示是否在传输时压缩文件。
```
$ scp -c blowfish -C local_file your_username@remotehost.edu:~
```

## -F
-F参数用来指定 ssh_config 文件，供 ssh 使用。
```
$ scp -F /home/pungki/proxy_ssh_config Label.pdf root@172.20.10.8:/root
```

## -i
-i参数用来指定密钥。
```
$ scp -vCq -i private_key.pem ~/test.txt root@192.168.1.3:/some/path/test.txt
```

## -l
-l参数用来限制传输数据的带宽速率，单位是 Kbit/sec。对于多人分享的带宽，这个参数可以留出一部分带宽供其他人使用。
```
$ scp -l 80 yourusername@yourserver:/home/yourusername/* .
```
上面代码中，scp命令占用的带宽限制为每秒 80K 比特位，即每秒 10K 字节。

## -p
-p参数用来保留修改时间（modification time）、访问时间（access time）、文件状态（mode）等原始文件的信息。
```
$ scp -p ~/test.txt root@192.168.1.3:/some/path/test.txt
```

## -P
-P参数用来指定远程主机的 SSH 端口。如果远程主机使用默认端口22，可以不用指定，否则需要用-P参数在命令中指定。
```
$ scp -P 2222 user@host:directory/SourceFile TargetFile
```
## -q
-q参数用来关闭显示拷贝的进度条。
```
$ scp -q Label.pdf mrarianto@202.x.x.x:.
```

## -r
-r参数表示是否以递归方式复制目录。

## -v
-v参数用来显示详细的输出。
```
$ scp -v ~/test.txt root@192.168.1.3:/root/help2356.txt
```

# rsync 命令
# 1.简介
rsync 是一个常用的 Linux 应用程序，用于文件同步。  
它可以在本地计算机与远程计算机之间，或者两个本地目录之间同步文件（但不支持两台远程计算机之间的同步）。它也可以当作文件复制工具，替代cp和mv命令。

# 2.安装
如果本机或者远程计算机没有安装 rsync，可以用下面的命令安装。
```
# Debian
$ sudo apt-get install rsync

# Red Hat
$ sudo yum install rsync

# Arch Linux
$ sudo pacman -S rsync
```
注意，传输的双方都必须安装 rsync。

# 基本用法
rsync 可以用于本地计算机的两个目录之间的同步。下面就用本地同步举例，顺便讲解 rsync 几个主要参数的用法。

## -r参数
本机使用 rsync 命令时，可以作为cp和mv命令的替代方法，将源目录拷贝到目标目录。
```
$ rsync -r source destination
```
上面命令中，-r表示递归，即包含子目录。注意，-r是必须的，否则 rsync 运行不会成功。source目录表示源目录，destination表示目标目录。上面命令执行以后，目标目录下就会出现destination/source这个子目录。  

如果有多个文件或目录需要同步，可以写成下面这样。
```
$ rsync -r source1 source2 destination
```
上面命令中，source1、source2都会被同步到destination目录。

## -a参数
-a参数可以替代-r，除了可以递归同步以外，还可以同步元信息（比如修改时间、权限等）。由于 rsync 默认使用文件大小和修改时间决定文件是否需要更新，所以-a比-r更有用。下面的用法才是常见的写法。
```
$ rsync -a source destination
```
目标目录destination如果不存在，rsync 会自动创建。执行上面的命令后，源目录source被完整地复制到了目标目录destination下面，即形成了destination/source的目录结构。  

如果只想同步源目录source里面的内容到目标目录destination，则需要在源目录后面加上斜杠。
```
$ rsync -a source/ destination
```
上面命令执行后，source目录里面的内容，就都被复制到了destination目录里面，并不会在destination下面创建一个source子目录。

## -n参数
如果不确定 rsync 执行后会产生什么结果，可以先用-n或--dry-run参数模拟执行的结果。
```
$ rsync -anv source/ destination
```
上面命令中，-n参数模拟命令执行的结果，并不真的执行命令。-v参数则是将结果输出到终端，这样就可以看到哪些内容会被同步。

## --delete参数
默认情况下，rsync 只确保源目录的所有内容（明确排除的文件除外）都复制到目标目录。它不会使两个目录保持相同，并且不会删除文件。如果要使得目标目录成为源目录的镜像副本，则必须使用--delete参数，这将删除只存在于目标目录、不存在于源目录的文件。
```
$ rsync -av --delete source/ destination
```
上面命令中，--delete参数会使得destination成为source的一个镜像。

# 排除文件
## --exclude参数
有时，我们希望同步时排除某些文件或目录，这时可以用--exclude参数指定排除模式。
```
$ rsync -av --exclude='*.txt' source/ destination
# 或者
$ rsync -av --exclude '*.txt' source/ destination
```
上面命令排除了所有 TXT 文件。  

注意，rsync 会同步以“点”开头的隐藏文件，如果要排除隐藏文件，可以这样写--exclude=".*"。  

如果要排除某个目录里面的所有文件，但不希望排除目录本身，可以写成下面这样。  
```
$ rsync -av --exclude 'dir1/*' source/ destination
```
多个排除模式，可以用多个--exclude参数。
```
$ rsync -av --exclude 'file1.txt' --exclude 'dir1/*' source/ destination
```
多个排除模式也可以利用 Bash 的大扩号的扩展功能，只用一个--exclude参数。
```
$ rsync -av --exclude={'file1.txt','dir1/*'} source/ destination
```
如果排除模式很多，可以将它们写入一个文件，每个模式一行，然后用--exclude-from参数指定这个文件。
```
$ rsync -av --exclude-from='exclude-file.txt' source/ destination
```

## --include参数
--include参数用来指定必须同步的文件模式，往往与--exclude结合使用。
```
$ rsync -av --include="*.txt" --exclude='*' source/ destination
```
上面命令指定同步时，排除所有文件，但是会包括 TXT 文件。

# 远程同步
## SSH 协议
rsync 除了支持本地两个目录之间的同步，也支持远程同步。它可以将本地内容，同步到远程服务器。
```
$ rsync -av source/ username@remote_host:destination
```
也可以将远程内容同步到本地。
```
$ rsync -av username@remote_host:source/ destination
```
rsync 默认使用 SSH 进行远程登录和数据传输。  

由于早期 rsync 不使用 SSH 协议，需要用-e参数指定协议，后来才改的。所以，下面-e ssh可以省略。
```
$ rsync -av -e ssh source/ user@remote_host:/destination
```
但是，如果 ssh 命令有附加的参数，则必须使用-e参数指定所要执行的 SSH 命令。
```
$ rsync -av -e 'ssh -p 2234' source/ user@remote_host:/destination
```
上面命令中，-e参数指定 SSH 使用2234端口。

## rsync 协议
除了使用 SSH，如果另一台服务器安装并运行了 rsync 守护程序，则也可以用rsync://协议（默认端口873）进行传输。具体写法是服务器与目标目录之间使用双冒号分隔::。
```
$ rsync -av source/ 192.168.122.32::module/destination
```
注意，上面地址中的module并不是实际路径名，而是 rsync 守护程序指定的一个资源名，由管理员分配。  

如果想知道 rsync 守护程序分配的所有 module 列表，可以执行下面命令。
```
$ rsync rsync://192.168.122.32
```
rsync 协议除了使用双冒号，也可以直接用rsync://协议指定地址。
```
$ rsync -av source/ rsync://192.168.122.32/module/destination
```

# 增量备份
rsync 的最大特点就是它可以完成增量备份，也就是默认只复制有变动的文件。  

除了源目录与目标目录直接比较，rsync 还支持使用基准目录，即将源目录与基准目录之间变动的部分，同步到目标目录。

具体做法是，第一次同步是全量备份，所有文件在基准目录里面同步一份。以后每一次同步都是增量备份，只同步源目录与基准目录之间有变动的部分，将这部分保存在一个新的目标目录。这个新的目标目录之中，也是包含所有文件，但实际上，只有那些变动过的文件是存在于该目录，其他没有变动的文件都是指向基准目录文件的硬链接。  
--link-dest参数用来指定同步时的基准目录。
```
$ rsync -a --delete --link-dest /compare/path /source/path /target/path
``` 
上面命令中，--link-dest参数指定基准目录/compare/path，然后源目录/source/path跟基准目录进行比较，找出变动的文件，将它们拷贝到目标目录/target/path。那些没变动的文件则会生成硬链接。这个命令的第一次备份时是全量备份，后面就都是增量备份了。

# 配置项
-a、--archive参数表示存档模式，保存所有的元数据，比如修改时间（modification time）、权限、所有者等，并且软链接也会同步过去。  

--append参数指定文件接着上次中断的地方，继续传输。  

--append-verify参数跟--append参数类似，但会对传输完成后的文件进行一次校验。如果校验失败，将重新发送整个文件。  

-b、--backup参数指定在删除或更新目标目录已经存在的文件时，将该文件更名后进行备份，默认行为是删除。更名规则是添加由--suffix参数指定的文件后缀名，默认是~。  

--backup-dir参数指定文件备份时存放的目录，比如--backup-dir=/path/to/backups。  

--bwlimit参数指定带宽限制，默认单位是 KB/s，比如--bwlimit=100。  

-c、--checksum参数改变rsync的校验方式。默认情况下，rsync 只检查文件的大小和最后修改日期是否发生变化，如果发生变化，就重新传输；使用这个参数以后，则通过判断文件内容的校验和，决定是否重新传输。  

--delete参数删除只存在于目标目录、不存在于源目标的文件，即保证目标目录是源目标的镜像。  

-e参数指定使用 SSH 协议传输数据。  

--exclude参数指定排除不进行同步的文件，比如--exclude="*.iso"。  

--exclude-from参数指定一个本地文件，里面是需要排除的文件模式，每个模式一行。  

--existing、--ignore-non-existing参数表示不同步目标目录中不存在的文件和目录。  

-h参数表示以人类可读的格式输出。  

-h、--help参数返回帮助信息。  

-i参数表示输出源目录与目标目录之间文件差异的详细情况。  

--ignore-existing参数表示只要该文件在目标目录中已经存在，就跳过去，不再同步这些文件。  

--include参数指定同步时要包括的文件，一般与--exclude结合使用。  

--link-dest参数指定增量备份的基准目录。  

-m参数指定不同步空目录。  

--max-size参数设置传输的最大文件的大小限制，比如不超过200KB（--max-size='200k'）。  

--min-size参数设置传输的最小文件的大小限制，比如不小于10KB（--min-size=10k）。  

-n参数或--dry-run参数模拟将要执行的操作，而并不真的执行。配合-v参数使用，可以看到哪些内容会被同步过去。  

-P参数是--progress和--partial这两个参数的结合。  

--partial参数允许恢复中断的传输。不使用该参数时，rsync会删除传输到一半被打断的文件；使用该参数后，传输到一半的文件也会同步到目标目录，下次同步时再恢复中断的传输。一般需要与--append或--append-verify配合使用。  

--partial-dir参数指定将传输到一半的文件保存到一个临时目录，比如--partial-dir=.rsync-partial。一般需要与--append或--append-verify配合使用。  

--progress参数表示显示进展。  

-r参数表示递归，即包含子目录。  

--remove-source-files参数表示传输成功后，删除发送方的文件。  

--size-only参数表示只同步大小有变化的文件，不考虑文件修改时间的差异。  

--suffix参数指定文件名备份时，对文件名添加的后缀，默认是~。  

-u、--update参数表示同步时跳过目标目录中修改时间更新的文件，即不同步这些有更新的时间戳的文件。  

-v参数表示输出细节。-vv表示输出更详细的信息，-vvv表示输出最详细的信息。  

--version参数返回 rsync 的版本。  

-z参数指定同步时压缩数据。  



