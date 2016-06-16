# Java项目如何与Node.js共存？

很多人都有Java背景，对于Node.js他们大多是观望，学习，可能还不是特别清楚，如果让Java项目和Node.js结合使用？

本文主要讲解的是这个问题，通过各种比较，最后给出技术栈升级的建议。

![Java](Java.png)

## 看一下Java的前后端定位

Java是传统的老牌后端，虽然它也可以jsp类的写点前端的东西，JavaEE里从model 1到model 2的差别就在于视图和控制器分离，引出jsp，再也不用servlet管全部了。

model 1

![1](1.png)

model 2

![2](2.png)

一般我们把WebRoot目录下面的理解为是前端，非WebRoot理解成后端。

烂大街的三大框架

![3](3.png)

这里又不是十分严格，比如后端给jsp传值，然后渲染，这时候前后端还是耦合在一起的，还不是严格意义上的前后端分离。

所以，我们彻底点，让后端只提供api，把webroot里面的api请求使用ajax替换。

app拆分成了2部分

- 后端 java api
- 前端，jsp + ajax

在彻底一点，把它们分别部署服务器上，比如tomcat

- tomcat 1（端口8080） 部署后端api
- tomcat 2（端口8081）  部署前端，jsp + ajax

![4](4.png)

这样就分开，但访问的时候可能会有跨域问题，于是我们还得把它们合并到一个域名里面，那就凑合nginx吧


```
  location / {
      proxy_redirect off;
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
	    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header Host $http_host;
	    proxy_set_header X-NginX-Proxy true;
	    proxy_pass http://127.0.0.1:8080;
      proxy_redirect off;
  } 
  
  # Load configuration files for the default server block.
  # include /etc/nginx/default.d/*.conf;
  # Load configuration files for the default server block.
  include /etc/nginx/default.d/*.conf;

  location /api {
      proxy_set_header X-Real-IP $remote_addr;
	    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header Host $http_host;
	    proxy_set_header X-NginX-Proxy true;
	    proxy_pass http://127.0.0.1:8081;
      proxy_redirect off;
  }
```

此时访问，所有的都放到http://127.0.0.1:80端口下面，是不是就完美解决了？

![5](5.png)

## 前端优化

前端，我们还能做什么优化呢？

> 把那些js和css、图片压缩，丢到cdn上

- 研究浏览器机制，提高渲染速度
- 模块化加载
- profile
- 各种框架，angularjs，react，具体看我写的前端4阶段吧
- 各种变态预处理写法

## 后端优化

默认是使用json api风格

### API 标准写法

摘抄：http://www.startupcto.com/backend-tech/building-an-api-best-practices


You'll generally want to wrap all your API responses in an 'envelope' which specifies metadata about the APIcall.

```
// sample JSON envelope
{
  "status": {
    "code": 10000,
    "message": 'Success'
  },
  "response": {
     ...results...
  }
}
```

Doing this allows for client handler code to behave the same way for all API calls, since it gets a responses back in a universal format.

语义上再好一点，推荐：

```
// sample JSON envelope
{
  "status": {
    "code": 10000,
    "message": 'Success'
  },
  "data": {
     ...results...
  }
}
```

可视化编辑校验： http://jsoneditoronline.org/


当然restful 风格api也是非常好的。

### 使用rpc拆分服务

比较好的做法是http api调用rpc，提供最终api

- 单一调用，简单接口
- 多个调用，可以封装成上层服务，也可以组合用

rpc框架，如dubbo，dubbo-x，motan，grpc等

![6](6.png)

### rpc拆分后，拆分db

一般拆分rpc都会按照某些业务主体划分，所以数据库是可以拆分出去

比如

- 库存
- 订单
- 评论
- 弹幕

其实，只要保证用户一致，其他数据库保存各自的数据就好。在数据分析的时候，汇总到一起即可，无论etl还是stream都可以。

服务和db拆分的好处是便于横向扩展，这样就可以做到动态伸缩，哪里有瓶颈治哪里，在架构调优上是有明显优势的。

### 服务多，就要服务治理、发现

