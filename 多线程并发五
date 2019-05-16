
> 开篇：ThreadLocal就不多说了，想必这个大家都很熟悉了，实现单线程共享和安全的有效途径，下面是我在GitHub上的一篇浅析ThreadLocal！

[ThreadLocal的简单剖析](https://github.com/wrail/MultiThread/blob/master/Java%E5%A4%9A%E7%BA%BF%E7%A8%8B%E5%B9%B6%E5%8F%91%EF%BC%88%E4%B8%89%EF%BC%89.md)

> 目标：通过SpringBoot2.x的过滤器，拦截器和ThreadLocal实现简单的线程封闭。并且促进理解RequestContextHolder（持有上下文的Request容器，也是一个很好用的工具类，它的实现关键也是依赖于两个ThreadLocal）。

## 图解本文进行测试案例
![在这里插入图片描述](https://img-blog.csdnimg.cn/2019051622074148.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQyNjA1OTY4,size_16,color_FFFFFF,t_70)
## 实现步骤
> 根据实现流程，写实现步骤
### 1.自定义一个RequestHolder
使用ThreadLocal自定义一个简单类型的RequestHolder，并增加增删查的方法
```
package com.wrial.main.example.threadLocal;

public class RequestHolder {

    private final static ThreadLocal<Long> requestHolder = new ThreadLocal<>();

    public static void add(Long id){
        requestHolder.set(id);
    }
    public static void remove(){
        requestHolder.remove();
    }
    public static Long get(){
        return requestHolder.get();
    }

}

```
### 2.自定义一个过滤器（Filter）
在SpringBoot中定义一个过滤器，其实SpringBoot的思想就是手动加自动配置，理解其思想就很容易写出自定义代码。需要自定义类继承Filter并且在配置类中进行配置。
```
package com.wrial.main;

import com.wrial.main.example.threadLocal.RequestHolder;
import lombok.extern.slf4j.Slf4j;
import javax.servlet.*;
import javax.servlet.http.HttpServletRequest;
import java.io.IOException;

@Slf4j
public class HttpFilter implements Filter {


    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {

         //打印路径和线程id日志
        log.info("do filter,{},{}",servlet.getServletPath(),Thread.currentThread().getId());

        //将线程Id放在自定义的RequestHolder
        RequestHolder.add(Thread.currentThread().getId());
       
        filterChain.doFilter(servletRequest,servletResponse);
    }

}

```
配置类中进行配置(FilterRegistrationBean)
```
  @Bean
    public FilterRegistrationBean httpFilter(){
        FilterRegistrationBean registrationBean = new FilterRegistrationBean();
        registrationBean.setFilter(new HttpFilter());
        registrationBean.addUrlPatterns("/threadLocal/*");
        return registrationBean;
    }

```
这样我们自定义的简单过滤器就ok了。
### 3.自定义一个拦截器（Interceptor）
编写一个拦截器实现HandlerInterceptorAdaptor接口

```
package com.wrial.main;

import com.wrial.main.example.threadLocal.RequestHolder;
import lombok.extern.slf4j.Slf4j;
import org.springframework.web.servlet.handler.HandlerInterceptorAdapter;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

@Slf4j
public class HttpInterceptor extends HandlerInterceptorAdapter {

    //请求前
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        log.info("preHandle");
        return true;
    }

    //请求后，postHandle是正常处理
    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
        //移除RequestHolder中的线程Id
        RequestHolder.remove();

        log.info("afterCompletion");
        return;
    }
}

```
配置类，使用WebMvcConfigurer来添加我们自定义的拦截器
```
 @Bean
    public WebMvcConfigurer webMvcConfigurer() {
        WebMvcConfigurer webMvcConfigurer = new WebMvcConfigurer() {
            @Override
            public void addInterceptors(InterceptorRegistry registry) {
                //添加自定义拦截器，并模式匹配所以路径
                registry.addInterceptor(new HttpInterceptor()).addPathPatterns("/**");
            }
        };
        return webMvcConfigurer;
    }

```
> 在以前的SpringBoot中是用WebMvcConfigurerAdaptor，SpringBoot2.x不推荐使用。当然，那样配置也是可以生效的。
#### 4.编写ThreadLocalController
```
package com.wrial.main.example.threadLocal;

import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.ResponseBody;

@Controller
//此请求映射路径在拦截器范围内
@RequestMapping("/threadLocal")
public class ThreadLocalController {

    @RequestMapping("test")
    @ResponseBody
    public Long test(){

        return RequestHolder.get();
    }
}

```
配置方法2：

#### 5.PostMan接口测试
访问：http://localhost:8080/threadLocal/test
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190516223846339.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQyNjA1OTY4,size_16,color_FFFFFF,t_70)
后台输出结果
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190516224033169.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQyNjA1OTY4,size_16,color_FFFFFF,t_70)
可以看到我们定义的拦截器和过滤器已经在接口访问过程中起作用了。

