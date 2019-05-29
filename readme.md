# luck7-ajax

可以拥有统一配置的项目层面的对ajax的再封装，使用简洁，可以同时存在多个

```javascript
// 创建ajax对象
const ajax = new Ajax(apiLists, commonConfigs, ajaxConfigs, apiMethods)
// 使用ajax对象发起请求
ajax.do(apiName, params) // 没有请求参数的时候，params可以不写
  .then(res => {
    console.log(res)
  }).catch(err => {
    console.error(err)
  })
```

## 参数

 * apiLists

  Array类型，api接口列表，格式为```key: {method, url, options}```，支持简写

 * commonConfigs

  Object类型，对ajax请求的配置(如headers、timeout等)，来自flyio框架，参考``请求配置项``，默认值如下
  ```javascript
  {
    method: 'POST',
    parseJson: true
  }
  ```

 * ajaxConfigs

  Object类型，框架专有的配置，用于对请求/返回数据进行处理，支持如下四个参数
  ```javascript
  {
    // publicParams: Function, // 公共参数，已移除，建议通过instance的拦截实现
    lang: Object, // 错误提示文本
    isStrict: true // 是否开启严格模式
  }
  ```

  错误提示文本默认值
  ```javascript
  {
    noList: '配置错误: 缺少接口配置(list)',
    methodError: '配置错误: #apiName#请求类型异常#method#',
    urlError: '配置错误: #apiName#缺少请求地址(url)',
    typeError: '使用错误: 接口参数类型错误',
    noConfig: '使用错误: 接口#apiName#未配置',
    netError: '服务器错误: 状态码：#status#'
  }
  ```
 * apiMethods

 跟``apiLists``中的key对应，代表相应的api请求后的处理方法，格式如下

 
  ```javascript
  {
    key () { // key的唯一特例是'_',代表通用处理方法,将对所有未配置method的api生效
      return {
        request (req) {return {}}, // 对request参数进行处理，返回处理后的请求参数
        response (res) {return {}}, // 对返回数据进行处理，返回处理后的数据(也可以在这儿交由vuex处理，返回空对象)
        error (err) {} // 对异常进行处理
      }
    }
  }
  ```
## 请求配置项

默认使用flyio，跟flyio的请求配置完全一致，此时支持如下配置选项

```javascript
{
  headers: {}, // http请求头，
  baseURL: "", // 请求基地址
  timeout: 0, // 超时时间，为0时则无超时限制
  // 是否自动将Content-Type为“application/json”的响应数据转化为JSON对象，默认为true    
  parseJson: true,
  params: {}, // 默认公共的url get参数     
  withCredentials: false // 跨域时是否发送cookie
}
```
 - 请求配置项支持**实例级配置**和**单次请求配置**

~~使用axios时，将支持所有axios支持的配置，详情参考[axios](https://github.com/mzabriskie/axios) 的"Request Config"~~

## 项目使用实例

```javascript
// src/config/ajax
import store from '@/store'
const list = {
  demo1: {
    url: '/mock1'
  },
  // 可以简写为 demo1: '/mock1'
  // 使用请求超时
  demo2: {
    method: 'GET', // 默认值为POST
    url: '/mock2', // 请求地址，必填且不能为空
    data: {}, // 请求参数，可以进行一些特殊写法，如写成'vuex.xxxx'，然后在methods中处理成对应的vuex数据
    options: { // 请求配置项
      timeout: 1000 // 请求超时时间
    }
  },
  // 使用URL参数（:path）
  demo3: '/:id/mock3',
  login: '/login',
}
const baseURL = '/mock'
const commonConfigs = { // ajax配置，详见axios、flyio，默认使用flyio
  // method: 'POST', // 框架默认值
  // interceptors: fun // 详见axios/flyip的interceptors说明
  baseURL
}
const ajaxConfigs = {
  // lang
}
const methods = {
  _ () {
  // _代表对所有的过滤，用于实现对request和respone的拦截
  // _跟其他的method配置(如login)不能同时生效
  // 为什么是function？为了实现一些情况下request和response共用变量，并且每次请求该变量都会变化的情况，如对数据进行加解密的情况
    return {
      request (data) {
        const userid = sessionStorage.getItem('userid')
        const token = sessionStorage.getItem('token')
        const backData = {userid, token}
        for (const k in data) {
          let v = data[k]
          if (typeof v === 'string' && v.substr(0, 5) === 'vuex.') { // 处理vuex数据
            const path = v.substr(5).split('.')
            v = path.reduce((acc, cur) => acc[cur], store.state)
          }
          backData[k] = v
        }
        return backData
      },
      response (res) {
        if (res.code !== 0) {
          throw new Error(res.msg)
        } else return res.data
      }
    }
  },
  login () {
    return {
      response (res) {
        store.commit('login/LOGIN_DATA', res)
      },
      error (err) {
        store.commit('ERR_MSG', '登录失败，错误详情：' + err.message)
      }
    }
  }
}
export {list, commonConfigs, ajaxConfigs, methods}

// src/utils/ajax.js
import isFunction from 'lodash/isFunction'
import Ajax from 'luck7-ajax'
import {list, commonConfigs, ajaxConfigs} from '../config/ajax'

const _ajax = new Ajax(list, commonConfigs, ajaxConfigs, methods)

const ajax = function (apiName, params) {
  return _ajax.do(apiName, params)
}
export default ajax

// main.js
ajax('demo1', {
  pageNo: 1,
  size: 10
}).then((data) => {
  console.log(data)
}).catch((e) => {
  console.error(e.message)
})

ajax(['demo2', 'demo3'], []) // 第二个数组不能省略
  .then((datas) => {
    console.log(datas[0], datas[1])
  }).catch((err) => {
    console.error(err.message)
  })
/*
上例等价于
Promise.all([
  ajax('demo2'),
  ajax('demo3')
]).then((datas) => {
  console.log(datas[0], datas[1])
}).catch((err) => {
  console.error(err.message)
})
*/

ajax(
  '/login', // 注意：这种使用方式在严格模式下会报错，只能在非严格模式下使用
  {
    username: 'admin',
    password: 'admin'
  }
).then((data) => {
  console.log(data)
}).catch((e) => {
  console.error(e.message)
})
```
