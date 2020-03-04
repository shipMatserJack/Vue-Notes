# Vue-Notes

***

*最佳实践
 程序化事件监听器
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

