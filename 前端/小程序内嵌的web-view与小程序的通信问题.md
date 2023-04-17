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
   1. 方便：设置 src 属性携带参数的方式比较简单，适合小规模的数据传输
   2. 兼容性好：可以支持绝大部分的浏览器和系统环境，无需考虑兼容性问题
   3. 安全：参数可以进行加密处理，增加信息安全性
2. 缺点：
   1. 传输数据量有限：由于 URL 长度的限制，传输的数据量有限，无法传输大量数据
   2. 不够灵活：src 属性携带参数的方式只能传输简单的数据类型，例如字符串、数字等，无法传输复杂的数据类型，例如对象、函数等
   3. 不够实时：src 属性携带参数的方式只有在页面加载时才会传输数据，无法实现实时通信

### 通信方案二：wx.miniProgram.postMessage({xxx:xxx})和 bindmessage 属性（web-view 向小程序传递消息）

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
   1. 实现数据传递
   2. 实现交互功能：通过向内嵌网页传递数据，可以实现小程序页面和内嵌网页之间的交互功能，比如在内嵌网页中调用小程序 API，或者在小程序页面中执行内嵌网页中的 `JavaScript` 代码
   3. 提高用户体验：使用内嵌网页可以提供更丰富的功能和交互方式，从而提高小程序的用户体验
2. 缺点：
   1. 使用 wx.miniProgram.postMessage 方法需要进行一些复杂的配置，需要在微信公众号中进行认证等操作
   2. web-view 的性能不如原生应用，可能会出现卡顿等问题
      1. 渲染性能问题：Web 技术本质上是单线程的，在渲染大量复杂的页面时可能会出现卡顿 此外，web-view 还需要将 `HTML` 、 `CSS` 和 `JavaScript` 解析成可执行的代码，这也会占用一定的时间 其中渲染线程可能会影响 `JavaScript` 的执行
      2. 网络延迟：web-view 需要从远程服务器加载 Web 内容，如果网络延迟较高，可能会导致页面加载缓慢或者出现卡顿
      3. 内存占用：web-view 所加载的页面可能包含大量的图片、视频等资源，这些资源会占用大量的内存 当内存占用过高时，可能会导致系统频繁地进行内存回收，从而引起卡顿
   3. 需要考虑 web-view 页面和小程序之间的安全性问题
      1. 信息泄露：web-view 页面和小程序之间共享了一些系统资源，比如本地存储、`Cookie` 等 如果没有进行适当的安全控制，就可能导致敏感信息泄露
      2. `XSS` 攻击：web-view 页面和小程序都支持 `JavaScript`，攻击者可能会在 web-view 页面中注入恶意的 `JavaScript` 代码，从而获取用户的敏感信息或者执行一些恶意操作
      3. `CSRF` 攻击：web-view 页面和小程序都可能存在 `CSRF` 攻击的风险，攻击者可以通过某些方式诱骗用户在 web-view 页面中发起某些操作，从而执行恶意操作
   4. wx.miniProgram.postMessage 方法可能会增加 web-view 页面的加载时间，影响用户体验
      1. web-view 是在小程序启动时才会初始化的 因此，在 web-view 页面加载过程中，如果需要与小程序进行通信，就需要等待 web-view 初始化完成后才能进行通信，从而增加了 web-view 页面的加载时间

## 在 web-view 页面 调用 wx.miniProgram.xxx

* 用于一些比较简单的场景

1. 调用的 API 有限
2. 使用微信 `JS-SDK` 中提供的一些 API，如分享、跳转等
3. 优点：
   1. wx.miniProgram.xxx 方法可以方便地实现在 web-view 页面中调用小程序提供的 API，如分享、支付等，增加了 web-view 页面的功能性
   2. wx.miniProgram.xxx 方法在使用过程中不需要进行微信公众号认证等复杂的配置，相对较为简单
   3. wx.miniProgram.xxx 方法的调用效率相对较高，因为小程序和 web-view 页面都是在微信客户端中运行的，所以可以直接进行通信
4. 缺点：
   1. 在 web-view 页面中直接使用 wx.miniProgram.xxx 方法需要用户已经安装了对应的小程序，否则将无法执行
   2. 由于小程序和 web-view 页面之间的通信是在微信客户端中进行的，所以可能会受到微信客户端版本的限制，需要进行兼容性测试

## 知识不迷路

1. web-view（非组件容器） 承载网页的容器 会自动铺满整个小程序页面
2. 小程序在调试模式，在右上角的“详情”中“本地设置"钩上“不校验合法域名、web-view（业务域名）、`TLS` 版本以及 `HTTPS` 证书”这个选项
3. web-view 网页与小程序之间不支持除 `JS-SDK` 提供的接口之外的通信
4. 网页内 `iframe` 的域名也需要配置到域名白名单

## 相关文档

* [web-view 文档](https://developers.weixin.qq.com/miniprogram/dev/component/web-view.html)
* 拓展：[第三方静态网站 H5 跳转小程序](https://developers.weixin.qq.com/miniprogram/dev/wxcloud/guide/staticstorage/jump-miniprogram.html)
