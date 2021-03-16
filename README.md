# 微服务分享文档

## vue的事件循环机制

> 先提另外一个问题，上次大转盘遇到的一个问题

问题描述: 

- 点击大转盘的时候抽奖的时候展示中奖弹窗，同时会调用`init`方法重新获取页面信息(为了获取抽奖按钮列表)，弹窗出现立即点击知道了关闭中奖弹窗，会出现弹窗延迟关闭的情况

排查问题：

- 最开始以为是接口请求造成的问题，后来发现并不是，分析代码，可能造成的这种的情况就是重新获取数据的时候，更新数据时间太耗时

解决问题：

- 最终是通过减少更新的数据，只更新指定数据，同时在数据更新完成之后再弹出中奖窗

---------
> vue数据驱动视图更新，是异步的，即修改数据的当下，视图不会立刻更新，而是等同一事件循环中的所有数据变化完成之后，生成vnode，再统一进行视图更新

在这里的问题，涉及到vue的一个事件循环机制问题：

只有当事件循环队列queue1中的事件处理完成之后(这里是在dom对象更新完成，并且渲染到页面之后才会推出事件队列)，才会执行事件循环2的队列

> 一个事件队列执行完成之后(dom对象已经修改完成)才会去渲染ui

参考案例: `/vue/single-video`项目

## 前端微服务

1.新建`micro_frontend`文件夹

2.新建app1和app2子项目(vue create 项目名)

- 修改子项目配置文件

```js

const package = require('./package.json')
module.exports = {
  // 告诉子应用在这个地址加载静态资源，否则会去基座应用的域名下加载(区分下线上和测试环境)
  publicPath: process.env.VUE_PUBLICPATHP_SINGLE === 'dev' ? '/' : (process.env.VUE_PUBLICPATHP_SINGLE === 'devMicro' ? '//localhost:9091' : '/app1-static'),
  // 开发服务器
  devServer: {
    port: 9091
  },
  configureWebpack: {
    // 导出umd格式的包，在全局对象上挂载属性package.name，基座应用需要通过这个全局对象获取一些信息，比如子应用导出的生命周期函数
    output: {
      // library的值在所有子应用中需要唯一
      library: package.name,
      libraryTarget: 'umd'
    }
  }
}

```

- 安装single-spa-vue(`npm i single-spa-vue -S`)

> single-spa-vue负责为vue应用生成通用的生命周期钩子，在子应用注册到single-spa的基座应用时需要用到

- 子项目入口文件修改

```js

// /src/main.js
import Vue from 'vue'
import App from './App.vue'
import router from './router'
import singleSpaVue from 'single-spa-vue'

Vue.config.productionTip = false

const appOptions = {
  el: '#microApp', // 要挂载到主项目上到元素(在主项目要有这个元素)
  router,
  render: h => h(App)
}

// 支持应用独立运行、部署，不依赖于基座应用
if (!window.singleSpaNavigate) {
  delete appOptions.el
  new Vue(appOptions).$mount('#app')
}

// 基于基座应用，导出生命周期函数
const vueLifecycle = singleSpaVue({
  Vue,
  appOptions
})

export function bootstrap (props) {
  console.log('app1 bootstrap')
  return vueLifecycle.bootstrap(() => {})
}

export function mount (props) {
  console.log('app1 mount')
  return vueLifecycle.mount(() => {})
}

export function unmount (props) {
  console.log('app1 unmount')
  return vueLifecycle.unmount(() => {})
}


```

- 子项目环境文件配置

```js
// .env

NODE_ENV=development
VUE_APP_BASE_URL=/
VUE_PUBLICPATHP_SINGLE=dev

// .env.micro

NODE_ENV=development
VUE_APP_BASE_URL=/app1
VUE_PUBLICPATHP_SINGLE=devMicro

// .env.buildMicro
NODE_ENV=development
VUE_APP_BASE_URL=/app1
VUE_PUBLICPATHP_SINGLE=prodMicro

```

- 路由文件修改配置

```js

// /src/router/index.js
// ...
const router = new VueRouter({
  mode: 'history',
  // 通过环境变量来配置路由的 base url
  base: process.env.VUE_APP_BASE_URL,
  routes
})
// ...

```

- package.json修改

```json
"scripts": {
    // 独立运行
    "serve": "vue-cli-service serve",
    // 作为子应用运行
    "serve:micro": "vue-cli-service serve --mode micro",
    // 构建子应用
    "build": "vue-cli-service build --mode buildMicro"
  },

```

3. 新建主应用layout

- vue create layout

- 安装single-spa(`npm i single-spa -S`)

- 主项目入口更改

```js
// src/main.js
import Vue from 'vue'
import App from './App.vue'
import router from './router'
import { registerApplication, start } from 'single-spa'

import Antd from 'ant-design-vue'

import 'ant-design-vue/dist/antd.css'

Vue.use(Antd)

Vue.config.productionTip = false

// 远程加载子应用
function createScript(url) {
  return new Promise((resolve, reject) => {
    const script = document.createElement('script')
    script.src = url
    script.onload = resolve
    script.onerror = reject
    const firstScript = document.getElementsByTagName('script')[0]
    firstScript.parentNode.insertBefore(script, firstScript)
  })
}

// 记载函数，返回一个 promise
function loadApp(url, globalVar) {
  // 支持远程加载子应用
  return async () => {
    await createScript(url + '/js/chunk-vendors.js')
    await createScript(url + '/js/app.js')
    // 这里的return很重要，需要从这个全局对象中拿到子应用暴露出来的生命周期函数
    return window[globalVar]
  }
}


// 子应用列表
const apps = [
  {
    // 子应用名称
    name: 'app1',
    // 子应用加载函数，是一个promise
    app: loadApp('http://localhost:8081', 'app1'),
    // 当路由满足条件时（返回true），激活（挂载）子应用
    activeWhen: location => location.pathname.startsWith('/app1'),
    // 传递给子应用的对象
    customProps: {}
  },
  {
    name: 'app2',
    app: loadApp('http://localhost:8082', 'app2'),
    activeWhen: location => location.pathname.startsWith('/app2'),
    customProps: {}
  },
]

// 注册子应用
for (let i = apps.length - 1; i >= 0; i--) {
  registerApplication(apps[i])
}

new Vue({
  router,
  mounted() {
    // 启动
    start()
  },
  render: h => h(App)
}).$mount('#app')

```

- 主项目需要使用到子项目的位置就需要进行如下修改

```js
// app.vue
// id对应子项目中的挂载配置
<div id="microApp"></div>

// 跳转直接通过route-link进行跳转
<router-link to='/app1'></router-link>

```

- 部署，直接运行各自的`npm run build`进行部署，然后配置好nginx就能访问

```conf

server {
    listen 8089;
    server_name 127.0.0.1;

    location /app1-static {
        alias /Users/baozi/Desktop/baozi/vue/micro_frontend/app1/dist;
    }
    location /app2-static {
        alias /Users/baozi/Desktop/baozi/vue/micro_frontend/app2/dist;
    }
    location / {
        root   /Users/baozi/Desktop/baozi/vue/micro_frontend/layout/dist;
        index  index.html index.htm review.html;
        try_files $uri /index.html;
    }
}

server {
    listen 8081;
    server_name 127.0.0.1;

    location / {
        root   /Users/baozi/Desktop/baozi/vue/micro_frontend/app1/dist;
        index  index.html index.htm review.html;
        try_files $uri /index.html;
    }
}


```
