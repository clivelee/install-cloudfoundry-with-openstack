#Cloudfoundry(v2) with Openstack(Folsom)安装手册

* Published: 2013年7月9日 上午11:21   
* CreateBy: Clive Lee 
* Version: 1.0

##1.准备工作

###已安装好的OpenStack(Folsom)

CloudFoundry(以下均简称为cf)需要用到以下OpenStack的参数，这些参数请务必保证正确:

	openstack_username: admin #openstack的管理员账号
	openstack_api_key: openstack123 #管理员密码
	openstack_tenant: CFv2  #CF安装所使用的项目名	
	openstack_auth_url: http://10.10.5.1:5000/v2.0/tokens#OpenStack的API地址 
	openstack_region: RegionOne #OpenStack的区域名


###安装CF的整体规划


|  机器名 |   静态ip   | 浮动ip |  功能描述 | 特殊要求 |
|--------| ------------ | ------------- | ------------ |
|bosh-cli| 自动分配   | 10.10.5.61 |  用于编译Bosh和CF的机器 | 硬盘需要大于20G |
|microbosh| 172.16.2.50 | - |  Microbosh安装节点 | Eph Disk需要大于20G  |
|core| 自动分配 |10.10.5.51 | cf的核心节点 | 需要挂载硬盘20G | 
|common| 自动分配 |10.10.5.52 | cf的监控和日志服务 | 需要挂载硬盘20G | 
|router| 自动分配 | 10.10.5.53 | cf的路由服务 | - | 
|dea| 自动分配 |10.10.5.54 | cf的dea服务 | - | 
|uaa| 自动分配 |10.10.5.55 | cf的用户中心服务 | - | 
|mysql_service| 自动分配 |10.10.5.56 | cf的MySQL数据服务 | 配置8G的硬盘 |
|redis_service| 自动分配 |10.10.5.57 | cf的Redis数据服务 | 配置8G的硬盘 | 

域名规划

|  域名 |   浮动ip   |  功能描述  |
|--------| ------------ | ------------- |
|login.mycloud.org| 10.10.5.52 | 登录入口 |  
|api.mycloud.org|  10.10.5.53 |  api访问入口 |
|uaa.mycloud.org| 10.10.5.55 | 用户认证中心|
|*.mycloud.org| 10.10.5.53 | 应用注册入口 |


在本文档中静态地址均使用的是172.16.2.x的ip地址，浮动地址用的10.10.5.x;

请安装者自己把这些地址改成实际安装过程中的地址。

本文所采用的系统为ubuntu 12.04,用户名为ubuntu;

##2.安装Bosh CLI
 
假定要装Bosh CLI的机器是10.10.5.61，现在登录到此服务器

###2.1 更新源

	sudo apt-get update

###2.2 安装Git账号


* 如果有Github账号，则从该账号所对应的key-pair上传到5.61上
	
```
scp github_ida.pub github_ida ubuntu@10.10.5.61:~/
```

* 没有Github账号的话，去gitbhub生成一个吧。然后再将key-pair放到10.10.5.61的~/.ssh目录下。


###2.3 安装Ruby

####安装rvm
为了方便ruby的版本切换，需要现在机器上安装rvm
	
	curl -L https://get.rvm.io | bash -s stable --autolibs=enabled --ruby=1.9.3

####设置环境变量
	
* 若用非root账号安装，则把下面这段加入到 ~/.bashrc里面

```
[[ -s "$HOME/.rvm/scripts/rvm" ]] && source "$HOME/.rvm/scripts/rvm"
```
* 若用root安装则在~/.bashrc中加入这段 

```
[[ -s "/usr/local/rvm/scripts/rvm" ]] && source "/usr/local/rvm/scripts/rvm"
```
	
####环境变量生效
	
	source ~/.bashrc
	
####如需要装指定版本的ruby

	rvm install 1.9.3-p327

###2.4 安装Bosh CLI

	gem install bosh_cli_plugin_micro -v "~> 1.5.0.pre" --no-ri --no-rdoc --source https://s3.amazonaws.com/bosh-jenkins-gems/
	

##3.安装Micobosh
当前还是在5.61这台服务器上.在安装microbosh之前,先看看bosh-cli上完整的工作目录结构.下面所有的过程都是在这个目录结构上进行的.

