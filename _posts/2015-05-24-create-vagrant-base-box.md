---
layout: post
title: 半自动创建Vagrant的Ubuntu base box
tags: DevOps vagrant project-vmlab
comment: true
image: 
published: true
---
DevOps的观念正深刻的改变软件开发的方式，使用Vagrant可以几分钟内部署单个LAMP的VM或一个Hadoop VM集群，使得开发测试环境和生产环境无限逼近。Vagrant使用的起点是base box。网上有很多共享的base box。如果你碰到下面情况，还是要自己创建base box。

- 定制VM的硬盘配置，增大硬盘到100GB，配置4个硬盘
- 安装特定的OS版本
- 定制VM的外部配件配置，例如USB/Audio，Vagrant推荐base box不要打开这些，因为服务器开发一般不需要它们。

下面的Vagrant官方文档描述了如何创建base box。但是步骤很多，如果能自动完成就造福大众了。

- http://docs.vagrantup.com/v2/boxes/base.html
- http://docs.vagrantup.com/v2/virtualbox/boxes.html

我写了一个脚本(https://github.com/yuanyangwu/vmlab)，可以半自动的创建Ubuntu 14.04的base box，可以自动完成官方文档中在Guest OS里的以下所有操作：

- 用户"root"密码，改为"vagrant"
- 用户"vagrant"的SSH authorized_keys，改为vagrant public certificate
- 用户"vagrant"执行sudo时不用输入密码
- openssh服务，关闭UseDNS，避免对SSH client进行DNS解析，加快SSH等入
- Ubuntu更新源，改为mirrors.aliyun.com，在中国可以快速更新
- 安装VirtualBox Guest Add-on

## 使用步骤

1. 安装VirtualBox、Vagrant，因为要用到ssh/scp，如果我们的机器是Windows还要装putty。准备好Ubuntu的安装ISO文件。
2. 启动VirtualBox创建VM   
   - 命名"vagrant-ubuntu-server1404-100g"
   - RAM 512MB。不要设太大可以让大多数机器默认使用，后面例子会演示如何在使用Vagrant时动态修改这个值
   - 创建硬盘时选择类型"VMDK"。我试过使用"VDI"类型，但是vagrant package会自动转为"VMDK"
   - 打开VM的Settings   
      * Audio，不要勾"Enable Audio"
      * USB，不要勾"Enable USB Controller"
      * Network，对Adapter 1，确保attached to "NAT"，然后点击Advanced > Port Forwarding按钮，添加规则：Protocol TCP，Host Port 2200，Guest Port 22
3. 启动VM，根据提示，选择Ubuntu的ISO文件，作为虚拟光驱插入VM。一路确认并输入以下设置，最后重启。
   - Hostname: "vagrant-ubuntu-server1404-100g"
   - User: "vagrant"
   - Password: "vagrant"
   - "Software selection", 用空格选择"OpenSSH server"
4. 配置Guest OS
   - 选择VirtualBox Guest Addon的ISO文件，作为虚拟光驱插入VM。ISO文件随VirtualBox安装，具体位置如下。如果没有找到，可以从http://download.virtualbox.org/virtualbox/\${VIRTUALBOX_VERSION}/VBoxGuestAdditions_\${VIRTUALBOX_VERSION}.iso下载。
     * Windows，C:\Program Files\Oracle\VirtualBox\VBoxGuestAdditions.iso
     * Linux，/usr/share/virtualbox/VBoxGuestAdditions.iso
     * Mac，??
   - 下载脚本https://raw.githubusercontent.com/yuanyangwu/vmlab/master/create_vagrant_base_box/vagrant-guest.sh
   - 把脚本传入Guest OS。这里利用了第2步里配置的Port Forwarding
     
      ```
      scp -P 2200 vagrant-guest.sh vagrant@localhost:/home/vagrant
      ```
   - SSH登入Guest OS

      ```
      ssh vagrant@localhost -p2200
      ```
   - 在Guest OS里运行脚本。运行时，根据提示，输入密码"vagrant"。运行结束会自动关闭Guest OS并断开SSH。

      ```
      sh vagrant-guest.sh
      ```
5. 打包Vagrant base box。100GB硬盘的VM对应的base box只有730MB，非常经济:)
    ```
    cd ~/vagrant_repo
    vagrant package --base vagrant-ubuntu-server1404-100g --output ubuntu.server1404.100g
    ```

## 测试验证

### 测试1 - 添加base box

```bash
vagrant box add --name ubuntu.server1404.100g ~/vagrant_repo/ubuntu.server1404.100g
```
可以用`vagrant box list`检查。

### 测试2 - 单个VM的Vagrant

- 新建文件Vagrantfile
  
```
Vagrant.configure(2) do |config|
  config.vm.box = "ubuntu.server1404.100g"
end
```
- 启动VM

```
vagrant up
```
- SSH登入VM

```
vagrant ssh
```
- 删除VM

```
vagrant destroy -f
```

### 测试3 - 多个VM的Vagrant
- 新建文件Vagrantfile。注意其中
  * 设置hostname
  * 连接"private_network"网络，并指定IP
  * 修改memory和CPU个数

```
servers = {
  :server1 => '192.168.33.10',
  :server2 => '192.168.33.11'
}

Vagrant.configure(2) do |config|
  config.vm.box = "ubuntu.server1404.100g"

  servers.each do |server_name, server_ip|
    config.vm.define server_name do |server_config|
      server_config.vm.hostname = server_name
      server_config.vm.network "private_network", ip: server_ip
      config.vm.provider "virtualbox" do |vb|
        vb.memory = "256"
        vb.cpus = 2
      end
    end
  end
end
```
- 启动VM。会把几个VM都启动

```
vagrant up
```

- SSH登入不同VM。可以看到
  * hostname的确修改成"server1"、"server2"。
  * 每个VM有2个网卡
    + 第1个网卡连接NAT（配有Port Forwarding，用于vagrant ssh）
    + 第2个网卡链接private_network（就是VirtualBox的Hostonly，VM之间，VM和Host OS间，都可以通信）

```
vagrant ssh server1
vagrant ssh server2
```


```
+---------+
|  VM     |
+=========+
|  eth0   +-----NAT
+---------+
|  eth1   +-----Hostonly
+---------+
```

- 删除VM

```
vagrant destroy -f
```