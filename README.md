# qiankun（乾坤）源码阅读

### 主应用的启动 start 和子应用的注册 registerMicroApps



首先，入口 index.ts 中引入和导出了**启动主应用**和**注册子应用**的方法

```
export { loadMicroApp, registerMicroApps, start } from './apis';
```



然后是 apis.ts 中的注册子应用函数 registerMicroApps

```
import { loadApp } from './loader';
import { mountRootParcel, registerApplication, start as startSingleSpa } from 'single-spa';

let microApps: RegistrableApp[] = [];

export let frameworkConfiguration: FrameworkConfiguration = {};
const frameworkStartedDefer = new Deferred<void>();

// 注册子应用
export function registerMicroApps<T extends object = {}>(
  apps: Array<RegistrableApp<T>>,
  lifeCycles?: FrameworkLifeCycles<T>,
) {
  // Each app only needs to be registered once
  // 对需要注册的子应用进行一遍过滤，筛选出没有注册过的子应用，防止重复注册
  const unregisteredApps = apps.filter(app => !microApps.some(registeredApp => registeredApp.name === app.name));

  microApps = [...microApps, ...unregisteredApps];

  // 调用 singe-spa 的 registerApplication 对没有注册过的模块逐个注册
  unregisteredApps.forEach(app => {
    const { name, activeRule, loader = noop, props, ...appConfig } = app;

    /**
     * singe-spa 注册子应用的函数
     * 
     * name: 子应用名称
     * app：回调函数，activeRule 激活的时候调用
     * activeWhen：子应用激活规则
     * customProps：主应用传给子应用的数据
     * 
     * 这些参数由 singe-spa 直接实现。在符合 activeRule 激活规则时将会激活子应用，执行 app 回调函数，返回一些生命周期
     */
    registerApplication({
      name,
      app: async () => {
        loader(true);
        await frameworkStartedDefer.promise;

        const { mount, ...otherMicroAppConfigs } = await loadApp(
          { name, props, ...appConfig },
          frameworkConfiguration,
          lifeCycles,
        );
        
        // 返回一些生命周期钩子函数
        return {
          mount: [async () => loader(true), ...toArray(mount), async () => loader(false)],
          ...otherMicroAppConfigs,
        };
      },
      activeWhen: activeRule,
      customProps: props,
    });
  });
}
```



然后是 apis.ts 中的启动主应用函数 start

```
import { mountRootParcel, registerApplication, start as startSingleSpa } from 'single-spa';

export let frameworkConfiguration: FrameworkConfiguration = {};
const frameworkStartedDefer = new Deferred<void>();

// 启动主应用
export function start(opts: FrameworkConfiguration = {}) {
  // 配置一些默认参数与合并 opts，并且赋值给全局变量 frameworkConfiguration，方便在注册子应用的时候用到
  frameworkConfiguration = { prefetch: true, singular: true, sandbox: true, ...opts };
  const { prefetch, sandbox, singular, urlRerouteOnly, ...importEntryOpts } = frameworkConfiguration;

  // 如果需要预加载
  if (prefetch) {
    doPrefetchStrategy(microApps, prefetch, importEntryOpts);
  }

  // sandbox：是否开启沙箱环境
  if (sandbox) {
    if (!window.Proxy) {
      console.warn('[qiankun] Miss window.Proxy, proxySandbox will degenerate into snapshotSandbox');
      // 快照沙箱不支持非 singular 模式
      if (!singular) {
        console.error('[qiankun] singular is forced to be true when sandbox enable but proxySandbox unavailable');
        frameworkConfiguration.singular = true;
      }
    }
  }

  // 主要调用了 single-spa 的 startSingleSpa 启动主应用
  startSingleSpa({ urlRerouteOnly });

  frameworkStartedDefer.resolve();
}
```

**总结：从注册子应用--->启动主应用--->是否符合条件激活子应用**

![qiankun启动主应用和注册子应用函数](/imgs/qiankun启动主应用和注册子应用.jpg)