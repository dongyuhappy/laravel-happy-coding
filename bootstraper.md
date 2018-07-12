
## LoadEnvironmentVariables，加载环境变量，默认为 .env 文件

- 判断配置缓存文件是否存在，也就是`boostrap/cache/config.php是否存在，如果已经存在就不会再次去解析 .env 文件;
- Application 应用对象上保留了 .env 文件的路径和文件名称，创建 `Dotenv` 对象，调用其 `load` 方法去读取解析加载环境变量数据，`Dotenv` 是属于第三方库 `vlucas/phpdotenv`；
- 判断 .env 文件是否可读;
- 使用 `file` 函数去读取 .env 文件，忽略 `EOL` 和空白行;
- 以 `#` 开头的为注释行，会被过滤掉；
- 行中含有 `=` 被认为是环境变量；
- 对环境变量和值做处理，这部分可阅读源码；
- 对于已经存在的环境变量，或者系统环境变量做忽略处理；
- 设置环境变量，具体包括 `apache_setenv`,`putenv`,`$_ENV`,`$_SERVER`。

有兴趣的同学可以关注下 `normaliseEnvironmentVariable` 方法，这里面就包含了嵌套配置`ALIAS_APP_NAME={$APP_NAME}` 的实现方式。


## LoadConfiguration 配置的加载

- 判断缓存文件 `boostrap/cache/config.php` 是否存在，如果存在直接 `require` 加载进来，然后标记 `$loadedFfromCache=true`；
- 创建 `Repository` 对象作为实例保存在 Application 容器；
- 如果 `$loadedFromCache` 不为 `true` 就会去加载真正的配置文件；
- 使用 `app.env` 的配置值作为 Application 的运行环境；
- 使用 `app.timezone` 设置 时区；
- 设置内部编码为 `UTF-8`；    

通过上面的过程我们可以看出：

- 配置数据对象 `Repository` 的实例以`config` 这个名字保存在 Application 容器对象上； 
- 配置文件是有缓存文件的，也就是把多个配置文件合并起来生成一个新的缓存配置文件；
- 时区和编码是在配置文件加载之后设置的；


加载配置文件的过程如下：

- 在创建 Application 的的时候在容器上保存了一个名为 `path.config` 的变量，这个就是配置文件的路径；
- 加载 `path.config` 下以 ` \*.php` 结尾的文件，然后使用 `Reposiory` 对象的 `set` 来加载设置配置数据；
- 由于 `Repository` 对象已经保存到 Application 容器对象上了，所以可以通过 `config` 标志从容器里面获取配置数据数据，然后在获取具体配置的值。

一般我们通过 `config` 函数来访问配置文件的数据,通过 `env` 函数来访问环境变量数据。

关于查找配置文件的细节可以关注 `symfony/finder` 这个扩展包，基础只是方面可以了解下 SPL 的 `DirectoryIterator` 类。

以上就是加载配置文件的过程，并没有看到相关生成配置缓存文件的逻辑，也就是在默认情况下是不会合并配置缓存文件的。

讨论：其实默认去变量配置文件目录下的 ** 全部 ** 配置文件并不是必须的，可以改为用的时候去判断要获取的数据是否在 `Repository`  对象上面，如果不存在才去加载，然后再保存到 `Repository` 对象上。


## HandleExceptions 异常处理设置

- 使用 `error_reporting(-1)` 包括所有的错误；
- `set_error_handler` 设置用户自定义错误处理函数，具体来说就是把错误转换为 `ErrorException` 异常对象抛出来；
- `set_exception_handler` 设置异常处理自定义函数，本质来说就是把异常交给在Application对象上注册的`Illuminate\Contracts\Debug\ExceptionHandler::class` 对象去处理，首先会使用 `report`方法去记录这个日志，然后`render` 去把异常信息输出给客户端；
- `register_shutdown_function` 注册脚本执行完成之后的回掉，主要还是对出现了异常的处理。

至此关于错误与异常的系统级别的处理已经设置完毕，但在这之前出现的未被处理的异常和错误不会进入这里设置的异常错误处理函数。


## RegisterFacades Facades（门脸/别名）系统

这个系统的核心是利用`spl_autoload_register`注册一个当被加载的类不存在的时候的回掉函数，它会去配置的别名列表里面找到真正的类，然后利用`class_alias` 做一个别名class的别名映射,最后利用每个Facades里面的 `getFacadeAccessor` 返回的实际对象在 Application 对象的标志，最后使用Application创建出具体的对象，然后调用其方法。个人是比较反对这种方式的，无论是在沟通还是代码结构的清晰程度上都显得复杂了不少。


## RegisterProviders 注册 ServiceProvider（服务提供者）

整个过程可以概括为，把框架，扩展包，项目的服务器提供者合并为一个数组（注意顺序），然后依次创建对象，执行其 `register` 方法。

Service Provider 按照 `$defer` 的不同可以分为及时加载和延迟加载。延迟加载的实现方式在是 Application 去 make 一个对象的时候，检查延迟服务提供者列表，如果存在就按照和普通提供者一样的方式去注册，创建。

## BootProviders 

目前功能是执行注册的 Service Provider 的 boot方法，可以在这个方法里面做一些关于 Service Provider 的初始化的操作。 





