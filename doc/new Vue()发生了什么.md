---
sidebar: auto
---

# New Vue

## Vue构造函数
```js
/**
 * Vue构造函数
 *
 * @param {*} options 选项参数
 */
function Vue(options) {
  if (process.env.NODE_ENV !== 'production' && !(this instanceof Vue)) {
    warn('Vue是一个构造函数，应该用“new”关键字调用');
  }
  this._init(options);
}
```
> 我们知道 **`new Vue()`** 将执行 Vue 构造函数, 进而执行 **`_init(),`** 那 _init 方法从何处而来？答案是Vue在初始化时添加了该方法,如果你对初始化还不是很清楚，建议你参考上文对初始化过程的梳理性文章：[「试着读读 Vue 源代码」初始化前后做了哪些事情❓](https://juejin.cn/post/6844903861468004359)。

## _init()
```js
import config from '../config';
import { initProxy } from './proxy';
import { initState } from './state';
import { initRender } from './render';
import { initEvents } from './events';
import { mark, measure } from '../util/perf';
import { initLifecycle, callHook } from './lifecycle';
import { initProvide, initInjections } from './inject';
import { extend, mergeOptions, formatComponentName } from '../util/index';

let uid = 0;
export function initMixin(Vue: Class<Component>) {
  Vue.prototype._init = function(options?: Object) {
    const vm: Component = this; // 当前 Vue 实例
    vm._uid = uid++; // 当前 Vue 实例唯一标识

    /**************************** 非生产环境下进行性能监控 --- start ****************************/
    let startTag, endTag;
    if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
      startTag = `vue-perf-start:${vm._uid}`;
      endTag = `vue-perf-end:${vm._uid}`;
      mark(startTag);
    }

    vm._isVue = true; // 一个标志，避免该对象被响应系统观测

    /****************** 对 Vue 提供的 props、data、methods等选项进行合并处理 ******************/
    // _isComponent 内部选项：在 Vue 创建组件的时候才会生成
    if (options && options._isComponent) {
      initInternalComponent(vm, options); // 优化内部组件实例化，因为动态选项合并非常慢，而且没有一个内部组件选项需要特殊处理。
    } else {
      vm.$options = mergeOptions(
        resolveConstructorOptions(vm.constructor), // parentVal
        options || {}, // childVal
        vm
      );
    }

    // 设置渲染函数的作用域代理，其目的是提供更好的提示信息（如：在模板内访问实例不存在的属性，则会在非生产环境下提供准确的报错信息）
    if (process.env.NODE_ENV !== 'production') {
      initProxy(vm);
    } else {
      vm._renderProxy = vm;
    }

    vm._self = vm; // 暴露真实的实例本身

    /**************************** 执行相关初始化程序及调用初期生命周期函数 ****************************/
    initLifecycle(vm); // 初始化生命周期
    initEvents(vm); // 初始化事件
    initRender(vm); // 初始化渲染
    callHook(vm, 'beforeCreate'); // 调用生命周期钩子函数 -- beforeCreate
    initInjections(vm); // resolve injections before data/props
    initState(vm); // 初始化 initProps、initMethods、initData、initComputed、initWatch
    initProvide(vm); // resolve provide after data/props
    callHook(vm, 'created'); // 此时还没有任何挂载的操作，所以在 created 中是不能访问DOM的，即不能访问 $el

    /**************************** 非生产环境下进行性能监控 --- end ****************************/
    if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
      vm._name = formatComponentName(vm, false);
      mark(endTag);
      measure(`vue ${vm._name} init`, startTag, endTag);
    }

    /**************************** 根据挂载点，调用挂载函数 ****************************/
    if (vm.$options.el) {
      vm.$mount(vm.$options.el);
    }
  };
}
```
- 根据_init方法所做的事情可大概梳理出以下要点：

  - ① 在非生产环境下开启性能监控程序(监控 ②、③、④ 执行过程耗时)。
  - ② 对 Vue 提供的 props、data、methods 等选项进行合并处理。
  - ③ 设置渲染函数的作用域代理。
  - ④ 执行相关初始化程序及调用初期生命周期函数。
  - ⑤ 根据挂载点，调用挂载函数。
> 注：性能监控：利用 `Web Performance API` 允许网页访问某些函数来测量网页和Web应用程序的性能; 这里是`Vue - mark`、`measure`具体代码实现，就不过多赘述了; 接下来着重看被监控的几个步骤主要做了什么？

## 流程分析
如果就单单看代码，可能就不太直观且不易理解；不如直接用 Demo 代入断点调试看看每一步是如何做的，那将会使你对代码的运行有更直观的理解与认识。
```js
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
    <title>vue.js DEMO</title>
    <script src="../../dist/vue.js"></script>
  </head>
  <body>
    <div id="app">
      <p>计算属性：{{messageTo}}</p>
      <p>数据属性：{{ message }}</p>
      <button @click="update">更新</button>
      <item v-for="item in list" :msg="item" :key="item" @rm="remove(item)" />
    </div>

    <script>
      new Vue({
        el: '#app',

        components: {
          item: {
            props: ['msg'],
            template: `<div style="margin-top: 20px;">{{ msg }} <button @click="$emit('rm')">x</button></div>`,
            created() {
              console.log('---componentA - 组件生命周期钩子执行 created---');
            }
          }
        },

        mixins: [
          {
            created() {
              console.log('---created - mixins---');
            },
            methods: {
              remove(item) {
                console.log('响应移除：', item);
              }
            }
          }
        ],

        data: {
          message: 'hello vue.js',
          list: ['hello,', 'the updated', 'vue.js'],
          obj: {
            a: 1,
            b: {
              c: 2,
              d: 3
            }
          }
        },

        computed: {
          messageTo() {
            return `${this.message} !;`;
          }
        },

        watch: {
          message(val, oldVal) {
            console.log(val, oldVal, 'message - 改变了');
          }
        },

        methods: {
          update() {
            this.message = `${this.list.join(' ')} ---- ${Math.random()}`;
          }
        }
      });
    </script>
  </body>
</html>
```
根据上述 `demo` 断点进入 `Vue` 构造函数 `options` 参数如下断点图所：
![img](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/6/8/16b361a624f75125~tplv-t2oaga2asx-watermark.awebp)

### 1. 选项合并处理
- 根据上述 `Demo` 我们着重分析执行代码即 **`mergeOptions`** 函数，根据代码可知该函数是对我们传入的`options`做了一层处理，然后赋值给实例属性`$options`。
- **`resolveConstructorOptions`**, 该函数主要判断构造函数是否存在父类，若存在父类需要对 `vm.constructor.options` 进行处理返回，若不存在直接返回vm.constructor.options; 根据上述Demo直接返回 vm.constructor.options。
- 注：在上文初始化过程对 **`vm.constructor.options`** 进行处理，其结果为：
```js
Vue.options = {
  components: {
    KeepAlive,
    Transition,
    TransitionGroup
  },
  directives: {
    model,
    show
  },
  filters: Object.create(null),
  _base: Vue
};
```
```js
// _isComponent 内部选项：在 Vue 创建组件的时候才会生成
if (options && options._isComponent) {
  initInternalComponent(vm, options); // 优化内部组件实例化，因为动态选项合并非常慢，而且没有一个内部组件选项需要特殊处理。
} else {
  vm.$options = mergeOptions(
    resolveConstructorOptions(vm.constructor), // parentVal
    options || {}, // childVal
    vm
  );
}
```
根据上述分析，程序进入 **`mergeOptions`** 函数内部，下面断点图展示了该函数的入参
![img](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/6/8/16b363ba24849960~tplv-t2oaga2asx-watermark.awebp)

#### mergeOptions
将两个 `option` 对象合并到一个新的 `options`，用于实例化和继承的核心实用程序中。
```js
export function mergeOptions(
  parent: Object,
  child: Object,
  vm?: Component
): Object {
  // 校验组件的名字是否符合要求:
  //                       限定组件的名字由普通的字符和中横线(-)组成，且必须以字母开头。
  //                       检测是否是内置的标签（如：slot） ||  检测是否是保留标签（html、svg等）。
  if (process.env.NODE_ENV !== 'production') {
    checkComponents(child);
  }

  // 如果 child 是一个函数的话，去其静态属性 options 重写 child;
  if (typeof child === 'function') {
    child = child.options;
  }

  /************************  规范化处理  ************************/
  normalizeProps(child, vm);
  normalizeInject(child, vm);
  normalizeDirectives(child);

  /************************  extends/mixins 递归处理合并  ************************/
  const extendsFrom = child.extends;
  if (extendsFrom) {
    parent = mergeOptions(parent, extendsFrom, vm);
  }
  if (child.mixins) {
    for (let i = 0, l = child.mixins.length; i < l; i++) {
      parent = mergeOptions(parent, child.mixins[i], vm);
    }
  }

  /************************  合并阶段  ************************/
  const options = {};
  let key;
  for (key in parent) {
    mergeField(key);
  }
  for (key in child) {
    if (!hasOwn(parent, key)) {
      mergeField(key);
    }
  }
  function mergeField(key) {
    const strat = strats[key] || defaultStrat;
    options[key] = strat(parent[key], child[key], vm, key);
  }

  return options;
}
```
**`规范化处理`**
```js
normalizeProps(child, vm);
normalizeInject(child, vm);
normalizeDirectives(child);
```
上述代码主要对 Vue 选项进行规范化处理，我们知道 Vue 的选项支持多种写法，但最终都需要化为统一格式，进行处理。
下面所列出的是各种写法与规范化之后的对比；
上述代码实现就不过多论述了，可直接根据上述导航到代码段去看即可。


- Props:
  - 如下几种写法：
    - `props: ['size', 'myMessage']`
    - `props: { height: Number }`
    - `props: { height: { type: Number, default: 0 } }`

  - 统一格式处理之后为：
    - `props: { size: { type: null }, myMessage: { type: null } }`
    - `props: { height: { type: Number } }`
    - `props: { height: { type: Number, default: 0 } }`

- Inject:
  - 如下几种写法：
    - `inject: ['foo'],`
    - `inject: { bar: 'foo' }`
  - 统一格式处理之后为：
    - `inject: { foo: { from: 'foo' } }`
    - `inject: { bar: { from: 'foo' } }`

- Directives:
  - 如下几种写法：
    - `directives: { foo: function() { console.log('自定义指令: v-foo') }`
  - 统一格式处理之后为：
    - `directives: { foo: { bind: function() { console.log('v-foo'), update: function() { console.log('v-foo') } } }`

**`合并阶段`**
代码到执行到这里，将开始真正的合并了，最终返回合并之后的`options`。
```js
const options = {};
let key;
for (key in parent) {
  mergeField(key);
}
for (key in child) {
  if (!hasOwn(parent, key)) {
    mergeField(key);
  }
}
function mergeField(key) {
  const strat = strats[key] || defaultStrat;
  options[key] = strat(parent[key], child[key], vm, key);
}
return options;
```
这里特别说明一下，`Vue` 为每一个选项合并都提供了选项合并的策略函数，`strats` 变量存放着这些函数。这里就不分别对每个策略函数进行展开论述了。
```js
const defaultStrat = function(parentVal: any, childVal: any): any {
  return childVal === undefined ? parentVal : childVal;
};

export function mergeDataOrFn(
  parentVal: any,
  childVal: any,
  vm?: Component
): ?Function {
  // ...
}

// optionMergeStrategies: Object.create(null),
const strats = config.optionMergeStrategies;

// el / propsData 合并策略函数
if (process.env.NODE_ENV !== 'production') {
  strats.el = strats.propsData = function(parent, child, vm, key) {
    // ...
  };
}

// data 合并策略函数
strats.data = function(
  parentVal: any,
  childVal: any,
  vm?: Component
): ?Function {
  // ...
};

// watch 合并策略函数
strats.watch = function(
  parentVal: ?Object,
  childVal: ?Object,
  vm?: Component,
  key: string
): ?Object {
  // ...
};

// props、methods、inject、computed 合并策略函数
strats.props = strats.methods = strats.inject = strats.computed = function(
  parentVal: ?Object,
  childVal: ?Object,
  vm?: Component,
  key: string
): ?Object {
  // ...
};

// provide 合并策略函数
strats.provide = mergeDataOrFn;
```
根据上述分析，`mergeOptions` 函数将返回规范化，且合并之后`options`，下面断点图展示了合并之后的options：
![img](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/6/8/16b36809778c2c2a~tplv-t2oaga2asx-watermark.awebp)

### 2. 执行相关初始化程序及调用初期生命周期函数
```js
initLifecycle(vm); // 初始化生命周期
initEvents(vm); // 初始化事件
initRender(vm); // 初始化渲染
callHook(vm, 'beforeCreate'); // 调用生命周期钩子函数 -- beforeCreate
initInjections(vm); // resolve injections before data/props
initState(vm); // 初始化 initProps、initMethods、initData、initComputed、initWatch
initProvide(vm); // resolve provide after data/props
callHook(vm, 'created'); // 此时还没有任何挂载的操作，所以在 created 中是不能访问DOM的，即不能访问 $el
```
**`initLifecycle`**
- 如下代码主要做了：
  - 找到第一个非抽象父级
  - 将当前实例添加到父实例的 `$children` 属性里
  - 并设置当前实例的 `$parent` 为父实例
  - 在当前实例上设置一些属性
```js
export function initLifecycle(vm: Component) {
  const options = vm.$options;
  /**
   * abstract - 是否是抽象组件
   * 抽象组件: 它自身不会渲染一个 DOM 元素，也不会出现在父组件链中。(如 keep-alive transition )
   */
  let parent = options.parent;
  if (parent && !options.abstract) {
    // 循环查找第一个非抽象的父组件
    while (parent.$options.abstract && parent.$parent) {
      parent = parent.$parent;
    }
    parent.$children.push(vm);
  }
  vm.$parent = parent;
  vm.$root = parent ? parent.$root : vm;

  vm.$children = [];
  vm.$refs = {};
  vm._watcher = null;
  vm._inactive = null;
  vm._directInactive = false;
  vm._isMounted = false;
  vm._isDestroyed = false;
  vm._isBeingDestroyed = false;
}
```
**`initEvents`**
```js
export function initEvents(vm: Component) {
  // 在当前实例添加 `_events` `_hasHookEvent` 属性
  vm._events = Object.create(null);
  vm._hasHookEvent = false; // 用于判断是否存在生命周期钩子的事件侦听器
  const listeners = vm.$options._parentListeners; // 初始化父附加事件
  if (listeners) {
    updateComponentListeners(vm, listeners);
  }
}
```
**`initRender`**
```js
export function initRender(vm: Component) {
  vm._vnode = null; // the root of the child tree
  vm._staticTrees = null; // v-once cached trees

  /***************************  解析并处理 slot  **************************/
  const options = vm.$options;
  const parentVnode = (vm.$vnode = options._parentVnode); // the placeholder node in parent tree
  const renderContext = parentVnode && parentVnode.context;
  vm.$slots = resolveSlots(options._renderChildren, renderContext);
  vm.$scopedSlots = emptyObject;

  /***************************  包装 createElement()   **************************/
  // render: (createElement: () => VNode) => VNode createElement
  // 将createElement fn绑定到这个实例，以便在其中获得适当的呈现上下文。
  // args顺序:标签、数据、子元素、normalizationType、alwaysNormalize内部版本由模板编译的呈现函数使用
  vm._c = (a, b, c, d) => createElement(vm, a, b, c, d, false);
  // 规范化总是应用于公共版本，用于用户编写的呈现函数。
  vm.$createElement = (a, b, c, d) => createElement(vm, a, b, c, d, true);

  /***************************  在实例添加 $attrs/$listeners   **************************/
  // $attrs和$listeners 用于更容易的临时创建。它们需要是反应性的，以便使用它们的 HOC 总是被更新
  const parentData = parentVnode && parentVnode.data;
  if (process.env.NODE_ENV !== 'production') {
    // 定义响应式的属性
    defineReactive(
      vm,
      '$attrs',
      (parentData && parentData.attrs) || emptyObject,
      () => {
        !isUpdatingChildComponent && warn(`$attrs is readonly.`, vm);
      },
      true
    );
    defineReactive(
      vm,
      '$listeners',
      options._parentListeners || emptyObject,
      () => {
        !isUpdatingChildComponent && warn(`$listeners is readonly.`, vm);
      },
      true
    );
  } else {
    defineReactive(
      vm,
      '$attrs',
      (parentData && parentData.attrs) || emptyObject,
      null,
      true
    );
    defineReactive(
      vm,
      '$listeners',
      options._parentListeners || emptyObject,
      null,
      true
    );
  }
  /***************************  在实例添加 $attrs/$listeners   **************************/
}
```
**`callHook`**
```js
export function callHook(vm: Component, hook: string) {
  pushTarget(); // 为了避免在某些生命周期钩子中使用 props 数据导致收集冗余的依赖 #7573
  const handlers = vm.$options[hook];
  if (handlers) {
    // 在合并选项处理时：生命周期钩子选项会被合并处理成一个数组
    for (let i = 0, j = handlers.length; i < j; i++) {
      try {
        handlers[i].call(vm);
      } catch (e) {
        // 捕获生命周期函数执行过程中可能抛出的异常
        handleError(e, vm, `${hook} hook`);
      }
    }
  }
  // 判断是否存在生命周期钩子的事件侦听器，在 initEvents 中初始化，若存在触发响应钩子函数
  if (vm._hasHookEvent) {
    vm.$emit('hook:' + hook);
  }
  popTarget();
}
```
> 这里额外提一下: 可以使用 hook: 加 生命周期钩子名称 的方式来监听组件相应的生命周期
```js
<child
  @hook:beforeCreate="handleChildBeforeCreate"
  @hook:created="handleChildCreated"
  @hook:mounted="handleChildMounted"
  @hook:生命周期钩子名称
/>
```
**`initInjections`**
```js
export function initInjections(vm: Component) {
  const result = resolveInject(vm.$options.inject, vm); // 作用：寻找父代组件提供的数据
  if (result) {
    // provide 和 inject 绑定并不是可响应的。
    // 这是刻意为之的。然而，如果你传入了一个可监听的对象，那么其对象的属性还是可响应的。
    toggleObserving(false); // 关闭响应式检测
    Object.keys(result).forEach(key => {
      // 对每个属性定义响应式属性，并在非生产环境下，提供警告程序。
      if (process.env.NODE_ENV !== 'production') {
        defineReactive(vm, key, result[key], () => {
          warn(
            `避免直接修改注入的值，因为当提供的组件重新呈现时，更改将被覆盖。正在修改的注入:“${key}”`,
            vm
          );
        });
      } else {
        defineReactive(vm, key, result[key]);
      }
    });
    toggleObserving(true); // 开启响应式检测
  }
}
```
**`initState`**
```js
/**
 * 初始化 props/ methods/ data/ computed/ watch/ 等选项。
 */
export function initState(vm: Component) {
  vm._watchers = [];
  const opts = vm.$options;
  if (opts.props) initProps(vm, opts.props);
  if (opts.methods) initMethods(vm, opts.methods);
  if (opts.data) {
    initData(vm);
  } else {
    observe((vm._data = {}), true /* asRootData */);
  }
  if (opts.computed) initComputed(vm, opts.computed);
  if (opts.watch && opts.watch !== nativeWatch) {
    initWatch(vm, opts.watch);
  }
}
```
> 注： 这里只是简单展示了其初始化顺序，其内部各个初始化方法将在`构建响应式系统`深挖。
这里只需要明白一点，即初始化顺序：`props` => `methods` => `data` => `computed` => `watch` (根据上述顺序，自然也就知道，为什么可以在data选项中使用props去初始化值)

**`initProvide`**
```js
export function initProvide(vm: Component) {
  const provide = vm.$options.provide;
  if (provide) {
    vm._provided = typeof provide === 'function' ? provide.call(vm) : provide;
  }
}
```
上述初始化部分的分析，只是简单的梳理了其执行过程，如果想对其内部实现做更为细致的认识，可以自行去看看代码实现或上述说明提到的源码解析的相关文章。

### 3. 根据挂载点，调用挂载函数
若存在挂载点，则执行挂载函数，渲染组件。挂载函数如何执行，实现机制如何，将在后文慢慢梳理出来。
```js
if (vm.$options.el) {
  vm.$mount(vm.$options.el);
}
```
总结：全文梳理了执行 new Vue() 调用 _init() 方法，接着又跟着代码执行过程探讨了内部实现。