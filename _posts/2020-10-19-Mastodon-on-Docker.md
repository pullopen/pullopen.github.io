---
title: 如何利用Docker搭建Mastodon实例（一）：基础搭建篇
layout: post
tags: [技术, Mastodon, Docker]
image: assets/img/docker-logo.png
catalog: true 
#gif: mygif
description: "Docker，一个也许更适合新手维护的搭建方案。"
customexcerpt: "Docker，一个也许更适合新手维护的搭建方案。"
---


## 为什么开始使用Docker搭建实例


开始使用Docker纯属一个手残意外导致的阴差阳错：我不小心在升级时按了DigitalOcean面板里的“关闭服务器”按钮，相当于跑着程序突然拔了主机电源，导致整个服务器出现大型罢工。最后由兔子帮（代）助（劳）我迁移了站点，方便起见部署在了Docker上。于是在此也提醒各位，想要关闭Mastodon服务，一定要使用

`systemctl stop 具体服务（mastodon-sidekiq、mastodon-web以及mastodon-streaming）`

的方法，千万不要硬关。

在使用了一段时间的Docker之后，我总结出以下Docker搭建的优缺点：

**优点：**

  1. 搭建、升级方便，命令简单，不用自行配置环境，比起用官方文档命令行搭建而言，更适合新手。

  2. 对内存较小的服务器，可以免除每次升级需要的编译（precompile）步骤给小机器带来的负担。

  3. Mastodon站点运行在一个隔离的小环境（容器）中，安全性较高，不怕新手操作把整个系统搞崩，出现问题之后只要重启即可，会按镜像自动复原。


**缺点：**

  1. 可用教程较少，许多命令需要重新学起。

  2. 魔改字数等相对而言不太方便，需要增加一个步骤，在下面会详细列出。

  3. 需要学习docker相关命令。


总体而言，对于一个新手，docker维护起来相对还是比较方便（皮实耐操）的。大家可以自行决定。但搜索互联网，官方并没有给出docker的搭建指南，而民间指南要么过时、要么有冗余步骤会占用大量时间。因此，希望这篇教程能给大家带来一些帮助。

本教程完全依赖兔子（@star@b612.me）大佬的手把手指导，万分感谢！

　　

　　

## 如何在Docker上从头搭建Mastodon

