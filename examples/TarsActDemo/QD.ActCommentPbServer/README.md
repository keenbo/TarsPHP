# TARS-PB-SERVER的说明

## 依赖

需要swoole版本4.3.4以上，开启openssl 开启http2
需要安装 protoc c库，需要安装 protobuf php 扩展
 参考 https://github.com/protocolbuffers/protobuf/tree/master/php
（google 官方文档如果看的比较辛苦 可以自己查询其他教程）

## 目录结构说明

1. scripts
存储业务所需要的脚本,如proto2php.sh 负责的就是根据proto文件,生成服务端所需要的代码。
这个脚本会调用pb 的 protoc 脚本把message 生成php 类。

2. src 
业务逻辑所在的目录,主要包含以下结构:
- conf: 业务需要的配置, 这里只是给出一个demo,如果从平台下发配置,默认也会写入到这个文件夹中;
- GPBMetadata: protoc 生成的文件，不用在意
- impl: 业务实际的实现代码,是对interface的对应实现, 具体实现的文件路径需要写入到services.php中;
- protocol: protoc 生成的class 文件和proto2php 生成的rpc interface 都会在这里
- servant: 使用tars2php生成的server端的代码, 这个目录名称完全可以自定义,只需要在使用的时候加以区分就可以了; 
- cservant: 调用其他client的代码,命名完全可以自定义,参考tars2php的代码生成;
- composer.json: 说明项目的依赖情况
- index.php: 整个服务的入口文件,可以自定义,但是必须要更改平台上的私有模板,增加entrance这个字段在server下面
- services.php: 声明interface的地址,声明实际实现的地址,这个两个地址会被分别用作实例化调用和注解解析

3. tars
这个tcp服务所依赖的就是这个文件夹下面的actComment.proto文件 
而tars.proto.php文件,则是生成servant下代码所必须的,这个会在下面的guideline中具体说明。

## 服务部署guideline

1. 进入运维管理 => 模板管理
平台会提供一份新的针对php的模板,命名为tars.tarsphp.default, !!!!!!!必须首先修改其中php的执行路径!!!!!!!。

2. 在平台上进入运维管理=>部署服务,填写对应的应用名和服务名称,注意,这个与下文中tars文件夹中的tars.proto.php
需要完全一致。

3. 选择服务类型为tars_php

4. 选择模板为刚刚建立的tcp服务模板,默认不启用set

5. 选择可用端口,填写服务器内网ip

6. 端口类型为TCP
!!!!协议类型TCP服务必须选择 非TARS!!!!!!

7. 线程数对应SWOOLE的进程数

8. 最大连接数和队列最大长度、队列超时时间,对php服务不生效

9. 点击添加和提交即可,接下来请进入开发guidline

## 开发guideline


1. 自行撰写了一份proto文件,可以是actComment.proto

2. 新建对应的目录结构,固定为scripts、src和tars

3. 在scripts下面新建proto2php.sh文件, 内容为
```
cd ../tars/

php ../src/vendor/phptars/tars2php/src/proto2php.php ./tars.proto.php
```

4. 将actComment.proto放入tars文件夹中,同时在tars文件夹下面新建tars.proto.php文件:
```
<?php
return array(
    'appName' => 'QD',
    'serverName' => 'ActCommentPbServer',
    'objName' => 'CommentObj',
    'withServant' => true, //决定是服务端,还是客户端的自动生成
    'tarsFiles' => array(
        './actComment.proto',
    ),
    'dstPath' => '../src/protocol', //这里指定的是 impl 基础interface 生成的位置
    'protocDstPath' => '../src', //这里指定的是 protoc 生成的问题
    'namespacePrefix' => 'Protocol',
);
```

APPName、serverName、objName 需要与tars平台上面申请的完全一致。withServant必须为true,同时指定tarsFiles的路径。
dstPath一般是`../src/?`,这里为`../src/servant`,这样生成的代码就会到这个文件夹。
namespacePrefix是对应代码的命名空间,这里是`Server\servant`,这个与composer.json中的psr-4的名称也是相互对应的。

注意 dstPath 和 protocDstPath 的不同： 
dstPath是用来生成 impl 基础interface 的，生成在 dstPath/APPName/serverName/objName 下
protocDstPath 用来代用protoc用来把message生成php 类的，生成在 protocDstPath/（package name 点转为/）/ 下

5. 执行scripts下面的proto2php.sh, 会在src/protocol下面生成一个二级文件夹,
这里就是QD/ActCommentPbServer 
* tars文件夹 - 存放proto文件
* CommentObjServant.php - interface文件


6. 在src下新建impl文件夹
在里面新建ActCommentServerImpl.php文件,这个文件,必须实现CommentObjServant.php其中对应的所有方法


7. 新建composer.json文件,内容如下:
```
{
    "name" : "tars-pb-server-demo",
    "description": "tars pb server",
    "require": {
        "phptars/tars-server": "~0.3",
        "phptars/tars-deploy": "~0.1",
        "phptars/tars-log": "~0.1",
        "phptars/tars2php": "~0.1",
        "ext-zip" : ">=0.0.1",
        "google/protobuf": "^3.8"
    },
    "autoload": {
        "psr-4": {
            "Server\\" : "./",
            "Protocol\\" : "./protocol",
            "GPBMetadata\\" : "./GPBMetadata"
        }
    },
    "repositories": [
        {
            "type": "composer",
            "url": "https://mirrors.aliyun.com/composer/"
        },
        {
            "packagist": false
        }
    ],
    "minimum-stability": "stable",
    "scripts" : {
        "deploy" : "\\Tars\\deploy\\Deploy::run"
    }
}

```
其中name、description、autoload中的psr-4可以修改为自己需要的,我们这里以此为例子。

8. 在src下新建index.php,内容如下:
```
<?php
require_once(__DIR__."/vendor/autoload.php");

use \Tars\cmd\Command;

//php tarsCmd.php  conf restart
$config_path = $argv[1];
$pos = strpos($config_path,"--config=");

$config_path = substr($config_path,$pos+9);

$cmd = strtolower($argv[2]);

$class = new Command($cmd,$config_path);
$class->run();

```
这个文件负责进行启动和入口加载工作

9. 新建services.php文件,内容如下:
```
<?php
/**
 * Created by PhpStorm.
 * User: liangchen
 * Date: 2018/2/24
 * Time: 下午2:51
 */

// 以namespace的方式,在psr4的框架下对代码进行加载
return [
    'CommentObj' => [
        'home-api' => '\Protocol\QD\ActCommentPbServer\CommentObjServant',
        'home-class' => '\Server\impl\ActCommentServerImpl',
        'protocolName' => 'pb', //http, json, tars or other
        'serverType' => 'grpc', //http(no_tars default), websocket, tcp(tars default), udp
    ],
];
```
home-api指的是interface文件对应的namespace名称和类名
home-class指的是实现文件对应的namespace名称和类名

10. composer install,加载对应的依赖包

11. 在src下新建conf目录存储配置,默认为ENVConf.php

12. 在 ActCommentServerImpl 实现业务逻辑

13. 如果需要进行其他的tars服务调用,请参考tars-client模块的文档

14. 完成代码开发后,在src目录下执行 composer run-script deploy 会自动进行代码打包

15. 将打包好的代码,上传到tars平台,并进行发布

16. 调用 client.php 访问一下
