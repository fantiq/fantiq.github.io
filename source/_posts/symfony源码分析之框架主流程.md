---
title: symfony源码分析之框架主流程
tags: [symfony,php框架,源码分析]
categories: [symfony源码分析]
date: 2017-06-29 14:37:33
---
这是基于 symfony3.3.0版本的源代码分析，主要包含以下部分：
1. 框架主流程
2. 容器生成及使用
3. 路由生成
4. 配置文件加载
5. 事件委派

在对源代码进行分析的时候，使用phpstrom配合xdebug扩展进行断点调试，对代码分析以及梳理起到了很大的帮助。phpstrom断点的配置可以参考 [这里](/2017/06/21/使用phpstorm进行断点调试/)

<!-- more -->

## 1 调用过程

#### web/app.php 
```
$kernel = new AppKernel('prod', false);
$response = $kernel->handle($request);
```

#### Symfony\Component\HttpKernel\Kernel 
```
public function handle(Request $request, $type = HttpKernelInterface::MASTER_REQUEST, $catch = true)
{
    if (false === $this->booted) {
    	// 初始化 容器 与 Bundle
        $this->boot();
    }
    // 这里通过容器的形式获取 Symfony\Component\HttpKernel\HttpKernel 并调用 handle方法 
    return $this->getHttpKernel()->handle($request, $type, $catch);
}
```
#### Symfony\Component\HttpKernel\HttpKernel 
```
private function handleRaw(Request $request, $type = self::MASTER_REQUEST)
{
    $this->requestStack->push($request);

// 匹配路由 查询需要执行的控制器
$event = new GetResponseEvent($this, $request, $type);
$this->dispatcher->dispatch(KernelEvents::REQUEST, $event);

// 获取控制器的实例以及要执行的方法
if (false === $controller = $this->resolver->getController($request)) {
    throw new NotFoundHttpException(sprintf('Unable to find the controller for path "%s". The route is wrongly configured.', $request->getPathInfo()));
}
// 这里的是为了触发事件 可以在事件委派的章节具体查看
$event = new FilterControllerEvent($this, $controller, $request, $type);
$this->dispatcher->dispatch(KernelEvents::CONTROLLER, $event);
$controller = $event->getController();

// 对控制器要执行的方法进行反射 获取方法中需要的参数 并通过容器来获取实例化的对象返回
$arguments = $this->argumentResolver->getArguments($request, $controller);

// 这里的是为了触发事件 可以在事件委派的章节具体查看
$event = new FilterControllerArgumentsEvent($this, $controller, $arguments, $request, $type);
$this->dispatcher->dispatch(KernelEvents::CONTROLLER_ARGUMENTS, $event);
$controller = $event->getController();
$arguments = $event->getArguments();

// call controller
//这里执行控制器
$response = call_user_func_array($controller, $arguments);

if (!$response instanceof Response) {
	// 当控制器返回的不是Response的话当作view来渲染处理
}

return $this->filterResponse($response, $request, $type);
}
```

## 2 主流程分析

对主流程代码进行精简，忽略事件委派的代码，流程如下

```
// 1. 匹配路由 查询需要执行的控制器
$this->dispatcher->dispatch(KernelEvents::REQUEST, $event);
// 2. 获取控制器的实例以及要执行的方法
$controller = $this->resolver->getController($request)
// 3. 对控制器要执行的方法进行反射 获取方法中需要的参数 并通过容器来获取实例化的对象返回
$arguments = $this->argumentResolver->getArguments($request, $controller);
// 4. 这里执行控制器
$response = call_user_func_array($controller, $arguments);
```

#### 2.1 匹配路由 查询需要执行的控制器
这里主要是触发了一个事件 `KernelEvents::REQUEST` , 这个事件中的一个handler `Symfony\Component\HttpKernel\EventListener\RouterListener`，进行了路由 、控制器的分析查询，如果路由没有生成还会生成路由代码。

```
# Symfony\Component\HttpKernel\EventListener\RouterListener
public function onKernelRequest(GetResponseEvent $event)
{
	// 路由已经解析过的话就直接返回
    if ($request->attributes->has('_controller')) {
        return;
    }

    try {
        if ($this->matcher instanceof RequestMatcherInterface) {
        	// 匹配路由 获取信息
            $parameters = $this->matcher->matchRequest($request);
        } else {
            $parameters = $this->matcher->match($request->getPathInfo());
        }
        //将分析出来的参数信息 控制器 方法 路由 添加到 request 的 attributes属性中
        // $parameters = [
        // 	'_controller' => "控制器::方法",
        // 	'_route' =''
        // ]
        $request->attributes->add($parameters);
        unset($parameters['_route'], $parameters['_controller']);
        $request->attributes->set('_route_params', $parameters);
    } catch (ResourceNotFoundException $e) {
    	// ......
    } catch (MethodNotAllowedException $e) {
    	// ......
    }
}
```
路由匹配的代码分析，框架会将路由信息分析出来并存储在 缓存文件 var/prod/appProdProjectContainerUrlMatcher.php 中，具体的路由生成代码可以到 [路由代码生成分析]() 具体查看

