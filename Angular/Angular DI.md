# Angular依赖注入

## 一、依赖注入

Angular 的DI系统由以下几个关键部分构成：

*   **@Injectable**装饰器 ：分别用于标记可注入的服务
    
*   **Injector**：负责创建并管理依赖对象的实例，提供依赖对象给请求它们的组件或服务。
    
*   **Providers**：定义如何创建依赖对象。
    
*   **@Inject** 装饰器：指定依赖的注入令牌。
    

## 二、核心概念深入分析

 `**@Injectable**` **装饰器**

`@Injectable` 装饰器用来标记一个类，使其可以被 Angular 的依赖注入系统注入到其他类中。

在 Angular 源码中，`@Injectable` 的实现位于 `packages/core/src/di/injectable.ts` 文件中：

```typescript
export const Injectable: InjectableDecorator = makeDecorator(
    'Injectable', undefined, undefined, undefined,
    (type: Type<any>, meta: Injectable) => compileInjectable(type as any, meta)
);

```

`makeDecorator` 是 Angular 中用于创建装饰器的工厂函数。`makeDecorator` 函数的结构非常通用，允许创建多种类型的装饰器，如 `@Component`, `@Directive`, `@Injectable`，`@Pipe`，`@NgModule` 。位于`packages\core\src\util\decorators.ts`文件中，函数大体如下

```typescript
function makeDecorator<T>( ){
  
  return noSideEffects(() => {
     // ...
    function DecoratorFactory()
        // ...
      const annotationInstance = new (DecoratorFactory as any)(...args);
      return function TypeDecorator(cls: Type<T>) {
        // ...
        return cls;
      };
    }
    // ... 
    return DecoratorFactory as any;
  });
}
```

`noSideEffects` 就是一个 自执行函数IIFE。

```javascript
function noSideEffects<T>(fn: () => T): T {
  return {toString: fn}.toString() as unknown as T;
}

```

1.  返回的是装饰器工厂函数 `DecoratorFactory`
    
2.  `TypeDecorator` 是`DecoratorFactory`返回的函数，它实际应用装饰器逻辑到目标类
    
3.  在 Angular 的 JIT（即时编译）模式下，`DecoratorFactory` 在运行时被调用，而在 AOT（提前编译）模式下，编译器会在编译时处理装饰器，下面就是编译后的`childService`
    

![image.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/8K4nyRZgb3kYqLbj/img/c8812659-b1f3-4075-8f77-b7233b9220b8.png)

#### `ɵfac`是 Angular 用于创建类实例的工厂函数

`ɵprov`是 Angular 用于定义可注入服务元数据的属性

```typescript
export function ɵɵdefineInjectable<T>(opts: {
  token: unknown,
  providedIn?: Type<any>|'root'|'platform'|'any'|'environment'|null, factory: () => T,
}): unknown {
  return {
    token: opts.token,
    providedIn: opts.providedIn as any || null,
    factory: opts.factory,
    value: undefined,
  } as ɵɵInjectableDeclaration<T>;
}
```

`ɵɵdefineInjectable` 的主要作用是注册一个可注入的服务并为 Angular 依赖注入系统提供必要的信息

4.  最后得到真实的service格式如下：
    

![image.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/8K4nyRZgb3kYqLbj/img/010063e2-bcf3-4e6b-a446-1c0b3d16860f.png)

**Injector和Providers**

`Injector`负责实例化和管理依赖。Providers定义如何创建依赖对象

Angular 中有两个注入器层次结构：

*   `ModuleInjector`层次结构 —— 使用`@NgModule()`或`@Injectable()`装饰器在此层次结构中配置`ModuleInjector`。
    
*   `ElementInjector`层次结构 —— 在每个 DOM 元素上隐式创建。除非你在`@Directive()`或`@Component()`的`providers`属性中进行配置，否则默认情况下，`ElementInjector`为空
    

#### ModuleInjector

可以通过以下两种方式之一配置 `ModuleInjector`   ：

*   使用`@Injectable()`的`providedIn`属性引用`root`、`platform`或者`any`。
    
*   使用`@NgModule()`的`providers`数组
    

从源码进行分析：

编译后的Module代码中，

![image.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/8K4nyRZgb3kYqLbj/img/f5d5d07c-80d1-45eb-9d51-be5ab48b5f33.png)

`ɵinj` 是 Angular 用于定义module注入依赖的元数据属性

创建Module实例：（主要关于注入依赖相关逻辑）

通过调用模块工厂函数`NgModuleFactory`的 `create` 方法，实例化`NgModuleRef`，位于`packages\core\src\render3\ng_module_ref.ts`文件中

```typescript
export class NgModuleFactory<T> extends viewEngine_NgModuleFactory<T> {
  constructor(public moduleType: Type<T>) {
    super();
  }

  override create(parentInjector: Injector|null): viewEngine_NgModuleRef<T> {
    return new NgModuleRef(this.moduleType, parentInjector);
  }
}
```

