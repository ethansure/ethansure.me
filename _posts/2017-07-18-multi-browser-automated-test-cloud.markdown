---
layout: post
title: "多浏览器自动化测试(2)-云服务测试"
author: "ethansure"
date: "2017-07-18 18:00:00"
comment: true
header-img: "img/post-bg-01.jpg"
---

在上一篇文章中，撸主已手把手教大家如何从零开始构建一个本地自动化测试工程。如果你没有看过上一篇文章，请先逐字阅读《[从入门到不放弃]多浏览器的自动化测试(1)-本地测试》。本文将在上一篇文章的基础上主要为大家介绍两个内容：一是如何免费地搭建多机的自动化测试环境，二是如何使用云测试服务进行360度无死角的自动化测试。信息量大，请各位阅后勿焚，动手牢记。

## 本地测试鞭长莫及

由于一台计算机支持的浏览器种类有限，如一台 mac 上可以安装 safari, chrome, firefox, opera 等，而且通常只能安装一个版本的产品，所以本地测试多用于检验功能逻辑是否正确，或者是检验特定浏览器的特定功能。对于未知的兼容性测试，单凭本地测试是没法进行的。下文中介绍的方法将提供给测试者一种全新的测试体验，通过远程测试的方式对自己的代码进行测试。

远程测试需要搞清楚两个概念，一是客户端 (Client)，一是服务端 (Server)，Client 是用于运行 test cases 代码的地方，Server 则是浏览器所在地。通过 Server 上的一些 servlet 来连接 Client 和 Server 上的浏览器，实现将 test 中的用例行为在远程端的浏览器上执行。 通过浏览器和 test 执行宿主机的分离，使得test能在更多的浏览器上执行，并且更易于扩展测试浏览器的数量。在下文的实践当中，读者会对 Client 和 Server 有更清楚的了解，在此不再赘述。

## 自己的云测试环境

