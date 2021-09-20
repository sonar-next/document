# Sonar-Scanner-Engine源码分析

---

title: Sonar-Scanner-Engine源码分析

date: 2021-09-18 

author: mamian521#gmail.com

---

## 介绍

根据 sonar-scanner-cli 和 sonar-scanner-api 的源码分析，我们得知，真正执行本地扫描的源码并不在这两处，而是在另外一个模块里，`sonar-scanner-engine` 

`地址：[https://github.com/SonarSource/sonarqube.git](https://github.com/SonarSource/sonarqube.git)`

本文基于 `branch-9.0` 分支

初始化 `Batch`

```java
@Override
  public Batch createBatch(Map<String, String> properties, final org.sonarsource.scanner.api.internal.batch.LogOutput logOutput) {
    EnvironmentInformation env = new EnvironmentInformation(properties.get(SCANNER_APP_KEY), properties.get(SCANNER_APP_VERSION_KEY));
    return Batch.builder()
      .setEnvironment(env)
      .setGlobalProperties(properties)
      .setLogOutput((formattedMessage, level) -> logOutput.log(formattedMessage, LogOutput.Level.valueOf(level.name())))
      .build();
  }
```

## 入口类

`org/sonar/batch/bootstrapper/Batch.java`

```java
// 入口方法
public synchronized Batch execute() {
    return doExecute(this.globalProperties, this.components);
  }

  public synchronized Batch doExecute(Map<String, String> scannerProperties, List<Object> components) {
    configureLogging();
    try {
      GlobalContainer.create(scannerProperties, components).execute();
    } catch (RuntimeException e) {
      throw handleException(e);
    }
    return this;
  }

private Batch(Builder builder) {
    components = new ArrayList<>();
    components.addAll(builder.components);
    if (builder.environment != null) {
      components.add(builder.environment);
    }
    if (builder.globalProperties != null) {
      globalProperties.putAll(builder.globalProperties);
    }
    if (builder.isEnableLoggingConfiguration()) {
      loggingConfig = new LoggingConfiguration(builder.environment).setProperties(globalProperties);

      if (builder.logOutput != null) {
        loggingConfig.setLogOutput(builder.logOutput);
      }
    }
  }
```

根据源码可以看到

`Batch` 初始化的时候，主要有两个参数，一个是读取配置信息，properties，另一个是 component 这里传入的是 `EnvironmentInformation` ，环境信息的类，主要保存的是scanner的版本。

然后开始执行 `execute` 方法，加了同步锁。

然后执行 `GlobalContainer` 的 `create` 和 `execute` 方法。顾名思义，一个是实例化类的方法，另一个是执行的方法。

```java
public static GlobalContainer create(Map<String, String> scannerProperties, List<?> extensions) {
    GlobalContainer container = new GlobalContainer(scannerProperties);
    container.add(extensions);
    return container;
  }

............
@Override
  public ComponentContainer add(Object... objects) {
    for (Object object : objects) {
      if (object instanceof ComponentAdapter) {
        addPicoAdapter((ComponentAdapter) object);
      } else if (object instanceof Iterable) {
        // 递归，重复注入
        add(Iterables.toArray((Iterable) object, Object.class));
      } else {
        addSingleton(object);
      }
    }
    return this;
  }

........
public ComponentContainer addComponent(Object component, boolean singleton) {
    Object key = componentKeys.of(component);
    if (component instanceof ComponentAdapter) {
      pico.addAdapter((ComponentAdapter) component);
    } else {
      try {
        pico.as(singleton ? Characteristics.CACHE : Characteristics.NO_CACHE).addComponent(key, component);
      } catch (Throwable t) {
        throw new IllegalStateException("Unable to register component " + getName(component), t);
      }
      declareExtension("", component);
    }
    return this;
  }
```

执行 add 方法时，可以看到一些关键字，例如 container/singleton ，其实可以推断出，他这个地方是使用了一个依赖注入的一个框架，类似于我们现在频繁使用的 `Spring` ，如果是可迭代的对象 `Iterable` ，那就会进入递归调用里，循环注入。如果是其他类型，例如我们传入的 `EnvironmentInformation` 就会注入为一个单例的对象。

这里使用到的框架为 picocontainer 