NgModuleRef函数关于注入器代码主要如下：其中`**createInjectorWithoutInjectorInstances(...)**`: 创建 `R3Injector` 实例，该函数使用模块类型、父注入器以及提供者数组来初始化注入器。

```typescript
export class NgModuleRef<T> extends viewEngine_NgModuleRef<T> implements InternalNgModuleRef<T> {
  // tslint:disable-next-line:require-internal-with-underscore
  _bootstrapComponents: Type<any>[] = [];
  // tslint:disable-next-line:require-internal-with-underscore
  _r3Injector: R3Injector;
  // ... 
  constructor(ngModuleType: Type<T>, public _parent: Injector|null) {
    super();
    // ... 
    this._r3Injector = createInjectorWithoutInjectorInstances(
      ngModuleType, _parent,
      [
        {provide: viewEngine_NgModuleRef, useValue: this}, 
        {
          provide: viewEngine_ComponentFactoryResolver,
          useValue: this.componentFactoryResolver
        }
      ],
      stringify(ngModuleType), new Set(['environment'])) as R3Injector;

  }
```

`createInjectorWithoutInjectorInstances(...)`: 创建 `R3Injector` 实例，该函数使用模块类型、父注入器以及提供者数组来初始化注入器。`provide: viewEngine_NgModuleRef` 和 `provide: viewEngine_ComponentFactoryResolver`: 将当前模块实例和组件工厂解析器作为提供者注入。

```typescript
export function createInjectorWithoutInjectorInstances(
    defType: /* InjectorType<any> */ any, parent: Injector|null = null,
    additionalProviders: StaticProvider[]|null = null, name?: string,
    scopes = new Set<InjectorScope>()): R3Injector {
  const providers = [
    additionalProviders || EMPTY_ARRAY,
    importProvidersFrom(defType),
  ];
  name = name || (typeof defType === 'object' ? undefined : stringify(defType));

  return new R3Injector(providers, parent || getNullInjector(), name || null, scopes);
}
```

`importProvidersFrom`会调用`packages\core\src\di\provider_collection.ts`文件中`internalImportProvidersFrom`可以通过`NgModule.imports` 递归找到的所有依赖模块的`providers` 数组和本身的`NgModule.providers`打平存放到一起

`R3Injector` 类的主要属性和方法

`records`：用于存储注入器内的所有提供者（providers）及其对应的`Record` 对象。`Record`包含提供者的工厂函数以及实例的状态，在R3Injector构造函数中通过`processProvider`函数将收集到的providers，存储在records Map对象中

`scopes`：注入器的作用域集合RootInjector的`scopes = new Set(['root','environment'])`,PlatFormInjector的`scopes = new Set(["platform"])` 

```typescript
export class R3Injector extends EnvironmentInjector {
  private records = new Map<ProviderToken<any>, Record<any>|null>();
  constructor(
    providers: Array<Provider|ImportedNgModuleProviders>, readonly parent: Injector,
    readonly source: string|null, readonly scopes: Set<InjectorScope>
  ) {
    super();
    forEachSingleProvider(providers, provider => this.processProvider(provider));
    this.records.set(INJECTOR, makeRecord(undefined, this));

    // And `EnvironmentInjector` if the current injector is supposed to be env-scoped.
    if (scopes.has('environment')) {
      this.records.set(EnvironmentInjector, makeRecord(undefined, this));
    }
  }
```

`processProvider`：统一将multi-provider和single-provider进行处理并存储在records中，`providerToRecord`会将不同类型的Provider转换为可统一处理的工厂函数

```typescript
private processProvider(provider: SingleProvider): void {
    // Determine the token from the provider. Either it's its own token, or has a {provide: ...}
    // property.
    provider = resolveForwardRef(provider);
    let token: any =
        isTypeProvider(provider) ? provider : resolveForwardRef(provider && provider.provide);

    // Construct a `Record` for the provider.
    const record = providerToRecord(provider);

    if (!isTypeProvider(provider) && provider.multi === true) {
      // If the provider indicates that it's a multi-provider, process it specially.
      // First check whether it's been defined already.
      let multiRecord = this.records.get(token);
      if (multiRecord) {
        // It has. Throw a nice error if
        if (ngDevMode && multiRecord.multi === undefined) {
          throwMixedMultiProviderError();
        }
      } else {
        multiRecord = makeRecord(undefined, NOT_YET, true);
        multiRecord.factory = () => injectArgs(multiRecord!.multi!);
        this.records.set(token, multiRecord);
      }
      token = provider;
      multiRecord.multi!.push(provider);
    } else {
      const existing = this.records.get(token);
      if (ngDevMode && existing && existing.multi !== undefined) {
        throwMixedMultiProviderError();
      }
    }
    this.records.set(token, record);
  }
```

`get()`: 获取与令牌（`token`）关联的提供者实例

