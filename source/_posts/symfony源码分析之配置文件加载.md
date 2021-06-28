---
title: symfony源码分析之配置文件加载
tags: [symfony,php框架,源码分析]
categories: [symfony源码分析]
date: 2017-06-29 20:51:13
---
syfmony框架支持多种形式的配置文件 `yaml` `xml` `php array` ，理解框架的配置加载解析逻辑有利于对框架源代码的理解。这里可以通过框架对config.yaml 的解析过程进行分析来梳理框架对配置文件的解析逻辑。
<!-- more -->


在初始化容器的时候 框架会对 `app/config/config.yml` 进行处理
```
# Symfony\Component\HttpKernel\Kernel
protected function buildContainer()
{
	if (null !== $cont = $this->registerContainerConfiguration($this->getContainerLoader($container))) {
	    $container->merge($cont);
	}
}
protected function getContainerLoader(ContainerInterface $container)
{
    $locator = new FileLocator($this);
    $resolver = new LoaderResolver(array(
        new XmlFileLoader($container, $locator),
        new YamlFileLoader($container, $locator),
        new IniFileLoader($container, $locator),
        new PhpFileLoader($container, $locator),
        new GlobFileLoader($locator),
        new DirectoryLoader($container, $locator),
        new ClosureLoader($container),
    ));

    return new DelegatingLoader($resolver);
}
# AppKernel
public function registerContainerConfiguration(LoaderInterface $loader)
{
    $loader->load($this->getRootDir().'/config/config_'.$this->getEnvironment().'.yml');
}
```

对以上代码进行顺序梳理
```
$loader = new DelegatingLoader(
	new LoaderResolver([
	    new XmlFileLoader($container, $locator),
	    new YamlFileLoader($container, $locator),
	    new IniFileLoader($container, $locator),
	    new PhpFileLoader($container, $locator),
	    new GlobFileLoader($locator),
	    new DirectoryLoader($container, $locator),
	    new ClosureLoader($container),
	])
);
$loader->load($this->getRootDir().'/config/config_'.$this->getEnvironment().'.yml');

# Symfony\Component\Config\Loader\DelegatingLoader
public function load($resource, $type = null)
{
	// $resolver => Symfony\Component\Config\Loader\LoaderResolver
    if (false === $loader = $this->resolver->resolve($resource, $type)) {
        throw new FileLoaderLoadException($resource, null, null, null, $type);
    }
    // 执行 支持配置文件格式的 loader 
    return $loader->load($resource, $type);
}

# Symfony\Component\Config\Loader\LoaderResolver
public function resolve($resource, $type = null)
{
	/*
	$this->loaders => [
		new XmlFileLoader($container, $locator),
	    new YamlFileLoader($container, $locator),
	    new IniFileLoader($container, $locator),
	    new PhpFileLoader($container, $locator),
	    new GlobFileLoader($locator),
	    new DirectoryLoader($container, $locator),
	    new ClosureLoader($container),
	]
	 */
    foreach ($this->loaders as $loader) {
    	// 调用每个loader的 support方法 检查 loader是否支持配置文件的类型
        if ($loader->supports($resource, $type)) {
            return $loader;
        }
    }

    return false;
}
```
其实在实例化这些loader的时候是针对当前处理的业务逻辑的
比如 实例化容器的时候要用到的Loader `Symfony\Component\DependencyInjection\Loader\*`
比如 路由匹配的时候用到的Loader `Symfony\Component\Routing\Loader\*`


