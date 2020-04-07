---
title: Empire源码分析
date: 2020-04-04 12:26:57
tags: Malware
---

一直在思考远控应该怎么设计，远控的源码究竟是什么样的。这次我会对Empire这个优秀的开源后渗透框架的源码进行分析，去挖掘这个框架背后的设计方法和原理。我想，分析完这个框架之后，我们就能够借鉴其思想，自己来实现一个远控程序来。

由于一些原因，这些分析只能在下班时间来写。对于这个框架，我打算分五篇文章写完，目录结构如下：

1. Empire整体结构 & 程序入口
2. Stager & Listener & Agent
3. 数据流通 & 数据加密
4. 插件 & 扩展性
5. 第三方库们

今天先开始第一篇，谈一谈Empire的目录结构和入口文件。从Github获取源码，只显示了二级目录并去除了一些安装部署相关的文件后，比较重要的几个目录结构如下：

```
Empire

├── data                     data目录用于存放静态文件、模板、数据库等

│  ├── agent

│  ├── empire-chain.pem

│  ├── empire-priv.key

│  ├── empire.db                 Empire使用了sqlite数据库存储

│  ├── misc

│  ├── module_source

│  ├── obfuscated_module_source

│  └── profiles

├── empire                    主程序入口，python文件

├── lib

│  ├── common

│  ├── listeners                 放置了不同的listener

│  ├── modules                  放置了各种payload，后渗透功能相关

│  ├── powershell                放置Invoke-Obfuscation项目文件，用于混淆powershell

│  └── stagers                  放置了各种平台下的stager

└── plugins                    放置了插件示例文件

```

可以看见项目文件还是比较清晰的，由于当前远控多数情况还是被控端主动连接到控制端的，为了区分方便，在下文中我会将被控端称之为Client，将控制端称为Server。

我们先来分析Server端，看一下这个项目是怎样运行的。首先关注一下Empire/empire这个文件，这是整个程序的入口。在逐渐阅读了Empire项目源码之后，我比较惊讶的是这个后渗透框架的本质其实是一个Flask Web App。

程序在最开始定义了一些数据库连接和查询相关的函数，这里的数据库使用的是sqlite。为了给RESTFUL API鉴权，这里还给出了生成token的函数。

```python
def refresh_api_token(conn):

"""

Generates a randomized RESTful API token and updates the value

in the config stored in the backend database.

"""

# generate a randomized API token

apiToken = ''.join(random.choice(string.ascii_lowercase + string.digits) for x in range(40))

execute_db_query(conn, "UPDATE config SET api_current_token=?", [apiToken])

return apiToken
```

岔个话题，这种写法还是蛮pythonic的，直接用了random.choice函数来随机选择，而且还使用了类似列表推导的语法

之后开始的start_restful_api()函数则是使用Flask框架来注册了一堆路由，用于RESTFUL API。有人可能会好奇这玩意是做什么用的，其实在EmpireProject的github账号中已经创建了Empire GUI客户端的项目，应该是用于客户端与服务端的交互过程，或者是留了接口能够以后被第三方工具调用

```python
####################################################################

#

# The Empire RESTful API.

#

# Adapted from http://blog.miguelgrinberg.com/post/designing-a-restful-api-with-python-and-flask

# example code at https://gist.github.com/miguelgrinberg/5614326

#

# Verb URI Action

# ---- --- ------

# GET http://localhost:1337/api/version return the current Empire version

#

# GET http://localhost:1337/api/config return the current default config


# GET http://localhost:1337/api/stagers return all current stagers

# GET http://localhost:1337/api/stagers/X return the stager with name X

# POST http://localhost:1337/api/stagers generate a stager given supplied options (need to implement)

# ...
```

官方已经给了很详细的注释，可以直接去读101行开始的源码，这一部分不再详细说明

从1362行来到了’__main__’，进入真正的逻辑，这里取了一些参数，我们只看最普通的执行情况，只有很简单的几句：

```python
else:

  \# normal execution

  main = empire.MainMenu(args=args)

  main.cmdloop()
```

进入了empire模块中MainMenu类的cmdloop()函数，这个empire模块才是框架比较核心的部分，之前的只是入口

进入Empire/lib/common/empire.py文件继续阅读

```python

\# custom exceptions used for nested menu navigation

class NavMain(Exception):

"""

Custom exception class used to navigate to the 'main' menu.

"""

   pass

```

