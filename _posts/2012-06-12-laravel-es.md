---
layout: post
title: "Laravel Scout，Elasticsearch，ik 全文搜索"
date: 2018-06-12
excerpt: "Laravel Scout，Elasticsearch，ik 全文搜索"
tags: [技术文档]
comments: false
feature: http://i.imgur.com/Ds6S7lJ.png
---

## Java 环境安装

1. 检查是否已安装

   ~~~ sh
   java -version
   ~~~

1. [oracle](http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html) 官网下载 jdk 安装包(大于 1.8 版本)，我下载的是 jdk-8u171-linux-x64.tar.gz

2. 在 /usr/ 目录下创建 java 目录，将安装包放在该目录下，解压

   ~~~ sh
   mkdir /user/java
   cd /usr/java
   tar -zxvf jdk-8u171-linux-x64.tar.gz
   ~~~

3. 设置环境变量

   ~~~ sh
   vi /etc/profile
   ~~~

   添加以下内容

   ~~~ sh
   #set java environment
   JAVA_HOME=/usr/java/jdk1.7.0_79
   JRE_HOME=/usr/java/jdk1.7.0_79/jre
   CLASS_PATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar:$JRE_HOME/lib
   PATH=$PATH:$JAVA_HOME/bin:$JRE_HOME/bin
   export JAVA_HOME JRE_HOME CLASS_PATH PATH
   ~~~

   修改生效

   ~~~ sh
   source /etc/profile
   ~~~

## elasticsearch-rtf 安装

