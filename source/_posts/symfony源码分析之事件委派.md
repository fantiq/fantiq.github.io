---
title: symfony源码分析之事件委派
tags: [symfony,php框架,源码分析]
categories: [symfony源码分析]
date: 2017-06-29 20:51:20
---
symfony中的事件委派是一个比较经典的设计模式，这个机制对代码的解耦有很大的帮助。理解事件委派的机制有助于对源代码的理解，比如路由的解析就是基于事件委派来实现的。
<!-- more -->
# 事件触发
这里是事件触发的代码，从这个代码作为切入点进行代码调用逻辑的梳理
```
# Symfony\Component\HttpKernel\HttpKernel

private function handleRaw(Request $request, $type = self::MASTER_REQUEST)
{
	$event = new GetResponseEvent($this, $request, $type);
	// 触发事件
	$this->dispatcher->dispatch(KernelEvents::REQUEST, $event);
}
```
# 委派器以及Listener
以上代码的`$this->dispatcher`是需要从容器里查找 在创建 `HttpKernel` 这个类的时候初始化了哪些参数。
从容器代码中查找分析，委派器类的实例化以及对事件增加 Listebner 的代码。

```
# var/cache/prod/appProdProjectContainer.php 
protected function getHttpKernelService()
{
    return $this->services['http_kernel'] = new \Symfony\Component\HttpKernel\HttpKernel(
    	// 这里是dispatcher参数 还是要在容器里取
    	${($_ = isset($this->services['event_dispatcher']) ? $this->services['event_dispatcher'] : $this->get('event_dispatcher')) && false ?: '_'}, 
    	// ......
    );
}

protected function getEventDispatcherService()
{
	// 实例化 dispatcher
    $this->services['event_dispatcher'] = $instance = new \Symfony\Component\EventDispatcher\ContainerAwareEventDispatcher($this);
    // 添加事件
    // 添加的是
    $instance->addListener('kernel.response', function (\Symfony\Component\HttpKernel\Event\GetResponseEvent $event) {
        return ${(
	        	$_ = isset($this->services['session_listener']) ? 
	        	$this->services['session_listener'] : 
	        	$this->get('session_listener')
        	) && false ?: '_'}->onKernelRequest($event);
    }, 128);
    $instance->addListener('kernel.request', /* ...... */);
    $instance->addListener('kernel.finish_request', /* ...... */);
    $instance->addListener('console.terminate', /* ...... */);
    // ......
}
```
# 委派器的工作方式
从上面的代码可以看出，委派器的Listener触发方式 `$this->dispatcher->dispatch(KernelEvents::REQUEST, $event);`。基于此对委派器工作的逻辑代码进行跟中
```
# Symfony\Component\EventDispatcher\ContainerAwareEventDispatcher
public function getListeners($eventName = null)
{
    if (null === $eventName) {
        foreach ($this->listenerIds as $serviceEventName => $args) {
            $this->lazyLoad($serviceEventName);
        }
    } else {
        $this->lazyLoad($eventName);
    }

    return parent::getListeners($eventName);
}
# Symfony\Component\EventDispatcher\EventDispatcher
public function dispatch($eventName, Event $event = null)
{
    if (null === $event) {
        $event = new Event();
    }
    // 针对委派器查询获取 对应的 Listener
    if ($listeners = $this->getListeners($eventName)) {
        $this->doDispatch($listeners, $eventName, $event);
    }

    return $event;
}
public function getListeners($eventName = null)
{
	// 对Listeners的处理 排序 过滤等操作
    if (null !== $eventName) {
        if (!isset($this->listeners[$eventName])) {
            return array();
        }

        if (!isset($this->sorted[$eventName])) {
            $this->sortListeners($eventName);
        }

        return $this->sorted[$eventName];
    }

    foreach ($this->listeners as $eventName => $eventListeners) {
        if (!isset($this->sorted[$eventName])) {
            $this->sortListeners($eventName);
        }
    }

    return array_filter($this->sorted);
}
protected function doDispatch($listeners, $eventName, Event $event)
{
	// 对委派器绑定的Listeners循环处理
    foreach ($listeners as $listener) {
        if ($event->isPropagationStopped()) {
            break;
        }
        // Listener绑定的是匿名函数，这里调用匿名函数，做到延迟加载
        call_user_func($listener, $event, $eventName, $this);
    }
}
```