```
# Symfony\Bundle\FrameworkBundle\Routing\Router
public function getRouteCollection()
{
    if (null === $this->collection) {
    	// 通过容器获取 routing.loader 对应的对象并调用方法 load 
    	// Symfony\Bundle\FrameworkBundle\Routing\DelegatingLoader::load 
    	// 参数 
    	//		$this->resource -> app/config/routing.yml 
    	//		$this->options['resource_type'] -> null 
        $this->collection = $this->container->get('routing.loader')->load($this->resource, $this->options['resource_type']);
        $this->resolveParameters($this->collection);
        $this->collection->addResource(new ContainerParametersResource($this->collectedParameters));
    }

    return $this->collection;
}

# Syfmony\Component\Routing\Router\Router
public function matchRequest(Request $request)
{
    $matcher = $this->getMatcher();
    if (!$matcher instanceof RequestMatcherInterface) {
        // fallback to the default UrlMatcherInterface
        return $matcher->match($request->getPathInfo());
    }

    return $matcher->matchRequest($request);
}

public function getMatcher()
{
    // $this->getConfigCacheFactory() -> Symfony\Component\Config\ResourceCheckerConfigCacheFactory
    // 获取存储路由信息的文件 var/prod/appProdProjectContainerUrlMatcher.php
    $cache = $this->getConfigCacheFactory()->cache($this->options['cache_dir'].'/'.$this->options['matcher_cache_class'].'.php',
        //当路由文件没有生成的时候 这个匿名方法执行，通过配置文件生成路由关系且存储到缓存文件
        function (ConfigCacheInterface $cache) {
            // ......
        }
    );

    //这里加载并实例化了缓存的路由信息 var/cache/dev/app[ENV]DebugProjectContainerUrlMatcher.php
    require_once $cache->getPath();
    return $this->matcher = new $this->options['matcher_cache_class']($this->context);
}
```


#### 2.2 获取控制器的实例以及要执行的方法

此处代码的调用连如下
```
Symfony\Bundle\FrameworkBundle\Controller\ControllerResolver extends 
Symfony\Component\HttpKernel\Controller\ContainerControllerResolver extends 
Symfony\Component\HttpKernel\Controller\ControllerResolver
```
最终执行的核心代码如下：

```
# Symfony\Component\HttpKernel\Controller\ControllerResolver
public function getController(Request $request)
{
	// 这里的变量 $controller 就是从$request 的 attributes属性中获取的，这个属性的数据是 事件 KernelEvents::CONTROLLER中的handler RouterListener 设置的
	// 下面的代码逻辑主要是对控制器不同类型的处理 匿名回调 数组 对象 等
    if (!$controller = $request->attributes->get('_controller')) {
        if (null !== $this->logger) {
            $this->logger->warning('Unable to look for the controller as the "_controller" parameter is missing.');
        }

        return false;
    }

    if (is_array($controller)) {
        return $controller;
    }

    if (is_object($controller)) {
        if (method_exists($controller, '__invoke')) {
            return $controller;
        }

        throw new \InvalidArgumentException(sprintf('Controller "%s" for URI "%s" is not callable.', get_class($controller), $request->getPathInfo()));
    }

    if (false === strpos($controller, ':')) {
        if (method_exists($controller, '__invoke')) {
            return $this->instantiateController($controller);
        } elseif (function_exists($controller)) {
            return $controller;
        }
    }

    $callable = $this->createController($controller);

    if (!is_callable($callable)) {
        throw new \InvalidArgumentException(sprintf('The controller for URI "%s" is not callable. %s', $request->getPathInfo(), $this->getControllerError($callable)));
    }

    return $callable;
}

```
以上代码的核心片段就是 `$controller = $request->attributes->get('_controller')`


#### 2.3 对控制器要执行的方法进行反射 获取方法中需要的参数 并通过容器来获取实例化的对象返回

控制器方法中的参数处理核心代码如下

```
# Symfony\Component\HttpKernel\Controller\ArgumentResolver
public function getArguments(Request $request, $controller)
{
    $arguments = array();
    //反射获得的控制器方法中的参数，且将参数封装到对象 Symfony\Component\HttpKernel\ControllerMetadata\ArgumentMetadata
    foreach ($this->argumentMetadataFactory->createArgumentMetadata($controller) as $metadata) {
        //将argument resolver 中的对象逐一进行尝试
        foreach ($this->argumentValueResolvers as $resolver) {
            if (!$resolver->supports($request, $metadata)) {
                continue;
            }

            $resolved = $resolver->resolve($request, $metadata);

            if (!$resolved instanceof \Generator) {
                throw new \InvalidArgumentException(sprintf('%s::resolve() must yield at least one value.', get_class($resolver)));
            }
            //$resolver 返回的是 Generator yield
            foreach ($resolved as $append) {
                $arguments[] = $append;
            }

            // continue to the next controller argument
            continue 2; // 跳转到上一层循环继续
        }

        $representative = $controller;

        if (is_array($representative)) {
            $representative = sprintf('%s::%s()', get_class($representative[0]), $representative[1]);
        } elseif (is_object($representative)) {
            $representative = get_class($representative);
        }

        throw new \RuntimeException(sprintf('Controller "%s" requires that you provide a value for the "$%s" argument. Either the argument is nullable and no null value has been provided, no default value has been provided or because there is a non optional argument after this one.', $representative, $metadata->getName()));
    }

    return $arguments;
}

public static function getDefaultArgumentValueResolvers()
{
    return array(
        new RequestAttributeValueResolver(),
        new RequestValueResolver(),
        new SessionValueResolver(),
        new DefaultValueResolver(),
        new VariadicValueResolver(),
    );
}
```














