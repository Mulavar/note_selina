DispatcherServlet分发请求并调度各个组件处理得到结果返回给用户。



```java
public class DispatcherServlet extends FrameworkServlet {
	/** List of HandlerMappings used by this servlet. */
	@Nullable
	private List<HandlerMapping> handlerMappings;

	/** List of HandlerAdapters used by this servlet. */
	@Nullable
	private List<HandlerAdapter> handlerAdapters;
}
```

HandlerMapping：维护请求到HandlerExecutionChain的映射关系；

HandlerExecutionChain：负责整个处理请求的逻辑链，包含Handler和一系列Interceptors;

HandlerAdapter：适配器，通过它统一适配Handler的处理入口。



```java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
    // ...
    HttpServletRequest processedRequest = checkMultipart(request);
    // 通过遍历handlerMappings查找对应的HandlerExecutionChain
    HandlerExecutionChain mappedHandler = getHandler(processedRequest);
    if (mappedHandler == null) {
        noHandlerFound(processedRequest, response);
        return;
    }
    
    // 首先从HandlerExecutionChain中获取对应的handler
    // 其次遍历handlerAdapters，对每个adapter调用其support方法找到对应的HandlerAdapter
    HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());
    
    // ...
    // 执行拦截器
    if (!mappedHandler.applyPreHandle(processedRequest, response)) {
        return;
    }

    // 执行handler的逻辑，返回一个ModelAndView
    ModelAndView mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

	// 执行拦截器
    mappedHandler.applyPostHandle(processedRequest, response, mv);
    
    // ...
}
```