```typescript
override get<T>(
  token: ProviderToken<T>, notFoundValue: any = THROW_IF_NOT_FOUND,
  flags = InjectFlags.Default): T {
  const previousInjector = setCurrentInjector(this);
  const previousInjectImplementation = setInjectImplementation(undefined);
  try {
    if (!(flags & InjectFlags.SkipSelf)) {
      let record: Record<T>|undefined|null = this.records.get(token);
      if (record === undefined) {
        const def = couldBeInjectableType(token) && getInjectableDef(token);
        if (def && this.injectableDefInScope(def)) {
          record = makeRecord(injectableDefOrInjectorDefFactory(token), NOT_YET);
        } else {
          record = null;
        }
        this.records.set(token, record);
      }
      if (record != null /* NOT null || undefined */) {
        return this.hydrate(token, record);
      }
    }
    const nextInjector = !(flags & InjectFlags.Self) ? this.parent : getNullInjector();.
    notFoundValue = (flags & InjectFlags.Optional) && notFoundValue === THROW_IF_NOT_FOUND ?
      null :
      notFoundValue;
    return nextInjector.get(token, notFoundValue);
  } catch (e: any) {
    if (e.name === 'NullInjectorError') {
      const path: any[] = e[NG_TEMP_TOKEN_PATH] = e[NG_TEMP_TOKEN_PATH] || [];
      path.unshift(stringify(token));
      if (previousInjector) {
        // We still have a parent injector, keep throwing
        throw e;
      } else {
        // Format & throw the final error message when we don't have any previous injector
        return catchInjectorError(e, token, 'R3InjectorError', this.source);
      }
    } else {
      throw e;
    }
  } finally {
    // Lastly, restore the previous injection context.
    setInjectImplementation(previousInjectImplementation);
    setCurrentInjector(previousInjector);
  }
}
```

*   如果在当前注入器中找不到提供者，则会根据 `InjectFlags` 继续在父注入器中查找。
    
*   `flags` 可以控制注入的行为，比如 `SkipSelf`、`Self`、`Optional` 等。
    

`injectableDefInScope`: 检查一个可注入的定义（`provideIn`属性值）是否在当前注入器的范围内。

**provideIn: "any" | "root" | "platform"**

*   `root`表示在根模块注入器（root`ModuleInjector`  ）提供依赖
    
*   `platform`表示在平台注入器提供依赖
    
*   指定模块表示在特定的特性模块提供依赖（注意循环依赖）
    
*   `any`所有急性加载的模块都会共享同一个服务单例，惰性加载模块各自有它们自己独有的单例
    

```typescript
private injectableDefInScope(def: ɵɵInjectableDeclaration<any>): boolean {
    if (!def.providedIn) {
      return false;
    }
    const providedIn = resolveForwardRef(def.providedIn);
    if (typeof providedIn === 'string') {
      return providedIn === 'any' || (this.scopes.has(providedIn));
    } else {
      return this.injectorDefTypes.has(providedIn);
    }
  }
```

Angular 应用的启动过程

`platformBrowserDynamic().bootstrapModule(AppModule).then(ref => {...})`

*   `bootstrapModule()`方法会创建一个由`AppModule`配置的注入器作为平台注入器的子注入器，也就是 root`ModuleInjector`
    

*   `platformBrowserDynamic()`方法创建一个由`PlatformModule`配置的注入器，该注入器包含特定平台的依赖项，这允许多个应用共享同一套平台配置
    

`hydrate`: 实例化提供者，并缓存其结果。这个方法确保提供者只被实例化一次，避免循环依赖问题

```typescript
private hydrate<T>(token: ProviderToken<T>, record: Record<T>): T {
  if (ngDevMode && record.value === CIRCULAR) {
    throwCyclicDependencyError(stringify(token));
  } else if (record.value === NOT_YET) {
    record.value = CIRCULAR;
    record.value = record.factory!();
  }
  if (typeof record.value === 'object' && record.value && hasOnDestroy(record.value)) {
    this._ngOnDestroyHooks.add(record.value);
  }
  return record.value as T;
}
```

`destroy()`:

*   销毁注入器并释放其所有实例和提供者的引用。
    
*   调用所有已实例化服务的 `ngOnDestroy` 生命周期钩子。
    

![image](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/8K4nyRZgb3kYqLbj/img/8a53016e-6ec4-4c53-bc36-2a0030198e6e.png)

#### ElementInjector

`Angular`会为每个 DOM 元素上都隐式创建了一个`ElementInjector`。默认情况下，`ElementInjector` 是空的，除非在`@Directive()`或`@Component()`中的`providers` 属性中进行了配置

**ChildComponent组件**

![image.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/8K4nyRZgb3kYqLbj/img/30620a56-3f63-4090-8c8b-8c7fea413d40.png)

![image.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/8K4nyRZgb3kYqLbj/img/7c0fc4e3-d155-4b84-8a3f-f176c0c9a3c8.png)

组件中的providers编译后是通过ɵɵProvidersFeature处理，并赋值在features字段上

