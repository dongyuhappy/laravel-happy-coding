本文尝试在代码的角度对 Laravel 的生命周期进行说明。

概括来说 Laravel 的生命周期由以下几个阶段组成：

- Application 初始化；
- Kernel 和 Exception Handler 注册到Application容器对象上，Kernel 分为 HttpKernel 和 ConsoleKernel；
- 使用 Application 对象创建 Kernel；
- 创建 Request 对象来包装客户端相关的请求数据;
- 把 Request 对象交予 Kernel 的 handle 方法处理，生成 Response 对象;
- 使用 Response 的 send 方法，把相关的输出返回给客户端；
- 调用 Kernel 的 terminate 方法进行相关的清理操作，至此整个请求结束。


上面在宏观的层面介绍了 Laravel 的生命周期，每个生命周期都实现了特定的功能，关于这些功能的细节将在下面的进行初步介绍。


## Application 初始化 

Application 是 Laravel 中最为核心的对象，如果没有它 Laravel 中的一切对象就都将不存在。

Application 对象是一个 **应用** **容器** 对象。换句话说它即是一个容器也是一个应用程序对象。作为一个表示「应用」的对象，提供了应用的基本信息，例如：路径相关，当前运行环境等相关基础信息；作为一个「容器」对象，提供了相关对象注册，创建，保存等相关的功能，在 Laravel 中各个组件中都可以非常方便的获取当前的 Application 对象，所以可以简单简单粗暴的把 Application 理解为一个全局的单例对象，所有需要全局共享的数据都可以保存到 Application 上。

Application 对象的初始化过程如下：

### 定义项目的目录结构

 设置基础路径 `$basePath` ，以 `$basePath` 路径为基础，** 定义了项目目录结构 **，并且把相关的目录结构保存到 Application 对象上。下文以`./` 表示基础路径 `$basePath`  具体的目录结构如下：

- `path`，对应的目录为 `./app`，里面包含了项目相关的核心代码，例如`Controler` 和 `Middleware` 都在这一层；
- `path.base`，目录为 `./` ，项目的基础目录；
- `path.resources` ，目录为 `./resources`， 模版文件和没有被编译过的css和js源码，图片等相关资源文件放这个这个目录，如果应用程序设计到多语音，那么多语言文件也在这个目录；
- `path.lang` ，目录为 `./resources/lang` ，多语言配置文件所在的目录；
- `path.public`，目录为 `./public`， web server 的根目录，web server 只能访问到这个目录，主要包含了项目的入口文件 index.php，编译好的 css 和 js 等相关资源文件都要拷贝的这个目录。
- `path.config`，目录为 `./config` ，配置文件所在的目录，
- `path.storage` 目录为 `./storage`，这个目录需要写入的权限，主要包含了3类在项目执行的过程中会产生的文件：缓存文件，session 文件，日志文件。
- `path.database`，目录为 `./database`，这个目录存放了数据库相关的文件，主要分为3类：数据库结构生成器文件，种子数据文件，创建种子数据工厂类文件。
- `path.bootstrap`，目录为 `./bootstrap`,这个目录包含了最外层的引导文件`app.php`，这个文件创建了 Application 对象，该目录的 `./bootstrap/cache` 还生成了缓存  `ServiceProvder`配置文件 


### 保存基础实例对象到 Application 上

设置了容器的单例对象，然后保存了两个实例对象到 Application 。

- `static::setInstance($this);` 把当前 Application 对象作为容器单例保存到容器
- `$this->instance(Container::class, $this);` 把当前 Application 对象保存到容器
- 创建 `PackageManifest` 对象，并把该数据保存到 Application 对象，这个对象的主要作用是分析 `./vendor/installed.json`  文件，把 laravel 相关的配置读取出来，例如 ServiceProvider等


### 注册基础的服务提供者（Service Provider）

注册了 3 个基础的服务提供者。
- EventServiceProvider，事件模块，事件模块是 Laravel 实现可扩展性的一个重要的工具;
- LogServiceProvider，日志模块;
- RoutingServiceProvider，路由模块，该模块的主要作用是把请求转发到具体的处理器上;

