### flask
- Flask 自开发伊始就被设计为可扩展的框架， 它具有一个包含基本服务的强健核心，其他功能则可通过扩展实现。
- Flask有两个主要依赖:路由、调试和Web服务器网关接口(Web Server Gateway Interface， WSGI)子系统由 [Werkzeug](http://werkzeug.pocoo.org/) 提供;模板系统由 [Jinja2](http://jinja.pocoo.org/)  提供。Werkzeug 和 Jinjia2 都是由 Flask 的核心开发者开发而成
- Flask 并不原生支持数据库访问、Web 表单验证和用户认证等高级功能。这些功能以及其 他大多数 Web 程序中需要的核心服务都以扩展的形式实现，然后再与核心包集成


### 创建虚拟环境
```
$ virtualenv  flask     #生成一个名为 flask 的环境
$ source  flask/bin/activate  #激活虚拟环境
$ pip install  flask    #虚拟环境中安装 flask ，此时能看到 flask 的核心包
```

### flask 的初始化
```
from flask import Flask 

app = Flask(__name__)
```
所有 Flask 程序都必须创建一个程序实例。Web 服务器使用一种名为 Web 服务器网关接口 (Web Server Gateway Interface，WSGI)的协议，把接收自客户端的所有请求都转交给这个对象处理。    
 `个人理解: web 请求所携带的信息都是发给 app 对象处理。`

### 路由和视图
```
@app.route('/')
def index():
    return '<h1>Hello World!</h>'
```
- 程序实例需要知道对每个 URL 请求运行哪些代码，所以保存了一个 URL 到 Python 函数的映射关系。处理 URL 和函数之间关系的程序称为路由。     
`在 Flask 程序中定义路由的最简便方式，是使用程序实例提供的 app.route 修饰器，把修 饰的函数注册为路由。    修饰器是 Python 语言的标准特性，可以使用不同的方式修改函数的行为。惯 常用法是使用修饰器把函数注册为事件的处理程序。`   
- index()这样的函数称为视图函数(view function)。视图函数返回的响应可以是包含 HTML 的简单字符串，也可以是复杂的表单
```
@app.route('/user/<name>')
def user(name):
    return '<h1>Hello, %s !</h1>' % name
```
- 动态匹配。可以定义类型 `/user/<int:id>  , flask 支持 int、 float、path  类型`

### 启动服务器
```
if __name__ == '__main__':
    app.run(debug=True)
```
- \_\_name\_\_ == '\_\_main\_\_'  代表只有执行当前拥有这份代码的源文件时才执行 app.run ，如果这个文件被其他文件引用，不会执行 app.run()
- 服务器启动后，一直轮训，等待并处理请求，直到程序停止   
`flask 提供的 web 服务器不适合在生产环境中使用`


### 一个完整的程序
```
from flask import Flask
app = Flask(__name__)

@app.route('/')
def index():
    return '<h1> Hello World ! </h1>'

@app.route('/user/<name>')
def user(name):
    return '<h1> Hello %s ! </h1>' % name

if __name__ == '__main__':
    app.run(debug=True)
```

###  程序和请求上下文
-  Flask 从客户端收到请求时，要让视图函数能访问一些对象，这样才能处理请求。`这些对象包括:请求对象，它封装客户端发送的 HTTP 请求`
- 要想让视图函数能够访问请求对象，一个显而易见的方式是将其作为参数传入视图函数。 `不过这会导致程序中的每个视图函数都增加一个参数。除了访问请求对象，如果视图函数在处理请求时还要访问其他对象，情况会变得更糟`
- 为了避免大量可有可无的参数把视图函数弄得一团糟，Flask 使用`上下文`临时把某些对象变为全局可访问。
```
from flask import request

@app.route('/')
def index():
    user_agent = request.headers.get('User-Agent')
    return '<p>Your brower is %s</>' % user_agent
```
`这个视图函数中我们如何把 request 当作全局变量使用。事实上，request 不可能是全局变量。试想，在多线程服务器中，多个线程同时处理不同客户端发送的不同请求时，每个线程看到的 request 对象必然不同。Falsk 使用上下文让特定的变量在一个线程中全局可访问，与此同时却不会干扰其他线程。`
- 上下文: 程序上下文和请求上下文      
    - 程序上下文
        - current_app : `当前激活程序的程序实例`
        - g           : `处理请求时用作临时存储的对象。每次请求都会重设这个变量`
    - 请求上下文
        - requst      : `请求对象，封装了客户端发出的 HTTP 请求中的内容`
        - session     : `用户会话，用于存储请求之间需要 "记住" 的值的词典`    

`Flask 在分发请求之前激活(或推送)程序和请求上下文，请求处理完成后再将其删除。程序上下文被推送后，就可以在线程中使用 current_app 和 g 变量。类似地，请求上下文被推送后，就可以使用 request 和 session 变量。如果使用这些变量时我们没有激活程序上下文或请求上下文，就会导致错误。    在程序实例上调用 app.app_context() 可获得一个程序的上下文。`

### 请求钩子
```有时在处理请求之前或之后执行代码会很有用。例如，在请求开始时，我们可能需要创建数据库连接或者认证发起请求的用户。为了避免在每个视图函数中都使用重复的代码， Flask 提供了注册通用函数的功能，注册的函数可在请求被分发到视图函数之前或之后调用。``` `请求钩子使用修饰器实现`
- before_first_request:注册一个函数，在处理第一个请求之前运行。
- before_request:注册一个函数，在每次请求之前运行。
- after_request:注册一个函数，如果没有未处理的异常抛出，在每次请求之后运行。
- teardown_request:注册一个函数，即使有未处理的异常抛出，也在每次请求之后运行。
```在请求钩子函数和视图函数之间共享数据一般使用上下文全局变量 g。例如，before_ request 处理程序可以从数据库中加载已登录用户，并将其保存到 g.user 中。随后调用视 图函数时，视图函数再使用 g.user 获取用户```

### 响应
- 如果视图函数返回的响应需要使用不同的状态码，那么可以把数字代码作为第二个返回
值，添加到响应文本之后。视图函数返回的响应还可接受第三个参数，这是一个由首部(header)组成的字典，可以 添加到 HTTP 响应中。
```
@app.route('/')
def index():
    return '<h1> Bad Request</h1>', 400
```
- Flask 视图函数还可以返回 Response 对 象。make_response() 函数可接受 1 个、2 个或 3 个参数(和视图函数的返回值一样)，并 返回一个 Response 对象。有时我们需要在视图函数中进行这种转换，然后在响应对象上调 用各种方法，进一步设置响应。
```
from flask import make_response
@app.route('')
def index():
    response = make_response('<h1>This document carries a cookie!</h1>')
    response.set_cookie('answer', '42')
    return response
```
- `重定向`的特殊响应类型
```
from flask import redirect 

@app.route('/')
def index():
    return redirect('http://www.example.com')
```
- 一种特殊的响应由 abort 函数生成，用于处理错误。
```
from flask import abort

@app.route('/user/<id>')
def get_user(id):
    user = load_user(id)
    if not user:
        abort(404)
    return '<h1>Hello, %s <h1>' %user.name
```
`注意，abort 不会把控制权交还给调用它的函数，而是抛出异常把控制权交给 Web 服务器。`

#### Flask-Script 扩展
- 安装    
`$ pip install flask-script`
- 使用 - 这样修改之后，程序可以使用一组基本命令行选项。
```
from flask.ext.script import Manager
manager = Manager(app)

#...

if __name__ == '__main__':
    manager.run()
```

### 模版
`用户在网站中注册了一个新账户。用户在表单中输入电子邮件地址和密码，然后点 击提交按钮。服务器接收到包含用户输入数据的请求，然后 Flask 把请求分发到处理注册 请求的视图函数。这个视图函数需要访问数据库，添加新用户，然后生成响应回送浏览 器。这两个过程分别称为业务逻辑和表现逻辑。`    
`把业务逻辑和表现逻辑混在一起会导致代码难以理解和维护,把表现逻辑移到模板中能够提升程序的可维护性。`    
- 默认情况下，Flask 在程序文件夹中的 templates 子文件夹中寻找模板。
- 渲染模板
```
from flask import Flask , render_template 

# ...

@app.route('/')
def index():
    return reder_template('index.html')

@app.route('/user/<name>')
def user(name):
    return render_template('user.html', name = name)
    # 渲染 template 目录下的 user.html ; render_template 函数的第一个参数是模板的文件名。随后的参数都是键值对，表示模板中变量对应的真实值
```

#### 模版: `变量`
- Jinja2 能识别所有类型的变量，甚至是一些复杂的类型，例如列表、字典和对象。
- 过滤器: `修改变量`
    - safe `渲染值时不转义，显示变量中的 HTML 代码`
    - capitalize `把值的首字母转换成大写，其他字母转换成小写`
    - lower `把值转换成小写形式`
    - upper `把值转换成大写形式`
    - title `把值中每个单词的首字母都转换成大写`
    - trim `把值的首尾空格去掉`
    - striptags `渲染之前把值中所有的 HTML 标签都删掉`

#### 模版: `控制结构`
- if 
```
{% if user %}
    Hello, {{ user }}!
{% else %}
    Hello, Stranger!
{% endif %}
```
- for 
```
<ul>
    {% for comment in comments %}
        <li> {{ comment }} </li>
    {% endfor %}
</ul>
```
- macro
```
# 直接使用
{% macro render_comment(comment) %}
    <li> {{ comment }} </li>
{% endmacro %}

<ul>
    {% for comment in comments %}
        {{ render_comment(comment) }}
    {% endfor %}
</ul>
```
```
# import macro
{% import 'macro.html' as macro %}
<ul>
    {% for comment in comments %}
        {{ macro.render_comment(comment) }}
    {% endfor %}
</ul>
```
`多处重复使用的模板代码片段可以写入单独的文件，再包含在所有模板中，以避免重复: {% include 'common.html' %}`

#### 模版: `继承`
- base.html 
```
<html>
<head>
    {% block head %}
    <title>{% block title %}{% endblock %} - My Application </title>
    {% endblock %}
</head>
    {% block body %}
    {% endblock %}
<body>
</body>
</html>
# block 标签定义的元素可在衍生模板中修改。在本例中，我们定义了名为 head、title 和 body 的块。
```
- 衍生模版
```
{% extends "base.html" %}   #extends 指令声明这个模板衍生自 base.html
{% block title %} Index {% endblock %}
{% block head %}
    {{ super() }}   #新定义的 head 块，在基模板中其内容不是空的，所以使用 super() 获取原来的内容
    <styple>
    </styple>
{% endblock %}
{% block body %}
<h1>Hello, World!</h1>
{% endblock %}
```

#### Flash-Bootstrap
- 安装
```
(venv) $ pip install flask-bootstrap
```

- 初始化
```
from flask.ext.bootstrap import Bootstrap
# Flask-Bootstrap 也从 flask.ext 命名空间中导入，然后把程序实例传入构造方法进行初始化
# ...
bootstrap = Bootstrap(app)
```
- 基于 Flask-bootstrap base.html 衍生
```
{% extends "bootstap/base.html" %}

{% block title %}Flasky{% endblock %}

{% block navbar %}
# http://www.runoob.com/bootstrap/bootstrap-navbar.html
<div class="navbar navbar-inverse" role="navigation">
    <div class="container">
        <div class="navbar-header">  #这会让文本看起来更大一号
            <button type="button" class="navbar-toggle" data-toggle="collapse" data-target=".navbar-collapse">
            # <span> 用于对文档中的行内元素进行组合。
                <span class="src-only"> Toggle navigation </span>
                <span class="icon-bar"></span>
                <span class="icon-bar"></span>
                <span class="icon-bar"></span>
            </button>
            <a class="navbar-brand"  href="/">Flasky</a>
        </div>
        <div class="navbar-collapse collapse">
            <ul class="nav navbar-nav">
                <li><a href="/">Home</a></li>
            </ul>
        </div>
    </div>
</div>
{% endblock %}

{% block content %}
<div class="contanier">
    <div class="page-header">
        <h1>Hello, {{ name }}!</h1>
    </div>
</div>
{% endblock %}
```
- Flask-Bootstrap基模版中定义的块
    - doc `整个 HTML 文档`
    - html_attribs `<html> 标签的属性`
    - html `<html> 标签中的内容` 
    - head `<head> 标签中的内容`
    - title `<title> 标签中的内容`
    - metas `一组 <metas> 标签`
    - styles `层叠样式表定义`
    - body_attribs `<body> 标签的属性`
    - body `<body> 标签中的内容`
    - navbar `用户定义的导航条`
    - content `用户定义的页面内容`
    - scripts `文档底部的 JavaScript 声明`
```
很多块都是 Flask-Bootstrap 自用的，如果直接重定义可能会导致一些问题。例 如，Bootstrap 所需的文件在 styles 和 scripts 块中声明。如果程序需要向已经有内容的块 中添加新内容，必须使用 Jinja2 提供的 super() 函数。    
例如，如果要在衍生模板中添加新 的 JavaScript 文件，需要这么定义 scripts 块:
{% block scripts %}
{{ super() }}
<scipt type="text/javascript" src="my-script.js"></scripts>
{% endblock %}
```

#### 自定义错误页面
- 错误代码自定义处理程序
```
Flask 允许程序使用基于模板的自定义错误页面。最常见的错误代码有两个:404，客户端请求未知页面或路由时显示;500，有未处理的异常时显示。

@app.errorhandler(404)
def page_not_found(e):
    return render_template('404.html'), 404

@app.errorhandler(500)
def internal_server_error(e):
    return render_template('500.html'), 500
```
- 生成一个格式一致的程序基模板
`基于 flask-bootstrap 定义一个新的基模版`
```
{% extends "bootstap/base.html" %}

{% block title %}Flasky{% endblock %}

{% block navbar %}
<div class="navbar navbar-inverse" role="navigation">
    <div class="container">
        <div class="navbar-header">  
            <button type="button" class="navbar-toggle" data-toggle="collapse" data-target=".navbar-collapse">
                <span class="src-only"> Toggle navigation </span>
                <span class="icon-bar"></span>
                <span class="icon-bar"></span>
                <span class="icon-bar"></span>
            </button>
            <a class="navbar-brand"  href="/">Flasky</a>
        </div>
        <div class="navbar-collapse collapse">
            <ul class="nav navbar-nav">
                <li><a href="/">Home</a></li>
            </ul>
        </div>
    </div>
</div>
{% endblock %}

{% block content %}
<div class="contanier">
    {% block page_content %}{% endblock %}   #这里是与上面唯一的不通。 这个模板的 content 块中只有一个 <div> 容器，其中包含了一个名为 page_content 的新的 空块，块中的内容由衍生模板定义。
</div>
{% endblock %}
```
- 使用程序基模版
    - 404.html
    ```
    {% extends "base.html" %}
    {% block title %}Flasky - Page Not Found{% endblock %}
    {% block page_content %}
    <div class="page-header">
         <h1>Not Found</h1>
    </div>
    {% endblock %}
    ```
    - 500.html
    ```
    ...
    ```
    - user.html
    ```
    ...
    ```

#### 链接
- 使用 url_for() 生成动态地址时，将动态部分作为关键字参数传入。例如，url_for ('user', name='john', _external=True)的返回结果是http://localhost:5000/user/john
- 传入 url_for() 的关键字参数不仅限于动态路由中的参数。函数能将任何额外参数添加到 查询字符串中。例如，url_for('index', page=2)的返回结果是/?page=2
- 寻找静态文件
```
### 定义收藏夹图标
{% block head %}
{{ super() }}
<link rel="shortcut icon" href="{{ url_for('static', filename = 'favicon.ico') }}" type='image/x-icon'>
<link rel="icon" href="{{ url_for('static', filename = 'favicon.ico') }}" type='image/x-icon'>
{% endblock %}
# 图标的声明会插入 head 块的末尾。使用 super() 保留基模板中定义的块的原始内容。
```

#### Flask-Moment 时间 js 库
- 安装
`(venv) $ pip install flask-moment`
- 初始化
```
from flask.ext.moment import Moment
moment = Moment(app)
```
- tmplate/base.html 模版引入 moment.js 库
```
{% block scripts %}
{{ super() }}
{{ moment.include_moment() }}
{% endblock %}
```

- 代码传入变量 current_time
```
from datetime import datetime

@app.route('/')
def index():
    return render_template('index.html', current_time=datetime.utcnow())
```

- 渲染 index.html 的时间戳
    - format('LLL') 根据客户端电脑中的时区和区域设置渲染日期和时间。参数决定了渲染的方 式，'L' 到 'LLLL' 分别对应不同的复杂度。format() 函数还可接受自定义的格式说明符。    
    - 第二行中的 fromNow() 渲染相对时间戳，而且会随着时间的推移自动刷新显示的时间。这 个时间戳最开始显示为“a few seconds ago”，但指定refresh参数后，其内容会随着时 间的推移而更新。如果一直待在这个页面，几分钟后，会看到显示的文本变成“a minute ago”“2 minutes ago”等。    
    - Flask-Moment 实现了 moment.js 中的 format()、fromNow()、fromTime()、calendar()、valueOf() 和 unix() 方法。   
    - Flask-Moment 渲染的时间戳可实现多种语言的本地化。语言可在模板中选择，把语言代码 传给 lang() 函数即可: `{{ moment.lang('es') }}`
```
<p>The local date and time is {{ moment(current_time).format('LLL') }}.</p>
<p>That was {{ moment(current_time).fromNow(refresh=True) }}</p>
```

### web 表单
- 安装
    - request.form 能获取 POST 请求中提交的表单数据。 尽管 Flask 的请求对象提供的信息足够用于处理 Web 表单，但有些任务很单调，而且要重复操作。比如，生成表单的 HTML 代码和验证提交的表单数据。 [Flask-WTF](http://pythonhosted.org/Flask-WTF/) 扩展可以把处理 Web 表单的过程变成一种愉悦的体验。
    ```
    (venv) $ pip install flask-wtf
    ```
- 跨站请求伪造保护
    - 为了实现 CSRF 保护，Flask-WTF 需要程序设置一个密钥。Flask-WTF 使用这个密钥生成加密令牌，再用令牌验证请求中表单数据的真伪。
    - app.config 字典可用来存储框架、扩展和程序本身的配置变量。使用标准的字典句法就能把配置值添加到 app.config 对象中。这个对象还提供了一些方法，可以从文件或环境中导 入配置值。SECRET_KEY 配置变量是通用密钥，可在 Flask 和多个第三方扩展中使用。不同的程序要使用不同的密钥，而且要保证其他人不 知道你所用的字符串。
    ```
    app = Flask(__name__)
    app.config['SECRET_KEY'] = 'hard to guess string'
    ```
- 表单类
    - 使用 Flask-WTF 时，每个 Web 表单都由一个继承自 Form 的类表示。这个类定义表单中的 一组字段，每个字段都用对象表示。字段对象可附属一个或多个验证函数。验证函数用来 验证用户提交的输入值是否符合要求
    - 一个简单的 Web 表单，包含一个文本字段和一个提交按钮。
    ```
    from flask.ext.wtf import Form
    form wtforms import StringField, SubmitField
    form wtforms.validators import Required
    #表单中的字段都定义为类变量，类变量的值是相应字段类型的对象。
    class NameForm(Form):
    #NameForm 表单中有一个名为 name 的文本字段和一个名为 submit 的提交按钮。
        name = StringField('what is your name?', varlidators=[Required()]) #验证函数 Required() 确保提交的字段不为空。
        #StringField 类表示属性为 type="text" 的 <input> 元素。
        submit = SubmitField('Submit')
        #SubmitField 类表示属性为 type="submit" 的 <input> 元素。
    ```
- WTForms 支持的 HTML 标准字段
    - StringField ` 文本字段`
    - TextAreaField `多行文本字段`
    - PasswordField `密码文本字段`
    - HiddenField `隐藏文本字段`
    - DateField  `文本字段，值为 datetime.date 格式 `
    - DateTimeField  `文本字段，值为 datetime.datetime 格式 `
    - IntegerField  `文本字段，值为整数`
    - DecimalField  `文本字段，值为 decimal.Decimal`
    - FloatField  `文本字段，值为浮点数 `
    - BooleanField  `复选框，值为 True 和 False`
    - RadioField  `一组单选框`
    - SelectField  `下拉列表`
    - SelectMultipleField  `下拉列表，可选择多个值`
    - FileField  `文件上传字段`
    - SubmitField  `表单提交按钮`
    - FormField `把表单作为字段嵌入另一个表单`
    - FieldList `一组指定类型的字段`
- WTForms 验证函数
    - Email `验证电子邮件地址 `
    - EqualTo `比较两个字段的值;常用于要求输入两次密码进行确认的情况`
    - IPAddress  `验证 IPv4 网络地址`
    - Length `验证输入字符串的长度`
    - NumberRange `验证输入的值在数字范围内`
    - Optional `无输入值时跳过其他验证函数`
    - Required `确保字段中有数据`
    - Regexp `使用正则表达式验证输入值`
    - URL `验证 URL`
    - AnyOf `确保输入值在可选值列表中`
    - NoneOf `确保输入值不在可选值列表中`

#### 把表单渲染成 HTML
- 表单字段是可调用的，在模板中调用后会渲染成 HTML。假设视图函数把一个 NameForm 实 Web 表单例通过参数 form 传入模板，在模板中可以生成一个简单的表单
```
<form method="POST">
    {{ form.hidden_tag() }}
    {{ form.name.label }} {{ form.name() }}
    {{ form.submit() }}
</form>
```
- 导入的 bootstrap/wtf.html 文件中定义了一个使用 Bootstrap 渲染 Falsk-WTF 表单对象 的辅助函数。wtf.quick_form() 函数的参数为 Flask-WTF 表单对象，使用 Bootstrap 的默认样式渲染传入的表单
```
{% extends "base.html" %}
{% import "bootstrap/wtf.html" as wtf %}
{% block title %}Flasky{% endblock %}
{% block page_content %}
<div class="page-header">
    <h1>Hello, {% if name %}{{ name }}{{% else %}}Stranger{% endif %}!</h1>
</div>
{{ wtf.quick_form(form) }}
{% endblock %}
```

#### 在视图函数中处理表单
- 视图函数 index() 不仅要渲染表单，还要接收表单中的数据
    - 把 POST 加入方法列表很有必要，因为将提交表单作为 POST 请求进行处理更加便利。表单 也可作为 GET 请求提交，不过 GET 请求没有主体，提交的数据以查询字符串的形式附加到 URL 中，可在浏览器的地址栏中看到。基于这个以及其他多个原因，提交表单大都作为 POST 请求进行处理。
```
@app.route('/', methods=['GET', 'POST']) # 如果没指定 methods 参数，就只把视图函数注册为 GET 请求 的处理程序。
def index():
    name = None
    form = NameForm()
    if form.validate_on_submit():
        name = form.name.data
        form.name.data = ''
    return render_template('index.html', form=form, name=name)
```
`用户第一次访问程序时，服务器会收到一个没有表单数据的 GET 请求，所以 validate_on_ submit() 将返回 False。if 语句的内容将被跳过，通过渲染模板处理请求，并传入表单对 象和值为 None 的 name 变量作为参数。用户会看到浏览器中显示了一个表单。    
用户提交表单后，服务器收到一个包含数据的 POST 请求。validate_on_submit() 会调用 name 字段上附属的 Required() 验证函数。如果名字不为空，就能通过验证，validate_on_ submit() 返回 True。现在，用户输入的名字可通过字段的 data 属性获取。在 if 语句中， 把名字赋值给局部变量 name，然后再把 data 属性设为空字符串，从而清空表单字段。最 后一行调用 render_template() 函数渲染模板，但这一次参数 name 的值为表单中输入的名 字，因此会显示一个针对该用户的欢迎消息。`

#### 重定向和用户会话
`用户输入名字后提交表单，然后点击浏览器的刷 新按钮，会看到一个莫名其妙的警告，要求在再次提交表单之前进行确认。之所以出现这 种情况，是因为刷新页面时浏览器会重新发送之前已经发送过的最后一个请求。如果这个 请求是一个包含表单数据的 POST 请求，刷新页面后会再次提交表单。大多数情况下，这并 不是理想的处理方式。
很多用户都不理解浏览器发出的这个警告。基于这个原因，最好别让 Web 程序把 POST 请 求作为浏览器发送的最后一个请求。
这种需求的实现方式是，使用重定向作为 POST 请求的响应，而不是使用常规响应。浏览器收到 这种响应时，会向重定向的 URL 发起 GET 请求，显示页面的内容。最后一个请求是 GET 请求，所以刷新命令能像预期的那样正常使用了。这个技 巧称为 Post/ 重定向 /Get 模式`
`这种方法会带来另一个问题。程序处理 POST 请求时，使用 form.name.data 获取用户输 入的名字，可是一旦这个请求结束，数据也就丢失了。程序可以把数据存储在用户会话中，在请求之间“记住”数据。用户会话是一种私有存 储，存在于每个连接到服务器的客户端中。它是请求上下 文中的变量，名为 session，像标准的 Python 字典一样操作。`
```
form flask import Flask, render_template, session, redirect, url_for

@app.route('/', methods=['GET', 'POST'])
def index():
    form = NameForm()
    if form.validate_on_submit():
        session['name'] = form.name.data
        return redirect(url_for('index'))
    return render_template('index.html', form=form, name=session.get('name'))
```
`url_for() 函数的第一个且唯一必须指定的参数是端点名，即路由的内部名字。默认情 况下，路由的端点是相应视图函数的名字。在这个示例中，处理根地址的视图函数是 index()，因此传给 url_for() 函数的名字是 index。
`

#### Flash 消息
- flash() 函数
`请求完成后，有时需要让用户知道状态发生了变化。这里可以使用确认消息、警告或者错 误提醒。一个典型例子是，用户提交了有一项错误的登录表单后，服务器发回的响应重新 渲染了登录表单，并在表单上面显示一个消息，提示用户用户名或密码错误。 这种功能是 Flask 的核心特性`
```
from flask import Flask, render_template, session, redirect, url_for, flash 

@app.route('/', methods=['GET','POST'])
def index():
    form = NameForm()
    if form.validate_on_submit():
        old_name = seesion.get('name')
        if old_name is not None and old_name != form.name.data:
            flash('Looks like you have changed you name!')
        session['name'] = form.name.data
        return redirect(url_for('index'))
    return render_template('index.html', form = form, name = session.get('name'))
```
`在这个示例中，每次提交的名字都会和存储在用户会话中的名字进行比较，而会话中存储 的名字是前一次在这个表单中提交的数据。如果两个名字不一样，就会调用 flash() 函数， 在发给客户端的下一个响应中显示一个消息。`

- get_flashed_message() 函数
`仅调用 flash() 函数并不能把消息显示出来，程序使用的模板要渲染这些消息。最好在 基模板中渲染 Flash 消息，因为这样所有页面都能使用这些消息。Flask 把 get_flashed_ messages() 函数开放给模板，用来获取并渲染消息`
```
# template/base.html: 渲染 Flash 消息
{% block content %}
<div class = "container"> 
    {% for message in get_flashed_message()  %}
    <div class="alert alert-warning">
        <button type='button' class='close' date-dismiss="alert">&times;</button>
        {{ message }}
    </div>
    {% endfor  %}
</div>
{% endblock %}
```
`在模板中使用循环是因为在之前的请求循环中每次调用 flash() 函数时都会生成一个消息， 所以可能有多个消息在排队等待显示。get_flashed_messages() 函数获取的消息在下次调 用时不会再次返回，因此 Flash 消息只显示一次，然后就消失了`


### 数据库
```
数据库按照一定规则保存程序数据，程序再发起查询取回所需的数据。     
Web 程序最常用基 于关系模型的数据库，这种数据库也称为 SQL 数据库，因为它们使用结构化查询语言。     
最近几年文档数据库和键值对数据库成了流行的替代选择，这两种数据库合称 NoSQL 数据库。    
```

####  SQL 数据库
- 关系型数据库把数据存储在表中，表模拟程序中不同的实体。
- 表的列数是固定的，行数是可变的。
- 列定义表所表示的实体的数据属性。 表中的行定义各列对应的真实数据。
- 表中有个特殊的列，称为主键，其值为表中各行的唯一标识符。
- 表中还可以有称为外键的 列，引用同一个表或不同表中某行的主键。
- 行之间的这种联系称为关系，这是关系型数据 库模型的基础。    
`关系型数据库存储数据很高效，而且避免了重复。但从另一方面来看，把数据分别存放在多个表中还是很复杂的。关系型数据库引擎为联结操作提供了必要的支持。`

#### NoSQL 数据库
- NoSQL 数据库一般使用 集合代替表，使用文档代替记录。
`这是执行反规范化操作得到的结果，它减少了表的数量，却增加了数据重复量。NoSQL 数据库采用的设计方式使联结变得困难，所以大多数数据库根本不支持这种操作。更新重复数据变的耗时，但是数据重复可以提升查询速度`

#### SQL还是NoSQL ?
- SQL 数据库擅于用高效且紧凑的形式存储结构化数据。这种数据库需要花费大量精力保证
数据的一致性
- NoSQL 数据库放宽了对这种一致性的要求，从而获得性能上的优势。

#### 数据库框架
- 易用性
`如果直接比较数据库引擎和数据库抽象层，显然后者取胜。抽象层，也称为对象关系 映射(Object-Relational Mapper，ORM)或对象文档映射(Object-Document Mapper， ODM)，在用户不知觉的情况下把高层的面向对象操作转换成低层的数据库指令。`
- 性能
`ORM 和 ODM 把对象业务转换成数据库业务会有一定的损耗。大多数情况下，这种性 能的降低微不足道，但也不一定都是如此。一般情况下，ORM 和 ODM 对生产率的提 升远远超过了这一丁点儿的性能降低，所以性能降低这个理由不足以说服用户完全放弃 ORM 和 ODM。真正的关键点在于如何选择一个能直接操作低层数据库的抽象层，以 防特定的操作需要直接使用数据库原生指令优化。`
- 可移植性
`选择数据库时，必须考虑其是否能在你的开发平台和生产平台中使用。可移植性还针对 ORM 和 ODM。尽管有些框架只为一种数据库引擎提供抽象层，但其 他框架可能做了更高层的抽象，它们支持不同的数据库引擎，而且都使用相同的面向对象接口。`

#### 使用Flask-SQLAlchemy管理数据库
- Flask-SQLAlchemy     
`一个 Flask 扩展，简化了在 Flask 程序中使用 SQLAlchemy 的操作。 SQLAlchemy 是一个很强大的关系型数据库框架，支持多种数据库后台。SQLAlchemy 提 供了高层 ORM，也提供了使用数据库原生 SQL 的低层功能。`
```
安装：
$ pip install flask-sqlalchemy

使用：
MySQL           mysql://username:password@hostname/database 
Postgres        postgresql://username:password@hostname/database 
SQLite(Unix)    sqlite:////absolute/path/to/database

```
`程序使用的数据库 URL 必须保存到 Flask 配置对象的 SQLALCHEMY_DATABASE_URI 键中。配 置对象中还有一个很有用的选项，即 SQLALCHEMY_COMMIT_ON_TEARDOWN 键，将其设为 True 时，每次请求结束后都会自动提交数据库中的变动。`
```
form flask.ext.sqlalchemy import SQLAlchemy

basedir = os.path.abspath(os.path.dirname(__file__))

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] =\
    'sqlite:///' + os.path.join(basedir, 'data.sqlite') 
app.config['SQLALCHEMY_COMMIT_ON_TEARDOWN'] = True

db = SQLAlchemy(app)
```

####  定义模型
`模型这个术语表示程序使用的持久化实体。在 ORM 中，模型一般是一个 Python 类，类中的属性对应数据库表中的列。Flask-SQLAlchemy 创建的数据库实例为模型提供了一个基类以及一系列辅助类和辅助函数,可用于定义模型的结构`
```
#  roles 表和 users 表可定义为模型 Role 和 User
# role: id , name 
# user: id , username, password, role_id

# 定义 Role 和 User 模型
class Role(db.Model):
    __tablename__ = 'roles'
    id   = db.Column(db.Interger, primary_key=True)
    name = db.Column(db.String(64), unique=True)

    def __repr__(self):
        return '<Role %>' % self.name

class User(db.Model):
    __tablename__ = 'users'
    id   = db.Column(db.Interger, primary_key=True)
    username = db.Column(db.String(64), unique=True, index=True)

    def __repr__(self):
        return '<User %>' % self.username
# 类变量 __tablename__ 定义在数据库中使用的表名
# db.Column 类构造函数的第一个参数是数据库列和模型属性的类型
# 虽然没有强制要求，但这两个模型都定义了 __repr()__ 方法，返回一个具有可读性的字符 串表示模型，可在调试和测试时使用。
# Flask-SQLAlchemy 要求每个模型都要定义主键，这一列经常命名为 id
```
```
# 最常用的SQLAlchemy列类型
 
类型名          Python类型          说  明
Integer         int                 普通整数，一般是 32 位 
SmallInteger    int                 取值范围小的整数，一般是 16 位
BigInteger      int or long         不限制精度的整数 
Float           float               浮点数
Numeric         decimal.Decimal     定点数
String          str                 变长字符串
Text            str                 变长字符串，对较长或不限长度的字符串做了优化
Unicode         unicode             变长 Unicode 字符串
UnicodeText     unicode             变长 Unicode 字符串，对较长或不限长度的字符串做了优化
Boolean         bool                布尔值
Date            datetime.date       日期
Time            datetime.time       时间
DateTime        datetime.datetime   日期和时间
Interval        datetime.timedelta  时间间隔
Enum            str                 一组字符串
PickleType      任何 python 对象    自动使用 Pickle 序列化
LargeBinary     str                 二进制文件
```
```
#最常使用的SQLAlchemy列选项
选项名          说  明
primary_key     如果设为 True，这列就是表的主键
unique          如果设为 True，这列不允许出现重复的值
index           如果设为 True，为这列创建索引，提升查询效率
nullable        如果设为 True，这列允许使用空值;如果设为 False，这列不允许使用空值
default         为这列定义默认值
```


####  关系
- 一对多关系
```
# 一个角色可属于多个用户，而每 个用户都只能有一个角色。

class Role(db.Model):
    # ...
    users = db.relationship('User', backref = 'role')
#db.relationship() 中的 backref 参数向 User 模型中添加一个 role 属性，从而定义反向关 系。这一属性可替代 role_id 访问 Role 模型，此时获取的是模型对象，而不是外键的值。 大多数情况下，db.relationship() 都能自行找到关系中的外键，但有时却无法决定把 哪一列作为外键。例如，如果 User 模型中有两个或以上的列定义为 Role 模型的外键， SQLAlchemy 就不知道该使用哪列。如果无法决定外键，你就要为 db.relationship() 提供 额外参数，从而确定所用外键。

class User(db.Model):
    # ...
    role_id = db.Column(db.Integer, db.Foreignkey('roles.id'))
#添加到 User 模型中的 role_id 列 被定义为外键，就是这个外键建立起了关系。传给 db.ForeignKey() 的参数 'roles.id' 表 明，这列的值是 roles 表中行的 id 值。
```
```
除了一对多之外，还有几种其他的关系类型。一对一关系可以用前面介绍的一对多关系 表示，但调用 db.relationship() 时要把 uselist 设为 False，把“多”变成“一”。多对 一关系也可使用一对多表示，对调两个表即可，或者把外键和 db.relationship() 都放在多这一侧。最复杂的关系类型是多对多，需要用到第三张表，这个表称为关系表。
```


### 数据库操作
- 创建数据库
```
db.create_all()

# 删除数据库
db.drop_all()
```
- 插入行
```
admin_role = Role(name='Admin')
mod_role = Role(name='Moderator')
user_role = Role(name='User')

user_john = User(username='john', role=admin_role)
user_susan = User(username='susan', role=user_role)
user_david = User(username='david', role=user_role)

# 对象只存在于 Python 中，还未写入数据库
# 通过数据库会话管理对数据库所做的改动，在 Flask-SQLAlchemy 中，会话由 db.session 表示
# 把对象写入数据库之前，先要将其添加到会话中
# 数据库 会话也称为事务

db.session.add(admin_role)
db.session.add(mod_role)
db.session.add(user_role)
db.session.add(user_john)
db.session.add(user_susan)
db.session.add(user_david)
# 数据库会话能保证数据库的一致性。提交操作使用原子方式把会话中的对象全部写入数据库。如果在写入会话的过程中发生了错误，整个会话都会失效。如果你始终把相关改动放在会话中提交，就能避免因部分更新导致的数据库不一致性
# 数据库会话也可回滚。调用 db.session.rollback() 后，添加到数据库会话 中的所有对象都会还原到它们在数据库时的状态


# 或者简写成:
db.session.add_all([admin_role, mod_role, user_role,
    user_john, user_susan, user_david])

# 为了把对象写入数据库，我们要调用 commit() 方法提交会话:
    db.session.commit()
```

- 修改行
```
# 在数据库会话上调用 add() 方法也能更新模型。

admin_role.name = 'Administrator'
db.session.add(admin_role)
db.session.commit()
```

- 删除行
```
db.session.delete(mod_role)
db.session.commit()
# 删除与插入和更新一样，提交数据库会话后才会执行
```

- 查询行
```
# flask-SQLAlchemy 为每个模型类都提供了 query 对象。最基本的模型查询是取回对应表中
的所有记录:
>>> Role.query.all()
      [<Role u'Administrator'>, <Role u'User'>]
>>> User.query.all()
      [<User u'john'>, <User u'susan'>, <User u'david'>]

#使用过滤器可以配置 query 对象进行更精确的数据库查询。下面这个例子查找角色为 "User" 的所有用户:
>>> User.query.filter_by(role=user_role).all() 
    [<User u'susan'>, <User u'david'>]

# 若要查看 SQLAlchemy 为查询生成的原生 SQL 查询语句，只需把 query 对象转换成字符串:
>>> str(User.query.filter_by(role=user_role))
    'SELECT users.id AS users_id, users.username AS users_username, users.role_id AS users_role_id FROM users WHERE :param_1 = users.role_id'
```
```
# 常用的SQLAlchemy查询过滤器
过滤器           说  明
filter()        把过滤器添加到原查询上，返回一个新查询
filter_by()     把等值过滤器添加到原查询上，返回一个新查询
limit()         使用指定的值限制原查询返回的结果数量，返回一个新查询
offset()        偏移原查询返回的结果，返回一个新查询
order_by()      根据指定条件对原查询结果进行排序，返回一个新查询
group_by()      根据指定条件对原查询结果进行分组，返回一个新查询
```
```
# 最常使用的SQLAlchemy查询执行函数
方法            说  明
all()            以列表形式返回查询的所有结果
first()          返回查询的第一个结果，如果没有结果，则返回 None
first_or_404()   返回查询的第一个结果，如果没有结果，则终止请求，返回 404 错误响应
get()            返回指定主键对应的行，如果没有对应的行，则返回 None
get_or_404()     返回指定主键对应的行，如果没找到指定的主键，则终止请求，返回 404 错误响应 
count()          返回查询结果的数量
paginate()       返回一个 Paginate 对象，它包含指定范围内的结果
```

- 关系
```
# 关系和查询的处理方式类似
# 从关系的两端查询角色和用户之间的一对多关系
     >>> users = user_role.users
     >>> users
     [<User u'susan'>, <User u'david'>]
     >>> users[0].role
     <Role u'User'>

#  这个例子中的 user_role.users 查询有个小问题。执行 user_role.users 表达式时，隐含的 查询会调用 all() 返回一个用户列表。query 对象是隐藏的，因此无法指定更精确的查询 过滤器。就这个特定示例而言，返回一个按照字母顺序排序的用户列表可能更好。修改了关系的设置，加入了lazy = 'dynamic'参数，从而禁止自动执行查询。
     class Role(db.Model):
         # ...
         users = db.relationship('User', backref='role', lazy='dynamic')
         # ...
# 这样配置关系之后，user_role.users 会返回一个尚未执行的查询，因此可以在其上添加过 滤器:
     >>> user_role.users.order_by(User.username).all()
     [<User u'david'>, <User u'susan'>]
     >>> user_role.users.count()
     2
```

####  在视图函数中操作数据库
```
# 首页路由的新版本， 把用户输入的名字写入了数据库。

@app.route('/', methods = ['GET', 'POST'])
def index():
    form = NameForm()
    if form.validate_on_submit():
        user = User.query.filter_by(username=form.name.data).first()
        if user is None:
            user = User(username = form.name.data)
            db.session.add(user)
            session['known'] = False
        else:
            session['known'] = True
        session['known'] = form.name.data
        form.name.data   = ''
        return redirect(url_for('index'))
    return render_template('index.html', \
        form  = form , name = session.get('name'), \
        known = session.get('known', False) )
#  提交表单后，程序会使用 filter_by() 查询过滤器在数据库中查 找提交的名字。变量 known 被写入用户会话中，因此重定向之后，可以把数据传给模板， 用来显示自定义的欢迎消息。
```
```
# templates/index.html 
{% extends "base.html" %}
{% import "bootstrap/wtf.html" as wtf %}

{% block title %}Flasky{% endblock %}

{% block page_content %} <div class="page-header">
<h1>Hello, {% if name %}{{ name }}{% else %}Stranger{% endif %}!</h1> {% if not known %}
<p>Pleased to meet you!</p>
{% else %}
    <p>Happy to see you again!</p>
{% endif %} </div>
{{ wtf.quick_form(form) }} {% endblock %}
```

####  集成Python shell
```
若想把对象添加到导入列表中，我们要为 shell 命令注册一个 make_context 回调函数，如

    from flask.ext.script import Shell
    def make_shell_context():
         return dict(app=app, db=db, User=User, Role=Role)
    manager.add_command("shell", Shell(make_context=make_shell_context)) 

make_shell_context() 函数注册了程序、数据库实例以及模型，因此这些对象能直接导入 shell
 
$ python hello.py shell
>>> app
<Flask 'app'>
>>> db
<SQLAlchemy engine='sqlite:////home/flask/flasky/data.sqlite'> >>> User
<class 'app.User'>
```

#### 使用Flask-Migrate实现数据库迁移
```
写的不清晰，自己研究
```


###  电子邮件
- Flask-Mail SMTP
```
# Flask-Mail SMTP服务器的配置
配  置           默认值            说  明
MAIL_SERVER     localhost 电子邮件服务器的主机名或 IP 地址 
MAIL_PORT       25 电子邮件服务器的端口
MAIL_USE_TLS    False 启用传输层安全(Transport Layer Security，TLS)协议 
MAIL_USE_SSL    False 启用安全套接层(Secure Sockets Layer，SSL)协议 
MAIL_USERNAME   None 邮件账户的用户名
MAIL_PASSWORD   None 邮件账户的密码
```
```
# 配置 Flask-Mail 使用 Gmail
import os
# ...
app.config['MAIL_SERVER'] = 'smtp.googlemail.com' 
app.config['MAIL_PORT'] = 587
app.config['MAIL_USE_TLS'] = True
app.config['MAIL_USERNAME'] = os.environ.get('MAIL_USERNAME')
app.config['MAIL_PASSWORD'] = os.environ.get('MAIL_PASSWORD')
```
```
from flask.ext.mail import Message

app.config['FLASKY_MAIL_SUBJECT_PREFIX'] = '[Flasky]' 
app.config['FLASKY_MAIL_SENDER'] = 'Flasky Admin <flasky@example.com>'

def send_email(to, subject, template, **kwargs):
    msg = Message(app.config['FLASKY_MAIL_SUBJECT_PREFIX'] + subject,
    sender=app.config['FLASKY_MAIL_SENDER'], recipients=[to]) msg.body = render_template(template + '.txt', **kwargs)
    msg.html = render_template(template + '.html', **kwargs)
    mail.send(msg)
```
```
# index() 视图函数很容易被扩展，这样每当表单接收新名字时，程序都会给管理员发送一 封电子邮件

@app.route('/', methods = ['GET', 'POST'])
def index():
    form = NameForm()
    if form.validate_on_submit():
        user = User.query.filter_by(username=form.name.data).first()
        if user is None:
            user = User(username = form.name.data)
            db.session.add(user)
            session['known'] = False

            if app.config['FLASKY_ADMIN']:
                app.config['FLASKY_ADMIN']
                 'New User', 'mail/new_user', user=user)

        else:
            session['known'] = True
        session['known'] = form.name.data
        form.name.data   = ''
        return redirect(url_for('index'))
    return render_template('index.html', \
        form  = form , name = session.get('name'), \
        known = session.get('known', False) )

```
```
# 如果你发送了几封测试邮件，可能会注意到 mail.send() 函数在发送电子邮件时停滞了几 秒钟，在这个过程中浏览器就像无响应一样。为了避免处理请求过程中不必要的延迟，我 们可以把发送电子邮件的函数移到后台线程中
from threading import Thread
def send_async_email(app,msg):
    with app.app_context():
        mail.send(msg)
# 注意，Flask-Mail 中的 send() 函数使用 current_app，因此要在激活的程序上下文中执行

def send_mail(to, subject, template, **kwargs):
    msg = Message(app.config['FLASKY_MAIL_SUBJECT_PREFIX'] + subject,
            sender=app.config['FLASKY_MAIL_SENDER'], recipients=[to]))
    msg.body = render_tempate(template + '.txt', **kwargs)
    msg.html = render_tempate(template + '.html', **kwargs)
    thr = Thread(target=send_async_email, args=[app.msg])
    thr.start()
    return thr
# 很多 Flask 扩展都假设已经存在激活的程序上下文和请求 上下文。Flask-Mail 中的 send() 函数使用 current_app，因此必须激活程序上下文。不过， 在不同线程中执行 mail.send() 函数时，程序上下文要使用 app.app_context() 人工创建。
# 不过要记住，程序要发送大量电子邮件时，使 用专门发送电子邮件的作业要比给每封邮件都新建一个线程更合适。例如，我们可以把执 行 send_async_email() 函数的操作发给 [Celery](http://www.celeryproject.org/) 任务队列。
```

###  大型程序的结构
#### 大型程序的结构
```
多文件 Flask 程序的基本结构
|-flasky 
     |-app/
         |-templates/
         |-static/
         |-main/
           |-__init__.py
           |-errors.py
           |-forms.py
           |-views.py
         |-__init__.py
         |-email.py
         |-models.py
       |-migrations/
       |-tests/
          |-__init__.py
          |-test*.py
       |-venv/
       |-requirements.txt 
       |-config.py 
       |-manage.py
这种结构有 4 个顶级文件夹:
• Flask 程序一般都保存在名为 app 的包中;
• 和之前一样，migrations文件夹包含数据库迁移脚本;
• 单元测试编写在tests包中;
• 和之前一样，venv文件夹包含Python虚拟环境。
同时还创建了一些新文件:
• requirements.txt 列出了所有依赖包，便于在其他电脑中重新生成相同的虚拟环境;
• config.py 存储配置;
• manage.py 用于启动程序以及其他的程序任务。
```

#### 配置选项 
```
# 程序经常需要设定多个配置。这方面最好的例子就是开发、测试和生产环境要使用不同的 数据库，这样才不会彼此影响。
# config.py 程序配置
import os 
basedir  = os.path.abspath(os.path.dirname(__file__))

class Config:
    SECRET_KEY = os.environ.get('SECRET_KEY') or 'hard to guess string' 
    SQLALCHEMY_COMMIT_ON_TEARDOWN = True
    FLASKY_MAIL_SUBJECT_PREFIX = '[Flasky]'
    FLASKY_MAIL_SENDER = 'Flasky Admin <flasky@example.com>' 
    FLASKY_ADMIN = os.environ.get('FLASKY_ADMIN')
    
    @staticmethod
    def init_app(app):
        pass

    class DevelopmentConfig(Config):
          DEBUG = True
          MAIL_SERVER = 'smtp.googlemail.com'
          MAIL_PORT = 587
          MAIL_USE_TLS = True
          MAIL_USERNAME = os.environ.get('MAIL_USERNAME')
          MAIL_PASSWORD = os.environ.get('MAIL_PASSWORD')
          SQLALCHEMY_DATABASE_URI = os.environ.get('DEV_DATABASE_URL') or \
              'sqlite:///' + os.path.join(basedir, 'data-dev.sqlite')

    class TestingConfig(Config):
          TESTING = True
          SQLALCHEMY_DATABASE_URI = os.environ.get('TEST_DATABASE_URL') or \
              'sqlite:///' + os.path.join(basedir, 'data-test.sqlite')

    class ProductionConfig(Config):
          SQLALCHEMY_DATABASE_URI = os.environ.get('DATABASE_URL') or \
              'sqlite:///' + os.path.join(basedir, 'data.sqlite')

    config = {
        'development': DevelopmentConfig,
        'testing': TestingConfig, 
        'production': ProductionConfig,
        'default': DevelopmentConfig 
    }

# 基类 Config 中包含通用配置，子类分别定义专用的配置。如果需要，你还可添加其他配 置类。
# 为了让配置方式更灵活且更安全，某些配置可以从环境变量中导入。例如，SECRET_KEY 的值， 这是个敏感信息，可以在环境中设定，但系统也提供了一个默认值，以防环境中没有定义。
# 配置类可以定义 init_app() 类方法，其参数是程序实例。在这个方法中，可以执行对当前 环境的配置初始化。现在，基类 Config 中的 init_app() 方法为空。
```

#### 使用工厂函数
```
工厂函数：(python 核心编程) 工厂函数看上去有点像函数，实质上他们是类，当你调用他们时，实质上是生产了该类型的一个实例，就像工厂生产货物。
```
```
# 在单个文件中开发程序很方便，但却有个很大的缺点，因为程序在全局作用域中创建，所 以无法动态修改配置。运行脚本时，程序实例已经创建，再修改配置为时已晚。这一点对 单元测试尤其重要，因为有时为了提高测试覆盖度，必须在不同的配置环境中运行程序。     
# 这个问题的解决方法是延迟创建程序实例，把创建过程移到可显式调用的工厂函数中。这 种方法不仅可以给脚本留出配置程序的时间，还能够创建多个程序实例，这些实例有时在 测试中非常有用。程序的工厂函数在 app 包的构造文件中定义
# 构造文件导入了大多数正在使用的 Flask 扩展。由于尚未初始化所需的程序实例，所以没 有初始化扩展，创建扩展类时没有向构造函数传入参数。create_app() 函数就是程序的工 厂函数，接受一个参数，是程序使用的配置名。配置类在 config.py 文件中定义，其中保存 的配置可以使用Flask app.config配置对象提供的from_object()方法直接导入程序。至 于配置对象，则可以通过名字从 config 字典中选择。程序创建并配置好后，就能初始化 扩展了。在之前创建的扩展对象上调用 init_app() 可以完成初始化过程。

# app/__init__.py

from flask import Flask, render_template
from flask.ext.bootstrap import Bootstrap
from flask.ext.mail import Mail
from flask.ext.sqlalchemy import SQLAlchemy
from config import config 

bootstrap = Bootstrap()
mail      = Mail()
moment    = Moment()
db        = SQLAlchemy()

def create_app(config_name):
    app   = Flask(__name__)
    app.config.from_object(config[config_name])
    config[config_name].init_app(app)

    bootstrap.init_app(app)
    mail.init_app(app)
    moment.init_app(app)
    db.init_app(app)

    # 附加路由和自定义的错误页面

    return app

# 工厂函数返回创建的程序示例，不过要注意，现在工厂函数创建的程序还不完整，因为没 有路由和自定义的错误页面处理程序。
```

####  在蓝本中实现程序功能
- 蓝本理解     
    - [知乎](https://www.zhihu.com/question/31748237)     
    - [explore flask](https://spacewander.github.io/explore-flask-zh/7-blueprints.html)     
```
# 转换成程序工厂函数的操作让定义路由变复杂了。在单脚本程序中，程序实例存在于全 局作用域中，路由可以直接使用 app.route 修饰器定义。但现在程序在运行时创建，只 有调用 create_app() 之后才能使用 app.route 修饰器，这时定义路由就太晚了。和路由 一样，自定义的错误页面处理程序也面临相同的困难，因为错误页面处理程序使用 app. errorhandler 修饰器定义。
# 幸好 Flask 使用蓝本提供了更好的解决方法。蓝本和程序类似，也可以定义路由。不同的 是，在蓝本中定义的路由处于休眠状态，直到蓝本注册到程序上后，路由才真正成为程序 的一部分。使用位于全局作用域中的蓝本时，定义路由的方法几乎和单脚本程序一样。
# 和程序一样，蓝本可以在单个文件中定义，也可使用更结构化的方式在包中的多个模块中 创建。为了获得最大的灵活性，程序包中创建了一个子包，用于保存蓝本

# app/main/__init__.py 
from flask import Blueprint 

main = Blueprint('main', __name__)

from . import  view, errors 

# 通过实例化一个 Blueprint 类对象可以创建蓝本。这个构造函数有两个必须指定的参数: 蓝本的名字和蓝本所在的包或模块。和程序一样，大多数情况下第二个参数使用 Python 的 __name__ 变量即可。
# 程序的路由保存在包里的 app/main/views.py 模块中，而错误处理程序保存在 app/main/ errors.py 模块中。导入这两个模块就能把路由和错误处理程序与蓝本关联起来。注意，这 些模块在 app/main/__init__.py 脚本的末尾导入，这是为了避免循环导入依赖，因为在 views.py 和 errors.py 中还要导入蓝本 main。

# 蓝本在工厂函数 create_app() 中注册到程序上

# app/__init__.py 
def create_app(config_name):
    # ...
    from .main import main as main_blueprint 
    app.register_blueprint(main_blueprint)

    return app

# app/main/errors.py
from flask import render_template
from .     import main

@main.app_errorhandler(404)
def page_not_found(e):
    return render_template('404.html'),404

@main.app_errorhandler(500)
def page_not_found(e):
    return render_template('500.html'),500

# 在蓝本中编写错误处理程序稍有不同，如果使用 errorhandler 修饰器，那么只有蓝本中的错误才能触发处理程序。要想注册程序全局的错误处理程序，必须使用 app_errorhandler。

# app/main/views.py
from datetime import datetime 
from flask    import render_template, session, redirect, url_for

from .        import main
from .forms   import NameForm
from ..       import db
from ..modles import User

@main.route('/', methods=['GET', 'POST'])
def index():
    form = NameForm()
    if form.validate_on_submit():
        # ...
        return redirect(url_for('.index'))
    return render_template('index.html',
                            form=form, name=session.get('name'),
                            known=session.get('known', False),
                            current_time=datetime.utcnow())
# 在蓝本中编写视图函数主要有两点不同:第一，和前面的错误处理程序一样，路由修饰器 由蓝本提供;第二，url_for() 函数的用法不同。你可能还记得，url_for() 函数的第一 个参数是路由的端点名，在程序的路由中，默认为视图函数的名字。例如，在单脚本程序 中，index() 视图函数的 URL 可使用 url_for('index') 获取。
# 在蓝本中就不一样了，Flask 会为蓝本中的全部端点加上一个命名空间，这样就可以在不同的蓝本中使用相同的端点名定义视图函数，而不会产生冲突。命名空间就是蓝本的名字 (Blueprint 构造函数的第一个参数)，所以视图函数 index() 注册的端点名是 main.index，其 URL 使用 url_for('main.index') 获取。
# url_for() 函数还支持一种简写的端点形式，在蓝本中可以省略蓝本名，例如 url_for('. index')。在这种写法中，命名空间是当前请求所在的蓝本。这意味着同一蓝本中的重定向 可以使用简写形式，但跨蓝本的重定向必须使用带有命名空间的端点名。
# 为了完全修改程序的页面，表单对象也要移到蓝本中，保存于 app/main/forms.py 模块。
```

#### 启动脚本
```
# manage.py 

#!/usr/bin/env python
import os
from app        import create_app, db
from app.models import User, Role
from flask.ext.script   import Manager, Shell
from flask.ext.migarate import Migrate, MigrateCommand

app     = create_app(os,get('FLASK_CONFIG') or 'default')
manager = Manager(app)
migrate = Migrate(app, db)

def make_shell_context():
    return dict(app=app, db=db, User=User, Role=Role)
manager.add_command("shell", Shell(make_context=make_shell_context))
manager.add_command("db", MigrateCommand)

if __name__ == '__main__':
    manager.run()

```

#### 需求文件
```
# 程序中必须包含一个 requirements.txt 文件，用于记录所有依赖包及其精确的版本号。如果 要在另一台电脑上重新生成虚拟环境，这个文件的重要性就体现出来了，例如部署程序时 使用的电脑。pip 可以使用如下命令自动生成这个文件:
     (venv) $ pip freeze >requirements.txt

# 如果你要创建这个虚拟环境的完全副本，可以创建一个新的虚拟环境，并在其上运行以下 命令:
     (venv) $ pip install -r requirements.txt
```

#### 单元测试
```
# tests/test_basics.py
import unittest
from flask  import current_app
from app    import current_app,db

class BasicsTestCase(unitest.TestCase):
    def setUp(self):
        self.app         = create_app('testing')
        self.app_context = self.app.app_context()
        self.app_context.push()
        db.create_all()

    def tearDown(self):
        db.session.remove()
        db.drop_all()
        self.app_context.pop()

    def test_app_exists(self):
        self.assertFalse(current_app is None)

    def test_app_is_testing(self):
        self.assertFalse(current_app.config['TESTING'])

# 这个测试使用 Python 标准库中的 unittest 包编写。setUp() 和 tearDown() 方法分别在各 测试前后运行，并且名字以 test\_ 开头的函数都作为测试执行。
# setUp() 方法尝试创建一个测试环境，类似于运行中的程序。首先，使用测试配置创建程 序，然后激活上下文。这一步的作用是确保能在测试中使用 current_app，像普通请求一 样。然后创建一个全新的数据库，以备不时之需。数据库和程序上下文在 tearDown() 方法 中删除。
# 第一个测试确保程序实例存在。第二个测试确保程序在测试配置中运行。若想把 tests 文 件夹作为包使用，需要添加 tests/__init__.py 文件，不过这个文件可以为空，因为 unittest 包会扫描所有模块并查找测试。

# 为了运行单元测试，你可以在 manage.py 脚本中添加一个自定义命令
# manage.py
@manager.command
def test():
    """Run the unit tests."""
    import unittest
    test  =  unittest.TestLoader().discover('tests')
    unittest.TextTestRunner(verbosity=2).run(tests)

# manager.command 修饰器让自定义命令变得简单。修饰函数名就是命令名，函数的文档字符 串会显示在帮助消息中。test() 函数的定义体中调用了 unittest 包提供的测试运行函数。
```

#### 创建数据库
```
# 重组后的程序和单脚本版本使用不同的数据库。
# 首选从环境变量中读取数据库的 URL，同时还提供了一个默认的 SQLite 数据库做备用。3 种配置环境中的环境变量名和 SQLite 数据库文件名都不一样。例如，在开发环境中，数据 库 URL 从环境变量 DEV_DATABASE_URL 中读取，如果没有定义这个环境变量，则使用名为 data-dev.sqlite 的 SQLite 数据库。
# 不管从哪里获取数据库 URL，都要在新数据库中创建数据表。如果使用 Flask-Migrate 跟 踪迁移，可使用如下命令创建数据表或者升级到最新修订版本:
     (venv) $ python manage.py db upgrade
```



### Tips
####  flask-wtf
- Web表单  `它主要在我们的网页中扮演着数据采集的功能`
- Flask-WTF offers simple integration with WTForms. This integration includes optional CSRF handling for greater security.     
- [somelink](http://krzer.com/2016/11/27/flask-wtf-introduction/)

####  编程中什么是「Context(上下文)」？
 每一段程序都有很多外部变量。只有像Add这种简单的函数才是没有外部变量的。一旦你的一段程序有了外部变量，这段程序就不完整，不能独立运行。你为了使他们运行，就要给所有的外部变量一个一个写一些值进去。这些值的集合就叫上下文。


#### sqlalchemy  `enum`
```
from flask import Flask
from flask_sqlalchemy import SQLAlchemy
import enum

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = 'mysql://root:root1234@localhost/kaka_db'
app.config['SQLALCHEMY_TRACK_MODIFICATIONS']=True
db = SQLAlchemy(app)

class UserType(enum.Enum):
    puTongUser      = 0
    guanLiYuan      = 1
    superGuanLiYuan = 2
    changJia        = 3

class User(db.Model):
    __talbename__ = 'user_table'
    id          = db.Column(db.Integer, primary_key=True, autoincrement=True)
    userName    = db.Column(db.String(80), unique=True, nullable=False)
    passWord    = db.Column(db.String(80), nullable=False)
    phone       = db.Column(db.String(80))
    email       = db.Column(db.String(80), unique=True)
    userType    = db.Column(db.Enum(UserType))
    code        = db.Column(db.String(80), unique=True)
    pushToken   = db.Column(db.String(80))
    token       = db.Column(db.String(80))
    regiserType = db.Column(db.Integer, unique=True)
    userMoney   = db.Column(db.Float)

    def __init__(self, username, password, phone = None, email = None, code = None, pushToken = None, userType = 0, registerType = 0, userMoney = 0.0):
        self.userName = username
        self.passWord = password
        self.phone = phone
        self.email = email
        self.code = code
        self.pushToken = pushToken
        self.userType = userType
        self.regiserType = registerType
        self.userMoney = userMoney

db.create_all()
db.session.commit()


if __name__ == '__main__':
    app.run(debug = True)

```
