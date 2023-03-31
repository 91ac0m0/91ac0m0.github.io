## MAR week4

路过的客官请进，听我唠嗑来了。周记周记，开设这个栏目的意图就是，多么闲闲碎碎、鸡毛蒜皮、香的辣的的东西都可以丢上来。

### 数日前——“急了急了”

事情要从 giacomo 同学着急忙慌向的跑向 web 手寻找帮助说起，她显然十分焦灼，先发了好多:sob: 才说明来意，然而磕磕绊绊言不达意，什么 ”mysql“”php“ 听得人云里雾里。好不容易，听者总算明白了 giacomo 需要在有限的时间写一个小网站，麻雀虽小可五脏俱全，giacomo 同学对于后端前端的认识还停留在她上辈子，当下时髦的新品对她而言是闻所未闻见所未见，”先干什么，后干什么，要学什么“是大难题。

web 手们仗义相助，说了一些 giacomo 不太懂，但安心了不少，可以着手去干了。

giacomo 同学本能逃避这艰难玩意，咕咕咕了几天终于开工。她选了她觉得相对容易的 flask（一个 python 框架，核心简单可扩展）、vue（用于构筑界面的 js 框架）进行学习，而且意外地发现原来 python 本身自带一个数据库 sqlite，难度又降低些许。 之后一周 giacomo 同学在这分清单上修修补补，一周后是这个模样：