```typescript
export function ɵɵProvidersFeature<T>(providers: Provider[], viewProviders: Provider[] = []) {
  return (definition: DirectiveDef<T>) => {
    definition.providersResolver =
      (def: DirectiveDef<T>, processProvidersFn?: ProcessProvidersFunction) => {
        return providersResolver(
          def,                                                             //
          processProvidersFn ? processProvidersFn(providers) : providers,  //
          viewProviders);
      };
  };
}
```

编译后的组件会执行`ɵɵdefineComponent`，将组件的元数据转换成真实可用的。函数执行完成后会转成`providersResolver`字段挂载在组件的`ɵcmp`属性上

当渲染组件过程中，组件内部会调用`resolveDirectives`去处理指令提供者、生命周期钩子和模板属性等，调用`providersResolver`函数处理providers

```typescript
export function resolveDirectives(
    tView: TView, lView: LView, tNode: TElementNode|TContainerNode|TElementContainerNode,
    localRefs: string[]|null): boolean {
  // ... 

  let hasDirectives = false;
  if (getBindingsEnabled()) {
    // ...
    if (directiveDefs !== null) {
      hasDirectives = true;
      initTNodeFlags(tNode, tView.data.length, directiveDefs.length);
      for (let i = 0; i < directiveDefs.length; i++) {
        const def = directiveDefs[i];
        if (def.providersResolver) def.providersResolver(def);
      }
      // ...
    }
    // ...
  }
  return hasDirectives;
}
```

`providersResolver`函数，会确保仅在视图的第一次创建阶段进行提供者处理，再次调用`resolveProvider`处理providers（正常情况下，`providers` ​和 `viewProviders` ​没有任何区别，只要当在组件中使用插槽会不同的表现）

```typescript
export function providersResolver<T>(
    def: DirectiveDef<T>, providers: Provider[], viewProviders: Provider[]): void {
  const tView = getTView();
  if (tView.firstCreatePass) {
    const isComponent = isComponentDef(def);

    // The list of view providers is processed first, and the flags are updated
    resolveProvider(viewProviders, tView.data, tView.blueprint, isComponent, true);

    // Then, the list of providers is processed, and the flags are updated
    resolveProvider(providers, tView.data, tView.blueprint, isComponent, false);
  }
}
```

`resolveProvider`这个函数负责解析不同类型的提供者，包括单一提供者和多重提供者，并根据它们的作用范围（视图提供者或普通提供者）将它们注册到视图或指令中

```typescript
function resolveProvider(
  provider: Provider, tInjectables: TData, lInjectablesBlueprint: NodeInjectorFactory[],
  isComponent: boolean, isViewProvider: boolean): void {
  provider = resolveForwardRef(provider);
  if (Array.isArray(provider)) {
    // Recursively call `resolveProvider`
    // Recursion is OK in this case because this code will not be in hot-path once we implement
    // cloning of the initial state.
    for (let i = 0; i < provider.length; i++) {
      resolveProvider(
        provider[i], tInjectables, lInjectablesBlueprint, isComponent, isViewProvider);
    }
  } else {
    const tView = getTView();
    const lView = getLView();
    let token: any = isTypeProvider(provider) ? provider : resolveForwardRef(provider.provide);
    let providerFactory: () => any = providerToFactory(provider);

    const tNode = getCurrentTNode()!;
    // ... 
    if (isTypeProvider(provider) || !provider.multi) {
      // Single provider case: the factory is created and pushed immediately
      const factory = new NodeInjectorFactory(providerFactory, isViewProvider, ɵɵdirectiveInject);
      const existingFactoryIndex = indexOf(
        token, tInjectables, isViewProvider ? beginIndex : beginIndex + cptViewProvidersCount,
        endIndex);
      if (existingFactoryIndex === -1) {
        diPublicInInjector(
          getOrCreateNodeInjectorForNode( tNode as TElementNode | TContainerNode | TElementContainerNode, lView),
          tView, 
          token
        );
        registerDestroyHooksIfSupported(tView, provider, tInjectables.length);
        tInjectables.push(token);
        tNode.directiveStart++;
        tNode.directiveEnd++;
        if (isViewProvider) {
          tNode.providerIndexes += TNodeProviderIndexes.CptViewProvidersCountShifter;
        }
        lInjectablesBlueprint.push(factory);
        lView.push(factory);
      } else {
        lInjectablesBlueprint[existingFactoryIndex] = factory;
        lView[existingFactoryIndex] = factory;
      }
    } else {
      // ...
      if (isViewProvider && !doesViewProvidersFactoryExist ||
          !isViewProvider && !doesProvidersFactoryExist) {
        diPublicInInjector(
          getOrCreateNodeInjectorForNode(tNode as TElementNode | TContainerNode | TElementContainerNode, lView),
          tView,
          token
        );
        const factory = multiFactory(
          isViewProvider ? multiViewProvidersFactoryResolver : multiProvidersFactoryResolver,
          lInjectablesBlueprint.length, isViewProvider, isComponent, providerFactory);
        if (!isViewProvider && doesViewProvidersFactoryExist) {
          lInjectablesBlueprint[existingViewProvidersFactoryIndex].providerFactory = factory;
        }
        registerDestroyHooksIfSupported(tView, provider, tInjectables.length, 0);
        tInjectables.push(token);
        tNode.directiveStart++;
        tNode.directiveEnd++;
        if (isViewProvider) {
          tNode.providerIndexes += TNodeProviderIndexes.CptViewProvidersCountShifter;
        }
        lInjectablesBlueprint.push(factory);
        lView.push(factory);
      } else {
       // ... 
      }
      if (!isViewProvider && isComponent && doesViewProvidersFactoryExist) {
        lInjectablesBlueprint[existingViewProvidersFactoryIndex].componentProviders!++;
      }
    }
  }
}
```