```
├── bosh-workspace
│   ├── bosh
│   ├── cf-release
│   ├── deployments
│   │   ├── bosh-deployments.yml
│   │   ├── bosh-openstack
│   |   |   ├── bosh-openstack.yml
│   │   ├── bosh_registry.log
│   │   ├── cf.yml
│   │   ├── microbosh-openstack
│   |   |   ├── bosh_micro_deploy.log
│   |   |   └── micro_bosh.yml
│   │   └── temp
│   ├── stemcells
│   │   └── last_successful_bosh-stemcell-openstack.tgz
```
简单说明一下这个目录结构:

* `bosh-workspace`在`~`目录下;
* `bosh`和`cf-release`都是从git下签出的源代码目录;
* `deployments`里面放的是在安装过程中的各种配置文件;* `stemcells`目录,里面存放的是wget到的几个镜像文件;


首先在当前用户目录下新建几个目录
	
	mkdir -p ~/bosh-workspace/deployments/microbosh-openstack
	cd ~/bosh-workspace/deployments/microbosh-openstack

在当前目录下新建`micro_bosh.yml`，并输入模板文件的内容:

```

---
name: microbosh-openstack

logging:
  level: DEBUG

network:
  type: manual
  ip: <static_ip>
  cloud_properties:
    net_id: <network_uuid>

resources:
  persistent_disk: 16384
  cloud_properties:
    instance_type: <flavor_name>

cloud:
  plugin: openstack
  properties:
    openstack:
      auth_url: http://<identity_server>:5000/v2.0 
      username: <username>
      api_key: <password> 
      tenant: <tenant>
      region: <region> # Optional
      default_security_groups: ["default", <microbosh_security_group>]
      default_key_name: <microbosh_keypair>
      private_key: <path_to_microbosh_keypar_private_key>

apply_spec:
  properties:
    director:
      max_threads: 3
      snapshot_schedule: false
      self_snapshot_schedule: false
    hm:
      resurrector_enabled: true
    ntp:
      - 3.cn.pool.ntp.org
      - 1.asia.pool.ntp.org
      - 3.asia.pool.ntp.org

```

替换模板中的<x>的内容，每一个配置的具体说明如下：

* `<static_ip>` 表示microbosh将要安装的服务器的ip地址；
* `<network_uuid>` 表示OpenStack的网络ID号，可以在OpenStack的控制节点上用`nova-manage network list`命令看到；
* `<flavor_name>` 表示microbosh这台机器所用的资源模板名，是在OpenStack中定义的；
* `<identity_server>` 表示OpenStack的控制节点ip地址；
* `<username>` 表示OpenStack所用的管理员账号名；
* `<password>` 表示OpenStack的管理账号密码；
* `<tenant>` 表示CloudFoundry所处的项目名，这个项目在OpenStack中预先指定好的；
* `<region>` 表示CloudFoundry所处的区域名，也是在OpenStack中分配好的；
* `<microbosh_security_group>` 表示microbosh使用的安全组，在OpenStack中定义；
* `<microbosh_keypair>` 表示microbosh所用的ssh-key名称，也是在OpenStack中创建；
* `<path_to_microbosh_keypar_private_key>` 将ssh-key存放在哪个路径上；

替换完后的microbosh.yml并确认无误后，就可以运行下面的命令：

```
mkdir -p ~/bosh-workspace/stemcells
cd ~/bosh-workspace/stemcells
wget http://bosh-jenkins-artifacts.s3.amazonaws.com/last_successful_micro-bosh-stemcell-openstack.tgz
```
这三条命令是从指定地址获取micro-bosh的最新stemcell,这个所谓的stemcell是干细胞的意思. 顾名思义就是microbosh的模板镜像.

下好了stemcell之后就可以部署microbosh了:

```
cd ~/bosh-workspace/deployments
bosh micro deployment microbosh-openstack
bosh micro deploy ~/bosh-workspace/stemcells/last_successful_micro-bosh-stemcell-openstack.tgz
```
如果一切顺利的话就应该看到如下提示

```
Deploying new micro BOSH instance `microbosh-openstack/micro_bosh.yml' to `https://172.16.2.50:25555' (type 'yes' to continue): yes

Verifying stemcell...
File exists and readable                                     OK
Using cached manifest...
Stemcell properties                                          OK

Stemcell info
-------------
Name:    micro-bosh-stemcell
Version: 703