![image-20230331022107075](https://s2.loli.net/2023/03/31/pHu3PlcgoiqzmBM.png)

虽然图中不免有错误的地方，这个过程中发生了什么呢？以下是她自述：

### flask 学习

过程中参考了浙大学霸的[博客](https://www.cnblogs.com/Here-is-SG/p/16737427.html)，非常有帮助。

#### vitualenv

虽然 win 的环境已经是一坨那啥了，但还是可以用一下 [virtualenv](https://www.cnblogs.com/cwp-bg/p/python.html)。

#### 运行应用

一个最简单的 hello world 应用是这样，通过几行命令就能让它运行起来

```python
# hello.py
from flask import Flask
app = Flask(__name__)

@app.route('/')
def hello_world():
    return 'Hello, World!'
```

在 win powershell 中输入

```bash
$env:FLASK_APP = "hello.py"  # 指定环境变量
$env:FLASK_DEBUG=1  # 调试
flask run
```

linux中

```bash
export FLASK_APP=app.py
flask run
```

加上一些路由、template、数据库就可以实现一个非常简单的小东西了。虽然看起来很简陋，但是加上各种插件和功能还可以实现不少东西。

#### flask-cors

flask-cors 插件用于解决跨域问题。[什么是跨域](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/CORS)，简单来说就是：出于安全考虑，浏览器的”同源策略“要求，一个页面请求的 url 有相同的协议、域名、端口，当url 的这些要素不同，称为跨域。在前后端未分离时，跨域问题比较少，如今前后端分离，前端常常有跨域请求。

全局配置cors：

```python
from flask import Flask
from flask_cors import CORS

app = Flask(__name__)
CORS(app)

@app.route("/")
def helloWorld():
  return "Hello, cross-origin-world!"
```

#### flask-sqlalchemy

是一个基于Python实现的`ORM`（对象关系映射）框架，说人话就是把table抽象成一个类，用“类和对象”操作数据库，这样一来数据库的操作就简单很多（python 库 sqlite3 也可以用，不过[复杂不少](https://zhuanlan.zhihu.com/p/407131061)）。（后来发现了一个更简单的，像操作 JSON 一样操作数据库的 [dataset](https://github.com/pudo/dataset)，下次试试）

##### models

models，即 class，可以单独定义在 models.py 文件当中，import 之后需要[设置上下文环境](https://stackoverflow.com/questions/73961938/flask-sqlalchemy-db-create-all-raises-runtimeerror-working-outside-of-applicat)

```python
#models.py

from flask import Flask
from flask_sqlalchemy import SQLAlchemy

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI']='sqlite:///databas.db' #会自动创建在 instance 文件夹下面
db = SQLAlchemy(app)

class student(db.Model):
    name = db.Column(db.String(80), primary_key = True)
    age = db.Column(db.Integer)
```

在命令行里测试一下

```bash
>>> from models import *
>>> app.app_context().push() #设置 context
>>> db.create_all()
>>> sb=student(name='sb',age=1)
>>> db.session.add(sb)
>>> db.session.commit()
```

这样就可以在数据库里看见了，使用 [DB browser for sqlite](https://sqlitebrowser.org/) 可以查看

![image-20230326120526326](https://s2.loli.net/2023/03/26/yBHm2soKxXLlrJZ.png) 

设置环境在 app.py 里可以这么写

```python
with app.app_context():
    db.create_all()
```

##### relationship

如果多个表之间需要建立关系，可以定义 relations。比如 user 和 challenge 是多对多关系（仿佛在数据库课堂的梦境中听过这个词），我们需要记录解除题目的用户避免重复答题，就可以这么写

```python
from flask import Flask
from flask_sqlalchemy import SQLAlchemy

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI']='sqlite:///databas.db'
db = SQLAlchemy(app)


user_chal = db.Table('user_chal',
                db.Column('challenge_id', db.Integer, db.ForeignKey('challenge.id')),
                db.Column('user_id', db.Integer, db.ForeignKey('user.id')))

class user(db.Model):
    id = db.Column(db.Integer, autoincrement=True, primary_key=True)
    username = db.Column(db.String(80))
    challenges = db.relationship('challenge', secondary=user_chal, back_populates='users')

class challenge(db.Model):
    id = db.Column(db.Integer, autoincrement=True, primary_key=True)
    answer = db.Column(db.String(80))
    users = db.relationship('user', secondary=user_chal, back_populates='challenges')

```

验证

```python
from models import *

app.app_context().push()
db.create_all()

u1=user(username='666')
u2=user(username='233')
c1=challenge(answer='?')
c2=challenge(answer='.')
c2.users.append(u1)
db.session.add(u1)
db.session.add(u2)
db.session.add(c1)
db.session.add(c2)
db.session.commit()

for i in c2.users:
	print(i.id)

for i in u1.challenges:
	print(i.id)
```

##### 语法

[column 参数](https://blog.csdn.net/weixin_44733660/article/details/104066220)

[增删改查](https://blog.csdn.net/qq_39147299/article/details/109495792)

#### gunicorn

WSGI 指定了 web 服务器和 Python web 应用之间的标准接口，如图：

![image-20230323094049093](https://s2.loli.net/2023/03/23/awPzsTrFMEUOBQn.png)

虽然 flask 有自带的WSGI，但只有一个 worker 响应所有 request，所以在生产环境可以用 gunicorn

> 在管理 worker 上，使用了 pre-fork 模型，即一个 master 进程管理多个 worker 进程，所有请求和响应均由 Worker 处理。Master 进程是一个简单的 loop, 监听 worker 不同进程信号并且作出响应。比如接受到 TTIN 提升 worker 数量，TTOU 降低运行 Worker 数量。如果 worker 挂了，发出 CHLD, 则重启失败的 worker, 同步的 Worker 一次处理一个请求。

启动方法：

```bash
apt install gunicorn

# demo.py path
gunicorn -b 0.0.0.0:8000 demo:app
```

[stop](https://cloud.tencent.com/developer/article/1493641)

```bash
pstree -ap|grep gunicorn

# restart
kill -HUP <主进程id>

# quit
kill -9 <主进程id>
```

### vue 学习

vite + vue3 + ts 偷抄了这个[博客](https://wbxyy.github.io/2021/07/15/learnVue3/)，不过云里雾里没太抄明白（）。vue 零零碎碎的东西不想写了，摆在我这个计网白痴面前的大问题是，前后端是如何通信的？

#### 如何与后端配合

前后端分离，网上的说法是后端提供接口，前端与接口交换数据并负责页面展示，但这样说有点抽象， [这篇文章](https://testdriven.io/blog/combine-flask-vue) 介绍了 flask + vue 如何整合的例子，讲的非常详细，一下子就明白了。我把代码改了一下，post 请求用 axois 实现。

在 main.js 中新增 axois 的配置

 ```js
import Vue from 'vue'
import App from './App.vue'
import axios from 'axios'   //新增

Vue.config.productionTip = false
Vue.prototype.$axios = axios    //新增

new Vue({
  render: h => h(App),
}).$mount('#app')

 ```

点击按钮执行函数

```vue
<template>
  <div id="app">
    <input v-model="post_msg.try">
    <button @click="post1">点击我</button> 
  </div>
</template>
```

发送 post 请求

```js
methods:{
      post1() {
        let data = this.post_msg
        this.$axios.post('http://127.0.0.1:5000/try_post', data).then(res=>{
        alert(res.data['msg']);
      });
      }
    }
```

然后分别在 backend 目录中 `flask run`，frontend 目录 `npm run serve`。这个过程就是访问 http://localhost:8080/ 看见前端页面，点击按钮，就会请求 http://localhost:5000/ 的数据。看到这里突然非常饿，忍不住开了一包汤达人罪恶一下。


 ![IMG20230326222329 (小)](https://s2.loli.net/2023/03/26/wbnPqsRF3EoDic8.jpg)

其实这里有点疑惑，直接把 api 写在 script 里面会不会不安全，[查了一下](https://www.zhihu.com/question/347765070)，关键不是不让 api 被人发现，而是怎么做好承压和鉴权。

#### axios 封装

另一个浙大学霸提醒我，axios 大部分时候会封装一下使用。

封装的目的一是配置拦截器

> 1. 请求拦截器 在请求发送前进行必要操作处理，例如添加统一cookie、请求体加验证、设置请求头等，相当于是对每个接口里相同操作的一个封装；
> 2. 响应拦截器 同理，响应拦截器也是如此功能，只是在请求得到响应之后，对响应体的一些处理，通常是数据统一处理等，也常来判断登录失效等。

![](https://img-blog.csdnimg.cn/20210726185956128.png) 

而是包装一下常见的 get post 之类的方法，调用的时候可以简单来写。

由于不熟悉 typescript （反正 js 也不熟悉），所以涉及 type 的报错就被卡了一阵，后来向同学解释这个报错 “typescript 是在 javascript 的基础上发展的，它一改 js 类型过于灵活的毛病，加入了严格的 type 检查机制，因为太严格了所以报错，但实际上执行没有问题”，同学沉思良久，说“就像我在学校门口被拦住了，学校里面的人认识我，但保安不让我进去”，大笑。

![image-20230331103849908](https://s2.loli.net/2023/03/31/1XL8CkH9KeVB4Ow.png)

### 后记

然后就水完了，“干这行的，哪有什么职业道德可言？” 。之后会再研究研究的。

- [ ] session、tokken、cookie 机制？
- [ ] 数据加密存储
- [ ] ui 怎么弄好看一点