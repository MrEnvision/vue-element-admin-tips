# 管理系统开发 Tips（Vue 项目）

> [vue-element-admin](https://github.com/PanJiaChen/vue-element-admin) 是基于 Vue 和 ElementUI 的后台前端解决方案，[简体中文](./README.zh-CN.md) | [English](./README.md) ，本文主要基于该项目总结「后台管理类系统」的开发过程中的一些技巧和值得学习的思想。

<p align="center">
  <img width="900" src="https://wpimg.wallstcn.com/a5894c1b-f6af-456e-82df-1151da0839bf.png">
</p>

# 1. 代码规范化配置

本项目主要基于 Eslint + husky + lint-staged 进行代码配置，推荐阅读 👉[「Eslint + Prettier + husky + lint-staged 前端代码规范」](https://github.com/MrEnvision/Front-end_learning_project/tree/master/coding_guide_setting)。

# 2. 自动化引入文件

如下代码所示，Vuex module 手动一个个去引入相应的文件比较麻烦：

```js
import moduleOne from './modules/moduleOne'
import moduleTwo from './modules/moduleTwo'
import moduleThree from './modules/moduleThree'

export default new Vuex.Store({
  modules: {
    moduleOne,
    moduleTwo,
    moduleThree
  }
})
```

现在通过自动搜索文件的方式来自动化引入 Vuex module，从而不需要再手动一个个去引入相应的文件：

```js
// 通过自动化搜索文件来引入 Vuex module
const modulesFiles = require.context('./modules', true, /\.js$/)
const modules = modulesFiles.keys().reduce((modules, modulePath) => {
  const moduleName = modulePath.replace(/^\.\/(.*)\.\w+$/, '$1') // set './app.js' => 'app'
  const value = modulesFiles(modulePath)
  modules[moduleName] = value.default
  return modules
}, {})

export default new Vuex.Store({
  modules,
})
```

# 3. Axios 全局拦截

```js
// utils/request.js
import axios from 'axios'
// 直接利用 Message 组件显示请求返回信息
import { Message } from 'element-ui'

const service = axios.create({
  baseURL: process.env.VUE_APP_BASE_API, // 真实请求 url = baseURL + requestURL
  timeout: 5000, // request timeout
})

// request 拦截器
service.interceptors.request.use(
  (config) => {
    // do something before request is sent
    // 例如，请求头携带Token等，config.headers["Token"] = 'XXX';
    return config
  },
  (error) => {
    // do something with request error
    console.log(error) // for debug
    return Promise.reject(error)
  }
)

// response 拦截器
service.interceptors.response.use(
  (response) => {
    const res = response.data
    // 以下判断仅供参考，视后端返回情况而定
    if (res.code !== 20000) {
      // code 不等于视为存在错误
      Message({
        message: res.message || 'Error',
        type: 'error',
      })
      return Promise.reject(new Error(res.message || 'Error'))
    } else {
      return response
    }
  },
  (error) => {
    // do something with response error
    console.log('error' + error) // for debug
    Message({
      message: error.message,
      type: 'error',
    })
    return Promise.reject(error)
  }
)

export default service
```

# 4. 路由守卫*

```js
// permission.js  路由守卫
import router from './router'
import { getToken } from '@/utils/auth' // get token from cookie

// 设置白名单
const whiteList = ['/login', '/auth-redirect'] // no redirect whitelist

router.beforeEach(async (to, from, next) => {
  // 1. 通过 token 判断用户是否登录
  const hasToken = getToken()
  if (hasToken) {
    // 2. 用户已登录
    if (to.path === '/login') {
      // 2.1 用户已登录且当前访问的是登录页面，则跳转至首页
      next({ path: '/' })
    } else {
      // 2.2 用户已登录且当前访问的不是登录页面，则 go directly
      // 此处还可以判断用户是否获得了他的权限角色，如果未获取则需要进行获取并动态生成路由
      next()
    }
  } else {
    // 3. 用户未登录（无token）
    if (whiteList.indexOf(to.path) !== -1) {
      // 3.1 用户未登录但访问的是白名单路径，则 go directly
      next()
    } else {
      // 3.2 用户未登录且访问的不是白名单路径，则转至登录页面（是否带有 redirect 信息视需求而定）
      next(`/login?redirect=${to.path}`)
    }
  }
})
```

# 5. 动态路由

路由根据权限可以分为基础路由和动态路由：

```js
// router.index.js
import Vue from 'vue'
import VueRouter from 'vue-router'

Vue.use(VueRouter)

/* constantRoutes: 基础路由，不需要考虑权限，所有用户角色都能访问 */
export const constantRoutes = []

/* asyncRoutes: 根据用户角色动态加载的路由 */
export const asyncRoutes = []

const router = new VueRouter({
  mode: 'history',
  base: process.env.BASE_URL,
  routes: constantRoutes,
})

export default router
```

如上述代码所示，默认情况下只有`constantRoutes`，动态路由的话需要用户在第一次登录进入系统的时候需要可以角色进行生成：

```js
// permission.js  路由守卫
import router from './router'
import { getToken } from '@/utils/auth' // get token from cookie
import { constantRoutes } from '@/router'

const whiteList = ['/login', '/auth-redirect'] // no redirect whitelist

router.beforeEach(async (to, from, next) => {
  const hasToken = getToken()
  if (hasToken) {
    if (to.path === '/login') {
      next({ path: '/' })
    } else {
      // 2.2 用户已登录且当前访问的不是登录页面，则 go directly
      // 此处还可以判断用户是否获得了他的权限角色，如果未获取则需要进行获取并动态生成路由(一般是第一次的时候)

      // 如果已经获取权限并且生成了动态路由则会在 vuex 中保存，此时就可以通过该字段来判断是否已执行权限获取
      const hasRoles = store.getters.roles && store.getters.roles.length > 0
      if (hasRoles) {
        // 2.2.1 已经获取过权限且生成动态路由了则 go directly
        next()
      } else {
        // 2.2.1 未获取用户权限和生成动态路由
        try {
          // get user info
          // note: roles must be a object array! such as: ['admin'] or ,['developer','editor']
          const { roles } = await store.dispatch('user/getInfo')

          // 动态分配路由 generate accessible routes map based on roles
          const accessRoutes = await store.dispatch('permission/generateRoutes', roles)

          // 添加动态路由 dynamically add accessible routes
          // router.options.routes = [...constantRoutes, accessRoutes];
          router.addRoutes(accessRoutes)

          // hack method to ensure that addRoutes is complete
          // set the replace: true, so the navigation will not leave a history record
          next({ ...to, replace: true })
        } catch (error) {
          // remove token and go to login page to re-login
          await store.dispatch('user/resetToken')
          next(`/login?redirect=${to.path}`)
        }
      }
    }
  } else {
    if (whiteList.indexOf(to.path) !== -1) {
      next()
    } else {
      next(`/login?redirect=${to.path}`)
    }
  }
})
```

路由中 meta 字段设置相应的 role 内容，其表示哪些用户角色允许访问该路由，从而能够根据用户角色动态更新路由：

```js
// 动态分配路由，对应 store.dispatch('permission/generateRoutes', roles)
import { asyncRoutes, constantRoutes } from '@/router'

function hasPermission(roles, route) {
  if (route.meta && route.meta.roles) {
    return roles.some(role => route.meta.roles.includes(role))
  } else {
    return true
  }
}

function filterAsyncRoutes(routes, roles) {
  const res = []

  routes.forEach(route => {
    const tmp = { ...route }
    if (hasPermission(roles, tmp)) {
      if (tmp.children) {
        tmp.children = filterAsyncRoutes(tmp.children, roles)
      }
      res.push(tmp)
    }
  })

  return res
}

const actions = {
  // 按角色分配路由权限
  generateRoutes(content, roles) {
    return new Promise(resolve => {
      let accessedRoutes
      if (roles.includes('admin')) {
        // 超级管理员
        accessedRoutes = asyncRoutes || []
      } else {
        // 普通管理员
        accessedRoutes = filterAsyncRoutes(asyncRoutes, roles)
      }
      // 更新路由
      content.commit('SET_ROUTES', accessedRoutes)
      resolve(accessedRoutes)
    })
  }
}
```

# 6. Mock 请求数据

本项目的数据均通过 [mock.js](https://github.com/nuysoft/Mock) 生成，推荐阅读：[「vue项目中mock.js的使用」](https://juejin.cn/post/6844903847660371982)，尤其关注一下几个方面的使用：

- Mock.mock( rurl, template )：当拦截到匹配 `rurl` 的 Ajax 请求时，将根据数据模板 `template` 生成模拟数据返回。
- Mock.mock( rurl, function( options ) )：当拦截到匹配 `rurl` 的 Ajax 请求时，函数 `function(options)` 执行并把结果返回。

# 7. 全局 Svg Icon 图标组件

推荐阅读：[「手摸手，带你优雅的使用 icon」](https://juejin.cn/post/6844903517564436493)，具体应用可详见项目：[「Vue3.0项目-简易后台管理系统」](https://github.com/MrEnvision/vue-admin#171-svg文件)。

```js
// ./icon.js，可将其引入 main.js 全局注册 svg-icon 组件
import Vue from 'vue'
import SvgIcon from '@/components/SvgIcon'// svg component

// register globally
Vue.component('svg-icon', SvgIcon)

// 解析 svg 文件自动导入，只需要把文件放在固定的搜索路径
const req = require.context('./svg', false, /\.svg$/)
const requireAll = requireContext => requireContext.keys().map(requireContext)
requireAll(req)
```

```vue
<template>
  <svg :class="svgClass" aria-hidden="true" v-on="$listeners">
    <use :xlink:href="iconName" />
  </svg>
</template>

<script>
export default {
  name: 'SvgIcon',
  props: {
    iconClass: {
      type: String,
      required: true
    },
    className: {
      type: String,
      default: ''
    }
  },
  computed: {
    iconName() {
      return `#icon-${this.iconClass}`
    },
    svgClass() {
      if (this.className) {
        return 'svg-icon ' + this.className
      } else {
        return 'svg-icon'
      }
    }
  }
}
</script>

<style scoped>
.svg-icon {
  width: 1em;
  height: 1em;
  vertical-align: -0.15em;
  fill: currentColor;
  overflow: hidden;
}
</style>
```

除此之外还需要 svg-sprite-loader 这个 webpack loader 来将所有 svg 打包成 svg-sprite，安装 svg-sprite-loader，在 vue.config.js 中配置具体详见[「vue.config.js 文件」](./vue.config.js) 或 [「Vue3.0项目-简易后台管理系统」](https://github.com/MrEnvision/vue-admin#171-svg文件)。

# 8. 模块化思想

对于大型项目来说一定要有模块化思想，不要把一堆东西都写在一起，例如本项目中 Vuex 和路由都采用了模块化进行划分，详见代码。



------

项目内容有错误或存在侵权，请提交 issues 进行指正，合作请邮件 <a href="mailto:EnvisionShen@gmail.com">EnvisionShen@gmail.com </a>联系。