*   处理单一提供者：如果提供者是单一的（非多提供者），函数会立即创建一个工厂并注册它。
    

*   处理多提供者：对于多提供者，函数会创建或更新一个多工厂（`multiFactory`），并确保不同类型的提供者（视图提供者和普通提供者）按正确的优先级顺序进行处理。
    
*   `getOrCreateNodeInjectorForNode`获取节点注入器在该节点位置
    
*   `diPublicInInjector(getOrCreateNodeInjectorForNode(),tView,token)`：内部通过`bloomAdd`函数将token添加到Bloom过滤器中，并token中添加字段`NG_ELEMENT_ID`（`NG_ELEMENT_ID`是一个唯一且递增的值，`NG_ELEMENT_ID`值获取到token）;
    
*   `lView.push(factory)`:`lView` 是一个数组，代表了当前视图的局部状态，包括视图中的所有节点和指令的实例。`lView`中的工厂函数用于创建和缓存依赖注入系统中需要的所有提供者实例。
    
*   `providerToFactory` 函数确保了不同类型的提供者都能被正确地转换为工厂函数
    

```typescript
export function providerToFactory(
    provider: SingleProvider, ngModuleType?: InjectorType<any>, providers?: any[]): () => any {
  let factory: (() => any)|undefined = undefined;
  if (ngDevMode && isImportedNgModuleProviders(provider)) {
    throwInvalidProviderError(undefined, providers, provider);
  }

  if (isTypeProvider(provider)) {
    const unwrappedProvider = resolveForwardRef(provider);
    return getFactoryDef(unwrappedProvider) || injectableDefOrInjectorDefFactory(unwrappedProvider);
  } else {
    if (isValueProvider(provider)) {
      factory = () => resolveForwardRef(provider.useValue);
    } else if (isFactoryProvider(provider)) {
      factory = () => provider.useFactory(...injectArgs(provider.deps || []));
    } else if (isExistingProvider(provider)) {
      factory = () => ɵɵinject(resolveForwardRef(provider.useExisting));
    } else {
      const classRef = resolveForwardRef(
          provider &&
          ((provider as StaticClassProvider | ClassProvider).useClass || provider.provide));
      if (ngDevMode && !classRef) {
        throwInvalidProviderError(ngModuleType, providers, provider);
      }
      if (hasDeps(provider)) {
        factory = () => new (classRef)(...injectArgs(provider.deps));
      } else {
        return getFactoryDef(classRef) || injectableDefOrInjectorDefFactory(classRef);
      }
    }
  }
  return factory;
}
```

**@Inject 装饰器**

`@Inject` 装饰器用于显式指定一个构造函数参数的依赖项，`packages\core\src\di\metadata.ts`

```typescript
const Inject: InjectDecorator = attachInjectFlag(
    // Disable tslint because `DecoratorFlags` is a const enum which gets inlined.
    // tslint:disable-next-line: no-toplevel-property-access
    makeParamDecorator('Inject', (token: any) => ({token})), 
  DecoratorFlags.Inject);
```

`makeParamDecorator` 是 Angular 用于创建参数装饰器的一个内部工具函数

`attachInjectFlag` 用于将`Inject`特定的装饰器标志（flag）附加到装饰器上

在代码中的使用

![image.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/8K4nyRZgb3kYqLbj/img/96271f76-9479-486c-a56d-8300e740120b.png)

编译后的代码：

![image.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/8K4nyRZgb3kYqLbj/img/fe09204c-e4d8-40bf-8b9b-86c54dc4b605.png)

`ɵɵdirectiveInject`位于`packages\core\src\render3\instructions\di.ts`文件中，函数主要用于在指令和组件的上下文中进行依赖注入

