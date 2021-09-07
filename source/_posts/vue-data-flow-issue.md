---
title: Vue 数据流踩坑：属性的改变不能立马传递给子组件
date: 2021-09-07 11:39:21
tags: Vue
---

这篇文章讲述自己 Vue 开发实践中遇到的一个小问题，通过这个小问题让自己对属性在组件之间如何传递产生了一些思考。首先，通过一个简单的例子说明，外部绑定的改变不能立马在组件内响应到；然后通过 `$nextTick` 修正了此问题；最后，提到 `$nextTick` 不是一个好的实践，从而引发对好的实践的思考。

<!-- more -->

今天遇到一个开发上的小问题，让我对 Vue 的数据流过程不得不产生关注。

## 简单复现

我遇到的问题，可以用以下的代码简单概括：

```javascript
// 子组件，它的目的就是显示属性 msg
const Hello = {
  props: ['name'],
  data () {
    return {
      msg: ''
    }
  },
  methods: {
    sayHello () {
      if (this.name) {
        this.msg = 'Hello, ' + this.name + '!'
      }
    }
  },
  template: '<span>{{ msg || "No content!" }}</span>'
}

// 父组件，它向子组件传递 name 属性
new Vue({
  components: {
    Hello
  },
  data () {
    return {
      name: ''
    }
  },
  mounted () {
    this.name = 'James'
    // 注意以下代码，它主动要求 Hello 组件调用 sayHello 方法。虽然此时
    // 父组件已经将 name 设置为 'James'，但这一变动并没有传递到子组件，
    // 因此子组件的显示内容依然为 'No content!'
    this.$refs.hello.sayHello()
  },
  template: '<Hello ref="hello" :name="name" />'
}).$mount('#app')
```

出现问题的原因在于，当在父组件内修改 `name` 属性时，这一变动并没有立即传递到子组件。如果要确保子组件已经响应变动，可以将代码放在 `$nextTick` 中：

```javascript
this.$nextTick(() => {
  this.$refs.hello.sayHello()
})
```

但即便如此，我仍然是不太满意的。因为以前的我是不关心数据流的问题的，现在又不得不注意了。而拿上面的代码来说，数据流的发展对我而言并不自然，我怎么确定在 `$nextTick` 回调中 `name` 的变动已经响应到子组件了呢？总之，我在调用 Hello 组件的 `sayHello` 方法时，不得不关注组件内部的状态，这真的不是一种很好的开发体验。

有些人可能意识到，用计算属性不就解决问题了吗？因为这是极度简化的场景，与真实场景有所出入，所以并不能从这个简单例子得到什么。试想一下，`this.msg` 的获取如果是通过异步方法呢？这时就不能用计算属性做到了。简单场景可以用 `$watch` 做到，但现实往往更复杂一点。

## 一个真实场景的示例

下面将演示一个真实场景，说明计算属性和 `$watch` 都不能很好地实现。事情是这样的，我将做一个博文写作的页面，这个页面分为两块：编辑区和笔记区。编辑区就放置一个 textarea，笔记区类似于 Todo List 的操作。

新建博文页面和更新博文页面都有编辑区域和笔记区域，其中笔记区域依赖 `NoteList` 组件实现，其完整的代码如下。

新建博文页面：

```vue
<!-- views/NewPost.vue -->
<template>
  <div class="layout">
    <div class="main">
      <input type="text" v-model="post.title" />
      <textarea v-model="post.body" />
      <button @click="publish">发布</button>
    </div>
    <div class="right">
      <NoteList ref="noteList" postId="post.id" />
    </div>
  </div>
</template>

<script>
import Post from '@/models/post'
import NoteList from '@/components/NoteList.vue'

export default {
  name: 'NewPost',
  components: {
    NoteList
  },
  data () {
    return {
      post: {}
    }
  },
  mounted () {
    // 开场时调用 fetchAll，此时 this.post.id 为 undefined
    this.$nextTick(() => {
      this.$refs.noteList.fetchAll()
    })
  },
  methods: {
    async publish () {
      this.post = await Post.create(this.post)

      // 当发布成功后，this.post.id 被设置为发布后的博文 id，此时
      // 不要调用 fetchAll，而应该调用 updatePostId 将所有笔记的
      // postId 绑定到 this.post.id
      this.$nextTick(() => {
        this.$refs.noteList.updatePostId()
      })
    }
  }
}
</script>
```

编辑博文页面：

```vue
<!-- views/EditPost.vue -->
<template>
  <div class="layout">
    <div class="main">
      <intput type="text" v-model="post.title" />
      <textarea v-model="post.body" />
      <button @click="update">更新</button>
    </div>
    <div class="right">
      <NoteList ref="noteList" :postId="post.id" />
    </div>
  </div>
</template>

<script>
import Post from '@/models/post'
import NoteList from '@/components/NoteList.vue'

export default {
  name: 'EditPost',
  components: {
    NoteList
  },
  data () {
    return {
      post: {}
    }
  },
  async mounted () {
    // 开场时不要调用 fetchAll
    const postId = parseInt(this.$route.params.id)
    this.post = await Post.find(postId)

    // 当 this.post 准备就绪后调用 fetchAll
    this.$nextTick(() => {
      this.$refs.noteList.fetchAll()
    })
  },
  methods: {
    async update () {
      this.post = await Post.update(this.post)
    }
  }
}
</script>
```

