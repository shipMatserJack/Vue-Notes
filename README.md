# Vue-Notes

***

## Vue2最佳实践(高级进阶)

### 1. 程序化事件监听器
到目前为止，您已经看到了$emit，和with一起使用的用法v-on，但Vue实例在其事件接口中还提供了其他方法。我们可以：

与一起听事件 $on(eventName, eventHandler)
只听一次活动 $once(eventName, eventHandler)
停止收听以下事件 $off(eventName, eventHandler)
通常，您不必使用它们，但是在需要手动侦听组件实例上的事件的情况下，它们是可用的。它们还可以用作代码组织工具。例如，您可能经常会看到这种用于集成第三方库的模式：

```
// Attach the datepicker to an input once
// it's mounted to the DOM.
mounted: function () {
  // Pikaday is a 3rd-party datepicker library
  this.picker = new Pikaday({
    field: this.$refs.input,
    format: 'YYYY-MM-DD'
  })
},
// Right before the component is destroyed,
// also destroy the datepicker.
beforeDestroy: function () {
  this.picker.destroy()
}
```

这有两个潜在的问题：

picker当可能只有生命周期挂钩需要访问它时，它需要将保存到组件实例。这并不可怕，但可以认为它很杂乱。
我们的设置代码与清理代码保持分开，这使得以编程方式清理我们设置的内容变得更加困难。
您可以使用程序化侦听器解决这两个问题：
```
mounted: function () {
  var picker = new Pikaday({
    field: this.$refs.input,
    format: 'YYYY-MM-DD'
  })

  this.$once('hook:beforeDestroy', function () {
    picker.destroy()
  })
}
```
使用这种策略，我们甚至可以将Pikaday与多个输入元素一起使用，每个新实例都会在其自身之后自动清除：

```
mounted: function () {
  this.attachDatepicker('startDateInput')
  this.attachDatepicker('endDateInput')
},
methods: {
  attachDatepicker: function (refName) {
    var picker = new Pikaday({
      field: this.$refs[refName],
      format: 'YYYY-MM-DD'
    })

    this.$once('hook:beforeDestroy', function () {
      picker.destroy()
    })
  }
}
```

### 2. 路由参数变化组件不更新

同一path的页面跳转时路由参数变化，但是组件没有对应的更新。

原因：主要是因为获取参数写在了created或者mounted路由钩子函数中，路由参数变化的时候，这个生命周期不会重新执行。

解决方案1：watch监听路由

```
watch: {
 // 方法1 //监听路由是否变化
  '$route' (to, from) {
   if(to.query.id !== from.query.id){
            this.id = to.query.id;
            this.init();//重新加载数据
        }
  }
}
//方法 2  设置路径变化时的处理函数
watch: {
'$route': {
    handler: 'init',
    immediate: true
  }
}
```
解决方案2 ：为了实现这样的效果可以给router-view添加一个不同的key，这样即使是公用组件，只要url变化了，就一定会重新创建这个组件。

```
<router-view :key="$route.fullpath"></router-view>
```
### 3. 递归组件实现逻辑
 定义子组件tree.vue, 组件template模板内引用tree自身完成递归，传参建议采用eventBus中央通信，以下为demo