```typescript
export function ɵɵdirectiveInject<T>(token: ProviderToken<T>, flags = InjectFlags.Default): T|null {
  const lView = getLView();
  // Fall back to inject() if view hasn't been created. This situation can happen in tests
  // if inject utilities are used before bootstrapping.
  if (lView === null) {
    // Verify that we will not get into infinite loop.
    ngDevMode && assertInjectImplementationNotEqual(ɵɵdirectiveInject);
    return ɵɵinject(token, flags);
  }
  const tNode = getCurrentTNode();
  return getOrCreateInjectable<T>(
      tNode as TDirectiveHostNode, lView, resolveForwardRef(token), flags);
}
```

*   `tNode`: 当前节点（`TDirectiveHostNode`）或 `null`。用于在节点的注入器中查找服务。
    

*   `lView`: 当前视图的上下文（`LView`）。包含当前视图的信息以及注入器的引用。
    
*   `token`: 提供者的令牌（`ProviderToken<T>`）。用于查找或创建服务的标识。
    

*   `flags`: 注入标志（`InjectFlags`），默认值为 `InjectFlags.Default`。用于控制注入行为的标志。`(Optional`、`Self`、`SkipSelf` 、`Host`)
    
*   `getOrCreateInjectable()`: 用于从当前视图的注入器（`injector`）中获取或创建指定 `token` 对应的依赖。
    

```typescript
export function getOrCreateInjectable<T>(
    tNode: TDirectiveHostNode|null, lView: LView, token: ProviderToken<T>,
    flags: InjectFlags = InjectFlags.Default, notFoundValue?: any): T|null {
  if (tNode !== null) {
    // If the view or any of its ancestors have an embedded
    // view injector, we have to look it up there first.
    if (lView[FLAGS] & LViewFlags.HasEmbeddedViewInjector) {
      const embeddedInjectorValue =
          lookupTokenUsingEmbeddedInjector(tNode, lView, token, flags, NOT_FOUND);
      if (embeddedInjectorValue !== NOT_FOUND) {
        return embeddedInjectorValue;
      }
    }

    // Otherwise try the node injector.
    const value = lookupTokenUsingNodeInjector(tNode, lView, token, flags, NOT_FOUND);
    if (value !== NOT_FOUND) {
      return value;
    }
  }

  // Finally, fall back to the module injector.
  return lookupTokenUsingModuleInjector<T>(lView, token, flags, notFoundValue);
}
```

*   `lookupTokenUsingEmbeddedInjector`： 如果有嵌入视图注入器，则使用该函数在嵌入视图注入器中查找指定的`token`。如果找到了依赖项，返回其值。
    
*   `lookupTokenUsingNodeInjector`： 如果没有找到嵌入视图注入器中的依赖项，则在当前节点的注入器中查找。节点注入器是与`tNode`关联的注入器，通常用于指令和组件的注入。如果在节点注入器中找到了依赖项，返回其值。
    
*   `lookupTokenUsingModuleInjector`：若在节点注入器找到返回一个特殊的值`NOT_FOUND`。会进入模块注入器，如果在模块注入器中找到了依赖项，返回其值
    

`**lookupTokenUsingNodeInjector**`**在节点的注入器中查找**

*   `bloomHashBitOrFactory(token)`：主要是取token的NG\_ELEMENT\_ID（provider时创建）
    
*   `bloomHash` 是一个函数表示这是一个特殊的对象（例如 ElementRef 或 TemplateRef），调用 `bloomHash(flags)` 来尝试获取依赖项的实例
    
*   `enterDI` 函数的主要任务是确保传入的`lView`和`tNode`是有效的
    
*   `bloomHash` 为数字时，表示常规的注入流程
    
*   `bloomHasToken(bloomHash, injectorIndex, tView.data)`bloomHash与injectorIndex和tView.data在bloom过滤池中有关联关系,，若返回true代表token由当前视图提供。
    
*   使用 `searchTokensOnInjector()` 函数查找依赖项的实例，若不存在沿着父注入器向上寻找
    

