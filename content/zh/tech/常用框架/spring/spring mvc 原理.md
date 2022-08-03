---
title: "Spring MVC 处理分发请求原理"
slug: "spring-mvc-request-principle"
date: "2022-07-12T23:00:00+23:00"
tags: ["Spring"]
---

在 Spring MVC 中是通过 DispatcherServlet 前端控制器来完成所有 Web 请求的转发、匹配、数据处理后，并转由页面进行展示，是 Spring MVC 实现中最核心的部分。

![image-20220712234222111](https://raw.githubusercontent.com/Coder-itCheng/blog-images/master/blog/image-20220712234222111-16576405442821.png "Spring MVC 分发请求流程图")

## DispatcherServlet

DispatcherServlet 是继承的 FrameworkServlet，而 FrameworkServlet 是 HttpServlet 的子类，FrameworkServlet 对诸如 doGet、doPost 等所有放法进行了重写，都调用 `processRequest` 方法，让后调用子类 DispatcherServlet 中 `doService`方法，最终调用 `doDispatch` 方法处理转发。

```java
public class DispatcherServlet extends FrameworkServlet {
    protected void doService(HttpServletRequest request, HttpServletResponse response) throws Exception {
        logRequest(request);
        Map<String, Object> attributesSnapshot = null;
        if (WebUtils.isIncludeRequest(request)) {
            attributesSnapshot = new HashMap<>();
            Enumeration<?> attrNames = request.getAttributeNames();
            while (attrNames.hasMoreElements()) {
                String attrName = (String) attrNames.nextElement();
                if (this.cleanupAfterInclude || attrName.startsWith(DEFAULT_STRATEGIES_PREFIX)) {
                    attributesSnapshot.put(attrName, request.getAttribute(attrName));
                }
            }
        }
        request.setAttribute(WEB_APPLICATION_CONTEXT_ATTRIBUTE, getWebApplicationContext()); // 把Spring上下文对象存放到Request的attribute中
        request.setAttribute(LOCALE_RESOLVER_ATTRIBUTE, this.localeResolver); // 把Spring国际化支持解析器对象存放到Request的attribute中
        request.setAttribute(THEME_RESOLVER_ATTRIBUTE, this.themeResolver); // 主题解析器对象
        request.setAttribute(THEME_SOURCE_ATTRIBUTE, getThemeSource()); // 主题对象
        if (this.flashMapManager != null) {
            FlashMap inputFlashMap = this.flashMapManager.retrieveAndUpdate(request, response);
            if (inputFlashMap != null) {
                request.setAttribute(INPUT_FLASH_MAP_ATTRIBUTE, Collections.unmodifiableMap(inputFlashMap));
            }
            request.setAttribute(OUTPUT_FLASH_MAP_ATTRIBUTE, new FlashMap());
            request.setAttribute(FLASH_MAP_MANAGER_ATTRIBUTE, this.flashMapManager);
        }

        try {
            doDispatch(request, response); // 真正的进行处理转发
        } finally {
            if (!WebAsyncUtils.getAsyncManager(request).isConcurrentHandlingStarted()) {
                if (attributesSnapshot != null) {
                    restoreAttributesAfterInclude(request, attributesSnapshot);
                }
            }
        }
    }
    protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
        HttpServletRequest processedRequest = request;
        HandlerExecutionChain mappedHandler = null; // 声明一个处理器执行链
        boolean multipartRequestParsed = false;
        WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
        try {
            ModelAndView mv = null;
            Exception dispatchException = null;
            try {
                processedRequest = checkMultipart(request); // 检查request对象判断请求是不是文件上传的请求
                // 判断是不是文件上传请求，若是则返回的processedRequest是MultipartHttpServletRequest，显然和原始的request对象不是同一个对象
                multipartRequestParsed = (processedRequest != request);
                // 从当前的请求中推断出HandlerExecuteChain处理器执行链对象
                mappedHandler = getHandler(processedRequest);
                if (mappedHandler == null) {
                    noHandlerFound(processedRequest, response);
                    return;
                }
                // 根据Handler选择HandlerAdpater对象，默认是@RequestMappingHandlerAdapter对象
                HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());
                String method = request.getMethod();
                boolean isGet = "GET".equals(method);
                if (isGet || "HEAD".equals(method)) {
                    long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
                    if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
                        return;
                    }
                }
                //触发拦截器的pre方法，返回false，就不进行处理了
                if (!mappedHandler.applyPreHandle(processedRequest, response)) {
                    return;
                }
                // 通过适配器真正的调用目标方法，RequestMappingHandlerAdapter.handle=>AbstractHandlerMethodAdapter#handle(HttpServletRequest, HttpServletResponse,Object)
                mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
                if (asyncManager.isConcurrentHandlingStarted()) {
                    return;
                }
                applyDefaultViewName(processedRequest, mv);
                // 触发拦截器链的post方法
                mappedHandler.applyPostHandle(processedRequest, response, mv);
            } catch (Exception ex) {
                dispatchException = ex;
            } catch (Throwable err) {
                dispatchException = new NestedServletException("Handler dispatch failed", err);
            }
            // 处理目标方法返回的结果，主要是渲染视图
            processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
        } catch (Exception ex) {// 抛出异常:处理拦截器的afterCompletion方法
            triggerAfterCompletion(processedRequest, response, mappedHandler, ex);
        } catch (Throwable err) {// 抛出异常:处理拦截器的afterCompletion方法
            triggerAfterCompletion(processedRequest, response, mappedHandler, new NestedServletException("Handler processing failed", err));
        } finally {
            if (asyncManager.isConcurrentHandlingStarted()) {
                if (mappedHandler != null) {
                    mappedHandler.applyAfterConcurrentHandlingStarted(processedRequest, response);
                }
            } else {
                if (multipartRequestParsed) {
                    cleanupMultipart(processedRequest); // 清除文件上传时候生成的临时文件
                }
            }
        }
    }
}
```

## HandlerMapping

在初始化完成时，在上下文环境中已定义的所有 HandlerMapping 都已被加载放到一个排好序的 List 中，存储着 HTTP 请求对应的映射数据，每个 HandlerMapping 可持有一系列从 URL 请求到 Controller 的映射。对于不同 Web 请求的映射，Spring MVC 提供了不同的 HandlerMapping 的实现，可让应用开发选取不同的映射策略，XML 方式通过在 `web.xml` 中配置的方式注册 Bean 默认使用 BeanNameUrlHandlerMapping 映射策略，通过 @Controller、@RequestMapping 注解的方式使用 RequestMappingHandlerMapping 映射策略。

```java
protected HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
   // Web容器中配置的所有的handlerMapping集合对象在本类中的initHandlerMappings()方法为DispatcherServlet类初始化赋值handlerMappings集合
   if (this.handlerMappings != null) {
      // 循环遍历所有的handlerMappings对象，依次调用handlerMappings的getHandler(request)来获取处理器执行链对象
      for (HandlerMapping hm : this.handlerMappings) {
         if (logger.isTraceEnabled()) {
            logger.trace("Testing handler map [" + hm + "] in DispatcherServlet with name '" + getServletName() + "'");
         }
         // 依次循环调用HandlerMapping的getHandler方法进行获取HandlerExecutionChain，调用所有的HandlerMapping的父类的AbstractHandlerMapping#getHandler(request)
         HandlerExecutionChain handler = hm.getHandler(request);
         if (handler != null) {
            return handler;
         }
      }
   }
   // 通过所有的handlerMapping对象 还没有获取到对应的HandlerExecutionChain，则认为该请求无法匹配
   return null;
}
```

首先通过 `UrlPathHelper` 对象解析出 request 中请求路径，让后通过该路径到 RequestMappingHanlderMapping 的路径映射注册表 `mappingRegistry` 中匹配，匹配到后通过 `getHandlerExecutionChain` 将其封装成一个 `HandlerExecutionChain`，然后匹配拦截器规则，将满足条件的拦截器添加到该封装对象中。

```java
public abstract class AbstractHandlerMapping extends WebApplicationObjectSupport implements HandlerMapping, Ordered {
    public final HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
        // 找到处理器对象，在本子类的AbstractHandlerMapping的子类RequestMappingHanlderMapping的生命周期回调接口InitializingBean中会把@RequestMapping注解信息和方法映射对象保存到路径映射注册表中
        Object handler = getHandlerInternal(request);
        if (handler == null) { // 判断上一步的handler是否为空
            handler = getDefaultHandler(); // 返回默认的handler
        }
        if (handler == null) {
            return null;
        }
        if (handler instanceof String) { // 若解析出的handler是String则通过Web容器创建handler对象
            String handlerName = (String) handler;
            handler = obtainApplicationContext().getBean(handlerName);
        }
        // 根据处理器来构建处理器执行链对象
        HandlerExecutionChain executionChain = getHandlerExecutionChain(handler, request);
        if (CorsUtils.isCorsRequest(request)) { // 处理跨域
            CorsConfiguration globalConfig = this.globalCorsConfigSource.getCorsConfiguration(request);
            CorsConfiguration handlerConfig = getCorsConfiguration(handler, request);
            CorsConfiguration config = (globalConfig != null ? globalConfig.combine(handlerConfig) : handlerConfig);
            executionChain = getCorsHandlerExecutionChain(request, executionChain, config);
        }
        return executionChain;
    }
    protected HandlerExecutionChain getHandlerExecutionChain(Object handler, HttpServletRequest request) {
        // 创建处理器执行链对象
        HandlerExecutionChain chain = (handler instanceof HandlerExecutionChain ? (HandlerExecutionChain) handler : new HandlerExecutionChain(handler));
        // 从请求中获取请求映射路径
        String lookupPath = this.urlPathHelper.getLookupPathForRequest(request);
        // 循环获取所有的拦截器对象
        for (HandlerInterceptor interceptor : this.adaptedInterceptors) {
            // 判断拦截器对象是不是实现HandlerInterceptor
            if (interceptor instanceof MappedInterceptor) {
                MappedInterceptor mappedInterceptor = (MappedInterceptor) interceptor;
                // 通过路径匹配看该拦截器是否会拦截本次请求路径
                if (mappedInterceptor.matches(lookupPath, this.pathMatcher)) {
                    chain.addInterceptor(mappedInterceptor.getInterceptor());
                }
            } else {
                chain.addInterceptor(interceptor);
            }
        }
        return chain; // 返回我们的拦截器链执行器对象
    }
}
public abstract class AbstractHandlerMethodMapping<T> extends AbstractHandlerMapping implements InitializingBean {
    protected HandlerMethod getHandlerInternal(HttpServletRequest request) throws Exception {
        // 获取UrlPathHelper对象，用于解析从request中解析出请求映射路径
        String lookupPath = getUrlPathHelper().getLookupPathForRequest(request);
        this.mappingRegistry.acquireReadLock(); // 通过映射注册表获取lock对象
        try {// 通过从Request对象中解析出来的lookupPath然后通过lookupPath获取HandlerMethod对象
            HandlerMethod handlerMethod = lookupHandlerMethod(lookupPath, request);
            return (handlerMethod != null ? handlerMethod.createWithResolvedBean() : null);
        } finally {// 释放锁对象
            this.mappingRegistry.releaseReadLock();
        }
    }
    protected HandlerMethod lookupHandlerMethod(String lookupPath, HttpServletRequest request) throws Exception {
        List<Match> matches = new ArrayList<>();
        // 根据路径去MappingRegistry注册表中的urlLookup的map对象中获取RequestMappingInfo对象，mappingRegistry在RequestMappingHandlerMapping的bean的初始化方法就进行解析保存到注册表中
        List<T> directPathMatches = this.mappingRegistry.getMappingsByUrl(lookupPath);
        // 判断通过解析出来的lookUpPath解析出来的RequestMappingInfo不为空 把RequestMappingInfo封装成为Match保存到集合中
        if (directPathMatches != null) {
            addMatchingMappings(directPathMatches, matches, request);
        }
        if (matches.isEmpty()) {
            addMatchingMappings(this.mappingRegistry.getMappings().keySet(), matches, request);
        }
        if (!matches.isEmpty()) {
            Comparator<Match> comparator = new MatchComparator(getMappingComparator(request)); // 创建Match的匹配器对象
            matches.sort(comparator); // 对匹配器进行排序
            Match bestMatch = matches.get(0); // 默认选择第一个为最匹配的
            if (matches.size() > 1) {
                if (CorsUtils.isPreFlightRequest(request)) {
                    return PREFLIGHT_AMBIGUOUS_MATCH;
                }
                Match secondBestMatch = matches.get(1); // 获取第二最匹配的
                if (comparator.compare(bestMatch, secondBestMatch) == 0) { // 若第一个和第二个是一样的则抛出异常
                    Method m1 = bestMatch.handlerMethod.getMethod();
                    Method m2 = secondBestMatch.handlerMethod.getMethod();
                    throw new IllegalStateException("Ambiguous handler methods mapped for HTTP path '" + request.getRequestURL() + "': {" + m1 + ", " + m2 + "}");
                }
            }
            request.setAttribute(BEST_MATCHING_HANDLER_ATTRIBUTE, bestMatch.handlerMethod); // 把最匹配的设置到request中
            handleMatch(bestMatch.mapping, lookupPath, request);
            return bestMatch.handlerMethod; // 返回最匹配的
        } else {
            return handleNoMatch(this.mappingRegistry.getMappings().keySet(), lookupPath, request);
        }
    }
}
```

> HandlerExecutionChain：HandlerInteceptor、HandlerInteceptor...、Handler

## HandlerAdapter

