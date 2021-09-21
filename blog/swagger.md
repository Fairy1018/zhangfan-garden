# flask搭配swagger使用轻松生成接口文档
1.Swagger 应用背景
2.Swagger 搭配 Flask 实现
    2.1 WSGI 回顾
    2.2 Restful 回顾
    2.3  flask-restplus重要类理解
3.flask-restplus && swagger 代码实现
4.Swagger UI 结果展示
5.总结
6.参考文献
<br>


## 一、Swagger 应用背景

上一个项目中，当 api 接口开发完并自测成功后，服务端同学会将 url、request param 和 response 人工添加到共享文档中，以方便客户端同学联调。

这样增加了额外的人工成本，除了手动添加过程有可能出错外，还带来了其他的麻烦。比如之后接口变更，共享文档也需要同步变动。这十分浪费开发时间。

swagger 很好的解决了以上问题。它能够自动生成 api 文档，并且以 html 页面的形式呈现 api 详细信息，查看更加直观。与此同时，swagger 还提供了运行入口，客户端同学能够直接在页面中验证输入和输出是否符合预期。

相比 postman ，swagger 更直观、更节省人力、更适合开发人员使用。 

## 二、Swagger 搭配 Flask 使用
python 提供了 flask-restplus 插件，能够很方便的实现 swagger UI。

 ### 2.1 WSGI 回顾
WSGI 全称是 Web Server Gateway Interface。
它封装了接收 http 请求、解析 http 请求、发送 http 响应的底层代码，开发人员无需关注 tcp 和 http 协议细节，只需专心写处理逻辑。
Flask 依赖的 WSGI 服务器是 Werkzeug。Werkzeug 是一个各种 WSGI 实用程序的集合。
一个 Flask 实例在启动后，会使用 Werkzeug 库中的 serving.BaseWSGIServer 处理请求。
### 2.2 Restful 回顾

Restful 全称是 Representational State Transfer，即表现层状态转化。
它是一种互联网软件架构。python 提供的 flask-restplus 插件则采用了Restful架构。
Restful中重要的概念如下：

* resource 资源

    指网络上的实体，或者具体的信息。一般通过 URI 访问（统一资源定位符，URL 是一种具体的表现形式）。

* Representation 表现层

    指资源的表现形式。比如文本常见的有 html、xml、json 格式。
    表现形式通过 http 请求头的 Content-type 和 Accept 字段指定。Flask 中常用 application/json 格式

*   State Transfer 状态转化

    指数据状态的转化。http 协议是无状态协议，一般由服务端保存状态。
    客户端通过操作让服务端的资源发生“状态转化”，常见的操作有：get、put、post、delete 
    理解了restful架构后，可以通过flask-restplus实现swagger了


### 2.3  flask-restplus 重要类理解
   首先掌握flask-restplus关键类的作用
	
*   **namespace**
	作用类似于 flask 中的 blueprints ，是一系列路由的集合和 app 相关的函数，之后函数会被注册到一个app实例上。
    它的特点是能够定义 app 的函数而无需提前定义 app 实例，并且使用与 Flask 相同的装饰器。

* 	**resource**

	表示一个抽象的 Restplus 资源。自定义的 resource 继承于 resource 类，并且暴露每个支持的 http 操作（如 get/post）。
    如果资源被一个不支持的 http 操作调用，api 会返回 405 响应（方法不被允许）。
    反之，如果 api 实例已添加自定义的 resource，此时调用正确的方法同时从url传入参数，则会对资源进行操作处理。
	 

* 	**model**

	是封装了 fields 的 dict 形式，用于存储 api 文档的元数据。model 能够集中管理 response ，同时显示在Swagger UI上。

## 三、flask-restplus && swagger 代码实现

### 3.1 生成namespace
`archive_api = Namespace('archive', description='archive API')`

### 3.2 生成metadata（model）
`archive_model = detective_api.model("archive_model", {`
` 'archiveData': fields.String(required=True, description="the serialization string"),`
`'archiveName': fields.String(required=True, description="the archive name,ex:EgyptArchive")`
`})`

### 3.3 对Resource类添加装饰器
`接口url     @api.route('/')`
`接口参数  url路径中参数   @api.doc(params={'UserId':"the description message"})`
`接口参数  header中参数   @api.expect(parser)  `
            `parser定义代码：parser=api.parser()  parser.add_argument('UserId',location='headers')`
`接口参数 body中参数      @api.doc(body=archive_model) `
`接口响应    @api.marshal_with(archive_model)`
    
### 3.4 创建 API
`detective_api = Api(version='1.0',title="Detective Server API",description="A authenticate doc for client-developer")`

### 3.5  添加命名空间 
` detective_api.add_namespace(archive_api,path='/v1/archive') `
注意：add_namespace 方法内部实现了注册 resource 到当前 api ，同时可通过 path参数 可以添加命名空间的 url 前缀

### 3.6 启动 Flask 实例
`app = Flask(__name__)`
`detective_api.init_app(app)`
`if __name__ == '__main__':`
`app.run(host='0.0.0.0', port=8000, debug=True)`
## 四、Swagger UI 结果展示
Swagger UI 的路径默认为域名的根路径，直接在浏览器中输入，可以看到：

![image.png](http://pfp.ps.netease.com/kmpvt/file/609e6dff6158bcbcd163ab41Wyh57vlg01?sign=Ge6XFD99zXys9n3Nw4vVJCpQAB0=&expire=1621268723)

点开 get 按钮，界面如下： 里面包含了get请求的 url、header 中的参数名和描述信息、响应值和描述

![image.png](http://pfp.ps.netease.com/kmpvt/file/609e6e8468d86444f0f5e585FltCtkrv01?sign=Ljzh93u_Fjyt2bUqZW-s4TVLGxM=&expire=1621268723)

点击右上角的try it out，可以填入参数，实际调用 url

![image.png](http://pfp.ps.netease.com/kmpvt/file/609e6f958c56743479a0e375x6VTQl9z01?sign=cTPLODkXj-lmQ8FPJXEpUCeuyHQ=&expire=1621268723)

## 五、总结
**在服务端同学自测或者和客户端同学联调时，可以使用 Swagger 进行。
Swagger 不需要额外的逻辑代码，能够自动生成接口文档。只需在浏览器打开界面即可，并且能够实现文档同步更新，无需手动维护。**

## 六、参考链接
- [理解RESTful架构](http://www.ruanyifeng.com/blog/2011/09/restful.html)
- [flask1.1.x doc](https://flask.palletsprojects.com/en/1.1.x/tutorial/factory/)
- [flask-restplus doc](https://flask-restplus.readthedocs.io/en/stable/api.html?highlight=resource#flask_restplus.Resource)
- [WSGI接口](https://www.liaoxuefeng.com/wiki/1016959663602400/1017805733037760)

