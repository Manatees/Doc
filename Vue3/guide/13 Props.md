## Prop 类型

在这里我们只看到以字符串数组形式列出的 prop

```javascript
props: ['title', 'likes', 'isPublished', 'commentIds', 'author']
```

但是，通常你希望每个prop都有指定的值类型。你可以以对象形式列出prop，给出各自的名称和类型

```javascript
props: {
    title: String,
    likes: Number,
    isPublished: Boolean,
    commentIds: Array,
    author: Object,
    callback: Function,
    contactsPromise: Promise
}
```



## 传递静态或动态的 Prop

```html
<!-- 静态值 -->
<blog-post title="My journey with Vue"></blog-post>
<!-- 动态值 -->
<blog-post :title="post.title"></blog-post>
<blog-post :title="post.title + ' by ' + post.author.name"></blog-post>
```

### 传入一个数字

```html
<!-- 即便 42 是静态的，仍然需要 v-bind 来告诉 Vue -->
<!-- 这是一个 表达式，而不是一个字符串 -->
<blog-post :likes="42"></blog-post>
<blog-post :likes="post.likes"></blog-post>
```

### 传入一个布尔值

```html
<blog-post is-published></blog-post>
<blog-post :is-published="false"></blog-post>
<blog-post :is-published="post.isPublished"></blog-post>
```

### 传入一个数组

```html
<blog-post :comment-ids="[234, 266, 273]"></blog-post>
<blog-post :comment-ids="post.commentIds"></blog-post>
```

### 传入一个对象

```html
<blog-post 
   :author="{
        name: 'Veronica',
        company: 'Veridian Dynamics'
    }"
></blog-post>

<blog-post :author="post.author"></blog-post>
```

### 传入一个对象的所有 property

如果你想要将一个对象的所有 propery 都作为 prop 传入，你可以使用不带参数的 `v-bind`

```javascript
post: {
    id: 1,
    title: 'My Journey with Vue'
}
```

```html
<blog-post v-bind="post"></blog-post>
```

等价于

```html
<blog-post :id="post.id" :title="post.title"></blog-post>
```



## 单向数据流

所有的prop都使得其父子prop之间形成了一个**单向下行绑定**：父级prop的更新会向下流动到子组件中，但是反过来则不行。这样会防止从子组件意外变更父组件的状态，从而导致你的应用的数据流向难以理解。

另外，每次父级组件发生更新时，子组件中所有的prop都将会刷新为最新的值。这意味着你**不**应该在一个子组件内部改变prop。如果你这样做了，Vue会在浏览器的控制台中发出警告。

有两种常见的试图变更一个prop的情形

1. 这个prop用来传递一个初始值，这个子组件接下来希望将其作为一个本地的prop数据来使用。这种情况下，最好定义一个本地的data property并将这个prop作为其初始值

```javascript
props: ['initialCounter'],
data() {
    return {
        counter: this.initialCounter
    }
}
```

2. 这个prop以一种原始的值传入且需要进行转换。在这种情况下，最好使用这个prop的值来定义一个计算属性

```javascript
props: ['size'],
computed: {
    normalizedSize: function() {
        return this.size.trim().toLowerCase()
    }
}
```

## Prop验证

```javascript
app.component('my-component', {
    props: {
        // 基础的类型检查， null 和 undefined 会通过任何类型验证
        propA: Number,
        // 多个可能的类型
        propB: [String, Number],
        // 必填的字符串
        propC: {
            type: String,
            required: true
        },
        // 带有默认值的数字
        propD: {
            type: Number,
            default: 100
        },
        // 带有默认值的对象
        propE: {
            type: Object,
            default: function() {
                return { message: 'hello'}
            }
        },
        // 自定义验证函数
        propF: {
            validator: function(value) {
                return ['success', 'warning', 'danger'].indexOf(value) !== -1
            }
        },
        // 具有默认值的函数
        propG: {
            type: function,
            // 与对象或数组默认值不同，这不是一个工厂函数，这是一个用作默认值的函数
            default: function() {
                return "Default function"
            }
        }
    }
})
```



## Prop 的大小写命名

HTML 中的 attribute 名是大小写不敏感的，所以浏览器会把所有大写字符解释为小写字符。这意味着当你使用DOM中的模板时，camelCase 的prop 名需要使用其等价的 kebab-case 命名

```javascript
const app = Vue.createApp({})
app.compnent('blog-post', {
    props: ['postTitle'],
    template: '<h3>{{ postTitle }}</h3>'
})
```

```html
<!-- kebab-case in HTML -->
<blog-post post-title="hello!"></blog-post>

```

> 如果你使用字符串模板，那么这个限制就不存在了。