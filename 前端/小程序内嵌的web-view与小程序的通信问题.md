# 小程序内嵌的 web-view 与小程序的通信问题

## 项目背景

小程序内嵌的 web-view 页面需要跳转到小程序页面

## 解决方案

1. 引入 [微信SDK](https://res.wx.qq.com/open/js/jweixin-1.6.0.js) 脚本文件
2. 在 `TypeScript` 文件中声明 `wx` 全局变量
   * 使用 `declare` 关键字声明了 `wx` 全局变量 这将告诉 `TypeScript` 编译器这个变量已经存在，可以在其他地方使用 由于 `wx` 对象是一个第三方库提供的全局变量，我们无法使用类型声明文件来描述其类型，因此我们将其类型声明为 `any`
3. 使用示例：

   ```html
    <template>
      <div>
        <button @click="handleJumpApplet">JumpApplet</button>
      </div>
    </template>
    <script lang="ts">
      import { defineComponent } from 'vue';
      export default defineComponent({
        setup() {
          function handleJumpApplet() {
            if (typeof wx !== undefined) {
              wx.miniProgram.navigateTo({ url: "/pages/mobile/view/index" });
              }
            }
          return {
            handleJumpApplet
          }
        }
      });
    </script>
   ```

## 由此引发小程序与 web-view 通信问题

### 通信方案一：URL 参数传递，web-view 的 src 属性(小程序向 web-view 传递消息)

* 通过在 web-view 标签的 src 属性中添加参数，从而将参数传递给 web-view 页面 在 web-view 页面中可以通过解析 URL 参数获取传递的数据
* 使用示例：
    小程序端：

    ```html
       <web-view src="https://example.com?param1=value1&param2=value2"></web-view>
    ```

    web-view 页面 解析路由参数即可获得参数

    ```js
      window.location.search
      // 即可获得 ? 后的所有参数，对其进行解析即可获得参数小程序传递给 web-view 的参数
    ```

1. 优点：
   1. 简单易用：设置 src 属性携带参数的方式比较简单，适合小规模的数据传输
2. 缺点：
   1. 传输数据量有限：由于 URL 长度的限制，传输的数据量有限，无法传输大量数据
   2. 不够灵活：src 属性携带参数的方式只能传输简单的数据类型，例如字符串、数字等，无法传输复杂的数据类型，例如对象、函数等
   3. 安全性风险：URL 参数传递通信的安全性依赖于参数的加密和签名等措施，如果这些措施不够严密，就可能存在安全漏洞，被攻击者利用进行攻击，因为 URL 参数传递通信中的参数是明文传递的，攻击者可以通过截取网络数据包的方式，获取参数的信息，进而了解通信双方的业务逻辑和敏感信息

### 通信方案二：wx.miniProgram.postMessage({xxx:xxx}) 和 bindmessage 属性（web-view 向小程序传递消息）

* web-view 将信息发送给小程序，由小程序内部监听信息，监听到信息后由小程序主动做出响应
* 使用示例：

  小程序端：

  ```html
    <web-view src="https://example.com" bindmessage="onMessage"></web-view>
  ```

  ```ts
    onMessage: function(event) {
      console.log(event.data); // "这是一条测试示例数据"
    }
  ```

  web-view 页面：

  ```html
    <template>
      <div>
        <button @click="handlePostMessageToApplet">postMessageToApplet</button>
      </div>
    </template>
    <script lang="ts">
        import { defineComponent } from 'vue';
        export default defineComponent({
          setup() {
            function handlePostMessageToApplet() {
              if (typeof wx !== undefined) {
                  wx.miniProgram.postMessage({ data: "这是一条测试示例数据" });
                 /**
                  * 注意：不能使用 window.postMessage 触发 web-view 的 bindmessage
                  * 因为：
                  * 1. window.postMessage 方法可以用于在跨域的 iframe 或者跨域的窗口之间传递消息，且它是 HTML5 的 API
                  * 2. web-view 网页与小程序之间不支持除 JSSDK 提供的接口之外的通信
                  */
                }
              }
            return {
              handleJumpApplet
            }
          }
        });
    </script>
  ```

1. 优点：
   1. 灵活性：可以自由定义传递的数据内容和格式，通信双方可以根据自己的需要进行自由的数据传递和解析
   2. 效率高：使用 wx.miniProgram.postMessage({xxx:xxx}) 和 bindmessage 属性进行通信时，数据传输的效率相对较高，通信双方可以实现高效的数据传递和响应，同时也可以避免 URL 参数传递通信中的参数长度限制的问题
2. 缺点：
   1. 使用 wx.miniProgram.postMessage 方法需要进行一些复杂的配置，需要在微信公众号中进行认证等操作
   2. 为什么需要考虑 web-view 页面和小程序之间的安全性问题？
      答：web-view 因为它们之间的通信机制存在安全漏洞，攻击者可以利用这些漏洞来获取用户的敏感信息
         1. 原理：web-view 页面和小程序之间的通信机制是通过 postMessage 方法实现的。攻击者可以利用 postMessage 方法的特性，伪造消息发送者的身份，并向小程序发送恶意消息。如果小程序没有对接收到的消息进行过滤和验证，就可能导致信息泄漏的风险
         2. 例如：攻击者可以伪造 web-view 页面发送的消息，将用户的个人信息、密码等敏感信息发送给小程序。如果小程序没有对接收到的消息进行验证，就可能将这些敏感信息保存到本地或发送到其它服务器，从而导致信息泄漏
         3. 解决方案：在小程序中对接收到的消息进行过滤和验证，确保只接受来自合法来源的消息，并防止恶意消息对小程序造成影响。同时，在 web-view 页面中发送消息时，也需要确保发送的消息合法，并对消息进行加密和签名等操作，以确保发送的消息只能被合法的接收方解密和验证
   3. wx.miniProgram.postMessage 方法为什么可能会增加 web-view 页面的加载时间？
      答：当使用 wx.miniProgram.postMessage 方法向 web-view 组件发送消息时，需要经过以下几个步骤：
        1. 调用 wx.miniProgram.postMessage 方法将消息发送给 web-view 组件
        2. web-view 组件的 `JavaScript` 上下文接收到消息后，将消息转发给 webview 进程
        3. webview 进程接收到消息后，将消息转发给浏览器进程
        4. 浏览器进程接收到消息后，将消息传递给 web-view 组件的 `JavaScript` 上下文，触发 message 事件
      * 综上可得：消息需要经过多个进程之间的传递，所以可能会增加 web-view 页面的加载时间。但是，具体是否会增加页面加载时间，取决于消息的大小和网络延迟等因素。如果消息很小且网络延迟较低，那么增加的时间非常短暂。但如果消息较大或者网络延迟较高，那么增加的时间可能会比较明显
      * 注意：在小程序中，web-view 组件是通过浏览器内核来渲染网页内容的，因此也需要涉及到浏览器进程和渲染进程

## 知识不迷路

1. web-view（非组件容器） 承载网页的容器 会自动铺满整个小程序页面
2. 小程序在调试模式，在右上角的“详情”中“本地设置"钩上“不校验合法域名、web-view（业务域名）、`TLS` 版本以及 `HTTPS` 证书”这个选项
3. web-view 网页与小程序之间不支持除 `JS-SDK` 提供的接口之外的通信
4. 网页内 `iframe` 的域名也需要配置到域名白名单

## 相关文档

* [web-view 文档](https://developers.weixin.qq.com/miniprogram/dev/component/web-view.html)
* 拓展：[第三方静态网站 H5 跳转小程序](https://developers.weixin.qq.com/miniprogram/dev/wxcloud/guide/staticstorage/jump-miniprogram.html)