Deploy Micro BOSH
  unpacking stemcell (00:00:04)                                                                     
  uploading stemcell (00:01:43)                                                                     
  creating VM from 26db559a-90b9-4793-bf0d-24ddc53cfb8b (00:00:55)                                  
  waiting for the agent (00:01:18)                                                                  
  create disk (00:00:01)                                                                            
  mount disk (00:00:05)                                                                             
  stopping agent services (00:00:01)                                                                
  applying micro BOSH spec (00:00:16)                                                               
  starting agent services (00:00:00)                                                                
  waiting for the director (00:00:15)                                                               
Done                    11/11 00:04:43                                                              
Deployed `microbosh-openstack/micro_bosh.yml' to `https://172.16.2.50:25555', took 00:04:43 to complete
```
从开始到这一步的过程应该还是很顺利的.




##4.安装releases

当安装好了microbosh之后,就可以进入cloudfoundry的安装过程了.

###4.1 安装cf-release

首先我们从github上签出最新的cf-release代码,下载完源代码后进入cf-release中时会出现要求你安装ruby-1.9.3-p392的提示. 为保证后面没什么篓子,我们选择了给它装上.(PS:吐槽一下ruby的版本管理,作为一门语言版本还发的这么频繁啊~~~)

```
cd ~/bosh-workspace
git clone -b release-candidate git://github.com/cloudfoundry/cf-release.git && cd cf-release
cd ~/bosh-workspace/cf-release
rvm install ruby-1.9.3-p392
./update
bosh create release --force
```

如果是第一次安装的话这个过程大概需要2个小时左右,中间还得稍微看着一下. 因为时不时的会有peer to peer网络中断的情况. 

不过好在bosh能够断点续传,发现这个错误之后直接运行`bosh create release --force`就行.

等release build得差不多之后,提示符会要求输入release name,我们直接输的是appcloud

安装完后bosh会在dev_releases下新建一个yml, 例如`appcloud-131.3-dev.yml`,我们需要把这个yml上传到microbosh上.

```
bosh target 172.16.2.50
bosh upload release dev_releases/appcloud-131.3-dev.yml
```
这里的172.16.2.50就是microbosh的ip.

等upload release安装完成后键入`bosh releases`命令看看appcloud的release是不是在列表中.

```
ubuntu@ubuntu:~$ bosh releases

+---------------------+------------+-------------+
| Name                | Versions   | Commit Hash |
+---------------------+------------+-------------+
| appcloud            | 131.3-dev* | e83251b1+   |
+---------------------+------------+-------------+
```

###4.2 安装cf-services-release



这个release里面主要包含了两个数据服务的安装文件mysql和postgresql.

这个安装步骤和cf-release一样都是先下代码,然后编译,最后上传:

```
git clone https://github.com/cloudfoundry/cf-services-release.git
cd cf-services-release
./update
bosh create release --force
echo "---\ndev_name: cf-services" > config/dev.yml
bosh -n upload release

```


###4.3 安装cf-contrib-services-release

这个release里面主要是有一些扩展的数据服务,如mongodb,redis和memcached等等;
安装步骤与其他的release一样.

```
git clone https://github.com/cloudfoundry/cf-services-contrib-release.git
cd cf-services-release
./update
bosh create release --force
echo "---\ndev_name: cf-contrib-services" > config/dev.yml
bosh -n upload release

```

最后安装完成后的结果如下:

```
ubuntu@ubuntu:~$ bosh releases

+---------------------+------------+-------------+
| Name                | Versions   | Commit Hash |
+---------------------+------------+-------------+
| appcloud            | 131.3-dev* | e83251b1+   |
| cf-contrib-services | 0.2-dev*   | 6ef9e134+   |
| cf-services         | 0.2-dev*   | 8ddb28fa+   |
+---------------------+------------+-------------+
(*) Currently deployed
(+) Uncommitted changes

