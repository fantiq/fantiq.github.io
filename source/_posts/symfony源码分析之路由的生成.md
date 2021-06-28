---
title: symfony源码分析之路由的生成
tags: [symfony,php框架,源码分析]
categories: [symfony源码分析]
date: 2017-06-29 20:50:59
---
路由主要有两部分组成，一部分是路由的匹配过程，一部分是路由表的生成过程，这里通过跟中代码看下路由表的生成过程。
symfony中的路由是通过 标注(annotation)的形式进行定义的，symfony是通过配置文件进行查找进行路由的解析，最终将路由存储到缓存文件 `var/cache/prod/appProdProjectContainerUrlMatcher.php` 中。

<!-- more -->

# 路由调用

在事件中触发路由的匹配 且将匹配的结果存入 request 中
```
# Symfony\Component\HttpKernel\EventListener\RouterListener
public function onKernelRequest(GetResponseEvent $event)
{
	try {
	// ......
        if ($this->matcher instanceof RequestMatcherInterface) {
            //$this->matcher 就是 Symfony\Bundle\FrameworkBundle\Routing\Router
            // 这里代码进行路由匹配 以及 路由缓存文件的生成
            $parameters = $this->matcher->matchRequest($request);
        } else {
            $parameters = $this->matcher->match($request->getPathInfo());
        }
        //在事件的这里会将分析出来的参数添加到 request 的 attributes属性中
        // 进行匹配的时候也会读取 request的attributes 属性进行匹配
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
对路由进行匹配 如果没有路由缓存文件 则生成
```
# Symfony\Component\Routing\Router
public function getMatcher()
{
	// ......

    // 加载路由缓存文件 var/cache/prod/appProdProjectContainerUrlMatcher.php
    $cache = $this->getConfigCacheFactory()->cache($this->options['cache_dir'].'/'.$this->options['matcher_cache_class'].'.php',
        function (ConfigCacheInterface $cache) {

            //当路由文件没有生成的时候 这个匿名方法执行，通过配置文件生成路由到缓存文件
            $dumper = $this->getMatcherDumperInstance();
            // ......

            //这里getRouteCollection 调用的是子类的方法 如下
            //$this->collection = $this->container->get('routing.loader')->load($this->resource, $this->options['resource_type']);
            $cache->write($dumper->dump($options), $this->getRouteCollection()->getResources());
        }
    );

    require_once $cache->getPath();
    //这里实例化了缓存的路由信息 var/cache/dev/appDevDebugProjectContainerUrlMatcher.php
    return $this->matcher = new $this->options['matcher_cache_class']($this->context);
}

# Symfony\Bundle\FrameworkBundle\Routing\Router
public function getRouteCollection()
{
    if (null === $this->collection) {
        //routeCollection 数据主要是在这里获取的，主要是容器 routing.loader => Symfony\Bundle\FrameworkBundle\Routing\DelegatingLoader();
        $this->collection = $this->container->get('routing.loader')->load($this->resource, $this->options['resource_type']);
        // ......
    }

    return $this->collection;
}
```
# 路由缓存文件生成 

如果要生成路由信息会执行如下代码流程
```
# Symfony\Bundle\FrameworkBundle\Routing\DelegatingLoader
public function load($resource, $type = null)
{
	// ......
    try {
        //获取route collection
        $collection = parent::load($resource, $type);
    } finally {
        $this->loading = false;
    }

    foreach ($collection->all() as $route) {
        try {
            $controller = $this->parser->parse($controller);
        } catch (\InvalidArgumentException $e) {
            // unable to optimize unknown notation
        }

        $route->setDefault('_controller', $controller);
    }

    return $collection;
}
# Symfony\Component\Config\Loader\DelegatingLoader
public function load($resource, $type = null)
{
	// ......

	// $loader => Syfmony\Component\Routing\Loader\YamlFileLoader
	return $loader->load($resource, $type);
}
```

重点在 `Symfony\Component\Routing\Loader\YamlFileLoader::load` 中的代码处理
```
# Symfony\Component\Routing\Loader\YamlFileLoader
public function load($file, $type = null)
{
	try {
        // 将yaml格式解析成 php 数组
        $parsedConfig = $this->yamlParser->parse(file_get_contents($path), Yaml::PARSE_KEYS_AS_STRINGS);
    } catch (ParseException $e) {
    	// ......
    }
    $collection = new RouteCollection();
    // ......
    foreach ($parsedConfig as $name => $config) {
    	// 如果路由配置中存在key  resource
    	// 对resource指定的文件目录中的路由定义进行处理
        if (isset($config['resource'])) {
            $this->parseImport($collection, $config, $path, $file);
        } else {
        	// 不存在直接进行路由解析
            $this->parseRoute($collection, $name, $config, $path);
        }
    }
}

