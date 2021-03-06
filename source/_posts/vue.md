---
title: 从零开始学习Vue
date: 2017-03-09 10:36

categories:
  - 技术

tags:
  - vue
---

## 搭建一个 Vue 项目 (node >= 6)

ElementUI 是一个基于 Vue 的组件库，并提供了一个[Vue 项目模板][1]

```
npm install
npm run dev //ElementUI使用的是webpack2.0，所有本地编写代码直接运行 npm run dev 默认在http://localhost:8010上
npm run build //若需发布编译打包以后的代码直接运行 npm run build
```

可以根据自己的习惯修改模板中文件的结构，在这里不再赘述

webpack2.0 还有一个新功能就是热加载，不需要刷新页面就可以看到你修改后的效果

本次使用的是 Intellig IDEA，并下载了 Vue.js 插件

## vue-devtools

推荐安装 vue-devtools,可以清楚的看清 vue 的整体结构

[在 Chrome 网上应用店中获取(需要翻墙)][2]

## vue-router 的安装使用

```
npm install vue-router --save
```

下面是一个 route 的例子，Test.vue 是一个简单的测试 route 的组件

```
/**
 * Created by hldev on 17-3-8.
 */
import Vue from 'vue'
import VueRouter from 'vue-router'
import Todo from '../components/todo/Todo.vue'
import Hello from '../components/Hello.vue'

//注册插件
Vue.use(VueRouter);

const routes = [{
    path: "/",
    component: Hello
}, {
    path: '/todo',
    component: Todo,
    children: [
        {
            path: 'baidu',
            component: Hello,
            children: [{
                path: 'news',
                component: Hello,
            }]
        },
        {
            path: 'weibo',
            component: Hello
        }
    ]
}];

const router = new VueRouter({
    mode: 'history',
    routes
});

export default router;
```

然后在入口文件中引用 router

```
import Vue from 'vue'
import VueRouter from 'vue-router'
import ElementUI from 'element-ui'
import 'element-ui/lib/theme-default/index.css'
import router from '../routes/route'
import Menu from '../components/home/Menu.vue'

//注册插件
Vue.use(ElementUI);
Vue.use(VueRouter);

new Vue({
    el: '#root',
    router,
    render: h => h(Menu)
});
```

vue-router 还可以监听\$route，只要它发生变化你可以在 method 里面写一个函数来触发

```
{
  ...
  watch: {
              '$route.query': 'getData'
      },
  methods: {
      getData(){
          this.$store.dispatch('findPage', {...this.$route.query})
          }
      }
  ...
}

```

## 组件

下面是一个简单的组件 FirstComponent.vue

```
//这是一个.vue格式的文件
<template>
    <div>
        <h1>{{msg}}</h1>
    </div>
</template>

<script>
    export default {
        data() {
            return {msg: this.$route.fullPath}
        },
        mounted(){
            console.log(this.$route.fullPath);
        }
    }
</script>
```

在其它组件中引入组件

```
<template>
    <div>
        <p>{{ msg }}</p>
        <first-component></first-component>
        <router-view></router-view>//是路由变化展示的地方
    </div>
</template>

<script>
    import FirstComponent from '../home/FirstComponent.vue.vue';
    export default {
        data() {
            msg: "组件的引用"
        },
        //注册组件
        components: {'first-component': FirstComponent}
        }
    }
</script>
```

### [组件的生命周期][3]

下面是一个写的一个运用生命周期的简单引用

```
<template>
    <div>
        <h1>{{msg}}</h1>
    </div>
</template>

<script>
    export default {
        data() {
            return {msg: this.$route.fullPath}
        },
        mounted(){
            console.log(`mounted`);
        },
        beforeDestroy(){
            console.log(`beforeDestroy`);
        },
    }
</script>
```

### 组件之间的通信

父组件

```$xslt
<template>
    <div>
        <h1>{{msg}}</h1>
        <el-input v-model="input" placeholder="请输入您要传给子组件的内容"></el-input>
        <h2>{{total}}</h2>
        <p>子组件</p>
        <child :message="input" @increment="incrementTotal"></child>
    </div>
</template>

<script>
    import Child from './Child.vue'
    export default {
        data() {
            return {msg: this.$route.fullPath, input: '', total: 0}
        },
        methods: {
            incrementTotal: function () {
                this.total += 1
            }
        },
        components: {'child': Child},
        mounted(){
            console.log(`mounted`);
        },
        beforeDestroy(){
            console.log(`beforeDestroy`);
        },
    }
</script>
```

子组件

```$xslt
<template>
    <div>
        <p>{{message}}</p>
        <el-button @click="increment">+1</el-button>
    </div>
</template>
<script>
    export default {
        props: ['message'],
        data() {
            return {input: ``,count: 0}
        },
        methods: {
            increment: function () {
                this.count += 1;
                //$emit(eventName) 触发事件
                this.$emit('increment')
            }
        }
    }
</script>
```

#### 父组件向子组件通信

- 使用 Prop 传递数据

组件实例的作用域是孤立的。这意味着不能并且不应该在子组件的模板内直接引用父组件的数据。可以使用 props 把数据传给子组件。 prop 是父组件用来传递数据的一个自定义属性。子组件应显式地用 props

也可以用 v-bind 动态绑定 props 的值到父组件的数据中。每当父组件的数据变化时，该变化也会传导给子组件： prop 是单向绑定的：只能从父组件传向子组件，prop 还可以验证

#### 子组件向父组件通信

- 父组件是使用 props 传递数据给子组件，子组件要把数据传递回去，应该自定义事件

如上面的 demo，子组件监听 button 的 click 事件并执行 increment 方法(@:click='increment')后触发 increment 事件父组件中监听到 increment 事件触发 incrementTotal 方法(@increment="incrementTotal")

## Vuex 的安装和使用

```
npm install vuex --save
```

Vuex 概念与 Redux 相似，一个 store 主要由 state，mutations,actions.getters 组成

大概流程是我们在程序里使用 dispatch 分发 action(可异步),action 提交(commit)一个 mutation(不能异步)进而改变 state 具体写法求看[vuex 文档][4]

如果项目太大，state 比较多就可以使用 Moludes，其实就是很多个小的 store 再在根节点上

```
import Vue from 'vue';
import Vuex from 'vuex';

Vue.use(Vuex);

export default new Vuex.Store({
    modules: {
        ...//引入你分好的小模块store
    },
})
```

## Vue 的 js 写法(render 函数)

```
import Vue from 'vue'

Vue.component('todo', {
    mounted(){
    },
    render: function (createElement) {
        return createElement(
            'div',
            {},
            [createElement(
                'h1',
                {
                    attrs: {
                        id: 'test'
                    },
                },
                'hello'
            ), createElement(
                'button',
                {
                },
                '+1'
            )]
        )
    },
});
```

[1]: https://github.com/ElementUI/element-starter
[2]: https://chrome.google.com/webstore/detail/vuejs-devtools/nhdogjmejiglipccpnnnanhbledajbpd
[3]: https://cn.vuejs.org/v2/api/#选项-生命周期钩子
[4]: https://vuex.vuejs.org/zh-cn/