一上来定义了几个类用于异常处理，其实这个是在菜单跳转中使用的。真正重要的是之后定义的几个大的类，MainMenu, SubMenu, AgentsMenu, AgentMenu, PowerShellAgentMenu, PythonAgentMenu, ListenersMenu, ListenerMenu, ModuleMenu, StagerMenu，下面一个一个讲

MainMenu是最核心的控制部分，程序启动后会首先进入主菜单。Empire菜单的控制逻辑其实不全是自己写的，而是继承了cmd模块中的Cmd类，这个类的详情可以看[这里](https://wiki.python.org/moin/CmdModule)，简要来说的话就是提供了写命令行应用的一些很方便的特性，比如能够自定义命令和语法、整体读取命令和返回结果的循环、TAB键的语法补全，我们在MainMenu类中看到形如do_xxx的语法均为定义指令，help_xxx的语法均为定义帮助命令，complete_xxx的语法均为处理TAB补全相关

在写远控框架的时候另一个比较重要的问题是，当我们发送的命令得到Client回复时，Server如何获知这个消息。Empire在这个问题上给出的答案是使用dispatcher模块。这个模块是生产者-消费者模式的一种实现，详情可以见[http://pydispatcher.sourceforge.net](http://pydispatcher.sourceforge.net/) 

```
# set up the event handling system

dispatcher.connect(self.handle_event, sender=dispatcher.Any)

```
77行这里指定了MainMenu的事件处理函数，这里对任何sender发出的信号都会接收并处理，我们去handle_event函数看一下

```python
def handle_event(self, signal, sender):

​    """

​    Whenver an event is received from the dispatcher, log it to the DB,

​    decide whether it should be printed, and if so, print it.

​    If self.args.debug, also log all events to a file.

​    """

​    \# load up the signal so we can inspect it

​    try:

​      signal_data = json.loads(signal)

​    except ValueError:

​      print(helpers.color("[!] Error: bad signal recieved {} from sender {}".format(signal, sender)))

​      return

​    \# this should probably be set in the event itself but we can check

​    \# here (and for most the time difference won't matter so it's fine)

​    if 'timestamp' not in signal_data:

​      signal_data['timestamp'] = helpers.get_datetime()

​    \# if this is related to a task, set task_id; this is its own column in

​    \# the DB (else the column will be set to None/null)

​    task_id = None

​    if 'task_id' in signal_data:

​      task_id = signal_data['task_id']

​    if 'event_type' in signal_data:

​      event_type = signal_data['event_type']

​    else:

​      event_type = 'dispatched_event'

​    event_data = json.dumps({'signal': signal_data, 'sender': sender})

...

```
注释写的挺清楚，收到信号时会记录到Database，并根据signal中的选项决定是否打印数据，在不同的菜单中显示不同的数据类型的相关逻辑就是这样控制的，除此之外，dispatcher也是listener与主控程序的通信方式。之前提到过，empire本质上是一个web应用，这个web应用是被主控程序启动的，主控程序与web应用这种无状态应用之间其实是存在一种隔离的，empire使用dispatcher机制来突破了这种限制（我原以为会是通过数据库读写，其实并没有），既然提到这里，我们也看一眼listener的相关实现

```python
\# lib/listeners/http_com.py

@app.before_request

def check_ip():

   """

   Before every request, check if the IP address is allowed.

   """

   if not self.mainMenu.agents.is_ip_allowed(request.remote_addr):

​     listenerName = self.options['Name']['Value']

​     message = "[!] {} on the blacklist/not on the whitelist requested resource".format(request.remote_addr)

​     signal = json.dumps({

​       'print': True,

​       'message': message

​     })

   dispatcher.send(signal, sender="listeners/http_com/{}".format(listenerName))

   return make_response(self.default_response(), 404)
```

其实这里价值在于观察程序的启动方式，观察菜单跳转和命令都是怎么实现的，看完这里我们完全也可以自己做一个相似的CC出来

菜单跳转这里是通过raise Exception实现的，在主菜单中永续循环读取命令，判断需要显示的菜单，如果需要跳转到agents/listeners等菜单会抛出一个异常，修改需要现实菜单的变量，主菜单捕获相应异常，根据变量值实例化一个新的菜单对象，然后再开始新的菜单对象的cmdloop()函数

第一章到此结束。先简单写一下逻辑结构，至此我们已经明确了empire的菜单实现，明确了菜单之间的跳转是怎么做到的，也明确了empire交互性良好的REPL模式命令行构建的秘密，从下一章开始我会将重点放在CC真正重要的部分。