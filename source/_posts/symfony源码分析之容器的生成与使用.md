---
title: symfony源码分析之容器的生成与使用
tags: [symfony,php框架,源码分析]
categories: [symfony源码分析]
date: 2017-06-29 20:50:36
---
symfony 的容器是有一个编译过程的，框架初始化的时候会执行 `Symfony\Component\HttpKernel\Kernel::initializationContainer`，这个方法会对代码进行检查，看是否需要生成新的容器代码。如果需要 Symfony 会将各个类的依赖关系通过编译生成静态的类并存储在缓存文件`var/cache/[ENV]/appProjectContainer`中。

其中很多是框架自己的依赖关系，这些依赖关系类似java的方式，通过xml文件的形式进行声明，这些xml存在与框架的代码中，`Syfmony\Bundle\FrameworkBundle\Resource\config\*.xml`。
<!-- more -->
## 0. 代码流程

容器相关的代码大致如下流程

```
# Symfony\Component\HttpKernel\Kernel
protected function initializeContainer()
{
	// 获取生成的容器代码文件 类名
    $class = $this->getContainerClass();
    $cache = new ConfigCache($this->getCacheDir().'/'.$class.'.php', $this->debug);
    $fresh = true;
    // 检查是否需要重新生成 容器 缓存类
    if (!$cache->isFresh()) {
    	// .....
        try {
            $container = null;
            // 重新生成容器缓存类
            // 实例化生成容器缓存代码的类
            $container = $this->buildContainer();
            // 调用方法开始生成容器代码
            $container->compile();
        } finally {
        	// ......
        }
        // 将生成的代码写入文件
        $this->dumpContainer($cache, $container, $class, $this->getContainerBaseClass());

        $fresh = false;
    }

    //加载生成的容器文件 实例化容器类 并且复制给 Kernel::$container
    require_once $cache->getPath();
    $this->container = new $class();
    $this->container->set('kernel', $this);

    // ......
}
```

下面着重分析 容器代码生成类的创建过程
```
protected function buildContainer()
{
	// 创建容器缓存代码的目录

    $container = $this->getContainerBuilder();
    $container->addObjectResource($this);
    // 容器预处理，在开始编译容器前，会给容器添加一些 pass (这个问题就是在2017phpcon大会上提过的，当时没解决，这里看代码解决了)
    // 这些pass主要处理框架配置中声明的容器关系

    $this->prepareContainer($container);

    // 解析容器配置文件 我们自己注入容器的类就要声明在配置文件 app/config/services.yml 中
    // 这里就是对这个配置文件进行解析，并将其中的关系生成代码到容器中
    if (null !== $cont = $this->registerContainerConfiguration($this->getContainerLoader($container))) {
        $container->merge($cont);
    }
    $container->addCompilerPass(new AddAnnotatedClassesToCachePass($this));
    $container->addResource(new EnvParametersResource('SYMFONY__'));

    return $container;
}
```

如上的分析可以将容器代码生成的过程精简到如下3个步骤

```
# Symfony\Component\HttpKernel\Kernel
protected function initializeContainer()
{
	// ...
    try {
        $container = null;
        $container = $this->buildContainer();
        3. 编译生成容器代码
        $container->compile();
    } finally {
    	// ......
    }
    // ......
}

protected function buildContainer()
{
	// 1. 实例化创建容器代码类的时候，预处理一些内容
    $this->prepareContainer($container);
    // 2. 加载项目容器的配置文件进行处理 app/config/services.yml
    if (null !== $cont = $this->registerContainerConfiguration($this->getContainerLoader($container))) {
        $container->merge($cont);
    }
}

```


## 1. 容器预处理 
```
# Symfony\Component\HttpKernel\Kernel
protected function prepareContainer(ContainerBuilder $container)
{
    $extensions = array();
    // 这里的 bundles 是在 initializeBundles 方法中实例化AppKernel中注册的Bundles 进行的赋值
    foreach ($this->bundles as $bundle) {
    	// 这里将注册的Bundle进行循环处理
    	// 取出bundle的extension 并 注册到 container中
        if ($extension = $bundle->getContainerExtension()) {
            $container->registerExtension($extension);
            $extensions[] = $extension->getAlias();
        }

        if ($this->debug) {
            $container->addObjectResource($bundle);
        }
    }

    foreach ($this->bundles as $bundle) {
    	// 触发bundle的build方法
        $bundle->build($container);
    }

    $this->build($container);

    // 这里的代码是为了确定将 MergeExtensionConfigurationPass 这个Pass类添加到容器的Pass中
    $container->getCompilerPassConfig()->setMergePass(new MergeExtensionConfigurationPass($extensions));
}
```