protected function parseImport(RouteCollection $collection, array $config, $path, $file)
{
	$subCollection = $this->import($config['resource'], $type, false, $file);
	$collection->addCollection($subCollection);
}
public function import($resource, $type = null, $ignoreErrors = false, $sourceResource = null)
{
	// ......
    return $this->doImport($resource, $type, $ignoreErrors, $sourceResource);
}
private function doImport($resource, $type = null, $ignoreErrors = false, $sourceResource = null)
{
    try {
        // $loader = \Symfony\Component\Routing\Loader\AnnotationDirectoryLoader
        $loader = $this->resolve($resource, $type);
        // ......
        try {
        	// 执行Loader中的 load 方法
            $ret = $loader->load($resource, $type);
        } finally {
        	// ......
        }

        return $ret;
    } catch (FileLoaderImportCircularReferenceException $e) {
    	// ......
    } catch (\Exception $e) {
    	// ......
    }
}
```
通过分析类文件中的标注解析出路由定义
```
# Syfmony\Component\Routing\Loader\AnnotationLoader
public function load($path, $type = null)
{
    $collection = new RouteCollection();

    // 这里读取 route中 resource 命名空间指定目录的所有文件
    $files = iterator_to_array(new \RecursiveIteratorIterator(
    	// 这里使用了php处理目录文件的迭代器
        new \RecursiveCallbackFilterIterator(
            new \RecursiveDirectoryIterator($dir, \FilesystemIterator::SKIP_DOTS | \FilesystemIterator::FOLLOW_SYMLINKS),
            function (\SplFileInfo $current) {
                return '.' !== substr($current->getBasename(), 0, 1);
            }
        ),
        \RecursiveIteratorIterator::LEAVES_ONLY
    ));
	// ......

    foreach ($files as $file) {
		// ......
        if ($class = $this->findClass($file)) {
            // 对控制器进行反射处理 如果是absrtact的则不会add到collection
            $refl = new \ReflectionClass($class);
            if ($refl->isAbstract()) {
                continue;
            }
            // 这里的loader对class的标注进行了解析 且 将路由解析的结果封装到RouteCollection返回
            $collection->addCollection($this->loader->load($class, $type));
        }
    }

    return $collection;
}

# Symfony\Component\Routing\Loader\AnnotationClassLoader
public function load($class, $type = null)
{
	// ......
    $class = new \ReflectionClass($class);
    if ($class->isAbstract()) {
    	// 如果是抽象类 抛出异常
    }
    // 实例化了 RouteCollection
    $collection = new RouteCollection();
    $collection->addResource(new FileResource($class->getFileName()));
    // 遍历类中的方法
    foreach ($class->getMethods() as $method) {
        $this->defaultRouteIndex = 0;
        // 对其中的标注进行解析
        foreach ($this->reader->getMethodAnnotations($method) as $annot) {
            if ($annot instanceof $this->routeAnnotationClass) {
            	// 解析结果 addRoute 中添加到$collection
                $this->addRoute($collection, $annot, $globals, $class, $method);
            }
        }
    }
    // ......
    return $collection;
}

# Doctrine\Common\Annotations\CachedReader
public function getMethodAnnotations(\ReflectionMethod $method)
{
	// ......
    if (false === ($annots = $this->fetchFromCache($cacheKey, $class))) {
    	// 获取方法中的注释
        $annots = $this->delegate->getMethodAnnotations($method);
        $this->saveToCache($cacheKey, $annots);
    }

    return $this->loadedAnnotations[$cacheKey] = $annots;
}
```