让我们忽略一些细节，重点关注以下几个部分：

- `<NoteList ref="noteList" postId="post.id" />`: 新建博文页面和更新博文页面都用同样的方式引用 `NoteList` 组件。新建博文组件也传递这个属性的原因是，当发布以后，`post.id` 将设置为发布后的博文 id，这样可以将属性的变动传递到 `NoteList` 组件，方便之后 `NoteList` 组件调用 `updatePostId` 方法。
- 两个页面的 `mounted` 方法都主动调用了 `NoteList` 组件的 `fetchAll` 方法，只不过时机不同。
- 从中可以看出 `NoteList` 组件组织数据的逻辑了：`postId` 作为属性传递，而 `fetchAll` 方法和 `updatePostId` 方法依赖 `NoteList` 组件的 `postId` 状态。所以，当调用 `fetchAll` 方法和 `updatePostId` 方法时，要确保 `postId` 的状态已就绪。这也就是调用这些方法始终放在 `$nextTick` 中的原因。

其实现在就可以回答为什么这里不能使用计算属性和 `$watch` 机制的理由了。首先显然不能用到计算属性，因为 `fetchAll` 是异步调用。说明不能使用 `$watch`  机制稍微费点口舌，但只要理解了在什么时机响应数据，就很好理解了：

> 我们观察 `NewPost` 页面内绑定到 `NoteList` 组件的属性 `postId` 的变化，它从 `null`（默认状态） 改变为发布后博文的 id（记为 `A`），*希望在 `null` 状态时调用 `fetchAll` 方法而在 `A` 状态时不调用*；我们继续观察 `EditPost` 页面内绑定到 `NoteList` 组件的属性 `postId` 的变化，它从 `null`（默认状态）改变为正在编辑的博文的 id（不妨仍然记为 `A`），*希望在 `null` 状态时不调用 `fetchAll`而在 `A` 状态时调用*。很明显，两者的监听逻辑不一致，所以不能够用 `$watch` 机制。

最后，贴一下 `NoteList` 组件的代码实现，以辅助于刚才所做的解释。已经理解的朋友完全可以跳过。

```vue
<!-- components/NoteList.vue -->
<template>
  <div>
    <input type="text" v-model="input" @keyup.enter="add" />
    <ul>
      <li v-for="note in notes" :key="note.id">{{ note.text }}</li>
    </ul>
  </div>
</template>

<script>
import Note from '@/models/note'

export default {
  name: 'NoteList',
  props: ['postId'],
  data () {
    return {
      input: '',
      notes: []
    }
  },
  methods: {
    async add () {
      const newNote = await Note.create({ text: this.input, postId: this.postId })
      this.notes.push(newNote)
    },
    async fetchAll () {
      this.notes = await Note.all({ postId: this.postId })
    },
    async updatePostId () {
      for (const note of this.notes) {
        note.postId = this.postId
        Object.assign(note, await Note.update(note))
      }
    }
  }
}
</script>
```

## 应该用何种方式

前面一节我们已经证明了不能使用计算属性和 `$watch` 机制实现此类场景，而是采用了 `$nextTick` 机制加以实现。但我认为 `$nextTick` 机制不是一个好的想法，它让我们从一个不用关注数据流的状态转向了不得不关注它的状态。关于这一点，是不是我们的组件设计出了问题？

我想是的，仔细推敲，`NoteList` 组件是有设计缺陷的，但具体有哪些缺陷却又不方便用言语说明。只能说以后留个心眼，当组件需要暴露一些方法给外面，那这些方法就不要依赖属性了，因为属性的数据流真的不好控制。可以将属性改为状态：

```javascript
export default {
  data () {
    return {
      postId: null,
      input: '',
      notes: []
    }
  },
  methods: {
    setPostId (postId) {
      this.postId = postId
    },
    async fetchAll () { /*...*/ },
    async updatePostId () { /*...*/ }
  }
}
```

另外，由于 `NoteList` 组件主要是新增和展示笔记的，它对 `postId` 似乎不太关心。考虑到暴露给外界调用的方法不多，可将方法声明调整为下面这样，不用引入 `postId` 状态：

```javascript
export default {
  methods: {
    async fetchAll ({ postId }) { /*...*/ },
    async updatePostId ({ postId }) { /*...*/ }
  }
}
```

使用 vuex 可视为最终方案。将 `NoteList` 绑定到全局 store 下的 `notes` 状态，这样我们只需要自由地修改 `notes` 状态就可以了。有时候，vuex 真的就是终极大法。