```
 <template>
  <div class="tree">
    <!-- 第一层开始 -->
    <van-collapse v-model="activeName" accordion v-if="orgList.length">
      <van-collapse-item
        v-for="(org, index) in orgList"
        :key="index"
        :name="org.name"
        :disabled="gridsOrg.map(v=>v.id).includes(org.id)"
      >
        <div slot="title" class="flex-start">
          <van-icon
            name="add"
            color="#1989fa"
            :style="{'margin-right':'5px'}"
            @click.stop="choose(org)"
          />
          {{`${org.name}`}}
        </div>
        <template v-if="org.children_organizations && org.users">
          <!-- 在这里引用tree自身，完成递归操作 -->
          <tree :orgList="org.children_organizations" :users="org.users" :gridsOrg="gridsOrg"></tree>
        </template>
      </van-collapse-item>
    </van-collapse>
    <van-cell-group v-if="users.length">
      <van-cell
        v-for="(item, index) in users"
        :key="index"
        :title="`${item.name}`"
        @click.stop="choose(item)"
      />
    </van-cell-group>
    <!-- 第一层结束 -->
  </div>
</template>

<script>
export default {
  name: "tree",
  props: {
    // 组织
    orgList: {
      type: Array,
      default: () => []
    },
    // 个人
    users: {
      type: Array,
      default: () => []
    },
    // 选中的组织
    gridsOrg: {
      type: Array,
      default: () => []
    }
  },
  data() {
    return {
      activeName: ""
    };
  },

  created() {},

  methods: {
    /**
     * @desc 选择拜访/组织
     * @param {Obj} data 拜访人信息
     */
    choose(data) {
      const isDisabled = this.$props.gridsOrg.map(v => v.id).includes(data.id);
      // 选择组织后收起折叠面板
      if (data.children_organizations && !isDisabled) {
        this.activeName = "";
      }
      // 发布订阅事件
      this.$root.eventBus.$emit("choosen", data);
    }
  }
};
</script>

<style>
</style>
```
### 4. eventBus解决递归组件下子组件无法通过$emit向根组件发送数据
解决方案：使用vue中央通信工具eventBus
1、使用场景：
    1)、兄弟组件传参
    2)、父子组件传参（递归组件下$emit传递参数根组件不能接收的情况下）
    3)、全局数据监听触发
2、实现思想：发布---订阅者模式

### 5.slot插槽在父子组件中的传参应用（v2.5.X）
1、应用场景：父组件<father />在template模版中获取子组件<child />传入的参数
2、代码实现
子组件
```
  <child>
    <slot name="child" :data="datas" />
  </chiild>
```
父组件接收
```
  <father>
    <template slot="child" slot-scope="props">
      <div>{{props.data}}</div>
    </template>
  </father>
```
以上父组件通过slot-scope接收子组件的data参数,vue2.6.X参考https://www.cnblogs.com/gxp69/p/10784299.html


### 6.监听组件的生命周期
比如有父组件 Parent 和子组件 Child，如果父组件监听到子组件挂载 mounted 就做一些逻辑处理，常规的写法可能如下：
```js
// Parent.vue
<Child @mounted="doSomething"/>

// Child.vue
mounted() {
  this.$emit("mounted");
}
```
此外，还有一种特别简单的方式，子组件不需要任何处理，只需要在父组件引用的时候通过@hook 来监听即可，代码如下：
```js
<Child @hook:mounted="doSomething" />
<Child @hook:updated="doSomething" />
```
当然这里不仅仅是可以监听 mounted，其它的生命周期事件，例如：created，updated 等都可以。

### 7.长列表性能优化
我们应该都知道 vue 会通过 object.defineProperty 对数据进行劫持，来实现视图响应数据的变化，然而有些时候我们的组件就是纯粹的数据展示，不会有任何改变，我们就不需要 vue 来劫持我们的数据，在大量数据展示的情况下，这能够很明显的减少组件初始化的时间。

所以，我们可以通过 object.freeze 方法来冻结一个对象，这个对象一旦被冻结，vue就不会对数据进行劫持了。
```js
export default {
  data: () => ({
    list: []
  }),
  async created() {
    const list = await axios.get('xxxx')
    this.list = Object.freeze(list)
  },
  methods: {
    // 此处做的操作都不能改变list的值
  }
}
```
另外需要说明的是，这里只是冻结了 list 的值，引用不会被冻结，当我们需要 reactive 数据的时候，我们可以重新给 list 赋值。


### 8. render函数解决多个根元素报错的问题
functional: true,
render(h, { props }) {
  return props.routes.map(route =>
    <li key={route.name}>
      <router-link to={route}>
        {route.title}
      </router-link>
    </li>
  )
}
