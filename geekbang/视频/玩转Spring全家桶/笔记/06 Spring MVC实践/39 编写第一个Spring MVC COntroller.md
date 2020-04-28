1. 认识Spring MVC，核心类就是DispatcherServlet，是所有请求的路口
 - Controller，就是我们常写的Controller类
 - xxxResoler，解析器，有ViewResolver、HandlerExceptionResolver、MultipartResolver等
 - HandlerMapping，就是根据请求路由到指定的Controller

2. @ResponseStatus(HttpStatus.CREATED)，用来指定返回码