首先，购买域名、购买服务器、配置SMTP服务等，在此不再赘述，大家可以参考本站[之前的教程](https://pullopen.github.io/2020/07/19/How-to-build-a-mastodon-instance.html){:target="_blank"}。

本文所用服务器操作系统为**Ubuntu 18.04或Debian 10**，请各位于购买时确认。

　　

#### 1. 配置系统

   * 配置ssh-key：

     {% highlight shell linenos %}
     mkdir -p ~/.ssh
     nano ~/.ssh/authorized_keys
     {% endhighlight %}

     将通过各种方法（如Xshell、PuTTy等软件）生成的ssh-rsa公钥粘贴入其中。随后通过ssh-key密钥方式登录。

     为了安全，官方推荐将ssh密码登录方式关闭（不影响通过VNC、DigitalOcean Console等方式登录，**请确保此时你的SSH是依靠密钥而不是密码登录，否则你设置完毕后会被踢出去**）：

     {% highlight shell linenos %}
     nano /etc/ssh/sshd_config
     {% endhighlight %}

     找到`PasswordAuthentication`一行，将其前面的#删掉（取消注释），在后面将yes改成no。

     重启sshd：

     {% highlight shell linenos %}
     systemctl restart sshd
     {% endhighlight %}


   * 安装常用命令：

     {% highlight shell linenos %}
     apt update && apt install wget rsync python git curl vim git ufw -y
     {% endhighlight %}


   * 配置SWAP，具体请参考[配置SWAP教程](https://www.digitalocean.com/community/tutorials/how-to-add-swap-space-on-ubuntu-16-04){:target="_blank"}。

     请让你的内存+SWAP至少达到4G以上。可以在root用户下通过`free -h`查看。
   
   
   * 配置防火墙

     {% highlight shell linenos %}
     sudo ufw allow OpenSSH
     sudo ufw enable
     sudo ufw allow http
     sudo ufw allow https
     {% endhighlight %}

   然后可以通过`sudo ufw status`检查防火墙状态，你应该会看到80和443端口的显示。

　　

#### 2. 安装docker和docker-compose

  注意：这里第一步使用了官方提供的一键脚本安装docker。如果你对此感到不放心，请通过[官网步骤](https://docs.docker.com/engine/install/ubuntu/){:target="_blank"}自行安装，同样也是复制粘贴命令行。


   {% highlight shell linenos %}
   bash <(curl -L https://get.docker.com/)
   curl -L "https://github.com/docker/compose/releases/download/1.27.4/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
   chmod +x /usr/local/bin/docker-compose
   {% endhighlight %}


　　

#### 3. 拉取Mastodon镜像

   * 拉取镜像

     {% highlight shell linenos %}
     mkdir -p /home/mastodon/mastodon
     cd /home/mastodon/mastodon
     docker pull tootsuite/mastodon:latest     #如果需要升级到某稳定版本，请将latest改成v3.2.1等版本号。
     wget https://raw.githubusercontent.com/tootsuite/mastodon/master/docker-compose.yml
     {% endhighlight %}

   * 修改`docker-compose.yml`配置文件

     {% highlight shell linenos %}
     nano docker-compose.yml
     {% endhighlight %}

     修改`db`部分，在该部分最后一行后加上

     {% highlight shell linenos %}
     environment:
        - POSTGRES_HOST_AUTH_METHOD=trust
     {% endhighlight %}


     随后依次找到`web`、`streaming`、`sidekiq`分类，在每一类的`image: tootsuite/mastodon`后添加`:latest`或者你刚才拉取的版本号，变成`image: tootsuite/mastodon:latest`或`image: tootsuite/mastodon:v3.2.1`等等。


     ctrl+X退出保存。

　　

#### 4. 配置Mastodon

   * 配置文件

     在`/home/mastodon/mastodon`文件夹中创建空白`.env.production`文件：

     {% highlight shell linenos %}
     touch .env.production
     {% endhighlight %}

     root用户内，运行

     {% highlight shell linenos %}
     docker-compose run --rm web bundle exec rake mastodon:setup
     {% endhighlight %}

     * 输入域名

     * Enable single user mode? 否

     * Using Docker to run Mastodon? 是

     * postsql和redis部分都直接回车

     * Store uploaded files on the cloud? 这个我们先填否，之后再参考[上云教程](https://pullopen.github.io/2020/07/22/Move-mastodon-media-to-Scaleway.html){:target="_blank"}配置。

     * Send e-mails from localhost? 否。然后填入邮件服务设置，具体参考[第一篇教程](https://pullopen.github.io/2020/07/19/How-to-build-a-mastodon-instance.html){:target="_blank"}。

     * This configuration will be written to .env.production
     Save configuration? 是

     然后会出现.env.production配置，**复制下来，先存到电脑里，等会儿要用。**

     然后会要你建立数据库和编译，都选是。最后建立管理员账号。

     一切成功之后，记得**立刻马上：**

     {% highlight shell linenos %}
     nano .env.production
     {% endhighlight %}

     把你刚才复制下来的配置保存进去。


   * 启动Mastodon

     {% highlight shell linenos %}
     docker-compose up -d
     {% endhighlight %}
  
   * 为相应文件夹赋权

     {% highlight shell linenos %}
     chown 991:991 -R ./public
     chown -R 70:70 ./postgres
     docker-compose down
     docker-compose up -d
     {% endhighlight %}

　　

#### 5. 安装并配置nginx

   * 安装nginx

     {% highlight shell linenos %}
     sudo apt install nginx -y
     {% endhighlight %}

   * 配置nginx

     {% highlight shell linenos %}
     nano /etc/nginx/sites-available/你的域名
     {% endhighlight %}

     网页打开[nginx模板](https://github.com/tootsuite/mastodon/blob/master/dist/nginx.conf){:target="_blank"}，将其中的example.com替换成自己域名，将20和43行的`/home/mastodon/live/public`改成`/home/mastodon/mastodon/public`，复制到服务器中保存。

     投射镜像文件：

     {% highlight shell linenos %}
     ln -s /etc/nginx/sites-available/你的域名 /etc/nginx/sites-enabled/
     {% endhighlight %}

     重启nginx：

     {% highlight shell linenos %}
     systemctl reload nginx
     {% endhighlight %}

     安装certbot：

     {% highlight shell linenos %}
     sudo snap install core; sudo snap refresh core    #如果没有snap则 apt install snapd 安装
     sudo snap install --classic certbot
     sudo ln -s /snap/bin/certbot /usr/bin/certbot
     sudo certbot --nginx -d 你的域名
     {% endhighlight %}

     再重启nginx

     {% highlight shell linenos %}
     systemctl reload nginx
     {% endhighlight %}


   如果不放心，可以再至`/home/mastodon/mastodon`文件夹，运行`docker-compose up -d`重启mastodon。静静等待几分钟后，点开你的域名，你的站点就上线啦！

　　

　　

在站点上线之后，你可以：

## 开启全文搜索

Docker的全文搜索开启十分方便，只需要：

{% highlight shell linenos %}
cd /home/mastodon/mastodon
nano docker-compose.yml
{% endhighlight %}

编辑`docker-compose.yml`，去掉`es`部分前所有的#号，并且去掉`web`部分中`es`前面的#号。

`nano .env.production`编辑`.env.production`文件，加上

{% highlight ruby linenos %}
ES_ENABLED=true
ES_HOST=es
ES_PORT=9200
{% endhighlight %}

三行，重启：

{% highlight shell linenos %}
docker-compose down
docker-compose up -d
{% endhighlight %}

待文件夹中出现elasticsearch文件夹后，赋权：

{% highlight shell linenos %}
chown 1000:1000 -R elasticsearch
{% endhighlight %}

再次重启：

{% highlight shell linenos %}
docker-compose down
docker-compose up -d
{% endhighlight %}

全文搜索即搭建完成。

然后

{% highlight shell linenos %}
docker-compose run --rm web bin/tootctl search deploy
{% endhighlight %}

建立之前嘟文的搜索索引即可。

　　

　　

## 修改配置文件

如果在之后需要再对.env.production配置进行修改，只需：

{% highlight shell linenos %}
cd /home/mastodon/mastodon
nano .env.production
{% endhighlight %}

进行相应修改，然后

{% highlight shell linenos %}
docker-compose down
docker-compose up -d
{% endhighlight %}

重启即可。

　　

　　

## 使用管理命令行

在docker中使用tootctl管理命令行的方式有三种：

  * **进入docker系统后操作**

     `docker ps`查看你的容器名字，如果你按照刚才设置，那你的容器名字一般为mastodon_web_1。

     {% highlight shell linenos %}
     cd /home/mastodon/mastodon
     docker exec -it mastodon_web_1 /bin/bash    #或者将“mastodon_web_1”替换为你的容器名。
     {% endhighlight %}

     进入docker系统mastodon用户，然后在其中进行相应的tootctl操作。

     注：如果需要进入docker系统的root用户进行一些软件安装，则需输入`docker exec --user root -it mastodon_web_1 /bin/bash`。

  * **在/home/mastodon/mastodon文件夹操作**

     首先进入/home/mastodon/mastodon，然后

     {% highlight shell linenos %}
     docker-compose run --rm web bin/tootctl 具体命令
     {% endhighlight %}

     进行操作。

  * **在任意位置操作**

     在任意位置：

     {% highlight shell linenos %}
     docker exec mastodon_web_1 tootctl 具体命令
     {% endhighlight %}

     需要注意的是，这则具体命令需要包括所有必须的参数，并且如果命令本身会要求你进行后续输入，则无法完成（比如self-distruct命令无法通过该步骤完成。）

　　

　　

## 升级

如果你要升级到最新版本，只需要：

{% highlight shell linenos %}
cd /home/mastodon/mastodon
docker pull tootsuite/mastodon:latest     #或者将latest改成版本号如v3.2.1
{% endhighlight %}

如果你升级的是特定版本，则需要编辑docker-compose.yml，将web、streaming、sidekiq三部分的版本号改成相应版本。如果是latest则无需改动。

然后

{% highlight shell linenos %}
docker-compose up -d
{% endhighlight %}

启动。

如果官方升级提示中包括其他步骤如`docker-compose run --rm web rails db:migrate`，则可在启动后进行。

在确认升级没问题之后，运行

{% highlight shell linenos %}
docker system prune -a
{% endhighlight %}

清除旧的docker镜像文件。

　　

　　

## 如果在操作过程中出现了任何问题……

如果没有对站点进行过魔改，只要在docker系统外，通过

{% highlight shell linenos %}
docker-compose down
docker-compose up -d
{% endhighlight %}

让系统通过docker镜像重新搭建容器即可。

　　

　　

## 利用Scaleway备份数据库

本步脚本由兔子写就，感谢ta！

注册Scaleway，申请token，创建Bucket，此三步参考[Scaleway上云教程](https://pullopen.github.io/2020/07/22/Move-mastodon-media-to-Scaleway.html){:target=blank}。

在服务器中安装rclone、fuse和zip：

{% highlight shell linenos %}
curl https://rclone.org/install.sh | sudo bash
apt install fuse zip -y
{% endhighlight %}

创建rclone配置文件夹

{% highlight shell linenos %}
mkdir -p  ~/.config/rclone/
{% endhighlight %}

新建配置文件：

{% highlight shell linenos %}
nano ~/.config/rclone/rclone.conf
{% endhighlight %}

填入下列内容：

{% highlight ruby linenos %}
[scaleway]
type = s3
provider = Scaleway
access_key_id = 你的ACCESS KEY
secret_access_key = 你的SECRET KEY
region = nl-ams（根据你bucket选择的地区，法国fr-par，荷兰nl-ams，波兰pl-waw）
endpoint = s3.nl-ams.scw.cloud  （同上）
acl = private
{% endhighlight %}

保存。

然后创建脚本

{% highlight shell linenos %}
nano /backup.sh
{% endhighlight %}

输入

{% highlight shell linenos %}
#!/bin/bash
source /etc/profile
now=$(date "+%Y%m%d-%H%M%S")
origin="/home/mastodon/mastodon"
target="scaleway:你的bucket名字"
echo `date +"%Y-%m-%d %H:%M:%S"` " now starting export"
/usr/bin/docker exec pg容器名 pg_dump -U postgres -Fc mastodon_production > ${origin}/backup.dump &&
echo `date +"%Y-%m-%d %H:%M:%S"` " succeed and upload to s3 now"
/usr/bin/zip -P 密码 ${origin}/backup_${now}.zip ${origin}/backup.dump &&
/usr/bin/rclone copy ${origin}/backup_${now}.zip ${target} &&
echo `date +"%Y-%m-%d %H:%M:%S"` " ok all done"
rm -f ${origin}/backup.dump ${origin}/backup_${now}.zip
/usr/bin/rclone --min-age 7d  delete ${target}
{% endhighlight %}

pg容器名一般为mastodon_db_1（通过docker ps查看），密码为你设立的解压密码。保存。

赋权：

{% highlight shell linenos %}
chmod 751 /backup.sh
{% endhighlight %}

然后试运行一下，看看Scaleway中有没有zip文件生成。如果出现zip文件则成功。之后恢复则可参考[迁移教程](https://pullopen.github.io/2020/10/21/migrate-Mastodon-to-Docker.html){:target=blank}。

{% highlight shell linenos %}
crontab -e
{% endhighlight %}

创建定时任务：

{% highlight shell linenos %}
3 22    * * *   /backup.sh >> /backup.log
{% endhighlight %}

具体时间自己设置，建议设置在半夜，注意服务器时区（通过`date`查看服务器时间）。

　　

　　

## 总结

以上就是从头通过Docker搭建Mastodon的方法，通过单纯复制粘贴很快就能搭建出来。在升级过程中也能免去编译过程对小机器造成的负担。（当然，SWAP还是要开的！）如果使用官方分支，升级只要几分钟。

目前网络上一些其他的docker搭建指南都提到需要docker-compose build这一步骤，并无必要，且耗时长、对服务器压力大。

然后有朋友就要问了：我之前是按照DigitalOcean一键镜像搭建的/用官方文档命令行搭建的，怎么迁移到docker上去呢？如果我想对代码进行一点改动要怎么做呢？这些问题，我们将在后续的文章中谈到。