采用consul作为服务发现软件

![7](7.png)

### 服务器监控apm

![9](9.png)

这是java写的

https://github.com/naver/pinpoint

### api多了，怎么办呢？

都是重复的，如日志、权限等，这时，我们需要api Gateway。

https://getkong.org/

通过nginx + lua实现，提供插件机制，非常好用。

![8](8.png)

### 容器化，自动化运维

![10](10.png)

- docker
- saltstack

### 总结

架构是相同的，其实语言是无所谓的。所以这套东西也适合Node.js哦。

## 看一下Node.js

首先，Node.js是运行在v8之上的js 运行环境。语法上和js同出一辙，差别在于，它可以进行更多其他后端语言完成的功能，如io、网络等。

一般，大前端指的是前端 + Node.js这样的组合，这时的Node.js仅作前端工具使用。

![F0](f0.png)

也就是说js才是混淆点

- 前端有js
- Node也是js语法

![F1](f1.png)

那再俗一点

- 前端有js（Node.js作前端工具）
- 后端 Node.js (express、koa等类似于javaEE的servlet)

![F2](f2.png)

这样是不是就清晰多了？

好了，我们在来对比Java的部分来说

- 后端，java api
- 前端，jsp + ajax

那么，结论应该是

- jsp + ajax = Node.js + ajax
- java api = Node.js api

java接口和jsp都是跑在tomcat这样的JavaEE服务器上的，而Node.js无论是jade/ejs还是用Node.js写api接口，它们都是Node.js自身的服务器的。

这样对比的目的是为了让大家情况它们的差别。

![B0](b0.png)

下面我们就看一下，组合吧

![B1](b1.png)

- Java api 做后端
- Node.js jade/ejs + ajax 作为前端（moa-frontend就是代表）

这是比较常见，很多叫这种方位proxy，这里面设计的问题就很明显了，ajax能直接调用java api，还有复杂的业务是无法java接口直接搞定的呢？

于是各种hack，比如采用 https://github.com/request/request ，但是这是http请求啊，http协议还是比较重的，每次请求至少50ms，如果多几个请求，就算加上缓存也是非常难受的，那么怎么办呢？

想想我们上面说的rpc吧，在express里不调用request，换成rpc是比较靠谱的，那么问题又来了，既要支持java又要支持node，rpc框架最好是支持多语言的，这时grpc就成了比较好的选择了，不然就一言不合自己写一个？哈哈

下面来个不长见的技术栈

![B2](b2.png)

- jsp + ajax 作为前端
- Node.js api 做后端

这其实也是可以，只是有点多余而已，对于熟悉jsp的人未尝不是好事呢。

其实Node可以做的更多的

## Modern Web Development的四个维度

![](overview.png)

### 使用预编译

- 模板引擎
- css预处理器: less|stylus|compass|sass
- js友好语言: coffee、typescript

### 前端开发4阶段

- html/css/js（基础）
- jQuery、jQuery-ui，Extjs（曾经流行）
- Backbone（mvc），Angularjs（当前流行）
- React组件化、Vuejs（未来趋势）

Vuejs综合Angular和React的优点，应该是下一个流行趋势

### 构建工具

- grunt
- gulp
- webpack
- npm scripts

### 平台工具

跨平台覆盖，从各种浏览器到移动端、pc客户端

- 移动端 cordova
- pc客户端 electron

## 总结

讲了java和Node的定位，总结了它们的组合可能，以期读者能够明白各位能做什么，合适做什么，同时有一些比较好的实践，最后给出Node.js能做的更好的现代web开发，希望对大家有所帮助。

对于上新的技术栈，我一向是保守的，一次只上一样新的，小步快走。比如java先做前后端分离，然后再考虑把前端用node替换，然后rpc改造，甚至极端点后端也可以使用Node.js做。


## 招聘

这篇文章本来之前就想写了，一直比较忙，没有得空，正好有朋友招聘Java大牛工程师，所以就有了此篇，希望对大家有用



职位一：J2ee系统架构师    薪资15000-20000

工作地：上海