我们可以在 `AppKernel::registerBundles` 中查看到注册的 Bundles 

```
public function registerBundles()
{
    $bundles = [
        new Symfony\Bundle\FrameworkBundle\FrameworkBundle(),
        new Symfony\Bundle\SecurityBundle\SecurityBundle(),
        new Symfony\Bundle\TwigBundle\TwigBundle(),
        new Symfony\Bundle\MonologBundle\MonologBundle(),
        new Symfony\Bundle\SwiftmailerBundle\SwiftmailerBundle(),
        new Doctrine\Bundle\DoctrineBundle\DoctrineBundle(),
        new Sensio\Bundle\FrameworkExtraBundle\SensioFrameworkExtraBundle(),
        new AppBundle\AppBundle(),
    ];

    return $bundles;
}
```
我们以 `Symfony\Bundle\FrameworkBundle\FrameworkBundle` 作为目标进行代码的继续跟踪
```
$extension = $bundle->getContainerExtension();
$container->registerExtension($extension);
```
跟中方法 `getContainerExtension` , 在类 `FrameworkBundle` 所继承的类`Syfmony\Component\HttpKernel\Bundle\Bundle`中找到

```
# Syfmony\Component\HttpKernel\Bundle\Bundle
public function getContainerExtension()
{
    if (null === $this->extension) {
    	// 获取extension
        $extension = $this->createContainerExtension();

        if (null !== $extension) {

            // check naming convention
            $basename = preg_replace('/Bundle$/', '', $this->getName());
            $expectedAlias = Container::underscore($basename);
            if ($expectedAlias != $extension->getAlias()) {
                throw new \LogicException(......);
            }

            $this->extension = $extension;
        } else {
            $this->extension = false;
        }
    }

    if ($this->extension) {
        return $this->extension;
    }
}
```

以下是获取extension的代码逻辑
```
protected function createContainerExtension()
{
	// 这里实例化一个类并返回
    if (class_exists($class = $this->getContainerExtensionClass())) {
        return new $class();
    }
}
protected function getContainerExtensionClass()
{
	// 类名的规则
	// 命名空间\DependencyInjection\(将类名的Bundle替换成Extension)
    $basename = preg_replace('/Bundle$/', '', $this->getName());

    return $this->getNamespace().'\\DependencyInjection\\'.$basename.'Extension';
}

# 以下两个方法主要还是当 $this->namespace $this->name 不存在的时候
# 通过方法 $this->parseClassName() 来解析出来
public function getNamespace()
{
    if (null === $this->namespace) {
        $this->parseClassName();
    }

    return $this->namespace;
}
final public function getName()
{
    if (null === $this->name) {
        $this->parseClassName();
    }

    return $this->name;
}
private function parseClassName()
{
	# 命名空间 类型的截取
    $pos = strrpos(static::class, '\\');
	// 截取类的命名空间
    $this->namespace = false === $pos ? '' : substr(static::class, 0, $pos);
    if (null === $this->name) {
		// 截取类的名称
        $this->name = false === $pos ? static::class : substr(static::class, $pos + 1);
    }
}
```
以上代码解析出来的就是 `Symfony\Bundle\FrameworkBundle\DependencyInjection\FrameworkExtension`。回到代码中
```
if ($extension = $bundle->getContainerExtension()) {
    $container->registerExtension($extension);
    $extensions[] = $extension->getAlias();
}
```
要执行容器的`registerExtension`方法 
```
# Symfony\Component\DependencyInjection\ContainerBuilder
public function registerExtension(ExtensionInterface $extension)
{
	// $extension->getAlias() 这个方法在 Symfony\Component\DependencyInjection\Extension\Extension 中
	// 主要的作用是将类名取出 将类名的Excention后缀去掉
    $this->extensions[$extension->getAlias()] = $extension;

    if (false !== $extension->getNamespace()) {
        $this->extensionsByNs[$extension->getNamespace()] = $extension;
    }
}
```