官网：[http://picocontainer.com/introduction.html](http://picocontainer.com/introduction.html)

介绍：PicoContainer是非常轻量级的Ioc容器，提供依赖注入和对象生命周期管理的功能，纯粹的小而美的Ioc容器。而Spring是Ioc+，提供如AOP等其他功能，是大而全的框架，不只是Ioc容器。

推测使用该框架的原因是，轻量级或者SonarQubu创始人比较熟悉，所以使用了该框架，看起来目前已经不怎么更新维护了，目前SonarQube也没有替换的动作。我们就把它当作成一个Spring框架来看即可。

接着看 `execute` 方法

```java
public void execute() {
    try {
      startComponents();
    } finally {
//    手动销毁注入过的容器
      stopComponents();
    }
  }

// 模版设计模式
/**
   * This method MUST NOT be renamed start() because the container is registered itself in picocontainer. Starting
   * a component twice is not authorized.
   */
  public ComponentContainer startComponents() {
    try {
      // 调用子类 GlobalContainer/ProjectScanContainer 的方法
      doBeforeStart();
      // 手动启动容器
      pico.start();
      // 调用子类 GlobalContainer/ProjectScanContainer 的方法
      doAfterStart();
      return this;
    } catch (Exception e) {
      throw PicoUtils.propagate(e);
    }
  }

......
@Override
  protected void doBeforeStart() {
    GlobalProperties bootstrapProps = new GlobalProperties(bootstrapProperties);
    GlobalAnalysisMode globalMode = new GlobalAnalysisMode(bootstrapProps);
    // 注入
    add(bootstrapProps);
    add(globalMode);
    addBootstrapComponents();
  }

// 依次注入
private void addBootstrapComponents() {
    Version apiVersion = MetadataLoader.loadVersion(System2.INSTANCE);
    SonarEdition edition = MetadataLoader.loadEdition(System2.INSTANCE);
    DefaultAnalysisWarnings analysisWarnings = new DefaultAnalysisWarnings(System2.INSTANCE);
    LOG.debug("{} {}", edition.getLabel(), apiVersion);
    add(
      // plugins
      ScannerPluginRepository.class,
      PluginClassLoader.class,
      PluginClassloaderFactory.class,
      ScannerPluginJarExploder.class,
      ExtensionInstaller.class,
      new SonarQubeVersion(apiVersion),
      new GlobalServerSettingsProvider(),
      new GlobalConfigurationProvider(),
      new ScannerWsClientProvider(),
      DefaultServer.class,
      new GlobalTempFolderProvider(),
      DefaultHttpDownloader.class,
      analysisWarnings,
      UriReader.class,
      PluginFiles.class,
      System2.INSTANCE,
      Clock.systemDefaultZone(),
      new MetricsRepositoryProvider(),
      UuidFactoryImpl.INSTANCE);
    addIfMissing(SonarRuntimeImpl.forSonarQube(apiVersion, SonarQubeSide.SCANNER, edition), SonarRuntime.class);
    addIfMissing(ScannerPluginInstaller.class, PluginInstaller.class);
    add(CoreExtensionRepositoryImpl.class, CoreExtensionsLoader.class, ScannerCoreExtensionsInstaller.class);
    addIfMissing(DefaultGlobalSettingsLoader.class, GlobalSettingsLoader.class);
    addIfMissing(DefaultNewCodePeriodLoader.class, NewCodePeriodLoader.class);
    addIfMissing(DefaultMetricsRepositoryLoader.class, MetricsRepositoryLoader.class);
  }

@Override
  protected void doAfterStart() {
// 安装插件
    installPlugins();
// 使用类加载器加载
    loadCoreExtensions();

    long startTime = System.currentTimeMillis();
    String taskKey = StringUtils.defaultIfEmpty(scannerProperties.get(CoreProperties.TASK), CoreProperties.SCAN_TASK);
    if (taskKey.equals("views")) {
      throw MessageException.of("The task 'views' was removed with SonarQube 7.1. " +
        "You can safely remove this call since portfolios and applications are automatically re-calculated.");
    } else if (!taskKey.equals(CoreProperties.SCAN_TASK)) {
      throw MessageException.of("Tasks support was removed in SonarQube 7.6.");
    }
    String analysisMode = StringUtils.defaultIfEmpty(scannerProperties.get("sonar.analysis.mode"), "publish");
    if (!analysisMode.equals("publish")) {
      throw MessageException.of("The preview mode, along with the 'sonar.analysis.mode' parameter, is no more supported. You should stop using this parameter.");
    }
    new ProjectScanContainer(this).execute();

    LOG.info("Analysis total time: {}", formatTime(System.currentTimeMillis() - startTime));
  }
```

梳理一下，这里就是把要启动的类作注入进去，然后手动调用启动的方法。

最后真正的执行方法是 `new ProjectScanContainer(this).execute();`

这个类 `ProjectScanContainer` 同样和 `GlobalContainer` 都继承自 `ComponentContainer` ，所以父类的 execute 方法里的 `doBeforeStart` `doAfterStart` 他们将其重写了。这里其实用到了一个设计模式，就是模版设计模式，父类实现了一个 execute 方法，但是子类不能覆盖，该方法内的其他方法，子类是可以覆盖的，这就是模版设计模式

![Untitled](Sonar-Scanner-Engine%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%20d54f744fd2f64bb5bb7189fcd43629f4/Untitled.png)

我们主要看下 ProjectScanContainer 这个类，这个是执行扫描的类，可以看到这几个方法

我们回顾一下父类的这几个方法

```java
// ComponentContaienr
  public ComponentContainer startComponents() {
    try {
      // 调用子类 GlobalContainer/ProjectScanContainer 的方法
      doBeforeStart();
      // 手动启动容器
      pico.start();
      // 调用子类 GlobalContainer/ProjectScanContainer 的方法
      doAfterStart();
      return this;
    } catch (Exception e) {
      throw PicoUtils.propagate(e);
    }
  }

```

1. doBeforeStart()
2. pico.start()
3. doAfterStart()

所以，我们按照这个顺序看一下这些方法

```java
// ProjectScanContainer
@Override
  protected void doBeforeStart() {

// 加载插件相关类
    addScannerExtensions();

// 重要，加载扫描的组件
    addScannerComponents();
// 仓库文件锁，检测是否有相同的扫描
    ProjectLock lock = getComponentByType(ProjectLock.class);
    lock.tryLock();
// 初始化工作目录
    getComponentByType(WorkDirectoriesInitializer.class).execute();
  }

private void addScannerExtensions() {
    getComponentByType(CoreExtensionsInstaller.class)
      .install(this, noExtensionFilter(), extension -> getScannerProjectExtensionsFilter().accept(extension));
    getComponentByType(ExtensionInstaller.class)
      .install(this, getScannerProjectExtensionsFilter());
  }

// 加载需要的组件
private void addScannerComponents() {
    add(
      ScanProperties.class,
      ProjectReactorBuilder.class,
      WorkDirectoriesInitializer.class,
      new MutableProjectReactorProvider(),
      ProjectBuildersExecutor.class,
      ProjectLock.class,
      ResourceTypes.class,
      ProjectReactorValidator.class,
      ProjectInfo.class,
      new RulesProvider(),
      new BranchConfigurationProvider(),
      new ProjectBranchesProvider(),
      new ProjectPullRequestsProvider(),
      ProjectRepositoriesSupplier.class,
      new ProjectServerSettingsProvider(),

      // temp
      new AnalysisTempFolderProvider(),

      // file system
      ModuleIndexer.class,
      InputComponentStore.class,
      PathResolver.class,
      new InputProjectProvider(),
      new InputModuleHierarchyProvider(),
      ScannerComponentIdGenerator.class,
      new ScmChangedFilesProvider(),
      StatusDetection.class,
      LanguageDetection.class,
      MetadataGenerator.class,
      FileMetadata.class,
      FileIndexer.class,
      ProjectFileIndexer.class,
      ProjectExclusionFilters.class,

      // rules
      new ActiveRulesProvider(),
      new QualityProfilesProvider(),
      CheckFactory.class,
      QProfileVerifier.class,

      // issues
      DefaultNoSonarFilter.class,
      IssueFilters.class,
      IssuePublisher.class,

      // metrics
      DefaultMetricFinder.class,

      // lang
      Languages.class,
      DefaultLanguagesRepository.class,

      // issue exclusions
      IssueInclusionPatternInitializer.class,
      IssueExclusionPatternInitializer.class,
      IssueExclusionsLoader.class,
      EnforceIssuesFilter.class,
      IgnoreIssuesFilter.class,

      // context
      ContextPropertiesCache.class,
      ContextPropertiesPublisher.class,

      SensorStrategy.class,

      MutableProjectSettings.class,
      ScannerProperties.class,
      new ProjectConfigurationProvider(),

      ProjectCoverageAndDuplicationExclusions.class,

      // Report
      ForkDateSupplier.class,
      ScannerMetrics.class,
      ReportPublisher.class,
      AnalysisContextReportPublisher.class,
      MetadataPublisher.class,
      ActiveRulesPublisher.class,
      AnalysisWarningsPublisher.class,
      ComponentsPublisher.class,
      TestExecutionPublisher.class,
      SourcePublisher.class,
      ChangedLinesPublisher.class,

      CeTaskReportDataHolder.class,

      // QualityGate check
      QualityGateCheck.class,

      // Cpd
      CpdExecutor.class,
      CpdSettings.class,
      SonarCpdBlockIndex.class,

      // PostJobs
      PostJobsExecutor.class,
      PostJobOptimizer.class,
      DefaultPostJobContext.class,
      PostJobExtensionDictionnary.class,

      // SCM
      ScmConfiguration.class,
      ScmPublisher.class,
      ScmRevisionImpl.class,

      // Sensors
      DefaultSensorStorage.class,
      DefaultFileLinesContextFactory.class,
      ProjectSensorContext.class,
      ProjectSensorOptimizer.class,
      ProjectSensorsExecutor.class,
      ProjectSensorExtensionDictionnary.class,

      // Filesystem
      DefaultProjectFileSystem.class,

      // CI
      new CiConfigurationProvider(),
      AppVeyor.class,
      AwsCodeBuild.class,
      AzureDevops.class,
      Bamboo.class,
      BitbucketPipelines.class,
      Bitrise.class,
      Buildkite.class,
      CircleCi.class,
      CirrusCi.class,
      DroneCi.class,
      GithubActions.class,
      GitlabCi.class,
      Jenkins.class,
      SemaphoreCi.class,
      TravisCi.class,

      AnalysisObservers.class);

    add(GitScmSupport.getObjects());
    add(SvnScmSupport.getObjects());

    addIfMissing(DefaultProjectSettingsLoader.class, ProjectSettingsLoader.class);
    addIfMissing(DefaultRulesLoader.class, RulesLoader.class);
    addIfMissing(DefaultActiveRulesLoader.class, ActiveRulesLoader.class);
    addIfMissing(DefaultQualityProfileLoader.class, QualityProfileLoader.class);
    addIfMissing(DefaultProjectRepositoriesLoader.class, ProjectRepositoriesLoader.class);
  }

  private void addScannerExtensions() {
    getComponentByType(CoreExtensionsInstaller.class)
      .install(this, noExtensionFilter(), extension -> getScannerProjectExtensionsFilter().accept(extension));
    getComponentByType(ExtensionInstaller.class)
      .install(this, getScannerProjectExtensionsFilter());
  }

......
@Override
  protected void doAfterStart() {
    GlobalAnalysisMode analysisMode = getComponentByType(GlobalAnalysisMode.class);
    InputModuleHierarchy tree = getComponentByType(InputModuleHierarchy.class);
    ScanProperties properties = getComponentByType(ScanProperties.class);
    properties.validate();

    properties.get("sonar.branch").ifPresent(deprecatedBranch -> {
      throw MessageException.of("The 'sonar.branch' parameter is no longer supported. You should stop using it. " +
        "Branch analysis is available in Developer Edition and above. See https://redirect.sonarsource.com/editions/developer.html for more information.");
    });

    BranchConfiguration branchConfig = getComponentByType(BranchConfiguration.class);
    if (branchConfig.branchType() == BranchType.PULL_REQUEST) {
      LOG.info("Pull request {} for merge into {} from {}", branchConfig.pullRequestKey(), pullRequestBaseToDisplayName(branchConfig.targetBranchName()),
        branchConfig.branchName());
    } else if (branchConfig.branchName() != null) {
      LOG.info("Branch name: {}", branchConfig.branchName());
    }

// 文件索引
    getComponentByType(ProjectFileIndexer.class).index();

    // Log detected languages and their profiles after FS is indexed and languages detected
    getComponentByType(QProfileVerifier.class).execute();

// 检查 module 模块，执行 ModuleContainer
    scanRecursively(tree, tree.root());

    LOG.info("------------- Run sensors on project");

// 扫描
    getComponentByType(ProjectSensorsExecutor.class).execute();

// 代码提交信息 git blame
    getComponentByType(ScmPublisher.class).publish();

// 代码重复率
    getComponentByType(CpdExecutor.class).execute();

// 生成报告，zip压缩，上传
    getComponentByType(ReportPublisher.class).execute();

// 是否要等待质量阀，内部走接口轮训等待后台任务结束
    if (properties.shouldWaitForQualityGate()) {
      LOG.info("------------- Check Quality Gate status");
      getComponentByType(QualityGateCheck.class).await();
    }

    getComponentByType(PostJobsExecutor.class).execute();

    if (analysisMode.isMediumTest()) {
// 测试后通知扫描完毕，观察者设计模式
      getComponentByType(AnalysisObservers.class).notifyEndOfScanTask();
    }
  }
```

这应该是相当重要的一个流程了，可以看到 doBeforeStart 里主要做的是先加载 plugin，这些 plugin 是之前下载好的 jar 包的插件，根据源码可知，主要是添加了 `ScannerSide` 注解的类。

第二步，在 `addScannerComponents` 方法中，将上述所有的组件加载注入

第三步，执行 `pico.start()` 方法，这里 pico 是我们之前提到的依赖注入的工具，上面这些类注入之后，部分的类继承自 `Startable` 接口，调用 `pico.start()` 之后，也就会触发这些组件的 `start()` 方法。

第四步，执行 `doAfterStart` 方法，在这个里边，才执行了扫描中各组件的 `execute` 方法。

我们重点看一下 doAfterStart 里的一些类。

我列举几个

- ProjectLock
- ProjectFileIndexer
- ModuleScanContainer
- ProjectSensorsExecutor
- ScmPublisher
- ExtensionInstaller
- CpdExecutor
- ReportPublisher
- PostJobsExecutor

我们逐个看一下。

### ProjectLock

先看下源码

```java
public class DirectoryLock {
  private static final Logger LOGGER = LoggerFactory.getLogger(DirectoryLock.class);
  public static final String LOCK_FILE_NAME = ".sonar_lock";

  private final Path lockFilePath;

  private RandomAccessFile lockRandomAccessFile;
  private FileChannel lockChannel;
  private FileLock lockFile;

  public DirectoryLock(Path directory) {
    this.lockFilePath = directory.resolve(LOCK_FILE_NAME).toAbsolutePath();
  }

  public boolean tryLock() {
    try {
      lockRandomAccessFile = new RandomAccessFile(lockFilePath.toFile(), "rw");
      lockChannel = lockRandomAccessFile.getChannel();
      lockFile = lockChannel.tryLock(0, 1024, false);

      return lockFile != null;
    } catch (IOException e) {
      throw new IllegalStateException("Failed to create lock in " + lockFilePath.toString(), e);
    }
  }
```

其大致的含义就是在项目里sonar的工作目录中建立一个 `.sonar_lock` 文件，用于检测当前项目是否正在扫描，注意，其用到的类，是 `FileChannel` 而不是普通的 `Files` ，这是 Java NIO 的一个应用，具体可以参考这个文章。[https://www.baeldung.com/java-lock-files](https://www.baeldung.com/java-lock-files)

```java
      lockFile = lockChannel.tryLock(0, 1024, false);
```

届时会创建一个1024byte大小的文件，空内容，一个文件作为独占锁。

这个主要的目的就是同时执行多个 sonar-scanner 的时候，多个进程的信息互相是看不到的，利用这个锁来保证同时在一个仓库下，只有同一个扫描进程。

### ProjectFileIndexer

在 SonarQube 里，有可以设置扫描的文件范围，排除的文件范围，测试文件的范围，测试文件的排除范围，gitignore，单元测试和重复文件的排除范围，所以这个类就是用来索引一遍文件，看哪些文件需要扫描，哪些是测试范围的文件。

内部主要调用了 `indexModulesRecursively` 方法

```java
public void index() {
    progressReport = new ProgressReport("Report about progress of file indexation", TimeUnit.SECONDS.toMillis(10));
    progressReport.start("Indexing files...");
    LOG.info("Project configuration:");
    projectExclusionFilters.log("  ");
    projectCoverageAndDuplicationExclusions.log("  ");
    ExclusionCounter exclusionCounter = new ExclusionCounter();

    if (useScmExclusion) {
      ignoreCommand.init(inputModuleHierarchy.root().getBaseDir().toAbsolutePath());
      indexModulesRecursively(inputModuleHierarchy.root(), exclusionCounter);
      ignoreCommand.clean();
    } else {
      indexModulesRecursively(inputModuleHierarchy.root(), exclusionCounter);
    }

    int totalIndexed = componentStore.inputFiles().size();
    progressReport.stop(totalIndexed + " " + pluralizeFiles(totalIndexed) + " indexed");

    int excludedFileByPatternCount = exclusionCounter.getByPatternsCount();
    if (projectExclusionFilters.hasPattern() || excludedFileByPatternCount > 0) {
      LOG.info("{} {} ignored because of inclusion/exclusion patterns", excludedFileByPatternCount, pluralizeFiles(excludedFileByPatternCount));
    }
    int excludedFileByScmCount = exclusionCounter.getByScmCount();
    if (useScmExclusion) {
      LOG.info("{} {} ignored because of scm ignore settings", excludedFileByScmCount, pluralizeFiles(excludedFileByScmCount));
    }
  }

private void indexModulesRecursively(DefaultInputModule module, ExclusionCounter exclusionCounter) {
    inputModuleHierarchy.children(module).stream().sorted(Comparator.comparing(DefaultInputModule::key)).forEach(m -> indexModulesRecursively(m, exclusionCounter));
    index(module, exclusionCounter);
  }

private void index(DefaultInputModule module, ExclusionCounter exclusionCounter) {
    // Emulate creation of module level settings
    ModuleConfiguration moduleConfig = new ModuleConfigurationProvider().provide(globalConfig, module, globalServerSettings, projectServerSettings);
    ModuleExclusionFilters moduleExclusionFilters = new ModuleExclusionFilters(moduleConfig);
    ModuleCoverageAndDuplicationExclusions moduleCoverageAndDuplicationExclusions = new ModuleCoverageAndDuplicationExclusions(moduleConfig);
    if (componentStore.allModules().size() > 1) {
      LOG.info("Indexing files of module '{}'", module.getName());
      LOG.info("  Base dir: {}", module.getBaseDir().toAbsolutePath());
      module.getSourceDirsOrFiles().ifPresent(srcs -> logPaths("  Source paths: ", module.getBaseDir(), srcs));
      module.getTestDirsOrFiles().ifPresent(tests -> logPaths("  Test paths: ", module.getBaseDir(), tests));
      moduleExclusionFilters.log("  ");
      moduleCoverageAndDuplicationExclusions.log("  ");
    }
    boolean hasChildModules = !module.definition().getSubProjects().isEmpty();
    boolean hasTests = module.getTestDirsOrFiles().isPresent();
    // Default to index basedir when no sources provided
    List<Path> mainSourceDirsOrFiles = module.getSourceDirsOrFiles()
      .orElseGet(() -> hasChildModules || hasTests ? emptyList() : singletonList(module.getBaseDir().toAbsolutePath()));
    indexFiles(module, moduleExclusionFilters, moduleCoverageAndDuplicationExclusions, mainSourceDirsOrFiles, Type.MAIN, exclusionCounter);
    module.getTestDirsOrFiles().ifPresent(tests -> indexFiles(module, moduleExclusionFilters, moduleCoverageAndDuplicationExclusions, tests, Type.TEST, exclusionCounter));
  }

private void indexFiles(DefaultInputModule module, ModuleExclusionFilters moduleExclusionFilters,
    ModuleCoverageAndDuplicationExclusions moduleCoverageAndDuplicationExclusions, List<Path> sources, Type type, ExclusionCounter exclusionCounter) {
    try {
      for (Path dirOrFile : sources) {
        if (dirOrFile.toFile().isDirectory()) {
          indexDirectory(module, moduleExclusionFilters, moduleCoverageAndDuplicationExclusions, dirOrFile, type, exclusionCounter);
        } else {
          fileIndexer.indexFile(module, moduleExclusionFilters, moduleCoverageAndDuplicationExclusions, dirOrFile, type, progressReport, exclusionCounter,
            ignoreCommand);
        }
      }
    } catch (IOException e) {
      throw new IllegalStateException("Failed to index files", e);
    }
  }

......
// FileIndexer
void indexFile(DefaultInputModule module, ModuleExclusionFilters moduleExclusionFilters, ModuleCoverageAndDuplicationExclusions moduleCoverageAndDuplicationExclusions,
    Path sourceFile, Type type, ProgressReport progressReport, ProjectFileIndexer.ExclusionCounter exclusionCounter, @Nullable IgnoreCommand ignoreCommand)
    throws IOException {
    // get case of real file without resolving link
    Path realAbsoluteFile = sourceFile.toRealPath(LinkOption.NOFOLLOW_LINKS).toAbsolutePath().normalize();
    Path projectRelativePath = project.getBaseDir().relativize(realAbsoluteFile);
    Path moduleRelativePath = module.getBaseDir().relativize(realAbsoluteFile);
    boolean included = evaluateInclusionsFilters(moduleExclusionFilters, realAbsoluteFile, projectRelativePath, moduleRelativePath, type);
    if (!included) {
      exclusionCounter.increaseByPatternsCount();
      return;
    }
    boolean excluded = evaluateExclusionsFilters(moduleExclusionFilters, realAbsoluteFile, projectRelativePath, moduleRelativePath, type);
    if (excluded) {
      exclusionCounter.increaseByPatternsCount();
      return;
    }
    if (!realAbsoluteFile.startsWith(project.getBaseDir())) {
      LOG.warn("File '{}' is ignored. It is not located in project basedir '{}'.", realAbsoluteFile.toAbsolutePath(), project.getBaseDir());
      return;
    }
    if (!realAbsoluteFile.startsWith(module.getBaseDir())) {
      LOG.warn("File '{}' is ignored. It is not located in module basedir '{}'.", realAbsoluteFile.toAbsolutePath(), module.getBaseDir());
      return;
    }

    String language = langDetection.language(realAbsoluteFile, projectRelativePath);

    if (ignoreCommand != null && ignoreCommand.isIgnored(realAbsoluteFile)) {
      LOG.debug("File '{}' is excluded by the scm ignore settings.", realAbsoluteFile);
      exclusionCounter.increaseByScmCount();
      return;
    }

    DefaultIndexedFile indexedFile = new DefaultIndexedFile(realAbsoluteFile, project.key(),
      projectRelativePath.toString(),
      moduleRelativePath.toString(),
      type, language, scannerComponentIdGenerator.getAsInt(), sensorStrategy);
    DefaultInputFile inputFile = new DefaultInputFile(indexedFile, f -> metadataGenerator.setMetadata(module.key(), f, module.getEncoding()));
    if (language != null) {
      inputFile.setPublished(true);
    }
    if (!accept(inputFile)) {
      return;
    }
    checkIfAlreadyIndexed(inputFile);
    componentStore.put(module.key(), inputFile);
    issueExclusionsLoader.addMulticriteriaPatterns(inputFile);
    String langStr = inputFile.language() != null ? format("with language '%s'", inputFile.language()) : "with no language";
    LOG.debug("'{}' indexed {}{}", projectRelativePath, type == Type.TEST ? "as test " : "", langStr);
    evaluateCoverageExclusions(moduleCoverageAndDuplicationExclusions, inputFile);
    evaluateDuplicationExclusions(moduleCoverageAndDuplicationExclusions, inputFile);
    if (properties.preloadFileMetadata()) {
      inputFile.checkMetadata();
    }
    int count = componentStore.inputFiles().size();
    progressReport.message(count + " " + pluralizeFiles(count) + " indexed...  (last one was " + inputFile.getProjectRelativePath() + ")");
  }

......
// InputComponentStore
public InputComponentStore put(String moduleKey, InputFile inputFile) {
    DefaultInputFile file = (DefaultInputFile) inputFile;
    updateNotAnalysedCAndCppFileCount(file);

    addToLanguageCache(moduleKey, file);
    inputFileByModuleCache.computeIfAbsent(moduleKey, x -> new HashMap<>()).put(file.getModuleRelativePath(), inputFile);
    inputModuleKeyByFileCache.put(inputFile, moduleKey);
    globalInputFileCache.put(file.getProjectRelativePath(), inputFile);
    inputComponents.put(inputFile.key(), inputFile);
    filesByNameCache.computeIfAbsent(inputFile.filename(), x -> new LinkedHashSet<>()).add(inputFile);
    filesByExtensionCache.computeIfAbsent(FileExtensionPredicate.getExtension(inputFile), x -> new LinkedHashSet<>()).add(inputFile);
    return this;
  }
```

可以看到，最后真正调用的是 `InputComponentStore#put()` 这个方法，详细看一下  InputComponentStore 这个类。

```java
private final SortedSet<String> globalLanguagesCache = new TreeSet<>();
private final Map<String, SortedSet<String>> languagesCache = new HashMap<>();
private final Map<String, InputFile> globalInputFileCache = new HashMap<>();
private final Map<String, Map<String, InputFile>> inputFileByModuleCache = new LinkedHashMap<>();
private final Map<InputFile, String> inputModuleKeyByFileCache = new HashMap<>();
private final Map<String, DefaultInputModule> inputModuleCache = new HashMap<>();
private final Map<String, InputComponent> inputComponents = new HashMap<>();
private final Map<String, Set<InputFile>> filesByNameCache = new HashMap<>();
private final Map<String, Set<InputFile>> filesByExtensionCache = new HashMap<>();
private final BranchConfiguration branchConfiguration;
private final SonarRuntime sonarRuntime;
private final Map<String, Integer> notAnalysedFilesByLanguage = new HashMap<>();
```

再看一下 DefaultInputFile 

```java
public class DefaultInputFile extends DefaultInputComponent implements InputFile {

  private static final intDEFAULT_BUFFER_SIZE= 1024 * 4;

  private final DefaultIndexedFile indexedFile;
  private final String contents;
  private final Consumer<DefaultInputFile> metadataGenerator;

// 是否检测到了语言
  private boolean published;
  private boolean excludedForCoverage;
  private boolean excludedForDuplication;
  private boolean ignoreAllIssues;
  // Lazy init to save memory
// 代码行中包含 //NOSONAR 则不会存储
  private BitSet noSonarLines;
  private Status status;
  private Charset charset;
  private Metadata metadata;
  private Collection<int[]> ignoreIssuesOnlineRanges;
// 可以被执行的代码行
  private BitSet executableLines;
```

根据这些可以知道

ProjectFileIndexer —>  FileIndexer —> InputComponentStore —> DefaultInputFile

在一个文件内，也是保存了很多信息，例如 DefaultInputFile 的成员变量。

有两个地方可以看下，一个是用 BitSet 存储 executableLines ，这里记录的是可以被执行的行的行号，这里在存储时没有使用 set 或 list， 优化了存储的空间。

`InputComponentStore` 在这个类里我们可以看到，put 的方式基本上就是把文件放在了这个类的成员变量中去。

### ModuleScanContainer

这个 ModuleScanContainer 同样继承自 ComponentContainer ，我们就把它当作是针对模块一级的组件扫描器即可，什么是Module一级，怎么区分的，这里应当指的是 Java 里的Module模块，例如一个Java项目中会分多个模块设计。

```java
public void execute() {
    Collection<ModuleSensorWrapper> moduleSensors = new ArrayList<>();
    withModuleStrategy(() -> moduleSensors.addAll(selector.selectSensors(false)));
    Collection<ModuleSensorWrapper> deprecatedGlobalSensors = new ArrayList<>();
    if (isRoot) {
      deprecatedGlobalSensors.addAll(selector.selectSensors(true));
    }

    printSensors(moduleSensors, deprecatedGlobalSensors);
    withModuleStrategy(() -> execute(moduleSensors));

    if (isRoot) {
      execute(deprecatedGlobalSensors);
    }
  }

private void execute(Collection<ModuleSensorWrapper> sensors) {
    for (ModuleSensorWrapper sensor : sensors) {
      String sensorName = getSensorName(sensor);
      profiler.startInfo("Sensor " + sensorName);
      sensor.analyse();
      profiler.stopInfo();
    }
  }
```

### ProjectSensorsExecutor

```java
public void execute() {
    List<ProjectSensorWrapper> sensors = selector.selectSensors();

    LOG.debug("Sensors : {}", sensors.stream()
      .map(Object::toString)
      .collect(Collectors.joining(" -> ")));
    for (ProjectSensorWrapper sensor : sensors) {
      String sensorName = getSensorName(sensor);
      profiler.startInfo("Sensor " + sensorName);
      sensor.analyse();
      profiler.stopInfo();
    }
  }

private String getSensorName(ProjectSensorWrapper sensor) {
    ClassLoader cl = getSensorClassLoader(sensor);
    String pluginKey = pluginRepo.getPluginKey(cl);
    if (pluginKey != null) {
      return sensor.toString() + " [" + pluginKey + "]";
    }
    return sensor.toString();
  }

  private static ClassLoader getSensorClassLoader(ProjectSensorWrapper sensor) {
    return sensor.wrappedSensor().getClass().getClassLoader();
  }

.....
public List<ProjectSensorWrapper> selectSensors() {
    Collection<ProjectSensor> result = sort(getFilteredExtensions(ProjectSensor.class, null));
    return result.stream()
      .map(s -> new ProjectSensorWrapper(s, sensorContext, sensorOptimizer))
      .filter(ProjectSensorWrapper::shouldExecute)
      .collect(Collectors.toList());
  }
```

从依赖中找到实现 ProjectSensor 接口的类，都有哪些？例如，单元测试收集，第三方Issue导入，重复代码解析，Metric 度量信息，

### ScmPublisher

这里就是从刚才 index过的文件中取出需要扫描的文件，然后查询 git blame 提交信息，汇总。

```java
public void publish() {
    if (configuration.isDisabled()) {
      LOG.info("SCM Publisher is disabled");
      return;
    }

    ScmProvider provider = configuration.provider();
    if (provider == null) {
      LOG.info("SCM Publisher No SCM system was detected. You can use the '" + CoreProperties.SCM_PROVIDER_KEY + "' property to explicitly specify it.");
      return;
    }

    List<InputFile> filesToBlame = collectFilesToBlame(writer);
    if (!filesToBlame.isEmpty()) {
      String key = provider.key();
      LOG.info("SCM Publisher SCM provider for this project is: " + key);
      DefaultBlameOutput output = new DefaultBlameOutput(writer, analysisWarnings, filesToBlame, client);
      try {
        provider.blameCommand().blame(new DefaultBlameInput(fs, filesToBlame), output);
      } catch (Exception e) {
        output.finish(false);
        throw e;
      }
      output.finish(true);
    }
  }

.......
// JGitBlameCommand
@Override
  public void blame(BlameInput input, BlameOutput output) {
    File basedir = input.fileSystem().baseDir();
    try (Repository repo = JGitUtils.buildRepository(basedir.toPath()); Git git = Git.wrap(repo)) {
      File gitBaseDir = repo.getWorkTree();

      if (cloneIsInvalid(gitBaseDir)) {
        return;
      }

      Stream<InputFile> stream = StreamSupport.stream(input.filesToBlame().spliterator(), true);
      ForkJoinPool forkJoinPool = new ForkJoinPool(Runtime.getRuntime().availableProcessors(), new GitThreadFactory(), null, false);
      forkJoinPool.submit(() -> stream.forEach(inputFile -> blame(output, git, gitBaseDir, inputFile)));
      try {
        forkJoinPool.shutdown();
        forkJoinPool.awaitTermination(Long.MAX_VALUE, TimeUnit.SECONDS);
      } catch (InterruptedException e) {
        LOG.info("Git blame interrupted");
      }
    }
  }

```

注意，这里会使用一个线程池去获取 blame 信息， `ForkJoinPool` ，设置的是电脑的所有核心数量。主要使用的是 jgit 这个库。

### ExtensionInstaller

```java
public ExtensionInstaller install(ComponentContainer container, ExtensionMatcher matcher) {

    // core components
    for (Object o : BatchComponents.all()) {
      doInstall(container, matcher, null, o);
    }

    // plugin extensions
    for (PluginInfo pluginInfo : pluginRepository.getPluginInfos()) {
      Plugin plugin = pluginRepository.getPluginInstance(pluginInfo.getKey());
      Plugin.Context context = new PluginContextImpl.Builder()
        .setSonarRuntime(sonarRuntime)
        .setBootConfiguration(bootConfiguration)
        .build();

      plugin.define(context);
      for (Object extension : context.getExtensions()) {
        doInstall(container, matcher, pluginInfo, extension);
      }
    }

    return this;
  }

.....
public static Collection<Object> all() {
    List<Object> components = new ArrayList<>();
    components.add(DefaultResourceTypes.get());
    components.addAll(CorePropertyDefinitions.all());
    components.add(ZeroCoverageSensor.class);
    components.add(JavaCpdBlockIndexerSensor.class);

    // Generic coverage
    components.add(GenericCoverageSensor.class);
    components.addAll(GenericCoverageSensor.properties());
    components.add(GenericTestExecutionSensor.class);
    components.addAll(GenericTestExecutionSensor.properties());
    components.add(TestPlanBuilder.class);

    // External issues
    components.add(ExternalIssuesImportSensor.class);
    components.add(ExternalIssuesImportSensor.properties());

    return components;
  }
```

### CpdExecutor

```java
// 默认超时时间，5分钟
private static final int TIMEOUT = 5 * 60 * 1000;
static final int MAX_CLONE_GROUP_PER_FILE = 100;
static final int MAX_CLONE_PART_PER_GROUP = 100;

public CpdExecutor(CpdSettings settings, SonarCpdBlockIndex index, ReportPublisher publisher, InputComponentStore inputComponentCache) {
    this(settings, index, publisher, inputComponentCache, Executors.newSingleThreadExecutor());
  }

public void execute() {
    execute(TIMEOUT);
  }

  void execute(long timeout) {
    List<FileBlocks> components = new ArrayList<>(index.noResources());
    Iterator<ResourceBlocks> it = index.iterator();

    while (it.hasNext()) {
      ResourceBlocks resourceBlocks = it.next();
      Optional<FileBlocks> fileBlocks = toFileBlocks(resourceBlocks.resourceId(), resourceBlocks.blocks());
      if (!fileBlocks.isPresent()) {
        continue;
      }
      components.add(fileBlocks.get());
    }

    int filesWithoutBlocks = index.noIndexedFiles() - index.noResources();
    if (filesWithoutBlocks > 0) {
      LOG.info("CPD Executor {} {} had no CPD blocks", filesWithoutBlocks, pluralize(filesWithoutBlocks));
    }

    total = components.size();
    progressReport.start(String.format("CPD Executor Calculating CPD for %d %s", total, pluralize(total)));
    try {
      for (FileBlocks fileBlocks : components) {
        runCpdAnalysis(executorService, fileBlocks.getInputFile(), fileBlocks.getBlocks(), timeout);
        count++;
      }
      progressReport.stopAndLogTotalTime("CPD Executor CPD calculation finished");
    } catch (Exception e) {
      progressReport.stop("");
      throw e;
    } finally {
      executorService.shutdown();
    }
  }

void runCpdAnalysis(ExecutorService executorService, DefaultInputFile inputFile, Collection<Block> fileBlocks, long timeout) {
    LOG.debug("Detection of duplications for {}", inputFile.absolutePath());
    progressReport.message(String.format("%d/%d - current file: %s", count, total, inputFile.absolutePath()));

    List<CloneGroup> duplications;
    Future<List<CloneGroup>> futureResult = executorService.submit(() -> SuffixTreeCloneDetectionAlgorithm.detect(index, fileBlocks));
    try {
      duplications = futureResult.get(timeout, TimeUnit.MILLISECONDS);
    } catch (TimeoutException e) {
      LOG.warn("Timeout during detection of duplications for {}", inputFile.absolutePath());
      futureResult.cancel(true);
      return;
    } catch (Exception e) {
      throw new IllegalStateException("Fail during detection of duplication for " + inputFile.absolutePath(), e);
    }

    List<CloneGroup> filtered;
    if (!"java".equalsIgnoreCase(inputFile.language())) {
      int minTokens = settings.getMinimumTokens(inputFile.language());
      Predicate<CloneGroup> minimumTokensPredicate = DuplicationPredicates.numberOfUnitsNotLessThan(minTokens);
      filtered = duplications.stream()
        .filter(minimumTokensPredicate)
        .collect(Collectors.toList());
    } else {
      filtered = duplications;
    }

    saveDuplications(inputFile, filtered);
  }

  final void saveDuplications(final DefaultInputComponent component, List<CloneGroup> duplications) {
    if (duplications.size() > MAX_CLONE_GROUP_PER_FILE) {
      LOG.warn("Too many duplication groups on file {}. Keep only the first {} groups.", component, MAX_CLONE_GROUP_PER_FILE);
    }
    Iterable<ScannerReport.Duplication> reportDuplications = duplications.stream()
      .limit(MAX_CLONE_GROUP_PER_FILE)
      .map(
        new Function<CloneGroup, Duplication>() {
          private final ScannerReport.Duplication.Builder dupBuilder = ScannerReport.Duplication.newBuilder();
          private final ScannerReport.Duplicate.Builder blockBuilder = ScannerReport.Duplicate.newBuilder();

          @Override
          public ScannerReport.Duplication apply(CloneGroup input) {
            return toReportDuplication(component, dupBuilder, blockBuilder, input);
          }

        })::iterator;
    publisher.getWriter().writeComponentDuplications(component.scannerId(), reportDuplications);
  }

  private Optional<FileBlocks> toFileBlocks(String componentKey, Collection<Block> fileBlocks) {
    DefaultInputFile component = (DefaultInputFile) componentStore.getByKey(componentKey);
    if (component == null) {
      LOG.error("Resource not found in component store: {}. Skipping CPD computation for it", componentKey);
      return Optional.empty();
    }
    return Optional.of(new FileBlocks(component, fileBlocks));
  }

  private Duplication toReportDuplication(InputComponent component, Duplication.Builder dupBuilder, Duplicate.Builder blockBuilder, CloneGroup input) {
    dupBuilder.clear();
    ClonePart originBlock = input.getOriginPart();
    blockBuilder.clear();
    dupBuilder.setOriginPosition(ScannerReport.TextRange.newBuilder()
      .setStartLine(originBlock.getStartLine())
      .setEndLine(originBlock.getEndLine())
      .build());
    int clonePartCount = 0;
    for (ClonePart duplicate : input.getCloneParts()) {
      if (!duplicate.equals(originBlock)) {
        clonePartCount++;
        if (clonePartCount > MAX_CLONE_PART_PER_GROUP) {
          LOG.warn("Too many duplication references on file " + component + " for block at line " +
            originBlock.getStartLine() + ". Keep only the first "
            + MAX_CLONE_PART_PER_GROUP + " references.");
          break;
        }
        blockBuilder.clear();
        String componentKey = duplicate.getResourceId();
        if (!component.key().equals(componentKey)) {
          DefaultInputComponent sameProjectComponent = (DefaultInputComponent) componentStore.getByKey(componentKey);
          blockBuilder.setOtherFileRef(sameProjectComponent.scannerId());
        }
        dupBuilder.addDuplicate(blockBuilder
          .setRange(ScannerReport.TextRange.newBuilder()
            .setStartLine(duplicate.getStartLine())
            .setEndLine(duplicate.getEndLine())
            .build())
          .build());
      }
    }
    return dupBuilder.build();
  }
```

`Future<List<CloneGroup>> futureResult = executorService.submit(() -> SuffixTreeCloneDetectionAlgorithm.detect(index, fileBlocks));` 

`Executors.newSingleThreadExecutor()` 

能看出这里是用了一个线程的线程池，这里能否加大提升扫描速度？不知道这个是否是线程安全的。

另外关于重复代码信息的真正实现，在另一个模块中，`sonar-duplication` ，我们下次再去看

### ReportPublisher

从 scanner-report 目录中读取生成的 scanner 报告，然后进行压缩和上传。

```java
@Override
  public void start() {
    reportDir = moduleHierarchy.root().getWorkDir().resolve("scanner-report");
    writer = new ScannerReportWriter(reportDir.toFile());
    reader = new ScannerReportReader(reportDir.toFile());
    contextPublisher.init(writer);

    if (!analysisMode.isMediumTest()) {
      String publicUrl = server.getPublicRootUrl();
      if (HttpUrl.parse(publicUrl) == null) {
        throw MessageException.of("Failed to parse public URL set in SonarQube server: " + publicUrl);
      }
    }
  }

public void execute() {
    File report = generateReportFile();
    if (properties.shouldKeepReport()) {
      LOG.info("Analysis report generated in " + reportDir);
    }
    if (!analysisMode.isMediumTest()) {
      String taskId = upload(report);
      prepareAndDumpMetadata(taskId);
    }

    logSuccess();
  }

....
private File generateReportFile() {
    try {
      long startTime = System.currentTimeMillis();
      for (ReportPublisherStep publisher : publishers) {
        publisher.publish(writer);
      }
      long stopTime = System.currentTimeMillis();
      LOG.info("Analysis report generated in {}ms, dir size={}", stopTime - startTime, humanReadableByteCountSI(FileUtils.sizeOfDirectory(reportDir.toFile())));

      startTime = System.currentTimeMillis();
      File reportZip = temp.newFile("scanner-report", ".zip");
      ZipUtils.zipDir(reportDir.toFile(), reportZip);
      stopTime = System.currentTimeMillis();
      LOG.info("Analysis report compressed in {}ms, zip size={}", stopTime - startTime, humanReadableByteCountSI(FileUtils.sizeOf(reportZip)));
      return reportZip;
    } catch (IOException e) {
      throw new IllegalStateException("Unable to prepare analysis report", e);
    }
  }
```

如果设置了 `sonar.scanner.keepReport` 或者 `sonar.verbose` 那么这个报告就可以保留，否则会删除报告信息。

![Untitled](Sonar-Scanner-Engine%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%20d54f744fd2f64bb5bb7189fcd43629f4/Untitled%201.png)

这里的 publish 有很多种，比如源码，激活的规则，变更行等，这些内容会写到 writer 这个变量里，这个变量是 `ScannerReportWriter` 这个类，这个类控制封装了对这些信息的写入操作。

我们可以看到，这些操作都是操作的 protobuf ，也就是说，最后SonarScanner 将所有需要上传到服务器的信息都封装为了 protobuf 的文件进行传输。

```java
// ScannerReportWriter
/**
   * Metadata is mandatory
   */
  public File writeMetadata(ScannerReport.Metadata metadata) {
    Protobuf.write(metadata, fileStructure.metadataFile());
    return fileStructure.metadataFile();
  }

  public File writeActiveRules(Iterable<ScannerReport.ActiveRule> activeRules) {
    Protobuf.writeStream(activeRules, fileStructure.activeRules(), false);
    return fileStructure.metadataFile();
  }

  public File writeComponent(ScannerReport.Component component) {
    File file = fileStructure.fileFor(FileStructure.Domain.COMPONENT, component.getRef());
    Protobuf.write(component, file);
    return file;
  }

  public File writeComponentIssues(int componentRef, Iterable<ScannerReport.Issue> issues) {
    File file = fileStructure.fileFor(FileStructure.Domain.ISSUES, componentRef);
    Protobuf.writeStream(issues, file, false);
    return file;
  }
```

上传报告

可以看出，最后是把 zip 压缩包通过 HTTP Protobuf 的方式进行了上传，然后得到一个 任务 id，这里使用到的框架是 `OkHttpClient` 

部分代码在 `sonar-scanner-protocol` 这个模块里

```java
/**
   * Uploads the report file to server and returns the generated task id
   */
  String upload(File report) {
    LOG.debug("Upload report");
    long startTime = System.currentTimeMillis();
    PostRequest.Part filePart = new PostRequest.Part(MediaTypes.ZIP, report);
    PostRequest post = new PostRequest("api/ce/submit")
      .setMediaType(MediaTypes.PROTOBUF)
      .setParam("projectKey", moduleHierarchy.root().key())
      .setParam("projectName", moduleHierarchy.root().getOriginalName())
      .setPart("report", filePart);

    String branchName = branchConfiguration.branchName();
    if (branchName != null) {
      if (branchConfiguration.branchType() != PULL_REQUEST) {
        post.setParam(CHARACTERISTIC, "branch=" + branchName);
        post.setParam(CHARACTERISTIC, "branchType=" + branchConfiguration.branchType().name());
      } else {
        post.setParam(CHARACTERISTIC, "pullRequest=" + branchConfiguration.pullRequestKey());
      }
    }

    WsResponse response;
    try {
      post.setWriteTimeOutInMs(DEFAULT_WRITE_TIMEOUT);
      response = wsClient.call(post).failIfNotSuccessful();
    } catch (HttpException e) {
      throw MessageException.of(String.format("Failed to upload report - %s", DefaultScannerWsClient.createErrorMessage(e)));
    }

    try (InputStream protobuf = response.contentStream()) {
      return Ce.SubmitResponse.parser().parseFrom(protobuf).getTaskId();
    } catch (Exception e) {
      throw new RuntimeException(e);
    } finally {
      long stopTime = System.currentTimeMillis();
      LOG.info("Analysis report uploaded in " + (stopTime - startTime) + "ms");
    }
  }

  void prepareAndDumpMetadata(String taskId) {
    Map<String, String> metadata = new LinkedHashMap<>();

    metadata.put("projectKey", moduleHierarchy.root().key());
    metadata.put("serverUrl", server.getPublicRootUrl());
    metadata.put("serverVersion", server.getVersion());
    properties.branch().ifPresent(branch -> metadata.put("branch", branch));

    URL dashboardUrl = buildDashboardUrl(server.getPublicRootUrl(), moduleHierarchy.root().key());
    metadata.put("dashboardUrl", dashboardUrl.toExternalForm());

    URL taskUrl = HttpUrl.parse(server.getPublicRootUrl()).newBuilder()
      .addPathSegment("api").addPathSegment("ce").addPathSegment("task")
      .addQueryParameter(ID, taskId)
      .build()
      .url();
    metadata.put("ceTaskId", taskId);
    metadata.put("ceTaskUrl", taskUrl.toExternalForm());

    ceTaskReportDataHolder.init(taskId, taskUrl.toExternalForm(), dashboardUrl.toExternalForm());
    dumpMetadata(metadata);
  }
```

### QualityGateCheck

如果不看源码，我绝对不知道 SonarQube 还有这个功能，我们可以原地等待 Server 端的执行结束。

`sonar.qualitygate.wait` 属性为 true 的时候就会执行这个类

设置这个属性可以应用于 流水线当中，在之前我们在执行 Sonar 的流水线是无法知晓扫描的结束的，只能通过回调hook的方式进行，因为扫描是异步的后台任务，通过这个方式，我们可以让流水线一直在前台等待，而且看了文档也确实是这样设计的，看了下版本，是 **SonarQube 8.1** 及之后的版本出现的功能。

```java
if (properties.shouldWaitForQualityGate()) {
  LOG.info("------------- Check Quality Gate status");
  getComponentByType(QualityGateCheck.class).await();
}

public void await() {
	if (!enabled) {
	  LOG.debug("Quality Gate check disabled - skipping");
	  return;
	}
	
	if (analysisMode.isMediumTest()) {
	  throw new IllegalStateException("Quality Gate check not available in medium test mode");
	}
	
	LOG.info("Waiting for the analysis report to be processed (max {}s)", properties.qualityGateWaitTimeout());
	String taskId = ceTaskReportDataHolder.getCeTaskId();
	
	Ce.Task task = waitForCeTaskToFinish(taskId);
	
	if (!TaskStatus.SUCCESS.equals(task.getStatus())) {
	  throw MessageException.of(String.format("CE Task finished abnormally with status: %s, you can check details here: %s",
	    task.getStatus().name(), ceTaskReportDataHolder.getCeTaskUrl()));
	}
	
	Status qualityGateStatus = getQualityGateStatus(task.getAnalysisId());
	
	if (Status.OK.equals(qualityGateStatus)) {
	  LOG.info("QUALITY GATE STATUS: PASSED - View details on " + ceTaskReportDataHolder.getDashboardUrl());
	} else {
	  throw MessageException.of("QUALITY GATE STATUS: FAILED - View details on " + ceTaskReportDataHolder.getDashboardUrl());
	}
	}

private Ce.Task waitForCeTaskToFinish(String taskId) {
  GetRequest getTaskResultReq = new GetRequest("api/ce/task")
    .setMediaType(MediaTypes.PROTOBUF)
    .setParam("id", taskId);

  long currentTime = 0;
  while (qualityGateTimeoutInMs > currentTime) {
    try {
      WsResponse getTaskResultResponse = wsClient.call(getTaskResultReq).failIfNotSuccessful();
      Ce.Task task = parseCeTaskResponse(getTaskResultResponse);
      if (TASK_TERMINAL_STATUSES.contains(task.getStatus())) {
        return task;
      }

      Thread.sleep(POLLING_INTERVAL_IN_MS);
      currentTime += POLLING_INTERVAL_IN_MS;
    } catch (HttpException e) {
      throw MessageException.of(String.format("Failed to get CE Task status - %s", DefaultScannerWsClient.createErrorMessage(e)));
    } catch (InterruptedException e) {
      Thread.currentThread().interrupt();
      throw new IllegalStateException("Quality Gate check has been interrupted", e);
    }
  }
  throw MessageException.of("Quality Gate check timeout exceeded - View details on " + ceTaskReportDataHolder.getDashboardUrl());
}
```



总结一下，`Sonar-Scanner-Engine` 这个模块基本上是扫描真正执行的代码。

- 里面用到了一个依赖注入的框架 `PicoContainer` ，
- 有一些常见的设计模式的使用，例如模版设计模式，适配器设计模式
- 通过源码，我们知道了 scanner 是通过 protobuf 和服务端进行通信的
- 最后报告压缩为 zip 格式上传到服务端
- 还发现了一个可以等待服务端扫描完毕的参数 `sonar.qualitygate.wait`
- `BitSet` ，`FileChannel#tryLock()` 的一些用法等



后续，我们可以再去看服务端`server` 模块，`sonar-core` 模块，`sonar-duplication` 模块，`sonar-plugin-api` 的设计和源码。