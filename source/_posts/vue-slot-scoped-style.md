---
title: Vue 的插槽可以使用父组件的 scoped style
date: 2021-09-18 18:19:38
tags: Vue
---

这次记录一个 Vue 框架使用过程中的一个问题，这个问题与组件的插槽有关。问题到最后会发现，插槽内可以使用父组件的 class 定义，即使这个定义父组件为其添加了 scoped 限制。这会带来麻烦，尤其是当我们无意识地为 class 取了一个跟父组件一样的名字的时候……

<!-- more -->

## 问题重现

这个问题需要用到 Vue 的*单文件组件*、*插槽*、*Scoped Style 块*。所以我不能确定是由于 Vue 引起的，还是 vue-loader  引起的。

版本：

- Vue：2.6.14
- vue-loader: 15.9.8

还是直接上代码，给出下面的一个组件（不妨命名为 `SlotComponent`），该组件带自定义插槽：

```vue
<template>
  <div class="container bg-blue">
    <div class="margin-25 bg-white">
      <div class="margin-25">
        <slot />
      </div>
    </div>
  </div>
</template>

<style scoped>
.bg-blue {
  background: blue;
}
.bg-white {
  background: white;
}
.container{
  border:1px solid #000;
  position: relative;
  width:200px;
  height:200px;
}
.margin-25{
  border:1px solid #000;
  position: absolute;
  left: 25px; right: 25px; top: 25px; bottom: 25px;
}
</style>
```

该组件从外向内包围了三个块，其效果直接显示为：

<div style="border:1px solid #000;position: relative;width:200px;height:200px;background:blue">
  <div style="border:1px solid #000;position: absolute;  left: 25px; right: 25px; top: 25px; bottom: 25px;background:white">
    <div style="border:1px solid #000;position: absolute;  left: 25px; right: 25px; top: 25px; bottom: 25px;background:white"></div>
  </div>
</div>
现在我在另一个组件内（不妨命名为 `DemoComponent`）使用该组件，同时在插槽内添加一个 class 为 `bg-blue`，该 class 为 DemoComponent 组件内定义的：

```vue
<SlotComponent>
  <div class="bg-blue" style="width: 100%; height: 100%;"></div>
</SlotComponent>
```

此时 `bg-blue` 起作用了：

<div style="border:1px solid #000;position: relative;width:200px;height:200px;background:blue">
  <div style="border:1px solid #000;position: absolute;  left: 25px; right: 25px; top: 25px; bottom: 25px;background:white">
    <div style="border:1px solid #000;position: absolute;  left: 25px; right: 25px; top: 25px; bottom: 25px;background:blue"></div>
  </div>
</div>

虽然 `DemoComponent` 没有定义 `bg-blue` 类，但 `SlotComponent` 内定义的 `bg-blue` 类依然对它起了作用。

## 这会带来什么问题？

有人会问，这样会带来什么影响呢？当我们在使用带插槽的组件的时候，就会有一个麻烦，我们必须时刻知道带插槽的组件的内部 class 类名，而我们自己定义的类名不能与它同名，否则将带来冲突。

比如，一个插槽组件，其内部结构是：

```html
<div class="container">
  <div class="side">
    <slot />    
  </div>
  <div class="main">
    <slot />
  </div>
  <div class="side">
    <slot />    
  </div>
</div>

<style scoped>
.container { /*...*/ }
.main { /* 插槽组件定义的 main 类 */ }
.side { /*...*/ }
</style>
```

这是典型的三栏布局，其具体的 style 实现这里就不再赘述了。假设这个组件名为 `Layout`，如果我们在自己的组件内使用了同名的 class 类名，就会受到影响：

```vue
<template>
  <Layout>
    <div class="main" slot="main"></div>
  </Layout>
</template>

<style scoped>
.main {
  /* 这里的类名与 Layout 组件同名了，也许开发者没有意识到，但是 Layout 的 main 类已经在影响这里了 */
}
</style>
```

如果你运气好，这种影响也许不会发生什么实质性的结果。但是如果你运气不好，可能就要花相当的时间才能发现问题的症结所在。我不知道其他人怎么看待这个问题，对我而言，它就是一个坑。

（感兴趣的同学可以自行打开开发者工具，看一下两个 scoped 定义分别为它们添加了几个 `data-v` 属性。）

所以，当开发一个带插槽的组件时，即使将所有的 class 定义都放在 scoped 块内，其影响依然会映射到插槽内。尤其是遇到像 `container`、`main`、`side` 这样常见的命名时。避免这一问题可以添加组件名前缀，例如上面的类名都可以重新命名为`layout-container`、 `layout-main`、`layout-side`。如果做的绝一点，所有组件的 class 定义都添加组件名前缀，这样就很难出问题了。不过讲真，都已经自己添加前缀了，使不使用 `scoped` 块就没那么重要了。这样的话，`scoped` 块提供给我们的遍历性就荡然无存，到头来还是自己在用最朴素的方法管理命名空间。

所以，这真的是坑。