![图1](http://cdn.ethansure.me/17-7-17/34594448.jpg)

既然测试代码要和浏览器环境分割开来，那么我们需要在前文的基础上将浏览器安装到其他的环境中，而不是将浏览器和测试的 Node 测试环境放在同一台机子。安装完成之后需要使用服务端的 Servlet 也就是 Selenium 提供的 webdriver server 将测试环境和浏览器连接起来。具体的步骤如下：

1. 寻找到一台可用的主机： 无论是实体机还是虚拟机都是可以的，不过需要主机可以接入到测试运行主机的网络。
2. 在主机上安装浏览器： 具体安装的浏览器类型和版本根据操作系统和测试需求而定， 例如可以在 windows 操作系统上安装 IE， firefox等浏览器，在 Linux 系统安装 chrome, firefox等浏览器， 在 Mac系统上安装 safari, chrome 等浏览器。
3. 下载对应浏览器的 driver 到Server主机上。因为 selenium 需要使用不同的 driver 来启动不同的浏览器，如同上一篇文章提到的bin目录下的 driver 可执行文件，此时要将需要测试浏览器对应的 driver 下载到 server 上，然后再通过测试工程的配置告诉 selenium-server-standalone 这些 driver 在哪，从而执行它们来操作浏览器。
 * chromedriver (用于 chrome)下载地址: https://sites.google.com/a/chromium.org/chromedriver/
 * geckodriver (可用于 firefox, safari)下载地址:https://github.com/mozilla/geckodriver/releases

4. 在主机上下载并启动 Selenium Server: 该 Server 实际上是一个 Java 小程序，用于 client 和 server 之间的通信（有关 selenium 原理的文章请关注《搞不懂不甘心》系列）。首先在 Selenium 的官网上下载 selenium-server-standalone-{VERSION}.jar， 然后启动该 Jar 包。

```
java -jar selenium-server-standalone-{VERSION}.jar
```
如果主机没有安装 JRE, 则需要再安装 java 的运行环境或者是直接安装 jdk 。

5. 修改测试项目的配置文件：还记得启动测试时需要指定的配置文件吗？这个配置文件 test.conf.js 非常重要，用于配置 selenium 以及测试的浏览器，当我们改变使用远程server的浏览器作为测试目标时，当然需要修改配置文件。我们需要将配置文件中的 selenium 项修改为如下形式：

```
selenium : {
    "start_process" : true,

    //server的ip地址
    "host" : "192.168.10.1",

    "port" : 4444,

    "cli_args": {

      //chromedriver 在server主机上的文件路径
      "webdriver.chrome.driver": "/home/bin/chromedriver",

      //geckodriver 在server主机上的文件路径
      "webdriver.gecko.driver" : "/home/bin/geckodriver"
    }
  }
```

对于test_settings的设置请参照上文，然后按照自己安装的浏览器版本进行修改。
6. 启动测试：一切准备好了之后，在client主机上，也就是测试代码运行的机子上便可启动测试。

```
"scripts": {
  ...
  "test": "./node_modules/.bin/nightwatch -c conf/test.conf.js -e A,B"
  ...
}
```

自己搭建测试云环境的过程其实并不复杂，只需要在将 selenium server 和浏览器安装到其他主机即可，对于 client 上的代码不需要改动，只需要改动配置中的 selenium 配置。但是很快测试者会发现，当我们需要测试更多的机子，用手工的方式去维护这些 server 是一件费时费力的事，也消耗了公司的计算资源。有没有更好的办法让我们既可以全面的测试自己的代码又可以不用费尽心思维护主机？答案是有，请继续阅读。

## 云测试服务
对于繁琐重复的工程任务，商家们总是能想到赚钱的办法，这不对于上文我们碰到的麻烦就有商家提供了相应的产品。该产品为测试者们提供无数个测试浏览器，测试者不需要关心这些浏览器在何处运行，应该怎么样维护，只需要一个服务地址便可以将自己的测试页面跑在这些浏览器上，其实这个服务地址和之前我们自己搭建的 Server ip 类似，只不过如果使用自己的测试云，使用不同的测试主机时，需要手动更改host。而这些商家提供了一个类似分销中心，用于流量分发，所以我们只需要用一个地址便可实现使用不同的主机进行测试。

目前提供此类服务的商家有很多，如 browserstack、saucelabs、crossbrowsertesting 等，大家可以根据自己手头黄金和测试的需要选择性价比高的服务。本文将使用 browserstack 作为例子为大家科普此类服务，不过它并不是撸主的金钱爸爸，请大家放下水文的猜疑。

![图2](http://cdn.ethansure.me/17-7-17/3776665.jpg)

根据我们自行搭建云测试环境的经验，我们将 browserstack 的测试后台架构猜想为下图所示。我们不关心该架构是否是真实的实现，但是这是合理的理论猜想，希望此图能让我们对此服务有个大概的技术了解：

![图3](http://cdn.ethansure.me/17-7-17/7066742.jpg)

browserstack 为用户提供了自动化测试、实时交互测试、截图等服务，关于具体的服务细节请移步官网。本节将主要介绍如何使用其自动化测试服务，会稍微提及实时交互测试的功能。那接下来便开始我们的云测试使用体验：

首先在其[网站](https://www.browserstack.com)上注册账号，点击最上方的导航栏中的 Automate，跳转页面后在新页面左侧最上方点击 ”Username and Access Keys”，便可看到用于使用云测试服务的用户名和key，我们将使用此auth来修改测试配置。

现在回到我们的测试项目，对 test.conf.js 的 selenium 项进行修改，并添加 common_capabilities 项，用于配置云服务的信息。

```
  selenium : {
    "start_process" : false,
    "host" : "hub-cloud.browserstack.com",
    "port" : 80
  },

  common_capabilities: {
    'build': 'nightwatch-browserstack',
    // Browserstack 的 username 对应配置项
    'browserstack.user': process.env.BROWSERSTACK_USERNAME',
    // Browserstack 的 key 对应配置项
    'browserstack.key': process.env.BROWSERSTACK_ACCESS_KEY,
    'browserstack.debug': true,
    'browserstack.local': true
  }
```

连接云测试服务的配置工作完成后，我们需要指定测试的浏览器种类和版本。如果有不指定的字段，云服务会有缺省值来填充，例如配置中没有指定操作系统，云服务则会自动选择最快的一个测试机，而不管浏览器所在的操作系统。再例如当没指定测试浏览器的版本时，云服务则会测试最新版本的浏览器。官网上的文档提供了所有可提供测试的浏览器种类和版本，为了说明方便，我们现在只指定浏览器种类，不规定版本。简要的浏览器配置项如下：

```
    ...
    safari: {
      desiredCapabilities: {
        browserName: 'safari'
      }
    },
    ie: {
      desiredCapabilities: {
        browserName: 'ie'
      }
    },
    ...
    ios: {
      desiredCapabilities: {
        browserName: 'iPhone'
      }
    }
    ...
  }
```

以上工作做完之后便可以启动测试了，是不是so easy。除了命令行返回的测试结果之外，browsertack 自动化测试还为我们提供了测试回放等。如果发现测试出错，可以通过商家提供的在线实时测试来进行调试，这也是一个非常方便的功能。

![图4](http://cdn.ethansure.me/17-7-17/38411086.jpg)

## 有的放矢地测试
阅读完自动化测试的文章，相信大家已经迫不及待想体验云测试的便利。在各位动手之前，有一些温馨提示需要告知大家。首先这些云测试服务因为由国外服务商提供，所以网络延时有些时候会过高，测试可能会出现超时的情况，请选择网络较好的主机来运行测试用例。其次是因为自动化测试会让大家写测试用例上瘾，反正测试扔上去测就好，但是撸主认为测试人员还是要清楚地划分测试的粒度，有些测试用例比如细粒度的单元测试和端对端的测试，有很多测试覆盖的都是同样的代码，这样的测试其实是浪费的，所以在明确目标之后，还需要精心设计测试用例。最后如有不懂请先 google，其他不能 google 的问题欢迎和撸主交流，文章若有错请指教。
