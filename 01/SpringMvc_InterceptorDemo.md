# Spring MVC 拦截器实战-实现方法调用性能信息摘要输出
## handler的实现：
### 1. 实现HandlerInterceptor核心接口
```java
@Slf4j
public class PerformanceInterceptor implements HandlerInterceptor {
    private ThreadLocal<StopWatch> localStopWatch = new ThreadLocal<StopWatch>();

    /**
     * 拦截器预处理，方法调用前，返回true继续后续，false则直接返回
     * @param request
     * @param response
     * @param handler
     * @return
     * @throws Exception
     */
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        StopWatch stopWatch = new StopWatch();
        stopWatch.start();
        localStopWatch.set(stopWatch);
        return true;
    }

    /**
     * 处理器调用结束，视图解析之前 
     * @param request
     * @param response
     * @param handler
     * @param modelAndView
     * @throws Exception
     */
    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {
        StopWatch stopWatch = localStopWatch.get();
        stopWatch.stop();
        stopWatch.start();
    }

    /**
     * 视图解析之后
     * @param request
     * @param response
     * @param handler
     * @param ex
     * @throws Exception
     */
    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
        StopWatch stopWatch = localStopWatch.get();
        stopWatch.stop();
        String method = handler.getClass().getSimpleName();
        if(handler instanceof HandlerMethod){
            String beanType = ((HandlerMethod)handler).getBeanType().getName();
            String methodName = ((HandlerMethod)handler).getMethod().getName();
            method = beanType+"."+methodName;
        }
        log.info("{};{};{};{};{}ms;{}ms;{}ms", request.getRequestURI(), method,
                response.getStatus(), ex == null ? "-" : ex.getClass().getSimpleName(),
                stopWatch.getTotalTimeMillis(), stopWatch.getTotalTimeMillis() - stopWatch.getLastTaskTimeMillis(),
                stopWatch.getLastTaskTimeMillis());
    }
}
```
### 拦截器注册到环境中
通过`WebMvcConfigurer`接口的`addInterceptors`方法实现注册
```java
@SpringBootApplication 
public class MvcDemoApplication implements WebMvcConfigurer {

    public static void main(String[] args) {
        SpringApplication.run(MvcDemoApplication.class,args);
    }

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new PerformanceInterceptor())
                .addPathPatterns("/coffee/**").addPathPatterns("/order/**");
    }
}
```
### 运行结果展示：
```java
2020-01-19 14:59:08.718  INFO 8692 --- [nio-8080-exec-4] c.n.s.i.PerformanceInterceptor           : /coffee/1;com.nd.springbucks.controller.CoffeeController.findById;200;-;67ms;65ms;2ms
```