## 插槽内容

Vue 实现了一套内容分发的API，将 `<slot>` 元素作为承载分发内容的出口

它允许你像这样合成组件

```html
<todo-button>
    Add todo
</todo-button>
```

然后在 `<todo-button>` 的模板中，你可能有

```html
<!-- todo-button 组件模板 -->
<button class="btn-primary">
    <slot></slot>
</button>
```

当组件渲染的时候，将会被替换为 "Add Todo"

```html
<!-- 渲染 HTML -->
<button class="btn-primary">
    Add todo
</button>
```

不过，字符串只是开始，插槽还可以包含任何模板代码，包括 HTML

```html
<todo-button>
    <!-- 添加一个 Font Awesome 图标 -->
	<i class="fas fa-plus"></i>
    Add todo
</todo-button>
```

或其他组件

```html
<todo-button>
    <!-- 添加一个图标的组件 -->
	<font-awesome-icon name="plus"></font-awesome-icon>
    Add todo
</todo-button>
```

如果 `<todo-button>` 的 template 中没有包含一个 `<slot>` 元素，则该组件起始标签和结束续签之间的任何内容都会被抛弃

```html
<!-- todo-button 组件模板 -->
<button class="btn-primary">
    Create a new item
</button>
```

```html
<todo-button>
	<!-- 以下文本不会渲染 -->
    Add todo
</todo-button>
```

## 渲染作用域

当你想在一个插槽中使用数据时，

```html
<todo-button>
	Delete a {{ item.name }}
</todo-button>
```

该插槽可以访问与模板其余部分相同的实例 property ，即相同的作用域。不能访问 `<todo-button>` 的作用域

<img src="G:\Doc\Vue3\guide\imgs\slot.png" alt="slot" style="zoom:67%;" />

> 父级模板里的所有内容都是在父级作用域中编译的，
>
> 子模板里的所有内容都是在子作用域中编译的。



## 后备内容

有时为一个插槽设置具体的后备（也就是默认的）内容是很有用的，它只会在没有提供内容的时候被渲染。

例，我们可能希望这个 `<button>` 绝大多数情况下都渲染文本 “Submit"。

```html
<button type="submit">
    <slot>Submit</slot>
</button>
```

在父级组件中使用 `<submit-button>` 并且不提供任何插槽内容时

```html
<submit-button></submit-button>
```

后备内容 "Submit" 将会被渲染

```html
<button type="submit">
    Submit
</button>
```

但如果提供内容

```html
<submit-button>Save</submit-button>
```

则这个内容将会被渲染，从而代替后备内容

```html
<button type="submit">
    Save
</button>
```

## 具名插槽

`<slot>` 元素有一个特殊的 attribute: `name` 。这个 attribute 可以用来定义额外的插槽

```html
<!-- base-layout 组件模板 -->
<div class="container">
	<headr>
    	<slot name="header"></slot>
    </headr> 
    <main>
        <slot></slot>
    </main>
    <footer>
        <slot name="footer"></slot>
    </footer>
</div>
```

> 一个不带 `name` 的 `<slot>` 出口会带有隐含的名字 default

在向具名插槽提供内容的时候，可以在一个 `<template>` 元素上使用 `v-slot` 指令，并以 `v-slot` 的参数的形式提供其名称

```html
<base-layout>
	<template v-slot:header>
    	<h1>
            Here might be a pag title
        </h1>
    </template>
    <template v-slot:default>
    	<p>
            A paragraph for the main content.
        </p>
        <p>
            And another one.
        </p>
    </template>
    
    <template v-slot:footer>
    	<p>
            Here's some contact info
        </p>
    </template>
</base-layout>
```

渲染的 HTML 将会是

```html
<div class="container">
    <header>
    	<h1>
            Here might be a page title
        </h1>
    </header>
    <main>
    	<p>
            A paragraph for the main content.
        </p>
        <p>
            And another one.
        </p>
    </main>
    <footer>
    	<p>
            Here's some contact info
        </p>
    </footer>
</div>
```



## 作用域插槽

有时让插槽内容能够访问子组件中才有的数据是很有用的。当一个组件被用来渲染一个项目数组时，这时一个常见的情况，我们希望能够自定义每个项目的渲染方式。

