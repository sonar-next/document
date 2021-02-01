# SonarScanner 源码分析

---
title: SonarScanner 源码分析
date: 2020-11-24 14:00:00
author: mamian521#gmail.com

---



## 介绍：

SonarQube是一个静态代码扫描平台，支持主流语言的静态代码扫描分析。

SonarQube主要分为几个模块：WebServer、CeServer、Scanner

今天主要对 Scanner 分析一下源码，其中Scanner是在本地执行扫描分析的工具，也就是说SonarScanner是在本地执行扫描分析之后将结果上传到服务端进行分析。本文重点介绍一下 Scanner 的源码和原理。

SonarScanner 的源码主要有三部分，一个是 SonarScannerCli 用来执行命令和接收参数(https://github.com/SonarSource/sonar-scanner-cli.git)，另一个是 SonarScannerApi ，主要用来下载插件包和加载类 (https://github.com/SonarSource/sonar-scanner-api.git)，还有真正用来执行扫描的 SonarScannerEngine (https://github.com/SonarSource/sonarqube/tree/master/sonar-scanner-engine)

本文主要分析一下前两部分。

### SonarScannerCli

目录结构：

![](https://tva1.sinaimg.cn/large/0081Kckwly1gl0nmbtmxkj30l00eomya.jpg)

可以看出，Cli 的源码并不多，入口是 `Main` 类的 `main` 函数

```bash
Logs logs = new Logs(System.out, System.err);
Exit exit = new Exit();
Cli cli = new Cli(exit, logs).parse(args);
Main main = new Main(exit, cli, new Conf(cli, logs, System.getenv()), new ScannerFactory(logs), logs);
main.execute();
```

主要的几个类：

Logs 用来记录日志

Exit 记录退出码

Cli 用来解析命令行执行时传递的变量

Conf 类用来接收 Cli 解析之后的参数

参数主要从几个地方获取

1. Scanner 全局设置(Scanner安装目录/`conf/sonar-scanner.properties`)
2. 环境变量 (key = `SONARQUBE_SCANNER_PARAMS` value 格式为 JSON )
3. 仓库文件 (`sonar-project.properties`)
4. 命令行参数 (通过 Java Properties 传递)

最后在 Main 里的 `execute` 方法中执行扫描

再来看看这个方法

### execute

```java
void execute() {
    // 记录执行耗时
    Stats stats = new Stats(logger).start();

    // 默认退出码为1
    int status = Exit.INTERNAL_ERROR;
    try {
      Properties p = conf.properties();
      // 检查是否有 skip 跳过扫描的设置
      checkSkip(p);
      // 根据参数设置 log 级别
      configureLogging(p);
      // 初始化 runner
      init(p);
      // 初始化 变量 ，下载插件
      runner.start();
      logger.info(String.format("Analyzing on %s", conf.isSonarCloud(null) ? "SonarCloud" : ("SonarQube server " + runner.serverVersion())));
      // 执行扫描
      execute(stats, p);
      status = Exit.SUCCESS;
    } catch (Throwable e) {
      displayExecutionResult(stats, "FAILURE");
      showError("Error during SonarScanner execution", e, cli.isDebugEnabled());
      status = isUserError(e) ? Exit.USER_ERROR : Exit.INTERNAL_ERROR;
    } finally {
      exit.exit(status);
    }

  }
```

里边最重要的是 3 个方法，

```java
init(p);
runner.start();
execute(stats, p);
```

### init

```java
private void init(Properties p) {
    SystemInfo.print(logger);
    if (cli.isDisplayVersionOnly()) {
      exit.exit(Exit.SUCCESS);
    }

    runner = runnerFactory.create(p, cli.getInvokedFrom());
  }
class ScannerFactory {

  private final Logs logger;

  public ScannerFactory(Logs logger) {
    this.logger = logger;
  }

  EmbeddedScanner create(Properties props, String isInvokedFrom) {
    String appName = "ScannerCLI";
    String appVersion = ScannerVersion.version();
    if (!isInvokedFrom.equals("") && isInvokedFrom.contains("/")) {
      appName = isInvokedFrom.split("/")[0];
      appVersion = isInvokedFrom.split("/")[1];
    }

    return EmbeddedScanner.create(appName, appVersion, new DefaultLogOutput())
      .addGlobalProperties((Map) props);
  }
```

`EmbeddedScanner runner` 是真正的执行的扫描的类，创建该类的实例化时，提供了一个 `create` 方法，实现了工厂设计模式。

到 `ScannerFactory` 就进入到了 ScannerApi 这个库里

可以看到，更多的细节处理在 SonarScannerApi 中，SonarScannerCli 这个库主要是进行了命令的解析及处理，后续的工作由 SonarScannerApi 来完成。

### SonarScannerApi

目录结构：

![](https://tva1.sinaimg.cn/large/0081Kckwly1gl0nnrbz3oj30ps11otd7.jpg)

我们先看一下 `EmbeddedScanner` 这个入口类

```java
/**
 * Entry point to run SonarQube analysis programmatically.
 *
 * @since 2.2
 */
public class EmbeddedScanner {
  private static final String BITBUCKET_CLOUD_ENV_VAR = "BITBUCKET_BUILD_NUMBER";
  private static final String SONAR_HOST_URL_ENV_VAR = "SONAR_HOST_URL";
  private static final String SONARCLOUD_HOST = "<https://sonarcloud.io>";
  private final IsolatedLauncherFactory launcherFactory;
  private IsolatedLauncher launcher;
  private final LogOutput logOutput;
  private final Map<String, String> globalProperties = new HashMap<>();
  private final Logger logger;
  private final Set<String> classloaderMask = new HashSet<>();
  private final Set<String> classloaderUnmask = new HashSet<>();
  private final System2 system;

  EmbeddedScanner(IsolatedLauncherFactory bl, Logger logger, LogOutput logOutput, System2 system) {
    this.logger = logger;
    this.launcherFactory = bl;
    this.logOutput = logOutput;
    this.classloaderUnmask.add("org.sonarsource.scanner.api.internal.batch.");
    this.system = system;
  }

  public static EmbeddedScanner create(String app, String version, final LogOutput logOutput, System2 system2) {
    Logger logger = new LoggerAdapter(logOutput);
    return new EmbeddedScanner(new IsolatedLauncherFactory(logger), logger, logOutput, system2)
      .setGlobalProperty(InternalProperties.SCANNER_APP, app)
      .setGlobalProperty(InternalProperties.SCANNER_APP_VERSION, version);
  }

  public static EmbeddedScanner create(String app, String version, final LogOutput logOutput) {
    return create(app, version, logOutput, new System2());
  }
```

`create` 用来初始化和实例化 `EmbeddedScanner` 对象，主要有这几参数：

IsolatedLauncherFactory  (启动器工厂类)

Logger (日志接口)

LogOutput (日志输出接口)

System2 (和 System 类相同，额外包了一层)

里边重要的是 `IsolatedLauncherFactory` ，又是一个 工厂类，初始化了 IsolatedLauncherFactory 的属性，调用了构造方法。

综上，`init` 方法进行的操作主要是对几个关键类的实例化操作。

### runner.start()

我们再回到 `EmbeddedScanner.start` 方法里

```java
/**
   * Download scanner-engine JAR and start bootstrapping classloader. After that it is possible to call {@link #serverVersion()}
   */
  public void start() {
    // 设置全局变量
    initGlobalDefaultValues();
    doStart();
  }
```

1. 设置全局参数
2. 调用 runner 的 `doStart` 方法

通过注释我们可以了解到，其功能是下载扫描器的引擎jar包，并利用类加载器加载。看一下 `doStart` 方法

```java
protected void doStart() {
    // 检查 IsolatedLauncher 有没有实例化，如果已经实例化，说明已经启动，抛出异常
    checkLauncherDoesntExist();
    // 创建 load class 的规则
    ClassloadRules rules = new ClassloadRules(classloaderMask, classloaderUnmask);
    // 初始化
    launcher = launcherFactory.createLauncher(globalProperties(), rules);
  }
```

我们重点看下最后一行

利用了 launcher 的工厂类来创建 `IsolatedClassloader` 对象

```java
public IsolatedLauncher createLauncher(Map<String, String> props, ClassloadRules rules) {
    if (props.containsKey(InternalProperties.SCANNER_DUMP_TO_FILE)) {
      String version = props.get(InternalProperties.SCANNER_VERSION_SIMULATION);
      if (version == null) {
        version = "5.6";
      }
      return new SimulatedLauncher(version, logger);
    }
    ServerConnection serverConnection = ServerConnection.create(props, logger);
    JarDownloader jarDownloader = new JarDownloaderFactory(serverConnection, logger, props.get("sonar.userHome")).create();

    return createLauncher(jarDownloader, rules);
  }
```

首先判断是否传递了 `sonar.scanner.dumpToFile` 参数，如果传递后续不再执行扫描，而是把参数全部写入到指定的文件里，推测这个作用是为了调试，所以取名`Simulated` 模拟的含义。

接着和服务端也就是 SonarQube 的服务进行连接。

```java
  public static ServerConnection create(Map<String, String> props, Logger logger) {
    String serverUrl = props.get("sonar.host.url");
    String userAgent = format("%s/%s", props.get(SCANNER_APP), props.get(SCANNER_APP_VERSION));
    return new ServerConnection(serverUrl, userAgent, logger);
  }
```

我们可以看到，使用`OkHttp` 这个库来进行接口请求。

紧接着是 `JarDownloaderFactory` 工厂类的 `create` 方法

```java
JarDownloader create() {
    FileCache fileCache = new FileCacheBuilder(logger)
      .setUserHome(userHome)
      .build();

    BootstrapIndexDownloader bootstrapIndexDownloader = new BootstrapIndexDownloader(serverConnection, logger);
    ScannerFileDownloader scannerFileDownloader = new ScannerFileDownloader(serverConnection);
    JarExtractor jarExtractor = new JarExtractor();
    return new JarDownloader(scannerFileDownloader, bootstrapIndexDownloader, fileCache, jarExtractor, logger);
  }
```

这些方法依旧是为了实例化对象。

接着在 `IsolatedLauncherFactory` 中调用 `IsolatedLauncher createLauncher(final JarDownloader jarDownloader, final ClassloadRules rules)` 方法。

```java
IsolatedLauncher createLauncher(final JarDownloader jarDownloader, final ClassloadRules rules) {
    return AccessController.doPrivileged((PrivilegedAction<IsolatedLauncher>) () -> {
      try {
        // 下载 插件 jar 包
        List<File> jarFiles = jarDownloader.download();
        logger.debug("Create isolated classloader...");
        // 创建自定义类加载器
        cl = createClassLoader(jarFiles, rules);
        // 创建代理类
        IsolatedLauncher objProxy = IsolatedLauncherProxy.create(cl, IsolatedLauncher.class, launcherImplClassName, logger);
        tempCleaning.clean();

        return objProxy;
      } catch (Exception e) {
        // Catch all other exceptions, which relates to reflection
        throw new ScannerException("Unable to execute SonarScanner analysis", e);
      }
    });
  }
```

IsolatedLauncherProxy

```java
public class IsolatedLauncherProxy implements InvocationHandler {
  private final Object proxied;
  private final ClassLoader cl;
  private final Logger logger;

  private IsolatedLauncherProxy(ClassLoader cl, Object proxied, Logger logger) {
    this.cl = cl;
    this.proxied = proxied;
    this.logger = logger;
  }

  public static <T> T create(ClassLoader cl, Class<T> interfaceClass, String proxiedClassName, Logger logger) throws ReflectiveOperationException {
    Object proxied = createProxiedObject(cl, proxiedClassName);
    // interfaceClass needs to be loaded with a parent ClassLoader (common to both ClassLoaders)
    // In addition, Proxy.newProxyInstance checks if the target ClassLoader sees the same class as the one given
    Class<?> loadedInterfaceClass = cl.loadClass(interfaceClass.getName());
    return (T) create(cl, proxied, loadedInterfaceClass, logger);
  }

  public static <T> T create(ClassLoader cl, Object proxied, Class<T> interfaceClass, Logger logger) {
    Class<?>[] c = {interfaceClass};
    return (T) Proxy.newProxyInstance(cl, c, new IsolatedLauncherProxy(cl, proxied, logger));
  }

  @Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    ClassLoader initialContextClassLoader = Thread.currentThread().getContextClassLoader();

    try {
      // 设置类加载器
      Thread.currentThread().setContextClassLoader(cl);
      logger.debug("Execution " + method.getName());
      return method.invoke(proxied, args);
    } catch (UndeclaredThrowableException | InvocationTargetException e) {
      throw unwrapException(e);
    } finally {
      // 还原类加载器
      Thread.currentThread().setContextClassLoader(initialContextClassLoader);
    }
  }

  private static Throwable unwrapException(Throwable e) {
    Throwable cause = e;

    while (cause.getCause() != null) {
      if (cause instanceof UndeclaredThrowableException || cause instanceof InvocationTargetException) {
        cause = cause.getCause();
      } else {
        break;
      }
    }
    return cause;
  }

  private static Object createProxiedObject(ClassLoader cl, String proxiedClassName) throws ClassNotFoundException, InstantiationException, IllegalAccessException {
    Class<?> proxiedClass = cl.loadClass(proxiedClassName);
    return proxiedClass.newInstance();
  }
}
```

`IsolatedLauncherProxy` 是用了JDK的动态代理，通过传入的 ClassLoader ，接口，被代理类的名称 `org.sonarsource.scanner.api.internal.batch.BatchIsolatedLauncher` ，最后生成 `BatchIsolatedLauncher` 的实例，然后 `BatchIsolatedLauncher` 调用了 SonarScannerEngine 的 `Batch` 类，完成扫描。

`JarDownloader`主要是去下载 Jar 包，下载之前优先去本地检查 jar 包的缓存。

```java
List<File> download() {
    List<File> files = new ArrayList<>();
    logger.debug("Extract sonar-scanner-api-batch in temp...");
    files.add(jarExtractor.extractToTemp("sonar-scanner-api-batch").toFile());
    files.addAll(getScannerEngineFiles());
    return files;
  }

  private List<File> getScannerEngineFiles() {
    Collection<JarEntry> index = bootstrapIndexDownloader.getIndex();
    return index.stream()
      .map(jar -> fileCache.get(jar.getFilename(), jar.getHash(), scannerFileDownloader))
      .collect(Collectors.toList());
  }
Collection<JarEntry> getIndex() {
    String index;
    try {
      logger.debug("Get bootstrap index...");
      // 从链接获取文件名和hash
      index = conn.downloadString("/batch/index");
      logger.debug("Get bootstrap completed");
    } catch (Exception e) {
      throw new IllegalStateException("Fail to get bootstrap index from server", e);
    }
    return parse(index);
  }

  // 解析
  private static Collection<JarEntry> parse(String index) {
    Collection<JarEntry> entries = new ArrayList<>();

    String[] lines = index.split("[\\r\\n]+");
    for (String line : lines) {
      try {
        line = line.trim();
        String[] libAndHash = line.split("\\\\|");
        String filename = libAndHash[0];
        String hash = libAndHash[1];
        entries.add(new JarEntry(filename, hash));
      } catch (Exception e) {
        throw new IllegalStateException("Fail to parse entry in bootstrap index: " + line);
      }
    }

    return entries;
  }

  static class JarEntry {
    private String filename;
    private String hash;

    JarEntry(String filename, String hash) {
      this.filename = filename;
      this.hash = hash;
    }

    public String getFilename() {
      return filename;
    }

    public String getHash() {
      return hash;
    }
  }
```

在 BootstrapIndexDownloader 类中，先去获取最新的插件版本 `getIndex()`，地址为 `[/batch/index](<http://sonarqubeurl//batch/index>)` ，以我的 SonarQube 为例，返回的是 `sonar-scanner-engine-shaded-7.9.4-all.jar|6daf938ede67767970bafc194078293b` 接着调用 `parse()` 方法进行解析，前者是文件名，后者是 hash 数值，一般应该为 md5 。

如果本地之前没有下载过，则跳转到 `/batch/file?name=${filename}` 地址去下载。

解压 jar 包的内容，如下：

```java
META-INF              com                   freemarker            linux                 okhttp3               org                   sq-version.txt
ch                    darwin                google                net                   okio                  sonar-api-version.txt win32
```

Jar包文件的内容主要有两部分，一部分是 .class 文件，是用来执行扫描和跟服务器进行数据传输的，一部分是  proto 文件，说明scanner和服务端通信是用了 `protobuffer` 协议。

接着使用到了Java的类加载器机制，将刚刚下载到的 Jar 包装到类加载器中去加载，自定义了一个类加载器 `IsolatedClassloader`

```java
/**
 * Special {@link java.net.URLClassLoader} to execute batch, which restricts loading from parent.
 */
class IsolatedClassloader extends URLClassLoader {
  private final ClassloadRules rules;

  /**
   * The parent classloader is used only for loading classes and resources in unmasked packages
   */
  IsolatedClassloader(ClassLoader parent, ClassloadRules rules) {
    super(new URL[0], parent);
    this.rules = rules;
  }

  void addFiles(List<File> files) {
    try {
      for (File file : files) {
        addURL(file.toURI().toURL());
      }
    } catch (MalformedURLException e) {
      throw new IllegalStateException("Fail to create classloader", e);
    }
  }

  /**
   * Same behavior as in {@link java.net.URLClassLoader#loadClass(String, boolean)}, except loading from parent.
   */
  @Override
  protected synchronized Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {
    // First, check if the class has already been loaded
    Class<?> c = findLoadedClass(name);
    if (c == null) {
      try {
        // Load from parent
        if (getParent() != null && rules.canLoad(name)) {
          c = getParent().loadClass(name);
        } else {

          // Load from system

          // I don't know for other vendors, but for Oracle JVM :
          // - ClassLoader.getSystemClassLoader() is sun.misc.Launcher$AppClassLoader. It contains app classpath.
          // - ClassLoader.getSystemClassLoader().getParent() is sun.misc.Launcher$ExtClassLoader. It contains core JVM
          ClassLoader systemClassLoader = getSystemClassLoader();
          if (systemClassLoader.getParent() != null) {
            systemClassLoader = systemClassLoader.getParent();
          }
          c = systemClassLoader.loadClass(name);
        }
      } catch (ClassNotFoundException e) {
        // If still not found, then invoke findClass in order
        // to find the class.
        c = findClass(name);
      }
    }
    if (resolve) {
      resolveClass(c);
    }
    return c;
  }
```

看一下 `IsolatedClassloader` 发现它继承了`URLClassLoader`

在常规加载器的内容上，判断了该类是否允许被加载`rules.canLoad`，不过查看源码，并没有地方被调用。

接着使用 JDK 动态代理技术，创建 `org.sonarsource.scanner.api.internal.batch.BatchIsolatedLauncher` 的代理类并返回，到这里，start 方法的流程结束了

### execute()

最后回到 Main 类的 execute方法，调用了 `EmbeddedScanner` 的 `execute` 方法

```java
protected void doExecute(Map<String, String> properties) {
    launcher.execute(properties, (formattedMessage, level) -> logOutput.log(formattedMessage, LogOutput.Level.valueOf(level.name())));
  }
```

这里边只做了一件事就是调用 `IsolatedLauncher` 的 `execute` 方法，通过查看源码可以知道这是个接口.

```java
public interface IsolatedLauncher {
    void execute(Map<String, String> var1, LogOutput var2);

    String getVersion();
}
```

并且，该源码内继承该接口的类只有一个，就是刚刚的 模拟类 `SimulatedLauncher` ，通过刚刚的源码可以知道这个类只是用来测试或者模拟用的，并不是真正的执行类，那真正的执行代码在哪？

查看代码之后发现在 `org/sonarsource/scanner/api/internal/batch/BatchIsolatedLauncher.java`

```java
@Override
  public void execute(Map<String, String> properties, org.sonarsource.scanner.api.internal.batch.LogOutput logOutput) {
    factory.createBatch(properties, logOutput).execute();
  }
class DefaultBatchFactory implements BatchFactory {
  private static final String SCANNER_APP_KEY = "sonar.scanner.app";
  private static final String SCANNER_APP_VERSION_KEY = "sonar.scanner.appVersion";

  @Override
  public Batch createBatch(Map<String, String> properties, final org.sonarsource.scanner.api.internal.batch.LogOutput logOutput) {
    EnvironmentInformation env = new EnvironmentInformation(properties.get(SCANNER_APP_KEY), properties.get(SCANNER_APP_VERSION_KEY));
    return Batch.builder()
      .setEnvironment(env)
      .setGlobalProperties(properties)
      .setLogOutput((formattedMessage, level) -> logOutput.log(formattedMessage, LogOutput.Level.valueOf(level.name())))
      .build();
  }
}
```

`BatchIsolatedLauncher` 的 `execute` 方法内，先用工厂类创建了一个 `Batch` 对象，设置参数等信息，然后执行 Batch 的 execute 方法。

```java
public synchronized Batch execute() {
    configureLogging();
    doStart();
    boolean threw = true;
    try {
      doExecuteTask(globalProperties);
      threw = false;
    } finally {
      doStop(threw);
    }
    return this;
  }
```

到这里就开始真正的代码扫描的任务执行了，会进入到下一个模块，`sonar-scanner-engine`

该代码位于 `SonarQube` 源码仓库下边的 `sonar-scanner-engine` 模块。

https://github.com/SonarSource/sonarqube/tree/master/sonar-scanner-engine

后续再单独针对这块代码的分析。

## 小结

SonarScanner 设计的很巧妙，安装 SonarScanner 只需要一个jar 包，大小500KB左右，分析代码需要的插件代码会自动从 SonarQube 的服务器下载，使用自定义的ClassLoader来加载从服务器上下载的jar包，然后使用这个启动器类调用了 SonarScannerEngine 的 Batch 启动并执行代码扫描。这种方式对于客户端(Scanner)来说不需要维护和更新插件，插件的更新在服务端就可以完成，减少了分发更新的成本。

参考：

[sonarqube+sonar-scanner-engine扫描引擎主要执行步骤](https://www.jianshu.com/p/c22213591f47)

[Sonar 质量扫描的输出日志--对应源码的跟踪（一）{源码解析sonar-scanner-maven3.2}](https://blog.csdn.net/lxlmycsdnfree/article/details/80283697)

[sonar-scanner的执行流程和对ClassLoader,动态代理的使用 - 梦中彩虹 - 博客园](https://www.cnblogs.com/jiaoyiping/p/9691620.html)