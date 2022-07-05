# 一、DNS 是什么？
DNS （Domain Name System 的缩写）的作用非常简单，就是根据域名查出IP地址。

# 二、查询过程
虽然只需要返回一个IP地址，但是DNS的查询过程非常复杂，分成多个步骤。  
工具软件dig可以显示整个查询过程。
```
$ dig math.stackexchange.com
```

# 三、DNS服务器
首先，本机一定要知道DNS服务器的IP地址，否则上不了网。通过DNS服务器，才能知道某个域名的IP地址到底是什么。  

DNS服务器的IP地址，有可能是动态的，每次上网时由网关分配，这叫做DHCP机制；也有可能是事先指定的固定地址。Linux系统里面，DNS服务器的IP地址保存在/etc/resolv.conf文件。

# 四、域名的层级
比如，域名math.stackexchange.com显示为math.stackexchange.com.。这不是疏忽，而是所有域名的尾部，实际上都有一个根域名。  

举例来说，www.example.com真正的域名是www.example.com.root，简写为www.example.com.。因为，根域名.root对于所有域名都是一样的，所以平时是省略的。  

根域名的下一级，叫做"顶级域名"（top-level domain，缩写为TLD），比如.com、.net；再下一级叫做"次级域名"，比如www.example.com里面的.example，这一级域名是用户可以注册的；再下一级是主机名（host），比如www.example.com里面的www，又称为"三级域名"，这是用户在自己的域里面为服务器分配的名称，是用户可以任意分配的。  

总结一下，域名的层级结构如下。
```
主机名.次级域名.顶级域名.根域名
 即
host.sld.tld.root
```

# 五、根域名服务器
DNS服务器根据域名的层级，进行分级查询。  
需要明确的是，每一级域名都有自己的NS记录，NS记录指向该级域名的域名服务器。这些服务器知道下一级域名的各种记录。  

所谓"分级查询"，就是从根域名开始，依次查询每一级域名的NS记录，直到查到最终的IP地址，过程大致如下。
```
从"根域名服务器"查到"顶级域名服务器"的NS记录和A记录（IP地址）
从"顶级域名服务器"查到"次级域名服务器"的NS记录和A记录（IP地址）
从"次级域名服务器"查出"主机名"的IP地址
```
仔细看上面的过程，你可能发现了，没有提到DNS服务器怎么知道"根域名服务器"的IP地址。回答是"根域名服务器"的NS记录和IP地址一般是不会变化的，所以内置在DNS服务器里面。  

# 六、分级查询的实例
dig命令的+trace参数可以显示DNS的整个分级查询过程。
```
$ dig +trace math.stackexchange.com
```

# 七、NS 记录的查询
dig命令可以单独查看每一级域名的NS记录。
```
$ dig ns com
$ dig ns stackexchange.com
```
+short参数可以显示简化的结果。
```
$ dig +short ns com
$ dig +short ns stackexchange.com
```

# 八、DNS的记录类型
域名与IP之间的对应关系，称为"记录"（record）。根据使用场景，"记录"可以分成不同的类型（type），前面已经看到了有A记录和NS记录。  
常见的DNS记录类型如下。
```
（1） A：地址记录（Address），返回域名指向的IP地址。

（2） NS：域名服务器记录（Name Server），返回保存下一级域名信息的服务器地址。该记录只能设置为域名，不能设置为IP地址。

（3）MX：邮件记录（Mail eXchange），返回接收电子邮件的服务器地址。

（4）CNAME：规范名称记录（Canonical Name），返回另一个域名，即当前查询的域名是另一个域名的跳转，详见下文。

（5）PTR：逆向查询记录（Pointer Record），只用于从IP地址查询域名，详见下文。
```
一般来说，为了服务的安全可靠，至少应该有两条NS记录，而A记录和MX记录也可以有多条，这样就提供了服务的冗余性，防止出现单点失败。  

CNAME记录主要用于域名的内部跳转，为服务器配置提供灵活性，用户感知不到。  
PTR记录用于从IP地址反查域名。dig命令的-x参数用于查询PTR记录。
```
$ dig -x 192.30.252.153
```

# 九、其他DNS工具
除了dig，还有一些其他小工具也可以使用。

## 1.host 命令

host命令可以看作dig命令的简化版本，返回当前请求域名的各种记录。
```
$ host github.com
```
host命令也可以用于逆向查询，即从IP地址查询域名，等同于dig -x <ip>。
```
$ host 192.30.252.153
```

## 2.nslookup 命令
nslookup命令用于互动式地查询域名记录。
```
$ nslookup

> facebook.github.io
```
3）whois 命令

## 3.whois命令用来查看域名的注册情况。
```
$ whois github.com
```