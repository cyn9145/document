# 服务-openvpn

[TOC]

## openvpn概述

  [openvpn官网](https://openvpn.net)

## 服务安装

1. 安装必要软件

    * 安装[阿里云](http://mirrors.aliyun.com)的yum源

        ```
        rpm -ivh http://mirrors.aliyun.com/epel/epel-release-latest-7.noarch.rpm
        ```

    * 安装编译时需要的软件

        ```
        yum install -y wget net-tools lrzsz gcc unzip
        ```
    * 下载软件包

        ```
        wget https://swupdate.openvpn.org/community/releases/openvpn-2.4.0.zip
        ```
2. 安装依赖

    * 安装openvpn必要的依赖

        ```
        yum install -y openssl-devel lzo-devel pam-devel net-tools
        ```
3. 安装软件

    * 配置openvpn

        ```
        cd ./openvpn-2.4.0 
        ./configure --prefix=/usr/local/openvpn-2.4.0 && \
        make && make install
        ```
4. 配置目录等文件

    * 添加openvpn配置文件目录，以及数据目录
    
        ```
        mkdir -p /usr/local/openvpn-2.4.0/{conf,date,scripts}
        ```
        
        > conf目录为配置文件目录
        > data目录为账号密码文件目录
        > scripts为账号密码认证模式下，认证脚本的路径

## 服务配置文件

 openvpn的所需要的配置文件可以在源码包中找到
    
```
openvpn-2.4.0/sample/
├── Makefile
├── Makefile.am
├── Makefile.in
├── sample-config-files
├── sample-keys
├── sample-plugins
├── sample-scripts
└── sample-windows
```

1. 服务器端配置

    * 将服务端所需要的证书文件等拷贝到服务端的`conf`目录下

        ```
        cd openvpn-2.4.0/sample/sample-keys
        cp ca.crt ca.key dh2048.pem server.crt server.key ta.key /usr/local/openvpn-2.4.0/conf/
        ```
    * 将所需要的配置文件拷贝到配置配置文件夹`conf`下
        
        ```
        cd openvpn-2.4.0/sample/sample-config-files/
        cp server.conf /usr/local/openvpn-2.4.0/conf/
        ```
        
        并将监听IP地址改为外网IP
        
        ```
        [root@linux-node01 ~]# grep '^local' /usr/local/openvpn-2.4.0/conf/server.conf           
    local 192.168.56.11
        ```
        
    * 修改连接IP

        ```
        push "route 192.168.3.0 255.255.255.0"
        ```
        **注意：** 如果不修改，阿里云无法链接，因此添加到推送的目的IP段
        
2. 客户端配置
    * 将客户端所需要的证书文件等拷贝到客户端的`conf`目录下

        ```
        cd openvpn-2.4.0/sample/sample-keys
        cp ca.crt client.crt client.key ta.key server.crt /usr/local/openvpn-2.4.0/conf/   
        ```
    * 将所需要的配置文件拷贝到配置配置文件夹`conf`下
        
        ```
        cd openvpn-2.4.0/sample/sample-config-files/
        cp clinet.conf /usr/local/openvpn-2.4.0/conf/
        ```
        并修改远程服务端IP地址
        
        ```
        [root@linux-node02 ~]# grep '^remote' /usr/local/openvpn-2.4.0/conf/client.conf   
        remote 192.168.56.11 1194
        ```

3. 启动服务

    * 服务端

        ```
        cd /usr/local/openvpn-2.4.0/conf
        ../sbin/openvpn ./server.conf 
        ```
    * 客户端
        
        ```
        cd /usr/local/openvpn-2.4.0/conf
        ../sbin/openvpn ./client.conf 
        ```

    到此openvpn已经安装完成
    可通过ping服务端的内网地址进行验证
    
    > 注意，由于服务端以及客户端的配置文件中需要的证书文件以及日志文件都未带路径，因此只能在server.conf的conf文件夹下启动，为了方便，请将证书文件，以及日志，登录IP等文件修改为`绝对路径 + 文件名`这样就可以在任何目录下启动了
    
# 改进认证方式

如果服务端使用证书认证，安全得到了保证，但伸缩性不行，因此需要将OpenVPN的的认证方式改为帐号密码模式

## 使用密码验证

1. 服务端修改

    * 修改服务端配置文件

        添加配置到服务端配置文件末尾
        
        ```
        client-cert-not-required
        auth-user-pass-verify /usr/local/openvpn-2.4.0/scripts/checkpsw.sh via-env
        username-as-common-name
        script-security 3
        ```
        
        添加账号密码文件
        
        ```
        echo 'test 123456' >> /usr/local/openvpn-2.4.0/date/psw-file
        ```
        
        为了增加VPN的账号密码文件的安全性，请将`psw-file`文件改为400权限，并将所属用户改为nobody
        
        ```
        chmod 400 /usr/local/openvpn-2.4.0/date/psw-file
        chown nobody.nobody /usr/local/openvpn-2.4.0/date/psw-file
        ```
        
        将用户密码校验脚本放置在oepnvpn的目录中
    
        > ls /usr/local/openvpn-2.4.0/scripts/checkpsw.sh
        
        ```
        #!/bin/sh
        ###########################################################
        # checkpsw.sh (C) 2004 Mathias Sundman <mathias@openvpn.se>
        #
        # This script will authenticate OpenVPN users against
        # a plain text file. The passfile should simply contain
        # one row per user with the username first followed by
        # one or more space(s) or tab(s) and then the password.
        
        
        PASSFILE="/usr/local/openvpn-2.4.0/date/psw-file"
        LOG_FILE="/var/log/openvpn-password.log"
        TIME_STAMP=`date "+%Y-%m-%d %T"`
        
        ###########################################################
        
        if [ ! -r "${PASSFILE}" ]; then
        echo "${TIME_STAMP}: Could not open password file \"${PASSFILE}\" for reading." >> ${LOG_FILE}
        exit 1
        fi
        
        CORRECT_PASSWORD=`awk '!/^;/&&!/^#/&&$1=="'${username}'"{print $2;exit}' ${PASSFILE}`
        
        if [ "${CORRECT_PASSWORD}" = "" ]; then 
        echo "${TIME_STAMP}: User does not exist: username=\"${username}\", password=\"${password}\"." >> ${LOG_FILE}
        exit 1
        fi
        
        if [ "${password}" = "${CORRECT_PASSWORD}" ]; then 
        echo "${TIME_STAMP}: Successful authentication: username=\"${username}\"." >> ${LOG_FILE}
        exit 0
        fi
        
        echo "${TIME_STAMP}: Incorrect password: username=\"${username}\", password=\"${password}\"." >> ${LOG_FILE}
        exit 1
        ```

    * 修改客户端配置文件

        修改到客户端配置文件
    
        ```
        ca ca.crt
        #cert client.crt
        #key client.key
        auth-user-pass
        ```

## 使用pam_mysql认证

虽然账号密码认证方式已经很方便了，通过修改账号密码文件的权限也可以保证vpn的安全性，但是仍有改进的方式，既然使用了账号密码模式，那么就可以将账号密码存储在数据库中统一管理，类似vsftp采用数据库存储账号密码一样，这样我们就可以通过只管理数据库而管理vpn、vsftp等工具了

1. 服务端修改配置为


    ```
    client-cert-not-required
    username-as-common-name
    script-security 3
    plugin /usr/local/openvpn-2.4.0/lib/openvpn/plugins/openvpn-plugin-auth-pam.so  openvpn
    ```

2. 安装测试pam联通性工具

    如果安装了`testsaslauthd` 忽略步骤1
    ```
    yum install -y cyrus-sasl cyrus-sasl-plain cyrus-sasl-devel cyrus-sasl-lib cyrus-sasl-gssapi
    ```
3. 安装pam_mysql

    [pam_mysql-0.7RC1.tar.gz](https://sourceforge.net/projects/pam-mysql/)下载地址
    
    ```
    yum install -y mysql-devel mysql
    
    tar xf pam_mysql-0.7RC1.tar.gz
    
    ./configure -with-pam-mods-dir=/lib64/security
    make && make install
    ```
    
4. 创建数据库表用于管理账号密码

    创建openvpn数据库
    
    ```
    create database openvpn;
    ```
    
    创建openvpn用户表
    
    ```
    use openvpn;
    create table vpnuser (
       name         char (100) not null,
       password     char (255) default null,
       active       int (10) not null default 1,
       primary key  (name)
       );
    ```
    创建openvpn登录记录表
    
    ```
    create table logtable (
       msg     char (254),
       user    char (100),
       pid     char (100),
       host    char (100),
       rhost   char (100),
       time    char (100)
       );
    ```
    插入一条vpn用户
    
    ```
    insert into vpnuser (name,password) values ('test',password('test'));
    ```
    授权一个用户用于连接数据库
    
    ```
    grant all on openvpn.* to openvpn@'localhost' identified by 'openvpn';
    ```

5. 添加一个openvpn的pam配置

    > cat /etc/pam.d/openvpn
    
    ```
    auth sufficient pam_mysql.so user=openvpn passwd=openvpn host=localhost db=openvpn table=vpnuser usercolumn=name passwdcolumn=password [where=vpnuser.active=1] sqllog=0 crypt=2 sqllog=true logtable=logtable logmsgcolumn=msg logusercolumn=user logpidcolumn=pid loghostcolumn=host logrhostcolumn=rhost logtimecolumn=time
    account required pam_mysql.so user=openvpn passwd=openvpn host=localhost db=openvpn table=vpnuser usercolumn=name passwdcolumn=password [where=vpnuser.active=1] sqllog=0 crypt=2 sqllog=true logtable=logtable logmsgcolumn=msg logusercolumn=user logpidcolumn=pid loghostcolumn=host logrhostcolumn=rhost logtimecolumn=time
    ```
6. 测试联通性

    ```
    systemctl start saslauthd
    [root@linux-node01 conf]# testsaslauthd -u test -ptest -s /etc/pam.d/openvpn 
    0: OK "Success."
    ```
    
    如果如上就表示pam认证通过
    
7. 如果用pam_mysql认证，`2.0.9`版本以后的有一个bug及时pam测试联通通过之后客户端依旧无法连接，这是由于`open_pam_auth.so`插件有bug，因此需要下载一个低版本的，如`2.0.9`版本

    认证失败日志如下，如果有如下错误可是采用此方法
    
    ```
    Sun Feb  5 18:57:10 2017 192.168.56.12:52723 TLS: Initial packet from [AF_INET]192.168.56.12:52723, sid=251aae9b 70602686
    Sun Feb  5 18:57:10 2017 192.168.56.12:52723 peer info: IV_VER=2.4.0
    Sun Feb  5 18:57:10 2017 192.168.56.12:52723 peer info: IV_PLAT=linux
    Sun Feb  5 18:57:10 2017 192.168.56.12:52723 peer info: IV_PROTO=2
    Sun Feb  5 18:57:10 2017 192.168.56.12:52723 peer info: IV_NCP=2
    Sun Feb  5 18:57:10 2017 192.168.56.12:52723 peer info: IV_LZ4=1
    Sun Feb  5 18:57:10 2017 192.168.56.12:52723 peer info: IV_LZ4v2=1
    Sun Feb  5 18:57:10 2017 192.168.56.12:52723 peer info: IV_LZO=1
    Sun Feb  5 18:57:10 2017 192.168.56.12:52723 peer info: IV_COMP_STUB=1
    Sun Feb  5 18:57:10 2017 192.168.56.12:52723 peer info: IV_COMP_STUBv2=1
    Sun Feb  5 18:57:10 2017 192.168.56.12:52723 peer info: IV_TCPNL=1
    AUTH-PAM: BACKGROUND: user 'test' failed to authenticate: Permission denied
    Sun Feb  5 18:57:10 2017 192.168.56.12:52723 PLUGIN_CALL: POST /usr/local/openvpn-2.4.0/lib/openvpn/plugins/openvpn-plugin-auth-pam.so/PLUGIN_AUTH_USER_PASS_VERIFY status=1
    Sun Feb  5 18:57:10 2017 192.168.56.12:52723 PLUGIN_CALL: plugin function PLUGIN_AUTH_USER_PASS_VERIFY failed with status 1: /usr/local/openvpn-2.4.0/lib/openvpn/plugins/openvpn-plugin-auth-pam.so
    Sun Feb  5 18:57:10 2017 192.168.56.12:52723 TLS Auth Error: Auth Username/Password verification failed for peer
    Sun Feb  5 18:57:10 2017 192.168.56.12:52723 Control Channel: TLSv1.2, cipher TLSv1/SSLv3 ECDHE-RSA-AES256-GCM-SHA384
    Sun Feb  5 18:57:10 2017 192.168.56.12:52723 Peer Connection Initiated with [AF_INET]192.168.56.12:52723
    Sun Feb  5 18:57:11 2017 192.168.56.12:52723 PUSH: Received control message: 'PUSH_REQUEST'
    Sun Feb  5 18:57:11 2017 192.168.56.12:52723 Delayed exit in 5 seconds
    Sun Feb  5 18:57:11 2017 192.168.56.12:52723 SENT CONTROL [UNDEF]: 'AUTH_FAILED' (status=1)
    Sun Feb  5 18:57:16 2017 192.168.56.12:52723 SIGTERM[soft,delayed-exit] received, client-instance exiting
    CSun Feb  5 19:00:49 2017 event_wait : Interrupted system call (code=4)
    ```
    下载`2.09`版本
    
    ```
    wget http://build.openvpn.net/downloads/releases/openvpn-2.0.9.zip
    ```
    编译插件
    
    ```
    cd openvpn-2.0.9/plugin/auth-pam/
    make
    cp openvpn-auth-pam.so /usr/local/openvpn-2.4.0/lib/openvpn/plugins/
    ```
    
    服务端配置文件修改为
    
    ```
    plugin /usr/local/openvpn-2.4.0/lib/openvpn/plugins/openvpn-auth-pam.so  openvpn
    ```
    这个时候再启动股服务端，则客户端可连接成功

# 防火墙规则修改

如果OpenVPN服务器启动了防火墙，则需要添加防火墙规则

>注意：根据自己使用的协议，以及使用的端口，修改防火墙规则

1. 开启路由转发功能

    >vim /etc/sysctl.conf

    找到net.ipv4.ip_forward = 0
    把0改成1

    ```
    sysctl -p
    ```

2. 设置防火墙规则

    设置iptables（这一条至关重要，通过配置nat将vpn网段IP转发到server内网）

    ```
    iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o eth0 -j MASQUERADE
    ```
    设置openvpn端口通过：

    ```
    iptables -A INPUT -p UDP --dport 1194 -j ACCEPT
    iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
    ```
    保存iptable设置：

    ```
    service iptables save
    ```

# 附录

## EasyRSA生成证书

1. 下载软件

    ```
    wget https://github.com/OpenVPN/easy-rsa/releases/download/2.2.2/EasyRSA-2.2.2.tgz
    ```
    
2. 配置变量

    备份`vars`变量文件
    
    ```
    cd EasyRSA-2.2.2
    cp vars.example vars.example
    ```

    修改变量
    
    ```
    export KEY_COUNTRY="CN"
    export KEY_PROVINCE="BeiJing"
    export KEY_CITY="BeiJing"
    export KEY_ORG="DrakeDog"
    export KEY_EMAIL="cyn9145@icloud.com"
    export KEY_OU="drakedog.org"
    ```
    
3. 生成ca证书文件

    ```
    [root@linux-node01 EasyRSA-2.2.2]# ./clean-all
    [root@linux-node01 EasyRSA-2.2.2]# . vars
    [root@linux-node01 EasyRSA-2.2.2]# ./build-ca 
    Generating a 2048 bit RSA private key
    ...............................................................................................+++
    ..................................................................+++
    writing new private key to 'ca.key'
    -----
    You are about to be asked to enter information that will be incorporated
    into your certificate request.
    What you are about to enter is what is called a Distinguished Name or a DN.
    There are quite a few fields but you can leave some blank
    For some fields there will be a default value,
    If you enter '.', the field will be left blank.
    -----
    Country Name (2 letter code) [CN]:
    State or Province Name (full name) [BeiJing]:
    Locality Name (eg, city) [BeiJing]:
    Organization Name (eg, company) [DrakeDog]:
    Organizational Unit Name (eg, section) [drakedog.org]:
    Common Name (eg, your name or your server's hostname) [DrakeDog CA]:vpn.drakedog.org
    Name [EasyRSA]:
    Email Address [cyn9145@icloud.com]:
    [root@linux-node01 EasyRSA-2.2.2]# 
    ```

4. 生成dh证书文件

    测过程需要较长时间
    
    ```
    [root@linux-node01 EasyRSA-2.2.2]# ./build-dh 
    Generating DH parameters, 2048 bit long safe prime, generator 2
    This is going to take a long time
    ..........................................................................+...............................................................................
    
    ... ...
    
    .............................................................................................................................................+...................................................................+.......................................................+..........................................................+............................+..................................................................++*++*
    [root@linux-node01 EasyRSA-2.2.2]# 
    ```
    
5. 生成服务器证书

    生成证书可需要密码，这里密码默认空，直接回车
    
    ```
    [root@linux-node01 EasyRSA-2.2.2]# ./build-key-server vpn.drakedog.org
    Generating a 2048 bit RSA private key
    .+++
    ..............................................................................................+++
    writing new private key to 'vpn.drakedog.org.key'
    -----
    You are about to be asked to enter information that will be incorporated
    into your certificate request.
    What you are about to enter is what is called a Distinguished Name or a DN.
    There are quite a few fields but you can leave some blank
    For some fields there will be a default value,
    If you enter '.', the field will be left blank.
    -----
    Country Name (2 letter code) [CN]:
    State or Province Name (full name) [BeiJing]:
    Locality Name (eg, city) [BeiJing]:
    Organization Name (eg, company) [DrakeDog]:
    Organizational Unit Name (eg, section) [drakedog.org]:
    Common Name (eg, your name or your server's hostname) [vpn.drakedog.org]:
    Name [EasyRSA]:
    Email Address [cyn9145@icloud.com]:
    
    Please enter the following 'extra' attributes
    to be sent with your certificate request
    A challenge password []:
    An optional company name []:
    Using configuration from /root/EasyRSA-2.2.2/openssl-1.0.0.cnf
    Check that the request matches the signature
    Signature ok
    The Subject's Distinguished Name is as follows
    countryName           :PRINTABLE:'CN'
    stateOrProvinceName   :PRINTABLE:'BeiJing'
    localityName          :PRINTABLE:'BeiJing'
    organizationName      :PRINTABLE:'DrakeDog'
    organizationalUnitName:PRINTABLE:'drakedog.org'
    commonName            :PRINTABLE:'vpn.drakedog.org'
    name                  :PRINTABLE:'EasyRSA'
    emailAddress          :IA5STRING:'cyn9145@icloud.com'
    Certificate is to be certified until Feb  3 13:25:53 2027 GMT (3650 days)
    Sign the certificate? [y/n]:y
    
    
    1 out of 1 certificate requests certified, commit? [y/n]y
    Write out database with 1 new entries
    Data Base Updated
    [root@linux-node01 EasyRSA-2.2.2]# 
    ```
6. 生成客户端证书

    生成证书可需要密码，这里密码默认空，直接回车
    
    ```
    [root@linux-node01 EasyRSA-2.2.2]# ./build-key caoyanan
    Generating a 2048 bit RSA private key
    ...........+++
    ...............................................+++
    writing new private key to 'caoyanan.key'
    -----
    You are about to be asked to enter information that will be incorporated
    into your certificate request.
    What you are about to enter is what is called a Distinguished Name or a DN.
    There are quite a few fields but you can leave some blank
    For some fields there will be a default value,
    If you enter '.', the field will be left blank.
    -----
    Country Name (2 letter code) [CN]:
    State or Province Name (full name) [BeiJing]:
    Locality Name (eg, city) [BeiJing]:
    Organization Name (eg, company) [DrakeDog]:
    Organizational Unit Name (eg, section) [drakedog.org]:
    Common Name (eg, your name or your server's hostname) [caoyanan]:
    Name [EasyRSA]:
    Email Address [cyn9145@icloud.com]:
    
    Please enter the following 'extra' attributes
    to be sent with your certificate request
    A challenge password []:
    An optional company name []:
    Using configuration from /root/EasyRSA-2.2.2/openssl-1.0.0.cnf
    Check that the request matches the signature
    Signature ok
    The Subject's Distinguished Name is as follows
    countryName           :PRINTABLE:'CN'
    stateOrProvinceName   :PRINTABLE:'BeiJing'
    localityName          :PRINTABLE:'BeiJing'
    organizationName      :PRINTABLE:'DrakeDog'
    organizationalUnitName:PRINTABLE:'drakedog.org'
    commonName            :PRINTABLE:'caoyanan'
    name                  :PRINTABLE:'EasyRSA'
    emailAddress          :IA5STRING:'cyn9145@icloud.com'
    Certificate is to be certified until Feb  3 13:30:42 2027 GMT (3650 days)
    Sign the certificate? [y/n]:y
    
    
    1 out of 1 certificate requests certified, commit? [y/n]y
    Write out database with 1 new entries
    Data Base Updated
    [root@linux-node01 EasyRSA-2.2.2]# 
    ```
    
7. 生成ta.key证书

    ta.key证书由openvpn生成
    
    ```
    cd /usr/local/openvpn-2.4.0/
    ./sbin/openvpn --genkey --secret ~/EasyRSA-2.2.2/keys/ta.key
    ```

7. 生成完证书可以结果如下

    ```
    [root@linux-node01 openvpn-2.4.0]# cd ~/EasyRSA-2.2.2/
    [root@linux-node01 EasyRSA-2.2.2]# tree keys/
    keys/
    ├── 01.pem
    ├── 02.pem
    ├── ca.crt
    ├── ca.key
    ├── caoyanan.crt
    ├── caoyanan.csr
    ├── caoyanan.key
    ├── dh2048.pem
    ├── index.txt
    ├── index.txt.attr
    ├── index.txt.attr.old
    ├── index.txt.old
    ├── serial
    ├── serial.old
    ├── ta.key
    ├── vpn.drakedog.org.crt
    ├── vpn.drakedog.org.csr
    └── vpn.drakedog.org.key
    
    0 directories, 18 files
    [root@linux-node01 EasyRSA-2.2.2]# 
    ```
    将所需要的证书存放到openvpn相应的位置即可
    
9. 复制一个博客内容

    如果使用esayras3制作证书
    
    ```
    最近研究如何在路由器上面实现openvpn的功能，其中便涉及到使用easyrsa来制作证书的问题，
    针对最新的openvpn-2.3.11源码包,easyrsa已经不包含在里面，需要单独下载，下载网址为https://github.com/OpenVPN/easy-rsa，下载下来是一个easy-rsa-master.zip压缩包，
    已上传为附件，在linux上面将其解压得到easy-rsa-master，进入easyrsa3，将vars.example复制一份命名为vars，此文件为制作证书时所使用到的配置文件，根据我的需要，我只打开了如下选项： 
    set_var EASYRSA_DN  "org" 
    set_var EASYRSA_REQ_COUNTRY "CN" 
    set_var EASYRSA_REQ_PROVINCE    "Guangdong" 
    set_var EASYRSA_REQ_CITY    "Shenzhen" 
    set_var EASYRSA_REQ_ORG "XXX" 
    set_var EASYRSA_REQ_EMAIL   "me@myhost.mydomain" 
    /*************************************/ 
    如果openvpn client的配置文件中使用了ns-cert-type server则要打开此选项，制作server证书时会将一些信息写入证书，如不打开此选项，则openvpn client会提示server certificate verify fail 
    set_var EASYRSA_NS_SUPPORT  "yes" 
    /*************************************/  
    下面就可以制作证书了，每条命令执行之后都有些信息输出，如出错，会提示相关错误信息 
    1 ./easyrsa init-pki 
    初始化，会在当前目录创建PKI目录，用于存储一些中间变量及最终生成的证书 
    
    2 ./easyrsa build-ca 
    创建根证书，首先会提示设置密码，用于ca对之后生成的server和client证书签名时使用，然后会提示设置Country Name，State or Province Name，Locality Name，Organization Name，Organizational Unit Name，Common Name，Email Address，可以键入回车使用默认的，也可以手动更改 
    
    3 ./easyrsa gen-req server nopass 
    创建server端证书和private key，nopass表示不加密private key，然后会提示设置Country Name，State or Province Name，Locality Name，Organization Name，Organizational Unit Name，Common Name，Email Address，可以键入回车使用默认的，也可以手动更改 
    
    4 ./easyrsa sign server server 
    给server端证书做签名，首先是对一些信息的确认，可以输入yes，然后输入build-ca时设置的那个密码 
    
    5 ./easyrsa gen-dh 
    创建Diffie-Hellman，时间会有点长，耐心等待 
    
    6 创建client端证书，需要单独把easyrsa3文件夹拷贝出来一份，删除里面的PKI目录，然后进入到此目录 
    ./easyrsa init-pki 
    初始化，会在当前目录创建PKI目录，用于存储一些中间变量及最终生成的证书 
    
    7 ./easyrsa gen-req client nopass 
    创建client端证书和private key，nopass表示不加密private key，然后会提示设置Country Name，State or Province Name，Locality Name，Organization Name，Organizational Unit Name，Common Name，Email Address，可以键入回车使用默认的，也可以手动更改 
    
    8 回到制作server证书时的那个easyrsa3目录，导入client端证书，准备签名 
    ./easyrsa import-req client.req所在路径 client 
    client.req应该在刚才制作client端证书的easyrsa3/pki/reqs/下面 
    
    9 ./easyrsa sign client client 
    给client端证书做签名，首先是对一些信息的确认，可以输入yes，然后输入build-ca时设置的那个密码 
    
    注意：ca、server和client的Common Name最好不要设置为一样，我没有验证，不过网上有人说设置一样后，openvpn连接时会有问题 
    
    至此，server和client端证书已制作完毕 
    openvpn server端需要的是 
    easyrsa3/pki/ca.crt   <制作server证书的文件夹> 
    easyrsa3/pki/private/server.key <制作server证书的文件夹> 
    easyrsa3/pki/issued/server.crt <制作server证书的文件夹> 
    easyrsa3/pki/dh.pem 
    
    openvpn client端需要的是 
    easy-rsa/easyrsa3/pki/ca.crt <制作server证书的文件夹> 
    easy-rsa/easyrsa3/pki/issued/client.crt <制作server证书的文件夹> 
    easy-rsa/easyrsa3/pki/private/client.key <制作client证书的文件夹>
    ```
    
    博客地址为：http://openwrt.iteye.com/blog/2305318
    
## OpenVPN配置文件解释

1. server.conf配置文件

    ```
     本文将介绍如何配置OpenVPN服务器端的配置文件。在Windows系统中，该配置文件一般叫做server.ovpn；在Linux/BSD系统中，该配置文件一般叫做server.conf。虽然配置文件名称不同，但其中的配置内容与配置方法却是相同的。

    本文根据官方提供的server.ovpn示例文件直接翻译得出。Windows、Linux、BSD等系统的服务器端配置文件均可参考本文。
    #################################################
    # 针对多客户端的OpenVPN 2.0 的服务器端配置文件示例
    #
    # 本文件用于多客户端<->单服务器端的OpenVPN服务器端配置
    #
    # OpenVPN也支持单机<->单机的配置(更多信息请查看网站上的示例页面)
    #
    # 该配置支持Windows或者Linux/BSD系统。此外，在Windows上，记得将路径加上双引号，
    # 并且使用两个反斜杠，例如："C:\\Program Files\\OpenVPN\\config\\foo.key"
    #
    # '#' or ';'开头的均为注释内容
    #################################################
    
    #OpenVPN应该监听本机的哪些IP地址？
    #该命令是可选的，如果不设置，则默认监听本机的所有IP地址。
    ;local a.b.c.d
    
    # OpenVPN应该监听哪个TCP/UDP端口？
    # 如果你想在同一台计算机上运行多个OpenVPN实例，你可以使用不同的端口号来区分它们。
    # 此外，你需要在防火墙上开放这些端口。
    port 1194
    
    #OpenVPN使用TCP还是UDP协议?
    ;proto tcp
    proto udp
    
    # 指定OpenVPN创建的通信隧道类型。
    # "dev tun"将会创建一个路由IP隧道，
    # "dev tap"将会创建一个以太网隧道。
    #
    # 如果你是以太网桥接模式，并且提前创建了一个名为"tap0"的与以太网接口进行桥接的虚拟接口，则你可以使用"dev tap0"
    #
    # 如果你想控制VPN的访问策略，你必须为TUN/TAP接口创建防火墙规则。
    #
    # 在非Windows系统中，你可以给出明确的单位编号(unit number)，例如"tun0"。
    # 在Windows中，你也可以使用"dev-node"。
    # 在多数系统中，除非你部分禁用或者完全禁用了TUN/TAP接口的防火墙，否则VPN将不起作用。
    ;dev tap
    dev tun
    
    # 如果你想配置多个隧道，你需要用到网络连接面板中TAP-Win32适配器的名称(例如"MyTap")。
    # 在XP SP2或更高版本的系统中，你可能需要有选择地禁用掉针对TAP适配器的防火墙
    # 通常情况下，非Windows系统则不需要该指令。
    ;dev-node MyTap
    
    # 设置SSL/TLS根证书(ca)、证书(cert)和私钥(key)。
    # 每个客户端和服务器端都需要它们各自的证书和私钥文件。
    # 服务器端和所有的客户端都将使用相同的CA证书文件。
    #
    # 通过easy-rsa目录下的一系列脚本可以生成所需的证书和私钥。
    # 记住，服务器端和每个客户端的证书必须使用唯一的Common Name。
    #
    # 你也可以使用遵循X509标准的任何密钥管理系统来生成证书和私钥。
    # OpenVPN 也支持使用一个PKCS #12格式的密钥文件(详情查看站点手册页面的"pkcs12"指令)
    ca ca.crt
    cert server.crt
    key server.key  # 该文件应该保密
    
    # 指定迪菲·赫尔曼参数。
    # 你可以使用如下名称命令生成你的参数：
    #   openssl dhparam -out dh1024.pem 1024
    # 如果你使用的是2048位密钥，使用2048替换其中的1024。
    dh dh1024.pem
    
    # 设置服务器端模式，并提供一个VPN子网，以便于从中为客户端分配IP地址。
    # 在此处的示例中，服务器端自身将占用10.8.0.1，其他的将提供客户端使用。
    # 如果你使用的是以太网桥接模式，请注释掉该行。更多信息请查看官方手册页面。
    server 10.8.0.0 255.255.255.0
    
    # 指定用于记录客户端和虚拟IP地址的关联关系的文件。
    # 当重启OpenVPN时，再次连接的客户端将分配到与上一次分配相同的虚拟IP地址
    ifconfig-pool-persist ipp.txt
    
    # 该指令仅针对以太网桥接模式。
    # 首先，你必须使用操作系统的桥接能力将以太网网卡接口和TAP接口进行桥接。
    # 然后，你需要手动设置桥接接口的IP地址、子网掩码；
    # 在这里，我们假设为10.8.0.4和255.255.255.0。
    # 最后，我们必须指定子网的一个IP范围(例如从10.8.0.50开始，到10.8.0.100结束)，以便于分配给连接的客户端。
    # 如果你不是以太网桥接模式，直接注释掉这行指令即可。
    ;server-bridge 10.8.0.4 255.255.255.0 10.8.0.50 10.8.0.100
    
    # 该指令仅针对使用DHCP代理的以太网桥接模式，
    # 此时客户端将请求服务器端的DHCP服务器，从而获得分配给它的IP地址和DNS服务器地址。
    #
    # 在此之前，你也需要先将以太网网卡接口和TAP接口进行桥接。
    # 注意：该指令仅用于OpenVPN客户端，并且该客户端的TAP适配器需要绑定到一个DHCP客户端上。
    ;server-bridge
    
    # 推送路由信息到客户端，以允许客户端能够连接到服务器背后的其他私有子网。
    # (简而言之，就是允许客户端访问VPN服务器自身所在的其他局域网)
    # 记住，这些私有子网也要将OpenVPN客户端的地址池(10.8.0.0/255.255.255.0)反馈回OpenVPN服务器。
    ;push "route 192.168.10.0 255.255.255.0"
    ;push "route 192.168.20.0 255.255.255.0"
    
    # 为指定的客户端分配指定的IP地址，或者客户端背后也有一个私有子网想要访问VPN，
    # 那么你可以针对该客户端的配置文件使用ccd子目录。
    # (简而言之，就是允许客户端所在的局域网成员也能够访问VPN)
    
    # 举个例子：假设有个Common Name为"Thelonious"的客户端背后也有一个小型子网想要连接到VPN，该子网为192.168.40.128/255.255.255.248。
    # 首先，你需要去掉下面两行指令的注释：
    ;client-config-dir ccd
    ;route 192.168.40.128 255.255.255.248
    # 然后创建一个文件ccd/Thelonious，该文件的内容为：
    #     iroute 192.168.40.128 255.255.255.248
    #这样客户端所在的局域网就可以访问VPN了。
    # 注意，这个指令只能在你是基于路由、而不是基于桥接的模式下才能生效。
    # 比如，你使用了"dev tun"和"server"指令。
    
    # 再举个例子：假设你想给Thelonious分配一个固定的IP地址10.9.0.1。
    # 首先，你需要去掉下面两行指令的注释：
    ;client-config-dir ccd
    ;route 10.9.0.0 255.255.255.252
    # 然后在文件ccd/Thelonious中添加如下指令：
    #   ifconfig-push 10.9.0.1 10.9.0.2
    
    # 如果你想要为不同群组的客户端启用不同的防火墙访问策略，你可以使用如下两种方法：
    # (1)运行多个OpenVPN守护进程，每个进程对应一个群组，并为每个进程(群组)启用适当的防火墙规则。
    # (2) (进阶)创建一个脚本来动态地修改响应于来自不同客户的防火墙规则。
    # 关于learn-address脚本的更多信息请参考官方手册页面。
    ;learn-address ./script
    
    # 如果启用该指令，所有客户端的默认网关都将重定向到VPN，这将导致诸如web浏览器、DNS查询等所有客户端流量都经过VPN。
    # (为确保能正常工作，OpenVPN服务器所在计算机可能需要在TUN/TAP接口与以太网之间使用NAT或桥接技术进行连接)
    ;push "redirect-gateway def1 bypass-dhcp"
    
    # 某些具体的Windows网络设置可以被推送到客户端，例如DNS或WINS服务器地址。
    # 下列地址来自opendns.com提供的Public DNS 服务器。
    ;push "dhcp-option DNS 208.67.222.222"
    ;push "dhcp-option DNS 208.67.220.220"
    
    # 去掉该指令的注释将允许不同的客户端之间相互"可见"(允许客户端之间互相访问)。
    # 默认情况下，客户端只能"看见"服务器。为了确保客户端只能看见服务器，你还可以在服务器端的TUN/TAP接口上设置适当的防火墙规则。
    ;client-to-client
    
    # 如果多个客户端可能使用相同的证书/私钥文件或Common Name进行连接，那么你可以取消该指令的注释。
    # 建议该指令仅用于测试目的。对于生产使用环境而言，每个客户端都应该拥有自己的证书和私钥。
    # 如果你没有为每个客户端分别生成Common Name唯一的证书/私钥，你可以取消该行的注释(但不推荐这样做)。
    ;duplicate-cn
    
    # keepalive指令将导致类似于ping命令的消息被来回发送，以便于服务器端和客户端知道对方何时被关闭。
    # 每10秒钟ping一次，如果120秒内都没有收到对方的回复，则表示远程连接已经关闭。
    keepalive 10 120
    
    # 出于SSL/TLS之外更多的安全考虑，创建一个"HMAC 防火墙"可以帮助抵御DoS攻击和UDP端口淹没攻击。
    # 你可以使用以下命令来生成：
    #   openvpn --genkey --secret ta.key
    #
    # 服务器和每个客户端都需要拥有该密钥的一个拷贝。
    # 第二个参数在服务器端应该为'0'，在客户端应该为'1'。
    ;tls-auth ta.key 0 # 该文件应该保密
    
    # 选择一个密码加密算法。
    # 该配置项也必须复制到每个客户端配置文件中。
    ;cipher BF-CBC        # Blowfish (默认)
    ;cipher AES-128-CBC   # AES
    ;cipher DES-EDE3-CBC  # Triple-DES
    
    # 在VPN连接上启用压缩。
    # 如果你在此处启用了该指令，那么也应该在每个客户端配置文件中启用它。
    comp-lzo
    
    # 允许并发连接的客户端的最大数量
    ;max-clients 100
    
    # 在完成初始化工作之后，降低OpenVPN守护进程的权限是个不错的主意。
    # 该指令仅限于非Windows系统中使用。
    ;user nobody
    ;group nobody
    
    # 持久化选项可以尽量避免访问那些在重启之后由于用户权限降低而无法访问的某些资源。
    persist-key
    persist-tun
    
    # 输出一个简短的状态文件，用于显示当前的连接状态，该文件每分钟都会清空并重写一次。
    status openvpn-status.log
    
    # 默认情况下，日志消息将写入syslog(在Windows系统中，如果以服务方式运行，日志消息将写入OpenVPN安装目录的log文件夹中)。
    # 你可以使用log或者log-append来改变这种默认情况。
    # "log"方式在每次启动时都会清空之前的日志文件。
    # "log-append"这是在之前的日志内容后进行追加。
    # 你可以使用两种方式之一(但不要同时使用)。
    ;log         openvpn.log
    ;log-append  openvpn.log
    
    # 为日志文件设置适当的冗余级别(0~9)。冗余级别越高，输出的信息越详细。
    #
    # 0 表示静默运行，只记录致命错误。
    # 4 表示合理的常规用法。
    # 5 和 6 可以帮助调试连接错误。
    # 9 表示极度冗余，输出非常详细的日志信息。
    verb 3
    
    # 重复信息的沉默度。
    # 相同类别的信息只有前20条会输出到日志文件中。
    ;mute 20
    ```
