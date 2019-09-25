---
title: ASP.NET Boilerplate v0.2.0.0 分析
date: 2017-08-14 00:26:14
tags:
    - .NET
---

`AbpWebApplication`派生`HttpApplication`，成为应用入口。该类包含唯一属性：
``` c#
/// <summary>
/// Gets a reference to the <see cref="AbpBootstrapper"/> instance.
/// </summary>
private AbpBootstrapper AbpBootstrapper { get; set; }
```

### 启动

在`AbpWebApplication中`重载`Application_Start`函数：
```c#
/// <summary>
/// This method is called by ASP.NET system on web application's startup.
/// </summary>
protected virtual void Application_Start(object sender, EventArgs e)
{
    AbpBootstrapper = new AbpBootstrapper();
    AbpBootstrapper.Initialize();
}
```

### 关闭

在`AbpWebApplication中`重载`Application_End`函数：
```c#
/// <summary>
/// This method is called by ASP.NET system on web application's end.
/// </summary>
protected virtual void Application_End(object sender, EventArgs e)
{
    AbpBootstrapper.Dispose();
}
```

### AbpBootstrapper：初始化框架
在`AbpBootstrapper`中包含唯一私有字段 `AppApplicationManager`，调用`Initialize`进行配置：
```c#
/// <summary>
/// Initializes the system.
/// </summary>
public virtual void Initialize()
{
    IocManager.Instance.IocContainer.Install(FromAssembly.This());
    _applicationManager = IocHelper.Resolve<AbpApplicationManager>();
    _applicationManager.Initialize();
}
```

### AbpApplicationManager：记录框架和应用中所有的模块以及模块管理器，生命周期存同应用一样
```c#
/// <summary>
/// Initializes the application.
/// </summary>
public virtual void Initialize()
{
    var initializationContext = new AbpInitializationContext(_modules);
    _moduleManager.Initialize(initializationContext);
}
```

### AbpModuleManager：加载和卸载模块，包括调用每个模块的`PreInitialize`、`Initialize`、`PostInitialize`函数执行相应的初始化工作
```c#
public virtual void Initialize(IAbpInitializationContext initializationContext)
{
    _moduleLoader.LoadAll();

    var sortedModules = _modules.GetSortedModuleListByDependency();

    IocManager.Instance
        .AddConventionalRegisterer(new BasicConventionalRegisterer());

    sortedModules
        .ForEach(module => module.Instance.PreInitialize(initializationContext));

    IocManager.Instance.RegisterAssemblyByConvention(
        Assembly.GetExecutingAssembly(),
        new ConventionalRegistrationConfig()
            { InstallInstallers = false });
    sortedModules.ForEach(module => module.Instance.Initialize(initializationContext));

    sortedModules.ForEach(module => module.Instance.PostInitialize(initializationContext));
}
```

### 模块加载原理及顺序
1. 获取当前应用程序程序集及其以来程序集
2. 获取程序集中所有的`ApbModule`模块（是否实现`IAbpModule`接口），并放入模块集合中
3. 找出模块的依赖模块，重复第2步，直到所有模块处理完毕
4. 遍历模块集合设置每个模块的依赖模块信息
5. 执行每个模块的`PreInitialize`、再执行`Initialize`、最后执行`PostInitialize`
6. 加载完毕，执行应用逻辑。