Releases total: 3
```


###4.4 上传bosh-stemcell

除了安装release以外还需要上传一个stemcell,也就是cf运行创建虚拟机基础环境.这个步骤很简单,执行下面命令就行了:

```
wget http://bosh-jenkins-artifacts.s3.amazonaws.com/last_successful_bosh-stemcell-openstack.tgz
bosh upload stemcell ~/bosh-workspace/stemcells/last_successful_bosh-stemcell-openstack.tgz
```

###4.5 编辑cf.yml

当把release和stemcell都上传好了之后就可以进入最重要的编辑cf的配置文件的环节了.

cf的配置文件里面主要是用于定义即将安装的cloudfoundry的各种属性,例如名字,所要用到的发布版本,网络,任务以及任务的属性等等.
任务运行完成后就会产生若干个以任务名命名的虚拟机,而相同任务中的虚拟机上都具有相同的内容.

在编辑cf.yml的时候有许多要注意的地方:

* resource_pools里面一定要有足够数量的机器,否则再部署的时候就会抛出错误;
* 域名要事先规划好,保证api;uaa;login这两个域名都按照事先的规划;
* 应用域名都指向api所在的服务器*.domain.com => 10.10.5.53在这一步骤上我们也花费了不少的时间.
* 一个部署里面可以有多个release,比如cf-services-release和cf-release

在下面的实例文件当中一共有三个release, 因此在写每一个job时必须指明当前job是来自哪一个release.


```
---
name: cfos
director_uuid: 15e8b784-0c11-44b4-86fa-e55ab79099d1 

releases:
- name: appcloud
  version: 131.3-dev
- name: cf-services
  version: 0.2-dev
- name: cf-contrib-services 
  version: 0.2-dev 

compilation:
  workers: 4
  network: default
  reuse_compilation_vms: true
  cloud_properties:
    instance_type: m1.micro

update:
  canaries: 1
  canary_watch_time: 30000-600000
  update_watch_time: 30000-600000
  max_in_flight: 4
  max_errors: 1

networks:
  - name: floating
    type: vip
    cloud_properties:
      security_groups:
      - default
  - name: default
    type: dynamic
    cloud_properties:
      security_groups:
      - default

resource_pools:
  - name: small
    network: default
    size: 5
    stemcell:
      name: bosh-stemcell
      version: latest
    cloud_properties:
      instance_type: m1.micro

  - name: medium
    network: default
    size: 2 
    stemcell:
      name: bosh-stemcell
      version: latest
    cloud_properties:
      instance_type: m1.micro

jobs:
  - name: core
    template:
      - syslog_aggregator
      - nats
      - postgres
      - debian_nfs_server
    instances: 1
    release: appcloud
    resource_pool: medium
    persistent_disk: 16384
    networks:
      - name: default
        default: [dns, gateway]
      - name: floating
        static_ips:
          10.10.5.51
    properties:
      db: databases

  - name: common
    template:
      - login
      - health_manager_next
      - collector
    instances: 1
    release: appcloud
    resource_pool: small
    networks:
      - name: default
        default: [dns, gateway]
      - name: floating
        static_ips:
          10.10.5.52

  - name: dea
    template:
      - dea_next
    instances: 1
    release: appcloud
    resource_pool: small
    networks:
      - name: default
        default: [dns, gateway]
      - name: floating
        static_ips:
          10.10.5.54

  - name: uaa
    template:
      - uaa
    instances: 1
    release: appcloud
    resource_pool: medium
    networks:
      - name: default
        default: [dns, gateway]
      - name: floating
        static_ips:
          10.10.5.55

  - name: router
    template:
      - cloud_controller_ng
      - gorouter
    instances: 1
    resource_pool: small
    release: appcloud
    networks:
      - name: default
        default: [dns, gateway]
      - name: floating
        static_ips:
          - 10.10.5.53
    properties:
      ccdb: ccdb

  - name: mysql_service
    template:
    - mysql_gateway
    - mysql_node
    release: cf-services
    resource_pool: small
    persistent_disk: 8192
    instances: 1
    networks:
    - name: default
      default:
      - dns
      - gateway
    - name: floating
      static_ips:
      - 10.10.5.56
    properties:
      uaa_endpoint: http://uaa.dc.cfos.org
      uaa_client_id: cf
      uaa_client_auth_credentials:
        username: services
        password: c1oudc0w
      mysql_node:
        plan: free 

  - name: redis_service
    template:
    - redis_gateway
    - redis_node_ng
    release: cf-contrib-services
    resource_pool: small
    persistent_disk: 8192
    instances: 1
    networks:
    - name: default
      default:
      - dns
      - gateway
    - name: floating
      static_ips:
      - 10.10.5.57
    properties:
      uaa_endpoint: http://uaa.dc.cfos.org
      uaa_client_id: cf
      uaa_client_auth_credentials:
        username: services
        password: c1oudc0w
      plan: free