## 2. 解析容器配置文件
解析容器配置文件的代码
```
if (null !== $cont = $this->registerContainerConfiguration($this->getContainerLoader($container))) {
    $container->merge($cont);
}
```
主要是执行了方法 `registerContainerConfiguration` 这个方法在 `AppKernel` 类中
```
# AppKernel
public function registerContainerConfiguration(LoaderInterface $loader)
{
	// 这里的配置文件是 app/config/config_[ENV].yml 是Yml格式的文件
	// 这里的loader 应该是类 `YamlFileLoader`
    $loader->load($this->getRootDir().'/config/config_'.$this->getEnvironment().'.yml');
}

# 所以这里的代码应该是执行 
# SymfonyComponent\DependencyInjection\Loader\YamlFileLoader::load
public function load($resource, $type = null)
{
	// 加载配置文件，将yml格式的配置内容解析成php数组
    $content = $this->loadFile($path);
    $this->container->fileExists($path);
    // ......
    // 处理配置文件中的 key `imports`  对import 声明的文件进行load处理
    $this->parseImports($content, $path);

    // 处理配置文件中的参数部分
    if (isset($content['parameters'])) {
        if (!is_array($content['parameters'])) {
            throw new InvalidArgumentException(sprintf('The "parameters" key should contain an array in %s. Check your YAML syntax.', $resource));
        }

        foreach ($content['parameters'] as $key => $value) {
            $this->container->setParameter($key, $this->resolveServices($value, $resource, true));
        }
    }

    // extensions
    $this->loadFromExtensions($content);

    // services
    $this->anonymousServicesCount = 0;
    $this->setCurrentDir(dirname($path));
    try {
    	// 这里将services中的声明封装到Definition 将Definition 存储到Container中
        $this->parseDefinitions($content, $resource);
    } finally {
        $this->instanceof = array();
    }
}

private function parseDefinitions(array $content, $file)
{
	// 如果配置文件中不存在key services 这里直接返回
    if (!isset($content['services'])) {
        return;
    }

    // ......
    // 这里处理 配置文件中 services 下的 _default 配置信息，处理后会删除这个key
    $defaults = $this->parseDefaults($content, $file);

    //对配置文件中定义的services进行逐个分析
    foreach ($content['services'] as $id => $service) {
        $this->parseDefinition($id, $service, $file, $defaults);
    }
}

private function parseDefinition($id, $service, $file, array $defaults)
{

    //实例化 definition
    if ($this->isLoadingInstanceof) {
        $definition = new ChildDefinition('');
    } elseif (isset($service['parent'])) {
    	// ......
        $definition = new ChildDefinition($service['parent']);
    } else {
        $definition = new Definition();
        // ......
    }

    // 这里的大段逻辑是处理一些services相关的配置信息
    if (isset($service['class'])) {
        $definition->setClass($service['class']);
    }
    // ......

    if (array_key_exists('resource', $service)) {
        if (!is_string($service['resource'])) {
            throw new InvalidArgumentException(sprintf('A "resource" attribute must be of type string for service "%s" in %s. Check your YAML syntax.', $id, $file));
        }
        $exclude = isset($service['exclude']) ? $service['exclude'] : null;
        // 如果service的配置中存在key resource 进行类的注册
        $this->registerClasses($definition, $id, $service['resource'], $exclude);
    } else {
        $this->setDefinition($id, $definition);
    }
}

# Symfony\Component\DependencyInjection\Loader\FileLoader
public function registerClasses(Definition $prototype, $namespace, $resource, $exclude = null)
{
	// 一些判断
	// ......

	// 这里主要是处理 存在 resource 这个key的services注册
	// symfony的容器会将指定目录下都所有类都会注册到container
	// 这里的 findClasses 就是查询类的过程
    $classes = $this->findClasses($namespace, $resource, $exclude);
    // prepare for deep cloning
    $prototype = serialize($prototype);

    foreach ($classes as $class) {
    	// 这里调用 setDefinition 对resource下的类全部设置到 container中
        $this->setDefinition($class, unserialize($prototype));
    }
}
protected function setDefinition($id, Definition $definition)
{
	// ...... 

    //将definition添加到 container 里面
    $this->container->setDefinition($id, $definition instanceof ChildDefinition ? $definition : $definition->setInstanceofConditionals($this->instanceof));
}
```