### 定义核心容器的相关别名

本质来说就是把别名和真正的类之间做了一层映射关系，我觉得这个别名系统的缺点大于优点，优点无非就是所谓的写起来“简洁”，“简洁”的同时会带来阅读上的不直观，缺点首先是需要循环去定义映射关系，其实是这个映射关系（数组）需要使用额外的内存是保存。


## Kernel 的创建

这里只介绍 HttpKernel 相关的功能和初始化的过程。

Kernel 是位于 Application 上层的对象，相关的初始化操作和请求的处理都是由 Kernel 来处理的。

### 初始化的过程

Kernel 是使用 Application 容器对象创建的。

- Kernel 的构造函数定义了 Application 和 Router 类型的参数，这两个参数在 Application 都在已经在 Application 上注册或者创建好了;
- 把 `$middlewarePriority` 赋值给 Router 对象，这类中间件由优先级的设置； 
- 把 `$routeMiddleware` 定义的分组中间件赋值给 Router 对象；
- 把 `$routeMiddleware` 别名保存到 Router 对象上。

根据上面几个步骤来看，中间件的执行都是在 Router 上实现的，Kernel 只是负责定义而已。


### Kernel 的 Handle 方法执行的流程

Handle 方法的参数需求是需要一个 `Request` 对象，这个 `Request`  对象在 `index.php` 已经创建。

具体的流程如下：

- 设置 `Request`  对象的 http 方法是可以通过 `_method` 参数覆盖的，目前我还发现这个功能的使用实际的使用场景；
- 把 `Request` 对象保存的 Application 容器里；
- ** 执行 `Kernel` 上定义的引导器列表，相关的自定义的初始化操作都是通过这个步骤来实现的，** 比如：环境变量和配置文件的加载；服务提供者的注册和引导等；
- 创建 `Pipeline` 对象，把 `Request` 对象交给 `Pipeline` 对象处理；处理顺序先由 `$middleware` 属性定义的中间件列表处理完之后交给 Router 对象处理，处理完成之后会生成一个 `Response` 对象；   

- 异常的处理，新版 PHP 所有可以抛出/捕获的异常都实现了 `Throwable` 接口，首先对 `Exception` 异常进行捕获，然后交给 `renderException` 方法去处理返回， 对于非 `Exception` 的可抛出/捕获的异常（实际就是Error）会把它包装为 `FatalThrowableError` 对象，然后再给 `renderException` 方法处理返回；

- 最后派发 `RequestHandled` 事件， 该事件表明对 `Request`的相关处理已经完成， 事件系统已经在 Application 对象初始化的时候已经创建，所以这里可以直接用 Application 取出来使用。


在上面介绍的步骤中其中一个叫做「执行 `Kernel` 上定义的 `$bootstrapers` 列表」。下面对相关的 `bootstrapers` 列表进行说明。  

### `$bootstrappers`（引导器） 列表

引导器具体的执行者是 Application 应用程序对象，为避免多次执行，Application 会在执行完成之后进行标记为已经引导。具体的引导过程如下：

- 标记 `$hasBeenBootstrapped` 属性为  `true`；
- 遍历引导器列表，每个引导器在之后之前会派发`bootstrapping:引导器名称`的事件；
- 然后使用 Application 容器对象创建引导器对象，然后调用引导器对象的 `bootstrap` 方法；
- 最后派发 `bootstrapped:引导器名称`的事件。

透过事件系统，来实现可扩展性在这里就能体现出来。



具体定义的引导器列表为：

- LoadEnvironmentVariables， 加载「.env」 文件中定义的环境变量;
- LoadConfiguration，加载配置文件；
- HandleExceptions，设置相关系统级别的异常处理；
- RegisterFacades，注册 Facades 列表；
- RegisterProviders，加载注册服务提供者（Service Provider）;
- BootProviders，执行已经注册的服务提供者的`boot`方法;

至此相关的引导器已经执行完成，在后面的文章中会对每个引导器详细的介绍。