properties:
  domain: dc.cfos.org
  system_domain: dc.cfos.org
  system_domain_organization: "dc.cfos.org"
  app_domains:
    - dc.cfos.org

  networks:
    apps: default
    management: default


  service_plans:
    mysql:
      free:
        description: "MySql Service"
        job_management:
          high_water: 1400
          low_water: 100
        configuration:
          lifecycle:
            enable: false
          warden:
            enable: false
    redis:
      free:
        description: "Redis Service"
        free: true
        job_management:
          high_water: 410
          low_water: 40
        configuration:
          lifecycle:
            enable: false
          warden:
            enable: false

  mysql_gateway:
    cc_api_version: v2
    token: c1oudc0wc1oudc0w
    default_plan: free
    supported_versions: ["5.5"]
    version_aliases:
      current: "5.5"

  mysql_node:
    supported_versions: ["5.5"]
    default_version: "5.5"
    password: c1oudc0wc1oudc0w

  redis_gateway:
    cc_api_version: v2
    token: c1oudc0wc1oudc0w
    default_plan: free
    supported_versions: ["2.6"]
    version_aliases:
      current: "2.6"

  redis_node:
    command_rename_prefix: foobar
    supported_versions: ["2.6"]
    default_version: "2.6"


  nats:
    address: 10.10.5.51
    port: 4222
    user: nats
    password: "c1oudc0w"
    authorization_timeout: 5

  router:
    port: 8081
    status:
      port: 8080
      user: gorouter
      password: "c1oudc0w"

  dea: &dea
    max_memory: 4096
    memory_mb: 4084
    memory_overcommit_factor: 4
    disk_mb: 16384
    disk_overcommit_factor: 4

  dea_next: *dea

  service_lifecycle:
    serialization_data_server:
    - 10.10.5.51

  nfs_server:
    address: 10.10.5.51
    network: 10.10.5.0/23

  syslog_aggregator:
    address: 10.10.5.51
    port: 54321

  serialization_data_server:
    port: 8080
    logging_level: debug
    upload_token: 8f7COGvThwlmulIzAgOHxMXurBrG364k
    upload_timeout: 10


  collector:
    deployment_name: cloudfoundry
    use_tsdb: false
    use_aws_cloudwatch: false
    use_datadog: false

  databases: &databases
    db_scheme: postgres
    address: 10.10.5.51
    port: 5524
    roles:
      - tag: admin
        name: ccadmin
        password: "c1oudc0w"
      - tag: admin
        name: uaaadmin
        password: "c1oudc0w"
    databases:
      - tag: cc
        name: ccdb
        citext: true
      - tag: uaa
        name: uaadb
        citext: true

  ccdb: &ccdb
    db_scheme: postgres
    address: 10.10.5.51 
    port: 5524
    roles:
      - tag: admin
        name: ccadmin
        password: "c1oudc0w"
    databases:
      - tag: cc
        name: ccdb
        citext: true

  ccdb_ng: *ccdb

  uaadb:
    db_scheme: postgresql
    address: 10.10.5.51
    port: 5524
    roles:
      - tag: admin
        name: uaaadmin
        password: "c1oudc0w"
    databases:
      - tag: uaa
        name: uaadb
        citext: true


  cc: &cc
    logging_level: debug
    external_host: ccng
    srv_api_uri: http://api.dc.cfos.org
    cc_partition: default
    db_encryption_key: "b963127302433579"
    bootstrap_admin_email: "liji@cnic.cn"
    bulk_api_password: "c1oudc0w"
    uaa_resource_id: cloud_controller
    staging_upload_user: uploaduser
    staging_upload_password: c1oudc0w
    resource_pool:
      resource_directory_key: cc-resources
      fog_connection:
        provider: Local
        local_root: /var/vcap/shared
    packages:
      app_package_directory_key: cc-packages
    droplets:
      droplet_directory_key: cc-droplets
    stacks:
    - name: lucid64
      description: "Ubuntu 10.04"

  ccng: *cc

  login:
    protocol: http
    port : 80
    links:
      home: http://console.dc.cfos.org
      passwd: http://console.dc.cfos.org/password_resets/new
      signup: http://console.dc.cfos.org/register

  uaa:
    url: http://uaa.dc.cfos.org
    port: 80
    spring_profiles: postgresql
    no_ssl: true
    catalina_opts: -Xmx768m -XX:MaxPermSize=256m
    resource_id: account_manager
    jwt:
      signing_key: |
        -----BEGIN RSA PRIVATE KEY-----
        MIICXAIBAAKBgQDHFr+KICms+tuT1OXJwhCUmR2dKVy7psa8xzElSyzqx7oJyfJ1
        JZyOzToj9T5SfTIq396agbHJWVfYphNahvZ/7uMXqHxf+ZH9BL1gk9Y6kCnbM5R6
        0gfwjyW1/dQPjOzn9N394zd2FJoFHwdq9Qs0wBugspULZVNRxq7veq/fzwIDAQAB
        AoGBAJ8dRTQFhIllbHx4GLbpTQsWXJ6w4hZvskJKCLM/o8R4n+0W45pQ1xEiYKdA
        Z/DRcnjltylRImBD8XuLL8iYOQSZXNMb1h3g5/UGbUXLmCgQLOUUlnYt34QOQm+0
        KvUqfMSFBbKMsYBAoQmNdTHBaz3dZa8ON9hh/f5TT8u0OWNRAkEA5opzsIXv+52J
        duc1VGyX3SwlxiE2dStW8wZqGiuLH142n6MKnkLU4ctNLiclw6BZePXFZYIK+AkE
        xQ+k16je5QJBAN0TIKMPWIbbHVr5rkdUqOyezlFFWYOwnMmw/BKa1d3zp54VP/P8
        +5aQ2d4sMoKEOfdWH7UqMe3FszfYFvSu5KMCQFMYeFaaEEP7Jn8rGzfQ5HQd44ek
        lQJqmq6CE2BXbY/i34FuvPcKU70HEEygY6Y9d8J3o6zQ0K9SYNu+pcXt4lkCQA3h
        jJQQe5uEGJTExqed7jllQ0khFJzLMx0K6tj0NeeIzAaGCQz13oo2sCdeGRHO4aDh
        HH6Qlq/6UOV5wP8+GAcCQFgRCcB+hrje8hfEEefHcFpyKH+5g1Eu1k0mLrxK2zd+
        4SlotYRHgPCEubokb2S1zfZDWIXW3HmggnGgM949TlY=
        -----END RSA PRIVATE KEY-----
      verification_key: |
        -----BEGIN PUBLIC KEY-----
        MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQDHFr+KICms+tuT1OXJwhCUmR2d
        KVy7psa8xzElSyzqx7oJyfJ1JZyOzToj9T5SfTIq396agbHJWVfYphNahvZ/7uMX
        qHxf+ZH9BL1gk9Y6kCnbM5R60gfwjyW1/dQPjOzn9N394zd2FJoFHwdq9Qs0wBug
        spULZVNRxq7veq/fzwIDAQAB
        -----END PUBLIC KEY-----
    cc:
      client_secret: "c1oudc0w"
    admin:
      client_secret: "c1oudc0w"
    batch:
      username: batchuser
      password: c1oudc0w
    client:
      autoapprove:
        - cf
        - vmc
        - my
        - micro
        - support-signon
        - login
    clients:
      login:
        override: true
        scope: openid
        authorities: oauth.login
        secret: c1oudc0w
        authorized-grant-types: authorization_code,client_credentials,refresh_token
        redirect-uri: http://login.dc.cfos.org
      support-services:
        scope: scim.write,scim.read,openid,cloud_controller.read,cloud_controller.write
        secret: ssosecretsso
        authorized-grant-types: authorization_code,client_credentials
        redirect-uri: http://support-signon.dc.cfos.org
        authorities: portal.users.read
        access-token-validity: 1209600
        refresh-token-validity: 1209600
      vmc:
        override: true
        authorized-grant-types: password,implicit
        authorities: uaa.none
        scope: cloud_controller.read,cloud_controller.write,openid,password.write,cloud_controller.admin,scim.read,scim.write
      cf:
        override: true
        authorized-grant-types: password,implicit,refresh_token
        authorities: uaa.none
        scope: cloud_controller.read,cloud_controller.write,openid,password.write,cloud_controller.admin,scim.read,scim.write
      servicesmgmt:
        override: true
        secret: serivcesmgmtsecret
        scope: openid,cloud_controller.read,cloud_controller.write
        authorities: uaa.resource,oauth.service,clients.read,clients.write,clients.secret
        authorized-grant-types: authorization_code,client_credentials,password,implicit
        redirect-uri: http://servicesmgmt.dc.cfos.org/auth/cloudfoundry/callback
        autoapprove: true
    scim:
      users:
      - admin|c1oudc0w|scim.write,scim.read,openid,cloud_controller.admin
      - services|c1oudc0w|scim.write,scim.read,openid,cloud_controller.admin

  openstack:
    auth_url: http://10.10.5.1:5000/v2.0 # CHANGE
    username: admin # CHANGE
    api_key: openstack # CHANGE
    tenant: CFv2 # CHANGE
    default_security_groups: ["default"] # CHANGE
    default_key_name: ~/.microbosh/ssh/micro_bosh_rsa.pem # CHANGE
