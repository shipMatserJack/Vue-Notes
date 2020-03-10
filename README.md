# Vue-Notes

***

## 最佳实践

### 程序化事件监听器
到目前为止，您已经看到了$emit，和with一起使用的用法v-on，但Vue实例在其事件接口中还提供了其他方法。我们可以：

与一起听事件 $on(eventName, eventHandler)
只听一次活动 $once(eventName, eventHandler)
停止收听以下事件 $off(eventName, eventHandler)
通常，您不必使用它们，但是在需要手动侦听组件实例上的事件的情况下，它们是可用的。它们还可以用作代码组织工具。例如，您可能经常会看到这种用于集成第三方库的模式：

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

这有两个潜在的问题：

picker当可能只有生命周期挂钩需要访问它时，它需要将保存到组件实例。这并不可怕，但可以认为它很杂乱。
我们的设置代码与清理代码保持分开，这使得以编程方式清理我们设置的内容变得更加困难。
您可以使用程序化侦听器解决这两个问题：
mounted: function () {
  var picker = new Pikaday({
    field: this.$refs.input,
    format: 'YYYY-MM-DD'
  })

  this.$once('hook:beforeDestroy', function () {
    picker.destroy()
  })
}

使用这种策略，我们甚至可以将Pikaday与多个输入元素一起使用，每个新实例都会在其自身之后自动清除：

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

### 路由参数变化组件不更新

同一path的页面跳转时路由参数变化，但是组件没有对应的更新。

原因：主要是因为获取参数写在了created或者mounted路由钩子函数中，路由参数变化的时候，这个生命周期不会重新执行。

解决方案1：watch监听路由

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
解决方案2 ：为了实现这样的效果可以给router-view添加一个不同的key，这样即使是公用组件，只要url变化了，就一定会重新创建这个组件。

<router-view :key="$route.fullpath"></router-view>