1. 负责收集并分析客户的各项需求，包括业务领域和技术领域；
2. 识别相关的需求，并提供相应的技术解决方案（主要是J2EE技术领域）；
3. 进行客户应用系统的日常维护和支持，主要包括客户现场常驻开发、bug修订、功能完善等；
4. 建立良好的客户关系，获得客户对公司及个人的认可；
岗位要求：
1、有扎实的软件理论基础，深刻理解面向对象编程，良好的面向对象分析设计能力，熟悉开发模式、设计模式；
2、有5年以上大中型项目开发经验并持续至今，其中3年以上基于J2EE架构大型应用系统的模块设计开发经验；
3、熟悉JQ、Javascript、Html5、Css3、Ajax，能够熟练地综合运用以上技术开发动态页面
4、精通JAVA 语言，有良好的编程习惯，掌握Struts，Hibernate，Spring等开源框架原理；
5、具备ORACLE、MYSQL等大型数据库使用及设计、开发经验；
6、精通UML，熟练掌握UML设计工具，有良好文档编写习惯；
7、可承受较大的工作压力，有团队意识和钻研精神；


职位二：J2ee软件开发工程师   薪资10000-15000

工作地： 上海

1. 专科及以上学历，计算机软件或相关专业；
2. 具有三年以上的J2EE开发经验。熟悉Struts, Spring, Hibernate等框架；
3. 熟悉软件开发流程，熟悉需求分析、功能设计、技术设计、开发、测试、发布上线整个过程；
4. 具有一定的业务流程、功能的理解和分析能力，能够根据需求执行编码、测试等工作；
5. 熟悉常用的应用服务器的产品配置，例如Tomcat, WebSphere 或WebLogic；
6. 熟悉常用数据库的SQL语言，例如DB2, Oracle, SQL Server, Sybase ASE
7. 语言表达流畅、思维清晰、善于沟通，富有责任感，有良好团队协作能力；
8. 富有学习激情，具有较强学习能力，能够承受一定工作压力。


职位三：BIEE工程师    薪资10000-15000

工作地：上海

岗位职责：
1. 独立负责BIEE开发与维护；
2. 负责BIEE设计文档编写和规范制定；
3. 负责BIEE参数和性能调优；
4. 负责BIEE页面与数据中心；

任职资格：
1. 本科及以上学历，计算机相关专业，2年以上工作经验；
2. 熟悉掌握BIEE开发，具备独立开发经验；
3. 熟练掌握BIEE模型设计；
4. 熟悉掌握BIEE的布署、升级及运维；
5. 熟悉Linux、Weblogic集群；
6. 熟练使用Oracle数据库基本功能；
7. 熟悉概率和统计分析，会使用SPSS, SAS, R语言优先；
8. 有较强的学习能力，理解能力，沟通能力，表达能力；
9. 从事过金融行业者优先考虑。


公司简介

上海轶博信息技术有限公司，定位于服务金融行业、政府教育行业用户的专业信息化服务提供商，形成管理咨询、软件实施、软件开发、服务外包四个方面的核心竞争力。经过多年的经营和发展，上海轶博以管理咨询和软件开发为核心， 结合先进成熟的行业解决方案，提供专向业务咨询、业务流程梳理、软件开发，并通过与业内知名国内外企业的紧密合作，以先进理念给用户提供一流的业务咨询及实施服务。

上海轶博的核心成员来自于各大外资咨询公司和IT公司，轶博经过数年的发展，形成了互联网+、大数据、风险管控、人力资源、财务管理、公司协作等六大核心能力。全方面为客户提供从管理 咨询到系统落地的“端到端”解决方案。

公司网址：www.abigdata.com.cn

有感兴趣的朋友，可以联系我 i5ting@126.com

## 作者

桑世龙（网名i5ting），空弦科技 CTO，StuQ 明星讲师，开源项目 Moajs 作者，Node.js 技术布道者。
曾就职在新浪、网秦，曾做过前端、后端、数据分析、移动端负责人、首席架构师、技术总监，全栈技术实践者，目前主要关注技术架构和团队梯队建设方向。

![2.Pic](2.pic.jpg)