```

在这个配置文件中有几个地方需要改的:
1. 把所有的job里面指定的ip换成自己的实际ip
2. 把director的uuid换成当前microbosh的uuid,这个uuid可以通过`bosh status`查看
3. 换自己的域名


###4.6 部署cf.yml

等所有内容都编辑好之后就可以安装cf了.初次安装时,顺利执行大概需要将近40分钟左右的时间:

```
bosh deployment cf.yml
bosh deploy
```
如果以后配置文件有了变动,也是执行以上这两条命令.bosh会自动的检测出变化的内容,并执行.

如果中间出现了问题的话很有可能需要把刚刚部署的release删掉重来

	bosh release delete cf

如果一切顺利的话,就可以看到这样的画面.

```
Getting deployment properties from director...
Compiling deployment manifest...
Please review all changes carefully
Deploying `redis-mysql-mongodb.yml' to `microbosh-openstack' (type 'yes' to continue): yes

Director task 12

Preparing deployment
  binding deployment (00:00:00)                                                                     
  binding releases (00:00:00)                                                                       
  binding existing deployment (00:00:00)                                                            
  binding resource pools (00:00:00)                                                                 
  binding stemcells (00:00:00)                                                                      
  binding templates (00:00:00)                                                                      
  binding properties (00:00:00)                                                                     
  binding unallocated VMs (00:00:00)                                                                
  binding instance networks (00:00:00)                                                              
