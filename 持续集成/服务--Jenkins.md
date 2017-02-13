# 服务--Jenkins

[TOC]

## 环境准备

1. 安装JDK
2. 安装Tomcat

## 服务安装

1. 下载Jenkins运行包

    ```
    cd /usr/local/tomcat/webapps
    wget http://mirrors.jenkins.io/war-stable/latest/jenkins.war
    ```

2. 启动Tomcat

    ```
    cd /usr/local/tomcat
    ./bin/startup.sh
    ```

3. 访问Jenkins

    ```
    http://192.168.56.11:8080/jenkins
    ```

## 初始化Jenkins

1. 输入验证密码

    ```
    [root@linux-node01 ~]# cat /root/.jenkins/secrets/initialAdminPassword
    9c24a16a3d294d02be314b8840f1f511
    ```
    
    ![Jenkins_Setup_001](http://images.drakedog.cn/jenkins/Jenkins_Setup_001.png)

2. 选择插件(安装建议插件)

    ![Jenkins_Setup_002](http://images.drakedog.cn/jenkins/Jenkins_Setup_002.png)

    等待插件安装完毕
    有可能有的插件无法安装，选择`Retry`重新安装插件 或者 `Skip` 以后安装也可以
    
    ![Jenkins_Setup_003](http://images.drakedog.cn/jenkins/Jenkins_Setup_003.png)

3. 输入管理员的用户名，以及密码

    ![Jenkins_Setup_004](http://images.drakedog.cn/jenkins/Jenkins_Setup_004.png)
    
4. 出现此页面Jenkins安装完成

    ![Jenkins_Setup_005](http://images.drakedog.cn/jenkins/Jenkins_Setup_005.png)

## 配置Jenkins

无论在哪个位置，点击Jenkins的Logo就会回到主页，配置Jenkins的所有操作，都是基于此页面进行操作的

### 新加用户

1. 在`系统管理` > `管理用户` > `新建用户` 下新建需要的用户

    跟Jenkins初始化时，添加管理员用户，所需要的信息一致

    > 推荐新建用户列表为:(此为后续的Job权限管理作准备，也可以不按此用户分类)
    > 1. dev001
    > 2. test001
    > 3. opration001

    用户列表如下：
    ![Jenkins_UserAdd_001](http://images.drakedog.cn/jenkins/Jenkins_UserAdd_001.png)

### 新加Job

1. 点击`新建`,输入item name,并选择一个软件项目风格(此处选择了自由风格，后续项目风格，会依次介绍) > `OK`

    > 项目名尽量标示清楚此项目的，项目以及所用人群，以及子项目
    > `主项目名` + `对应角色` + `子项目名`

    ![Jenkins_JobAdd_001](http://images.drakedog.cn/jenkins/Jenkins_JobAdd_001.png)

2. 点击`保存`,此时一个Job就简单的创建成功了

    ![Jenkins_JobAdd_002](http://images.drakedog.cn/jenkins/Jenkins_JobAdd_002.png)
 

### 新加视图

添加试图有很多方式，为了对Job的权限管理，此处采用的直接添加试图的方式,此方式可以选择需要在该试图显示的Job,也可以在新建视图后，在编辑视图中添加所需要在该视图中显示的Job

1. 点击`+`按钮

    ![Jenkins_ViewAdd_001](http://images.drakedog.cn/jenkins/Jenkins_ViewAdd_001.png)

2. 选择视图类型,之后点击`OK`按钮

    ![Jenkins_ViewAdd_002](http://images.drakedog.cn/jenkins/Jenkins_ViewAdd_002.png)

3. 选择视图所包含的Job，点击`保存`

    ![Jenkins_ViewAdd_003](http://images.drakedog.cn/jenkins/Jenkins_ViewAdd_003.png)
    
4. 现在此视图已经包含了`zzr_dev_front`Job

    ![Jenkins_ViewAdd_004](http://images.drakedog.cn/jenkins/Jenkins_ViewAdd_004.png)

    > 注意：虽然此`zzr_dev_front`Job显示在了此视图中，但是现在所有的人仍是可以看到这个视图，以及这类工程的，需要后续的权限配置完成后，才能让不同的人查看到不同的view
    
### 权限配置

由于Jenkisn关系到整个网站的构建，因此对于不在此Job的用户，没有必要看到该Job了，对于Jenkins推荐流程，以及项目管理会在附录中

#### 全局权限配置

1. 在`系统管理` > `Configure Global Security` 下进行配置

    配置图如下：
    
    ![Jenkins_ConfGlobal_001](http://images.drakedog.cn/jenkins/Jenkins_ConfGlobal_001.png)
    
    具体策略如下图所示：
    
    > 注意管理员用户配置所有权限
    
    ![Jenkins_ConfGlobal_002](http://images.drakedog.cn/jenkins/Jenkins_ConfGlobal_002.png)
    
    > 特别注意: 由于图片原因可能导致显示不正确,非管理员用户只需要`Overall` 中的 ` Read`权限，别的所有权限均不需要

2. 点击`保存`，完成修改

#### 局部Job权限配置

1. 在`首页` > 点击需要修改权限的Job名 > `配置` 下设置

    ![jenkinsJenkins_ConfJob_001](http://images.drakedog.cn/jenkins/Jenkins_ConfJob_001.png)
    
    ![jenkinsJenkins_ConfJob_002](http://images.drakedog.cn/jenkinsJenkins_ConfJob_002.png)
    
    > 注意：管理员用户配置所有权限
    
    ![jenkinsJenkins_ConfJob_003](http://images.drakedog.cn/jenkinsJenkins_ConfJob_003.png)
    
此时Job的详细权限管理配置完毕，这样test的用户就无法看到dev项目的Job了

## 详细配置Job

# 附录

## Jenkins管理员登录不进去情况解决

1. 进入Jenkins服务器下,备份配置文件

    ```
    cd /root/.jenkins
    cp config.xml  config.xml.bak
    ```
2. 删除权限配置

    删除如下配置标签中的内容
    
    ```
        <useSecuity>true</useSecuity>
        <authorizationStrategy ....>
    
    		... ...
    
        </authorizationStrategy>
        <securityRealm ...>
    
    		... ...
    
        </securityRealm>
    ```
    
    此处我删除的如下：

    ```
      <useSecurity>ture</useSecurity>
      <authorizationStrategy class="hudson.security.ProjectMatrixAuthorizationStrategy">
        <permission>com.cloudbees.plugins.credentials.CredentialsProvider.Create:Admin</permission>
        <permission>com.cloudbees.plugins.credentials.CredentialsProvider.Delete:Admin</permission>
        <permission>com.cloudbees.plugins.credentials.CredentialsProvider.ManageDomains:Admin</permission>
        <permission>com.cloudbees.plugins.credentials.CredentialsProvider.Update:Admin</permission>
        <permission>com.cloudbees.plugins.credentials.CredentialsProvider.View:Admin</permission>
        <permission>hudson.model.Computer.Build:Admin</permission>
        <permission>hudson.model.Computer.Configure:Admin</permission>
        <permission>hudson.model.Computer.Connect:Admin</permission>
        <permission>hudson.model.Computer.Create:Admin</permission>
        <permission>hudson.model.Computer.Delete:Admin</permission>
        <permission>hudson.model.Computer.Disconnect:Admin</permission>
        <permission>hudson.model.Computer.Provision:Admin</permission>
        <permission>hudson.model.Hudson.Administer:Admin</permission>
        <permission>hudson.model.Hudson.ConfigureUpdateCenter:Admin</permission>
        <permission>hudson.model.Hudson.Read:Admin</permission>
        <permission>hudson.model.Hudson.Read:dev001</permission>
        <permission>hudson.model.Hudson.RunScripts:Admin</permission>
        <permission>hudson.model.Hudson.UploadPlugins:Admin</permission>
        <permission>hudson.model.Item.Build:Admin</permission>
        <permission>hudson.model.Item.Cancel:Admin</permission>
        <permission>hudson.model.Item.Configure:Admin</permission>
        <permission>hudson.model.Item.Create:Admin</permission>
        <permission>hudson.model.Item.Delete:Admin</permission>
        <permission>hudson.model.Item.Discover:Admin</permission>
        <permission>hudson.model.Item.Move:Admin</permission>
        <permission>hudson.model.Item.Read:Admin</permission>
        <permission>hudson.model.Item.Workspace:Admin</permission>
        <permission>hudson.model.Run.Delete:Admin</permission>
        <permission>hudson.model.Run.Replay:Admin</permission>
        <permission>hudson.model.Run.Update:Admin</permission>
        <permission>hudson.model.View.Configure:Admin</permission>
        <permission>hudson.model.View.Create:Admin</permission>
        <permission>hudson.model.View.Delete:Admin</permission>
        <permission>hudson.model.View.Read:Admin</permission>
        <permission>hudson.model.View.Read:dev001</permission>
        <permission>hudson.scm.SCM.Tag:Admin</permission>
      </authorizationStrategy>
      <securityRealm class="hudson.security.HudsonPrivateSecurityRealm">
        <disableSignup>true</disableSignup>
        <enableCaptcha>false</enableCaptcha>
      </securityRealm>
    ```
3. 重启Tomcat

    此时进入Jenkins就不需要账号密码了
    
4. 然后在`系统管理` > `Configure Global Security` 下修改成如下权限配置，并保存

    ![Jenkins_ForgotPass_001](http://images.drakedog.cn/jenkins/Jenkins_ForgotPass_001.png)

5. 重置管理员账号密码，在`系统管理` > `管理用户` > 选择管理员用户 > `设置` > 修改密码，并保存

    ![Jenkins_ForgotPass_002](http://images.drakedog.cn/jenkins/Jenkins_ForgotPass_002.png)
    ![Jenkins_ForgotPass_003](http://images.drakedog.cn/jenkins/Jenkins_ForgotPass_003.png)
    ![Jenkins_ForgotPass_004](http://images.drakedog.cn/jenkins/Jenkins_ForgotPass_004.png)

6. 以上步骤已经重置完管理员账号密码，现在将权限配置文件恢复

    ```
    cd /root/.jenkins
    cp config.xml.bak  config.xml
    ```
    
7. 重启Tomcat

这样，就完成了忘记管理员用户密码，并重置密码，而且不会丢失权限

## 推荐Jenkins项目权限管理图
