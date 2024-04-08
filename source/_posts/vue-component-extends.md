---
title: Vue Component 继承与复用
category: JavaScript
date: 2018-11-17 14:45:00
---
在做Web前端开发的时候会有大量的页面复用的地方，从UI布局到JS的逻辑。早年做后端开发的时候，我们通常可以通过面向对象的编程法式，使用抽象类、接口等等，那么现在前端是否也可以如此呢？

答案自然是肯定的，所以我们找工作面试的时候常被问及关于JS继承的问题，随之ES6出现了期盼已久的Class，一切都在往更为成熟的方向发展。接下我们以[Vue](https://vuejs.org/)为例，看看怎么去做继承这件事情。

<!-- more -->

### 需求描述

简单实现2个列表页面，一个是管理员列表、一个用户列表

![图一 管理员列表页面，筛选有用户名、状态，列表有用户名、手机号码、状态、"修改"操作按钮](/images/vue-component-extends/1.jpg)

![图二 用户页列表面有，筛选有用户名、手机号、状态，列表有用户名、手机号码、创建时间、状态、"删除"操作按钮](/images/vue-component-extends/2.jpg)

从上可以看出2个页面整体页面结构相同，在具体细节上会有些少于不同，第一反应就是使用前文提到的继承之类的东西去实现它。
![图三 简单的继承图](/images/vue-component-extends/3.jpg)

### 环境

- Vue 版本 2.5

- [Element-UI](http://element.eleme.io/) （仅仅使得Demo看上去不那么丑）

- [测试代码](https://github.com/miser/vue-compoent-extends-experiment)


### 步骤

通过vue cli工具创建项目
```javascript
vue create vue-component-extedns
```

此刻我们可以拥有一个Vue的默认开发目录结构和代码，我开始对其进行修改


__引入Element-UI__

```javascript
// main.js 
import Vue from 'vue'
import ElementUI from 'element-ui'
import 'element-ui/lib/theme-chalk/index.css'
import App from './App.vue'

Vue.config.productionTip = false
Vue.use(ElementUI)

new Vue({
  render: h => h(App)
}).$mount('#app')

```

在components目录下分别创建ListPageAbstract.vue、AdminPageAbstract.vue、ButtonClick.vue和Title.vue

*下面代码很多可以先跳过，看后续的介绍*

```html
// ButtonClick.vue
<template>
<div>
  <el-button size="small" plain type="primary" @click.stop="click">
    {{ label }}
  </el-button>
</div>
</template>
<script type="text/javascript">
  export default {
    props: ['click', 'label', 'opt'],
    mounted () {
      console.log('ButtonClick mounted')
    }
  }
</script>

// Title.vue
<template>
  <div class="title">{{ title }}</div>
</template>
<script type="text/javascript">
export default {
  props: ['title']
}
</script>

// ListPageAbstract.vue
<template>
  <div>
    <Title :title="title" />
    <div>
      <el-form v-if="config && config.filter" ref='form' :inline="true" :model="filterForm">
        <el-form-item v-if="config.filter.conditions.indexOf('name') >= 0" label="名字" prop="name">
          <el-input v-model="filterForm.name"></el-input>
        </el-form-item>
        <el-form-item v-if="config.filter.conditions.indexOf('phone') >= 0" label="手机" prop="date">
          <el-input v-model="filterForm.date"></el-input>
        </el-form-item>
        <el-form-item v-if="config.filter.conditions.indexOf('date') >= 0" label="时间" prop="date">
          <el-input v-model="filterForm.date"></el-input>
        </el-form-item>
        <el-form-item v-if="config.filter.conditions.indexOf('status') >= 0" label="状态" prop="status">
          <el-select v-model="filterForm.status" placeholder="请选择">
            <el-option v-for="option in statusOptions" :label="option.text" :value="option.value" :key="option.value"></el-option>
          </el-select>
        </el-form-item>
        <!-- <div>
        <slot name="filter-slot"></slot>
        </div> -->
        <el-form-item>
          <el-button type="primary" @click.stop="config.filter.action">提交</el-button>
        </el-form-item>
      </el-form>
    </div>
    <el-table v-if="config" :data="list" >
      <el-table-column v-for="(item, index) in config.table.column" :key="index"
      :prop="item.key" :label="item.label">
      </el-table-column>
      <el-table-column v-if="config.table.action" :label="config.table.action.headerLabel">
      <template slot-scope="scope">
      <Button :click="config.table.action.click.bind(null, scope.row)" :label="config.table.action.label" :opt="config.table.action" />
      </template>
      </el-table-column>
    </el-table>
  </div>
</template>
<script type="text/javascript">
import Button from './ButtonClick.vue'
import Title from './Title.vue'

export default {
  components: { Button, Title },
  data: function () {
    return {
      filterForm: { },
      statusOptions: [],
      list: [],
      title: null,
      config: null
    }
  },
  mounted: function () {
    this.config = this.createConfig()
    this.fetchOptions()
    this.fetchData()
  },
  methods: {
    createConfig () {
      let config = {}
      config.filter = {
        conditions: [ 'name', 'status' ],
        action: () => {
          this.fetchData(this.filterForm)
        }
      }
      config.table = {
        column: [
          { key: 'name', label: '用户名' },
          { key: 'phone', label: '手机号码' },
          { key: 'status', label: '状态' }
        ],
        action: {
          headerLabel: '操作',
          label: '修改',
          click: this.editRow
        }
      }
      return config
    },
    fetchOptions () {
      this.statusOptions = [
        { value: 1, text: 'status1' },
        { value: 2, text: 'status2' }
      ]
    },
    async fetchData () { },
    editRow (item) {
      console.log(`update data => ${item.name}`)
    }
  }
}
</script>
<style type="text/css">
.title {
  color: red;
  margin-bottom: 20px;
}
</style>

// AdminListPage.vue
<script type="text/javascript">
import ListPageAbstract from './ListPageAbstract'
// 模拟ajax请求
// 仅做了名字的模糊查询，其他参数忽略
function search (opt) {
  return new Promise((resolve) => {
    let list = [
      { name: 'Admin Peter', phone: '313141414', status: 'status1', date: '2018-10-10' },
      { name: 'Admin Marry', phone: '123931873', status: 'status2', date: '2018-11-11' },
      { name: 'Admin Sue', phone: '342391873', status: 'status1', date: '2018-01-01' },
      { name: 'Admin Join', phone: '143391873', status: 'status1', date: '2018-12-12' }
    ]
    if (opt.name) {
      list = list.filter(item => item.name.match(opt.name))
    }
    setTimeout(() => {
      resolve(list)
    }, 1000)
  })
}
export default {
  extends: ListPageAbstract,
  data () {
    return {
      title: '管理员列表'
    }
  },
  methods: {
    fetchOptions () {
      ListPageAbstract.methods.fetchOptions.call(this)
      console.log('to do other thing')
    },
    async fetchData () {
      let list = await search(this.filterForm)
      this.list = list
    }
  }
}
</script>
```

刷新页面就能呈现上述图一展示的样子和功能了。那么洋洋洒洒这么多代码做了些什么呢？

- Title.vue 用于显示页面的标题（之后我们用它测试下继承于ListPageAbstract.vue的组件如何重写css的问题）
- ButtonClick.vue 展示操作按钮和执行操作事件
- ListPageAbstract.vue 抽象的列表组件，这里是作为例子，具体方法定义的粗细程度根据具体情况调节
- AdminListPage.vue 管理员列表的具体组件

AdminListPage通过[extends](https://cn.vuejs.org/v2/api/#extends)继承了ListPageAbstract的模板、样式和其JS代码，通过部分的重写或完善，很容易的实现了一个页面，看上去很美好。那么新的问题来了，我们也发现ListPageAbstract定义筛选的内容是有限的，目前仅仅有name、phone、date和status，如果想扩展该怎么办呢？用过Vue的朋友或许此刻会想到[Slot](https://cn.vuejs.org/v2/api/#slot)，接下来我们注释掉ListPageAbstract.vue里关于__filter-slot__的注释，并为AdminListPage.vue添加相关slot代码。

```html
// AdminListPage.vue add template 
<template slot="filter-slot">
  <div slot="filter-slot">
    other input filter
  </div>
</template>
<script type="text/javascript">
```

刷新页面，页面仅仅留下了“other input filter”一串字符串，并没有实现我们的需求；也有人提出其它修改意见

![图四 除了“other input filter”其它都没了](/images/vue-component-extends/4.jpg)

```html
// AdminListPage.vue 
<template slot="filter-slot">
  <Page>
    <div slot="filter-slot">
      other input filter
    </div>
  </Page>
</template>
export default {
  extends: ListPageAbstract,
  components: {
    Page: ListPageAbstract,
  }
  // ...
}
```

虽然页面UI层是预期显示了，但是如果对ListPageAbstract的mounted方法打断点会发现，它被执行了2次，因为被实例化了2次，并且页面上的元素事件使用的是ListPageAbstract里的，而不是我们在Admin里面重写的，显然方法并不可行。关于Vue模板级别的继承扩展问题在github上有很多的吐槽，但并没有列为未来的新feature [#6811](https://github.com/vuejs/vue/issues/6811)

既然我们讨论这个问题，自然也是可以解决的，在这我们不以filter查询条件的多少为例子，我们以更为简单的按钮为例，在列表里每一行的最后有一个“修改”按钮，而然我们在UserListPage里面，我们希望它变成一个“删除”按钮，并弹出确实删除的提示。新增ButtonPop.vue和UserListPage.vue

*又是很多代码，没兴趣可跳过*

```html
// ButtonPop.vue
<template>
  <div>
    <el-button size="small" plain type="primary"
      @click.stop="dialogVisible = true">{{ label }}</el-button>
    <el-dialog
      title="提示"
      :visible.sync="dialogVisible"
      width="30%">
      <span>确认删除数据吗？</span>
      <span slot="footer" class="dialog-footer">
      <el-button @click="dialogVisible = false">取 消</el-button>
      <el-button type="primary" @click="deleteInfo">确 定</el-button>
      </span>
    </el-dialog>
  </div>
</template>
<script type="text/javascript">
export default {
  props: ['click', 'label', 'opt'],
  data () {
    return {
      dialogVisible: false
    }
  },
  mounted () {
    console.log('ButtonPop mounted')
  },
  methods: {
    async deleteInfo () {
      await this.click()
      this.dialogVisible = false
    }
  }
}
</script>

// UserListPage.vue
<script type="text/javascript">
import ListPageAbstract from './ListPageAbstract.vue'
import Button from './ButtonPop.vue'
// 模拟ajax请求
// 仅做了名字的模糊查询，其他参数忽略
function search (opt) {
  return new Promise((resolve) => {
    let list = [
      { name: 'User Peter', phone: '313141414', status: 'status1', date: '2018-10-10' },
      { name: 'User Marry', phone: '123931873', status: 'status2', date: '2018-11-11' },
      { name: 'User Sue', phone: '342391873', status: 'status1', date: '2018-01-01' },
      { name: 'User Join', phone: '143391873', status: 'status1', date: '2018-12-12' }
    ]
    if (opt.name) {
      list = list.filter(item => item.name.match(opt.name))
    }
    setTimeout(() => {
      resolve(list)
    }, 1000)
  })
}
export default {
  extends: ListPageAbstract,
  components: { Button },
  data: function () { return {} },
  mounted: function () {
    this.title = '用户列表'
  },
  methods: {
    createConfig () {
      // 用户名、创建时间、手机号、状态
      let config = ListPageAbstract.methods.createConfig.call(this)
      config.filter.conditions.splice(1, 0, 'phone')

      let table = config.table
      table.column.push({ key: 'date', label: '时间' })
      table.action = {
        headerLabel: '操作',
        label: '删除',
        click: this.deleteRow
      }
      return config
    },
    async fetchData () {
      let list = await search(this.filterForm)
      this.list = list
    },
    deleteRow (item) {
      return new Promise((resolve, reject) => {
        console.log(`delete date: ${item.name}`)
        setTimeout(() => {
          this.list.splice(this.list.indexOf(item), 1)
          resolve()
        }, 1000)
      })
    }
  }
}
</script>
<style type="text/css">
.title {
  margin-bottom: 50px;
  color: blue;
}
</style>
```

然后在修改下App.vue里面的Page引用，从AdminListPage改为UserListPage

刷新页面，就和图二的样子一样了。整个代码并不复杂，核心就是

```javascript
  components: { Button }
```

它将父类的Button（ButtonClick）替换成了User页面需要的ButtonPop，实现了扩展。其实Filter查询条件也可以，只要我们做好组件的抽取等就行。

__阅读Vue的源码时候，在Vue组件实例化的过程中，会有很多对options的深层次merge，使得我们可以通过上诉方法实现对父组件的扩展。__

另外，细心的朋友观察代码或页面也发现“用户列表”4个字的颜色从红色变成了蓝色，与下面列表的间距也增大了不少，在User页面的style标签里就能很容易修改父组件的css样式。

在Vue中，mixin、slot都是非常好用的工具，或许我们有时候也能改变思路，通过组件的替换构建出一个更为容易扩展的框架。