Done                    9/9 00:00:00                                                                

Preparing package compilation

Compiling packages
  redis_node/0.1-dev (00:01:58)                                                                     
Done                    1/1 00:01:58                                                                

Preparing DNS
  binding DNS (00:00:00)                                                                            
Done                    1/1 00:00:00                                                                

Preparing configuration
  binding configuration (00:00:00)                                                                  
Done                    1/1 00:00:00                                                                

Deleting unneeded VMs
  11905ca6-2a35-4326-9e87-faa843cb2dc4 (00:00:05)                                                   
Done                    1/1 00:00:05                                                                

Updating job core
  core/0 (canary) (00:01:04)                                                                        
Done                    1/1 00:01:04                                                                

Updating job common
  common/0 (canary) (00:01:03)                                                                      
Done                    1/1 00:01:03                                                                

Updating job dea
  dea/0 (canary) (00:02:23)                                                                         
  dea/1 (00:03:23)                                                                                  
Done                    2/2 00:05:46                                                                

Updating job uaa
  uaa/0 (canary) (00:00:57)                                                                         
Done                    1/1 00:00:57                                                                

Updating job router
  router/0 (canary) (00:00:51)                                                                      
Done                    1/1 00:00:51                                                                

Updating job mysql_service
  mysql_service/0 (canary) (00:04:30)                                                               
Done                    1/1 00:04:30                                                                

Updating job redis_service
  redis_service/0 (canary) (00:01:06)                                                               
Done                    1/1 00:01:06                                                                

Updating job mongodb_service
  mongodb_service/0 (canary) (00:02:42)                                                             
Done                    1/1 00:02:42                                                                

Task 12 done
Started		2013-07-09 14:05:05 UTC
Finished	2013-07-09 14:25:13 UTC
Duration	00:20:08

```
如果出现问题的话,请参照我写的TroubleShooting或者上Google Group提问.



##6.使用Cloud Foundry

终于到这一步完成了Cloud Foundry的部署，下面开始使用它吧!详细的使用步骤请参看<<Cloudfoundry使用手册>>.




##参考文献


* [http://docs.cloudfoundry.com/docs/running/bosh/reference/bosh-cli.html](http://)
* [http://docs.cloudfoundry.com/docs/running/deploying-cf/openstack/deploying_microbosh.html](http://)
* [http://ryangrenz.com/blog/2013/06/11/installing-cf-release-onto-openstack-with-bosh/](http://)
* [http://www.github.com](http://)
* [https://groups.google.com/a/cloudfoundry.org/forum/?fromgroups=#!forum/bosh-users](http://)






