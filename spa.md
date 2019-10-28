# 背景
随着项目的成长，单页spa逐渐包含了许多业务线

- 商城系统
- 售后系统
- 会员系统
...
当项目页面超过一定数量（150+）之后，会产生一系列的问题

##### 可扩展性
项目编译的时间（启动server，修改代码）越来越长，而每次调试关注的可能只是其中1、2个页面
##### 需求冲突
所有的需求都定位到当前git，需求过多导致测试环境经常排队

# 目标

1. 依然是spa

由于改善的是开发环境，当然不希望拆分项目影响用户体验。如果完全将业务线拆分成2个独立页面，那么用户在业务线之间跳转时将不再流畅，
因为所有框架以及静态资源都会在页面切换的时候重载。因此要求跳转业务线的时候依然停留在spa内部，不刷新页面，共用同一个页面入口；

2. 业务线页面不再重复加载资源
因为大部分业务线需要用到的框架(vue, vuex...), 公共组件(dialog，toast)都已经在spa入口加载过了，不希望业务线重复加载这些资源。
业务线项目中应该只包含自己独有的资源，并能使用公共资源；

3. 业务线之间资源完全共享
业务线之间应该能用router互相跳转，能访问其他业务线包括全局的store

# 实现
现在要从主项目拆分一个业务线 检验模块（examination) 出来

主项目：包含系统核心页面 + 各种必须框架，依赖（vue, vuex...）
examination项目：包含examination自己内部的业务代码

跳转examination流程
1. 用户访问业务线页面 路由 /examination；
2. 主项目router未匹配，走公共*处理；
3. 公共router判定当前路由为业务线examination路由，请求examination的入口bundle.js；
4. examination入口js执行过程中，将自身的router与store注册到主项目；
5. 注册完毕，标记当前业务线hello为已注册；
6. 之后路由调用next。会自动继续请求 /examination 对应的页面chunk（js,css）页面跳转成功；
7. 此时 examination 已经与主项目完成融合，examination 可以自由使用全部的store，使用router可以自由跳转任何页面。
8. done

第一次请求 /examination 时，此时router中所有路由无法匹配，会走公共*处理
``` javascript
// 主项目
const router = new VueRouter({
  routes: [
    {
      path: '*',
      async beforeEnter(to, from, next) {
        // 此时需要利用业务线配置，判断当前路由是否属于业务线，是的话就请求业务线，将权限判断留在后台
        let isService = await service.handle(to, from, next);
        // 非业务线页面
        if(!isService) {
          next('/');
        }
      }
    }
  ]
});


// 主项目
// 业务线接入处理
export const handle = async (to, from, next) => {
  let path = to.path || "";
  let paths = path.split('/');
  let serviceName = paths[1];

  let config = config[serviceName];

  // 非业务线路由
  if(!config) {
    return false;
  }

  // 已经加载
  if(config.loaded) {
    next();
    return true;
  }

  for(var i=0; i < config.src.length; i++) {
    await loadScript(config.src[i]);
  }
  config.loaded = true;
  next(to);  // 继续请求页面
  return true;
}

// config.js 配置在后台？
const config = {
    "examination": {
        "src": [
          "https://abcyun.cn/dist/js/examination-bundle-12352342.js"
        ]
    },
    "其他": {...}
}

```

examination 的入口entry.js

将主项目已经加载的内容 挂载到 examination，避免重复打包重复加载

``` javascript
// 主项目
import Vue from 'vue';
import GlobalRouter from 'app/router.js';
import GlobalStore from 'app/vuex/index.js';
import Vuex from 'vuex'

// 挂载子项目函数 应该由业务线调用
function registerApp(appName, {
  store,
  router
}) {

  // router store都是动态注册
  if(router) {
    globalRouter.addRoutes(router);
  }
  if(store) {
    globalStore.registerModule(appName, Object.assign(store, {
      namespaced: true
    }));
  }
}

// 主项目挂载到window上，全局一个实例
window.app = Object.assign(window.app || {}, {
  Vue,
  Vuex,
  router: globalRouter,
  store: globalStore,
  util: {
    registerApp
  }
});
```

``` javascript
// examination entry.js

/* 检验 */
import {APP_NAME} from 'utils/global'; // 业务线配置统一管理，避免重名
import store from 'store/index';

const Examination = resolve => require([/*webpackChunkName: "registrations.index"*/'../views/examination'], resolve)
const ExaminationBlank = resolve => require([/*webpackChunkName: "registrations.index"*/'../views/examination/blank'], resolve)
const ExaminationEdit = resolve => require([/*webpackChunkName: "registrations.index"*/'../views/examination/edit-form'], resolve)
let router = [{
    path: `${APP_NAME}`,
    component: Examination,
    meta: {
        name: '检验',
        moduleId: '9'
    },
    children: [
        {
            path: '',
            component: ExaminationBlank,
            name: '检验空白页',
            meta: {
                moduleId: '9'
            }
        },
        {
            path: ':id?', //path: ':id(\\d+)?',
            component: ExaminationEdit,
            name: '检验详情',
            meta: {moduleId: '9'},
        }
    ]
}]

window.app.util.registerApp(APP_NAME, {router, store});
```

# 问题：
1. webpack插件：需要打包完成后，静态资源上传到oss后更新config.js子项目src（本地调试不需要）
2. 中间件 配置管理（权限，资源）
3. 局部的相关内容会重复，utils。
4. 图片完全无法复用
5. 接口api散落进各个子项目
