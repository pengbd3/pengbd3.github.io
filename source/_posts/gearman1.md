---
title: 使用gearman实现PHP的多进程任务处理
date: 2017-11-08 14:50:05
tags:
  - PHP学习
  - gearman
categories: "PHP学习"
---
##### 一、GEARMAN简介 #####
{% fi http://oih4t7o53.bkt.clouddn.com/gearman1/6.png %}
Gearman是一个分发任务的程序框架，gearman 提供三个部分 client(客户端) jobServer(任务分发服务) worker（作业），它会对作业进行排队自动分配到一系列机器上。gearman跨语言跨平台，很方便的实现异步后台任务
<!-- more -->
##### 二、安装GEARMAN源码包 #####
`github下载：https://github.com/gearman/gearmand/releases`
##### 三、解压编译安装GEARMAN源码包 #####
先安装gearmand相关依赖包  
```shell
yum install -y libevent-dev uuid-dev boost boost-devel gperf
tar -zxvf gearmand-1.1.17.tar.gz
cd gearmand-1.1.17
./configure prefix=/usr/local/gearmand # 指定安装路径，方便后续PHP扩展安装  

make && make install
# 加入系统变量
vim /etc/profile
PATH=$PATH:/usr/local/webserver/php/bin:/usr/local/webserver/mysql/bin
export PATH
source /etc/profile
```
安装完成以后可以使用 `gearmand -d` 命令测试服务是否能正常开启。`htop` 命令查看进程  
![安装成功后的进程展示](http://oih4t7o53.bkt.clouddn.com/gearman1/1.jpg)

##### 四、安装PHP的gearman扩展 #####
`pecl官网下载：http://pecl.php.net/package/gearman`
![gearman扩展下载页面](http://oih4t7o53.bkt.clouddn.com/gearman1/2.jpg)  
注：PHP 7扩展可以前往 `https://github.com/wcgallego/pecl-gearman/tree/master` 下载  
解压、编译、安装
```shell
 tar -zxvf gearman-1.1.2.tar.gz
 cd gearman-1.1.2
 ./configure --with-php-config=/usr/local/php/bin/php-config --with-gearman=/usr/local/gearmand
 ```  
注：最好带上--with-gearman参数，不然会报libgearman尚未安装的错误
![gearman扩展编译结果](http://oih4t7o53.bkt.clouddn.com/gearman1/3.jpg)  
然后进行编译安装  
`make && make install`  
![gearman扩展编译安装成功](http://oih4t7o53.bkt.clouddn.com/gearman1/4.jpg)  
``` 
php.ini # 增加配置  
extension_dir=/usr/local/php/lib/php/extensions/no-debug-non-zts-20090626/
extension=gearman.so
```
重启`PHP`，使用命令 `php -m | grep gearman` 检查扩展是否添加成功  
![gearman扩展成功安装实例图](http://oih4t7o53.bkt.clouddn.com/gearman1/5.jpg)  
##### 五、简单测试gearman功能 #####
gearman中请求的处理过程一般涉及三种角色：client->job->worker  
其中client是请求的发起者  
job是请求的调度者，用于把客户的请求分发到不同的worker上进行工作
worker是请求的处理者  

比如这里我们要处理client向job发送一个请求，来计算两个数之和，job负责调度worker来具体实现计算两数之和。  
创建client.php和work.php文件  
![gearman扩展成功安装实例图](http://oih4t7o53.bkt.clouddn.com/gearman1/7.jpg)   
参照官方事例 `http://gearman.org/examples/reverse/`  
 `client.php` 代码如下
```php
<?php
//创建一个客户端对象
$client = new GearmanClient();
//添加一个job服务
$client->addServer('127.0.0.1',4730); //默认参数主机地址：127.0.0.1 端口：4730

echo "发送job\n";
//同步发送数据
$result=$client->doNormal("worker",json_encode(array('name'=>'fate'),1));

if($result){
    echo "Success: $result\n";
}
```

 `worker.php` 代码如下
```php
<?php

//创建一个属于你自己的worker对象
$worker = new GearmanWorker();
//添加一个work服务（）
$worker->addServer('127.0.0.1', 4730);
//通知服务器，处理函数名为send任务
$worker->addFunction("worker", "sayName");
//死循环等待任务介入
while (true) {
    echo "等待job处理\n";
    //阻塞执行任务，等待任务处理完成
    $result = $worker->work();
    //如果执行失败将会跳出循环
    if ($worker->returnCode() != GEARMAN_SUCCESS) {
        break;
    }
}
/**
 * 任务实现
 * @param GearmanJob $job 任务执行者
 * @return string
 */
function sayName(GearmanJob $job)
{
    $workload = $job->workload();
    echo "返回任务的句柄:" . $job->handle() . "\n";
    echo "打印参数: $workload\n";
    $result = json_decode($workload, 1);
    $answer = "我叫:" . $result['name'];
    echo "返回值:$answer\n";
    //等待2秒，模拟服务处理时间
    sleep(2);
    return $answer;
}
```
一切准备工作就绪，现在让我们启动gearmand服务
``` $xslt
    # 创建gearmand服务用户
    adduser -M -G www gearmand
    # 创建进程代号文件夹
    mkdir -p /var/run/gearmand
    # 修改所属用户与所属组
    chown gearmand:gearmand /var/run/gearmand
    # 启动
    gearmand -d -p 4730 -u gearmand -P /var/run/gearmand/gearmand.pid -l /var/log/gearmand/gearmand.log 
```
使用 `htop` 命令查看是否启动成功
由于我是在虚拟机服务器做的测试，所以先将文件传入服务器中  
分别打开两个终端先后执行 `php /data/worker.php` `php /data/client.php` 

结果如下：  
![gearman服务端](http://oih4t7o53.bkt.clouddn.com/gearman1/9.jpg)
服务端
![gearman客户端](http://oih4t7o53.bkt.clouddn.com/gearman1/10.jpg)
客户端  
##### 六、gearman异步处理任务 #####
大多时候执行任务消耗时间比较长我们将考虑到异步去处理任务，下面我们将演示常见的邮件发送的事例  
参照官方事例 `http://gearman.org/examples/send-emails/`  
 `client.php`   
```php
<?php
//填写邮件相关内容
$email = 'fate@163.com';
$subject = "我是邮件主题!";
$body = '我是邮件内容!';
$data = array('email' => $email, 'subject' => $subject, 'body' => $body);
$client = new GearmanClient();
$client->addServer('127.0.0.1', 4730);
$result = $client->doBackground("worker", json_encode($data, 1));
echo "请求已提交服务器\n";
```
`worker.php`
```php
<?php

$worker = new GearmanWorker();
$worker->addServer('127.0.0.1', 4730);

$worker->addFunction("worker", 'sendEmail');

while (true) {
    echo "等待job处理\n";
    //阻塞执行任务，等待任务处理完成
    $result = $worker->work();
    //如果执行失败将会跳出循环
    if ($worker->returnCode() != GEARMAN_SUCCESS) {
        break;
    }
};

function sendEmail(GearmanJob $job)
{
    $workload = json_decode($job->workload(), 1);
    echo "正在发送邮件: " . print_r($workload, 1);
    // 这里存放你需要实际发送的邮件的实现方法
    // 等待3秒模拟发送邮件所需时长
    sleep(3);
    echo "发送邮件{$workload['email']}成功\n";
    //mail($workload->email, $workload->subject, $workload->body);
}
```
分别打开两个终端先后执行 `php /data/worker.php` `php /data/client.php` 
结果如下：  
![gearman服务端/客户端](http://oih4t7o53.bkt.clouddn.com/gearman1/11.jpg)  
##### 七、gearman同步并行处理任务 #####
通常某些时候我们会遇到任务并行处理的需求，例如抢购活动。这里我将采用gearman的addTask来实现累加和。
 `client.php`  
 ```php
<?php
//
$sum = 0;
$client = new GearmanClient();
$client->addServer('127.0.0.1', 4730);

//可以通过setCompleteCallback函数给计算结果返回给客户端
$client->setCompleteCallback(function(GearmanTask $task)use(&$sum){
    $sum = $task->data();
    echo "计算结果为:".$sum."\n";
});
//添加5个需要累加和的任务
$client->addTask('Sum', json_encode(array(1, 100)));
$client->addTask('Sum', json_encode(array(100, 200)));
$client->addTask('Sum', json_encode(array(200, 300)));
$client->addTask('Sum', json_encode(array(300, 400)));
$client->addTask('Sum', json_encode(array(400, 500)));

//运行队列中的任务，do系列不需要runTask()
$client->runTasks();


```
 `worker.php`  
 ```php
 <?php
 
 $worker = new GearmanWorker();
 $worker->addServer('127.0.0.1', 4730);
 
 $worker->addFunction("Sum", 'Add');
 
 while (true) {
     echo "等待job处理\n";
     //阻塞执行任务，等待任务处理完成
     $result = $worker->work();
     //如果执行失败将会跳出循环
     if ($worker->returnCode() != GEARMAN_SUCCESS) {
         break;
     }
 };
 
 function Add(GearmanJob $job)
 {
     $sum=0;
     $workload = json_decode($job->workload(), 1);
     sleep(1);
     for ($i=$workload[0];$i<=$workload[1];++$i){
         $sum+=$i;
     }
     return $sum;
 }
 ```
 结果如下：  
 ![gearman服务端/客户端](http://oih4t7o53.bkt.clouddn.com/gearman1/12.jpg)  
 细心的朋友会发现执行过程和doNormal的方法单个执行差不多，没有达到预期并行处理的效果。原因在与我们只开启一个worker，为了方便演示这里我复制4个ssh连接，分别在启动服务端可得到如下的结果:
  ![gearman服务端/客户端](http://oih4t7o53.bkt.clouddn.com/gearman1/13.jpg)   
  可发现基本是1秒处理完所有任务,这里并行处理任务就演示完成了。