这里着重分析下 registerClasses方法中的 `$this->findClasses`

```
# Symfony\Component\DependencyInjection\Loader\FileLoader
private function findClasses($namespace, $pattern, $excludePattern)
{
    // ......
    // 这里是对文件的处理
    foreach ($this->glob($pattern, true, $resource) as $path => $info) {
    	// ......
        if (!$r->isInterface() && !$r->isTrait() && !$r->isAbstract()) {
            $classes[] = $class;
        }
    }
    // ......

    return $classes;
}
# Symfony\Component\Config\Loader\FileLoader
protected function glob($pattern, $recursive, &$resource = null, $ignoreErrors = false)
{
    try {
        $prefix = $this->locator->locate($prefix, $this->currentDir, true);
    } catch (FileLocatorFileNotFoundException $e) {
    	// ......
    }
    // 这里返回的是 yield
    $resource = new GlobResource($prefix, $pattern, $recursive);
    // 所以这里需要foreach来触发
    foreach ($resource as $path => $info) {
        yield $path => $info;
    }
}

# Symfony\Component\Config\Resource\GlobResource
public function getIterator()
{
    if (false === strpos($this->pattern, '/**/') && (defined('GLOB_BRACE') || false === strpos($this->pattern, '{'))) {
        foreach (glob($this->prefix.$this->pattern, defined('GLOB_BRACE') ? GLOB_BRACE : 0) as $path) {
            if ($this->recursive && is_dir($path)) {
            	// 这里是读取文件的主要逻辑
            	// 使用的是 php 中提供的处理文件的迭代器
                $files = iterator_to_array(new \RecursiveIteratorIterator(
                    new \RecursiveCallbackFilterIterator(
                        new \RecursiveDirectoryIterator($path, \FilesystemIterator::SKIP_DOTS | \FilesystemIterator::FOLLOW_SYMLINKS),
                        function (\SplFileInfo $file) { return '.' !== $file->getBasename()[0]; }
                    ),
                    \RecursiveIteratorIterator::LEAVES_ONLY
                ));
            } elseif (is_file($path)) {
                yield $path => new \SplFileInfo($path);
            }
            // ......
        }

        return;
    }

    $finder = new Finder();
    // ......
    foreach ($finder->followLinks()->sortByName()->in($this->prefix) as $path => $info) {
        if (preg_match($regex, substr($path, $prefixLen)) && $info->isFile()) {
            yield $path => $info;
        }
    }
}
```

## 3. 编译生成容器代码
这段代码主要是执行 Pass代码 这里主要跟踪下 `MergeExtensionConfigurationPass` 这个Pass，他执行了 `FrameworkExtension::load` ，这个方法中加载了框架需要的各种类以及依赖关系

```
# Symfony\Component\DependencyInjection\Complier\Complier
 public function compile(ContainerBuilder $container)
{
    try {
    	// 这里触发了所有Passs的 process方法
        foreach ($this->passConfig->getPasses() as $pass) {
            $pass->process($container);
        }
    } catch (\Exception $e) {
    	// ......
    }
}

# Symfony\Component\HttpKernel\DependencyInjection\MergeExtensionConfigurationPass
public function process(ContainerBuilder $container)
{
	// ......
    parent::process($container);
}
# Symfony\Component\DependencyInjection\Compiler\MergeExtensionConfigurationPass
public function process(ContainerBuilder $container)
{
    foreach ($container->getExtensions() as $name => $extension) {
    	// ......
    	// 执行了 extension 的 load 方法
        $extension->load($config, $tmpContainer);
        // ......
    }

    $container->addDefinitions($definitions);
    $container->addAliases($aliases);
}

# Symfony\Bundle\FrameworkBundle\DependencyInjection\FrameworkExtension
public function load(array $configs, ContainerBuilder $container)
{
	/*
	这个方法中的代码很多，主要内容是加载了很多的 xml 配置文件 
	这些配置文件的作用是声明框架所需要的类之间的关系

	 */
}
```