因为我们要使用 ik 插件，所以我们直接使用项目：[https://github.com/medcl/elasticsearch-rtf](https://github.com/medcl/elasticsearch-rtf) ，当前的版本是 Elasticsearch 5.1.1，ik 插件也是直接自带了。

### 安装过程

1. 下载项目

   ~~~ sh
   git clone git://github.com/medcl/elasticsearch-rtf.git elasticsearch
   ~~~

2. 修改 `elasticsearch/config/elasticsearch.yml/elasticsearch.yml` 文件，将 `network.host` 改成 `0.0.0.0`

3. 启动服务

   ~~~ sh
   cd elasticsearch-rtf/bin
   # 启动 elasticsearch 服务
   ./elasticsearch
   ~~~

4. 测试

   ~~~ sh
   curl http://127.0.0.1:9200
   # 显示以下内容表示启动完成
   {
     "name" : "Rkx3vzo",
     "cluster_name" : "elasticsearch",
     "cluster_uuid" : "Ww9KIfqSRA-9qnmj1TcnHQ",
     "version" : {
       "number" : "5.1.1",
       "build_hash" : "5395e21",
       "build_date" : "2016-12-06T12:36:15.409Z",
       "build_snapshot" : false,
       "lucene_version" : "6.3.0"
     },
     "tagline" : "You Know, for Search"
   }
   ~~~

   ​

---

### 可能错误

- 由于elasticsearch5.0默认分配jvm空间大小为2g，修改jvm空间分配

  ~~~ sh
  # 报错信息如下
  Java HotSpot(TM) 64-Bit Server VM warning: INFO: os::commit_memory(0x000000008a660000, 1973026816, 0) failed; error='Cannot allocate memory' (errno=12)
  ~~~

  解决方案：

  ~~~ sh
  vim config/jvm.options
  ~~~
  添加内容

  ~~~
  -Xms512m
  -Xmx512m
  ~~~

- 因安全因素，不能在root用户下运行

  ~~~ sh
  # 报错信息如下
  java.lang.RuntimeException: can not run elasticsearch as root
  ~~~

  解决方案：切换到非 root 用户，sudo 启动

- max_map_count 原因

  ~~~ sh
  # 报错信息如下
  max virtual memory areas vm.max_map_count [65530] is too low
  ~~~

  解决方案：

  ~~~ sh
  # 方法 1 ，执行命令
  sudo sysctl -w vm.max_map_count=262144

  # 方法 2，修改 sysctl.conf 文件
  vim /etc/sysctl.conf
  ## 添加内容
  vm.max_map_count=262144
  ~~~


## Laravel 项目使用

### 在 Homestead 环境下使用 elasticsearch-rtf

我本地使用的是 Homestead 环境进行安装测试，需要进行端口映射配置。修改 `Homestead/Homestead.yml` 文件，添加以下代码：

~~~ yaml
ports:
    - send: 62000
      to: 9200
      protocol: udp
~~~

然后重启 Homestead。如果不生效，也可以修改 `Homestead/scripts/homestead.rb`文件，在文件前添加以下代码：

~~~
config.vm.network "forwarded_port", guest: 9200, host: 62000, auto_correct: true
~~~
### 安装 ElasticSearch Scout Engine 包和配置

1. 下载

   ~~~ sh
   composer require tamayo/laravel-scout-elastic
   ~~~


2. 生成配置

   ~~~ sh
   php artisan vendor:publish --provider="Laravel\Scout\ScoutServiceProvider"
   ~~~


3. `config/app.php` 添加对应的 ServiceProvider

   ~~~ php
   <?php
       return [
       // ...
      'providers' => [
        // ...
        Laravel\Scout\ScoutServiceProvider::class
        ScoutEngines\Elasticsearch\ElasticsearchProvider::class
      ],
       // ...
     ];
   ~~~


4. 修改生成的 `config/scout.php`

   ~~~ php
   'driver' => env('SCOUT_DRIVER', 'elasticsearch') // 默认 elasticsearch
   // 加上 elasticsearch 初始配置
   'elasticsearch' => [
       'index' => env('ELASTICSEARCH_INDEX', 'laravel_search'),
       'hosts' => [
           env('ELASTICSEARCH_HOST', 'http://127.0.0.1:9200'),
       ],
   ],
   ~~~


5. 添加 `.env` 环境变量

   ~~~ makefile
   # 我使用 Homestead 环境，配置如下
   ELASTICSEARCH_HOST=http://192.168.10.10:9200
   ~~~

### 初始化 ES

添加 InitEs 命令，初始化 ES 的一些数据

~~~ sh
php artisan make:command InitEs
~~~

InitEs.php 代码如下，主要做了两件事情：

- 创建对应的 index 索引
- 创建一个 template，你可以通过 [这里](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-templates.html)了解一下什么是 Index template

~~~ php
<?php

namespace App\Console\Commands;

use GuzzleHttp\Client;
use Illuminate\Console\Command;

class InitEs extends Command
{
    /**
     * The name and signature of the console command.
     *
     * @var string
     */
    protected $signature = 'es:init';

    /**
     * The console command description.
     *
     * @var string
     */
    protected $description = 'Init es to create index';

    /**
     * Create a new command instance.
     *
     */
    public function __construct()
    {
        parent::__construct();
    }

    /**
     * Execute the console command.
     *
     * @return mixed
     */
    public function handle()
    {
        $client = new Client();
        $this->createTemplate($client);
        $this->createIndex($client);
    }

    protected function createIndex(Client $client)
    {
        $url = config('scout.elasticsearch.hosts')[0] . ':9200/' . config('scout.elasticsearch.index');
        $client->put($url, [
            'json' => [
                'settings' => [
                    'refresh_interval' => '5s',
                    'number_of_shards' => 1,
                    'number_of_replicas' => 0,
                ],
                'mappings' => [
                    '_default_' => [
                        '_all' => [
                            'enabled' => false
                        ]
                    ]
                ]
            ]
        ]);
    }

    protected function createTemplate(Client $client)
    {
        $url = config('scout.elasticsearch.hosts')[0] . ':9200/' . '_template/rtf';
        $client->put($url, [
            'json' => [
                'template' => '*',
                'settings' => [
                    'number_of_shards' => 1
                ],
                'mappings' => [
                    '_default_' => [
                        '_all' => [
                            'enabled' => true
                        ],
                        'dynamic_templates' => [
                            [
                                'strings' => [
                                    'match_mapping_type' => 'string',
                                    'mapping' => [
                                        'type' => 'text',
                                        'analyzer' => 'ik_smart',
                                        'ignore_above' => 256,
                                        'fields' => [
                                            'keyword' => [
                                                'type' => 'keyword'
                                            ]
                                        ]
                                    ]
                                ]
                            ]
                        ]
                    ]
                ]
            ]
        ]);

    }

~~~


### 创建模型并填充数据

1. Model 修改

   ~~~ php
   <?php

   namespace App;

   use Illuminate\Database\Eloquent\Model;
   use Laravel\Scout\Searchable;

   class Demo extends Model
   {
       use Searchable;
       protected $table = 'demos';

       /**
        * 索引名称
        *
        * @return string
        */
       public function searchableAs()
       {
           return 'students_index';
       }

       /**
        * 可搜索的数据索引
        *
        * @return array
        */
       public function toSearchableArray()
       {
        // $array = $this->toArray();
           return [
               'title' => $this->title,
               'content' => $this->content
           ];
       }
   }
   ~~~


2. 把该表现有记录导入到搜索索引中

   ~~~ php
   php artisan scout:import "App\Demo"
   # 正在导入
   Imported [App\Demo] models up to ID: 500
   Imported [App\Demo] models up to ID: 1000
   Imported [App\Demo] models up to ID: 1500
   Imported [App\Demo] models up to ID: 2000
   Imported [App\Demo] models up to ID: 2500
   Imported [App\Demo] models up to ID: 3000
   Imported [App\Demo] models up to ID: 3500
   All [App\Demo] records have been imported.
   ~~~

### 使用 ES 进行检索

~~~ php
$re = App\Demo::search('测试')->get();
~~~

## 更多链接

- [Elasticsearch 官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html)
- [Laravel5.5 Scout 使用文档](https://laravel-china.org/docs/laravel/5.5/scout/1346)