```typescript
function lookupTokenUsingNodeInjector<T>(
    tNode: TDirectiveHostNode, lView: LView, token: ProviderToken<T>, flags: InjectFlags,
    notFoundValue?: any) {
  const bloomHash = bloomHashBitOrFactory(token);
  if (typeof bloomHash === 'function') {
    if (!enterDI(lView, tNode, flags)) {
      // Failed to enter DI, try module injector instead. If a token is injected with the @Host
      // flag, the module injector is not searched for that token in Ivy.
      return (flags & InjectFlags.Host) ?
          notFoundValueOrThrow<T>(notFoundValue, token, flags) :
          lookupTokenUsingModuleInjector<T>(lView, token, flags, notFoundValue);
    }
    try {
      const value = bloomHash(flags);
      if (value == null && !(flags & InjectFlags.Optional)) {
        throwProviderNotFoundError(token);
      } else {
        return value;
      }
    } finally {
      leaveDI();
    }
    // 常规的注入
  } else if (typeof bloomHash === 'number') {
    let previousTView: TView|null = null;
    let injectorIndex = getInjectorIndex(tNode, lView);
    let parentLocation: RelativeInjectorLocation = NO_PARENT_INJECTOR;
    let hostTElementNode: TNode|null =
        flags & InjectFlags.Host ? lView[DECLARATION_COMPONENT_VIEW][T_HOST] : null;

    // If we should skip this injector, or if there is no injector on this node, start by
    // searching the parent injector.
    if (injectorIndex === -1 || flags & InjectFlags.SkipSelf) {
      parentLocation = injectorIndex === -1 ? getParentInjectorLocation(tNode, lView) :
                                              lView[injectorIndex + NodeInjectorOffset.PARENT];

      if (parentLocation === NO_PARENT_INJECTOR || !shouldSearchParent(flags, false)) {
        injectorIndex = -1;
      } else {
        previousTView = lView[TVIEW];
        injectorIndex = getParentInjectorIndex(parentLocation);
        lView = getParentInjectorView(parentLocation, lView);
      }
    }

    // Traverse up the injector tree until we find a potential match or until we know there
    // *isn't* a match.
    while (injectorIndex !== -1) {
      const tView = lView[TVIEW];
      if (bloomHasToken(bloomHash, injectorIndex, tView.data)) {
        const instance: T|{}|null = searchTokensOnInjector<T>(
            injectorIndex, lView, token, previousTView, flags, hostTElementNode);
        if (instance !== NOT_FOUND) {
          return instance;
        }
      }
      parentLocation = lView[injectorIndex + NodeInjectorOffset.PARENT];
      if (parentLocation !== NO_PARENT_INJECTOR &&
          shouldSearchParent(
              flags,
              lView[TVIEW].data[injectorIndex + NodeInjectorOffset.TNODE] === hostTElementNode) &&
          bloomHasToken(bloomHash, injectorIndex, lView)) {
        // The def wasn't found anywhere on this node, so it was a false positive.
        // Traverse up the tree and continue searching.
        previousTView = tView;
        injectorIndex = getParentInjectorIndex(parentLocation);
        lView = getParentInjectorView(parentLocation, lView);
      } else {
        // If we should not search parent OR If the ancestor bloom filter value does not have the
        // bit corresponding to the directive we can give up on traversing up to find the specific
        // injector.
        injectorIndex = -1;
      }
    }
  }

  return notFoundValue;
}
```

searchTokensOnInjector遍历注入器树，从指定的节点和视图中查找与给定 `Token` 匹配的提供者。

```typescript
function searchTokensOnInjector<T>(
    injectorIndex: number, lView: LView, token: ProviderToken<T>, previousTView: TView|null,
    flags: InjectFlags, hostTElementNode: TNode|null) {
  const currentTView = lView[TVIEW];
  const tNode = currentTView.data[injectorIndex + NodeInjectorOffset.TNODE] as TNode;
  const canAccessViewProviders = previousTView == null ?
      (isComponentHost(tNode) && includeViewProviders) :
      (previousTView != currentTView && ((tNode.type & TNodeType.AnyRNode) !== 0));

  const isHostSpecialCase = (flags & InjectFlags.Host) && hostTElementNode === tNode;

  const injectableIdx = locateDirectiveOrProvider(
      tNode, currentTView, token, canAccessViewProviders, isHostSpecialCase);
  if (injectableIdx !== null) {
    return getNodeInjectable(lView, currentTView, injectableIdx, tNode as TElementNode);
  } else {
    return NOT_FOUND;
  }
}
```
```typescript
export function getNodeInjectable(
    lView: LView, tView: TView, index: number, tNode: TDirectiveHostNode): any {
  let value = lView[index];
  const tData = tView.data;
  if (isFactory(value)) {
    const factory: NodeInjectorFactory = value;
    if (factory.resolving) {
      throwCyclicDependencyError(stringifyForError(tData[index]));
    }
    const previousIncludeViewProviders = setIncludeViewProviders(factory.canSeeViewProviders);
    factory.resolving = true;
    const previousInjectImplementation =
        factory.injectImpl ? setInjectImplementation(factory.injectImpl) : null;
    const success = enterDI(lView, tNode, InjectFlags.Default);
    ngDevMode &&
        assertEqual(
            success, true,
            'Because flags do not contain \`SkipSelf\' we expect this to always succeed.');
    try {
      value = lView[index] = factory.factory(undefined, tData, lView, tNode);
      if (tView.firstCreatePass && index >= tNode.directiveStart) {
        ngDevMode && assertDirectiveDef(tData[index]);
        registerPreOrderHooks(index, tData[index] as DirectiveDef<any>, tView);
      }
    } finally {
      previousInjectImplementation !== null &&
          setInjectImplementation(previousInjectImplementation);
      setIncludeViewProviders(previousIncludeViewProviders);
      factory.resolving = false;
      leaveDI();
    }
  }
  return value;
}
```

`**lookupTokenUsingModuleInjector**`**在模块注入器查找**