> 使 `item` 可用于父级提供的 slot 内容， 我们可以添加一个 `<slot>` 元素并将其绑定为属性

```javascript
app.comonent('todo-list', {
    data() {
        return {
            items: ['Feed cat', 'Buy milk']
        },
      	template: `
			<ul>
				<li v-for="(item, index) in items">
					<slot :item="item"></slot>
                </li>
            </ul>
		`,
    }
})
```

绑定在 `<slot>` 元素上的 attribue 被称为 **插槽 prop** ，然后在父级作用域中，使用带值的`v-slot` 来定义我们提供的插槽 prop 的名字

```html
<todo-list>
	<template v-slot:default="slotProps">
    	<i class="fas fa-check"></i>
        <span class="green">{{ slotProps.item }}</span>
    </template>
</todo-list>
```

> 在这个例子中，我们选择将包含所有插槽prop的对象命名为 slotProps ，但你也可 以使用任意你喜欢的名字。

<img src="G:\Doc\Vue3\guide\imgs\scoped-slot.png" alt="scoped-slot" style="zoom:67%;" />

### 独占默认插槽的缩写语法

在上述情况下，当被提供的内容只有默认插槽时，组件的标签才可以被当作插槽模板来使用。这样我们就可以把 `v-slot` 直接用在组件上

```html
<todo-list v-slot:default="slotProps">
	<i class="fas fa-check"></i>
    <span class="green">{{ slotProps.item }}</span>
</todo-list>
```

这种定法还可以更简单，就像末指明的内容对应该默认插槽一样，不带参数的 `v-slot` 被假定对应默认插槽

```html
<todo-list v-slot="slotProps">
	<i class="fas fa-check"></i>
    <span class="green">{{ slotProps.item }}</span>
</todo-list>
```

> 注意默认插槽的缩写语法**不能**和具名插槽混用，因为它会导致作用域不明确
>
> 只要出现多个插槽，请始终为所有的插槽使用完整的基于 `template>` 语法

```html
<todo-list>
	<template v-slot:default="slotProps">
    	<i class="fas fa-check"></i>
        <span class="green">{{ slotProps.item }}</span>
    </template>
    <template v-slot:other="otherSlotProps">
    	....
    </template>
</todo-list>
```

### 解构插槽 prop

作用域插槽的内部工作原理是将你的插槽内容包括在一个传入单个参数的函数里

```javascript
function (slotProps) {
    // ... 插槽内容
}
```

这意味着 `v-slot` 的值实际上可以是任何能够作为函数定义中的参数的 Javascript 表达式。你也可以使用 ES2015 解构来传入具体的插槽 prop

```html
<todo-list v-slot="{item}">
	<i class="fas fa-check"></i>
    <span class="green"> {{ item }}</span>
</todo-list>
```

这样可以使模板更简洁，尤其是在该插槽提供了多个 prop 的时候。它同样开启了 prop 重命名等其他可能，例 将 `item` 重命名为 `todo`

```html
<todo-list v-slot="{item : todo}">
	<i class="fas fa-check"></i>
    <span class="green"> {{ tod }}</span>
</todo-list>
```

你甚至可以定义后备内容，用于插槽 prop 是 undefined 的情形

```html
<todo-list v-slot="{ item = 'PlaceHolder' }">
	<i class="fas fa-check"></i>
    <span class="green"> {{ item }}</span>
</todo-list>
```



## 动态插槽名

动态指令参数也可以用在 `v-slot` 上

```html
<base-layout>
	<template v-slot:[dynamicSlotName]>
    	...
    </template>
</base-layout>
```



## 具名插槽的缩写

把参数之前的所有内容(`v-slot:`) 替换为字符 `#` ，例 `v-slot:header` 可以被重写为 `#header`

```html
<base-layout>
	<template #header>
        <h1>Here might be a page title</h1>
    </template>
	<template #default>
        <p>A paragraph for the main content.</p>
        <p>And another one.</p>        
    </template>
	<template #footer>
        <p>Here's some contact info</p>
    </template>    
</base-layout>
```

如果你希望使用缩写的话，就必须始终以明确的插槽名取代之

```html
<todo-list #default="{ item }">
	<i class="fas fa-check"></i>
    <span class="green">{{ item }}</span>
</todo-list>
```