```typescript
function lookupTokenUsingModuleInjector<T>(
    lView: LView, token: ProviderToken<T>, flags: InjectFlags, notFoundValue?: any): T|null {
  if ((flags & InjectFlags.Optional) && notFoundValue === undefined) {
    // This must be set or the NullInjector will throw for optional deps
    notFoundValue = null;
  }

  if ((flags & (InjectFlags.Self | InjectFlags.Host)) === 0) {
    const moduleInjector = lView[INJECTOR];
    // switch to `injectInjectorOnly` implementation for module injector, since module injector
    // should not have access to Component/Directive DI scope (that may happen through
    // `directiveInject` implementation)
    const previousInjectImplementation = setInjectImplementation(undefined);
    try {
      if (moduleInjector) {
        return moduleInjector.get(token, notFoundValue, flags & InjectFlags.Optional);
      } else {
        return injectRootLimpMode(token, notFoundValue, flags & InjectFlags.Optional);
      }
    } finally {
      setInjectImplementation(previousInjectImplementation);
    }
  }
  return notFoundValueOrThrow<T>(notFoundValue, token, flags);
}
```

`R3_Injector.get()`: 获取与令牌（`token`）关联的提供者实例，用于在当前注入器、其父级注入器或全局注入器中查找依赖项

*   如果在当前注入器中找不到提供者，则会根据 `InjectFlags` 继续在父注入器中查找。
    
*   injectableDefInScope 会处理 providedIn：root | platform | any
    

```typescript
override get<T>(
  token: ProviderToken<T>, notFoundValue: any = THROW_IF_NOT_FOUND,
  flags = InjectFlags.Default): T {
  this.assertNotDestroyed();
  // Set the injection context.
  const previousInjector = setCurrentInjector(this);
  const previousInjectImplementation = setInjectImplementation(undefined);
  try {
    // Check for the SkipSelf flag.
    if (!(flags & InjectFlags.SkipSelf)) {
      // SkipSelf isn't set, check if the record belongs to this injector.
      let record: Record<T>|undefined|null = this.records.get(token);
      if (record === undefined) {
        // No record, but maybe the token is scoped to this injector. Look for an injectable
        // def with a scope matching this injector.
        const def = couldBeInjectableType(token) && getInjectableDef(token);
        if (def && this.injectableDefInScope(def)) {
          record = makeRecord(injectableDefOrInjectorDefFactory(token), NOT_YET);
        } else {
          record = null;
        }
        this.records.set(token, record);
      }
      // If a record was found, get the instance for it and return it.
      if (record != null /* NOT null || undefined */) {
        return this.hydrate(token, record);
      }
    }

    // Select the next injector based on the Self flag - if self is set, the next injector is
    // the NullInjector, otherwise it's the parent.
    const nextInjector = !(flags & InjectFlags.Self) ? this.parent : getNullInjector();
    // Set the notFoundValue based on the Optional flag - if optional is set and notFoundValue
    // is undefined, the value is null, otherwise it's the notFoundValue.
    notFoundValue = (flags & InjectFlags.Optional) && notFoundValue === THROW_IF_NOT_FOUND ?
      null :
      notFoundValue;
    return nextInjector.get(token, notFoundValue);
  } catch (e: any) {
    if (e.name === 'NullInjectorError') {
      const path: any[] = e[NG_TEMP_TOKEN_PATH] = e[NG_TEMP_TOKEN_PATH] || [];
      path.unshift(stringify(token));
      if (previousInjector) {
        // We still have a parent injector, keep throwing
        throw e;
      } else {
        // Format & throw the final error message when we don't have any previous injector
        return catchInjectorError(e, token, 'R3InjectorError', this.source);
      }
    } else {
      throw e;
    }
  } finally {
    // Lastly, restore the previous injection context.
    setInjectImplementation(previousInjectImplementation);
    setCurrentInjector(previousInjector);
  }
}
```

#### 整个的解析规则

![image](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/8K4nyRZgb3kYqLbj/img/f34dcdeb-1bfc-40c9-875c-1dee55d7833c.png)

**解析修饰符**

Angular 中可以使用 `@Optional()`，`@Self()` ， `@SkipSelf()`和`@Host()`来修饰 Angular 的解析行为。默认情况下，`Angular`始终从当前的`Injector`开始，并一直向上搜索，修饰符可以更改开始（默认是自己）或结束位置

*   **@Optional()**：`@Optional()` 允许 Angular 将你注入的服务视为可选服务。这样，如果无法在运行时解析它，Angular 只会将服务解析为 null，而不会抛出错误。
    
*   **@Self()：** 使用`@Self()`让 Angular 仅查看当前组件或指令的`ElementInjector`
    
*   **@SkipSelf()：**`@SkipSelf()`与`@Self()`相反，使用`@SkipSelf()`，Angular在父`ElementInjector`中开始搜索服务，而不是从当前`ElementInjector`中开始搜索服务
    
*   **@Host()**：`@Host()`属性装饰器会禁止在宿主组件以上的搜索，宿主组件通常就是请求该依赖的那个组件
