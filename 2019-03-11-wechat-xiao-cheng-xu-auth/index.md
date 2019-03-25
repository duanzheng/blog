## 用户身份创建

在小程序中调用 wx.login() 获取 code（用户登录凭证）

在小程序中获取用户信息的两种方式：

1. 未授权之前，使用 open-type=getUserInfo 的 button，在 bindgetuserinfo 回调中获
   得用户信息
2. 得到授权后，调用 wx.getUserInfo() 方法获取用户信息

用户信息中会被后面用到的数据有 encryptedData 和 iv，将这两个数据和前面得到的
code 传到服务端。

在服务端请求微信接口 https://api.weixin.qq.com/sns/jscode2session 获取到 openid
和 session_key，该请求还需要 appid 和 secret，这两个数据是从管理后台获取，可以把
他们保存在项目的配置文件中方便读取。openid 是该微信用户在该平台的唯一性标识，因
此我们可以把 openid 进行 hash 加密后的值作为用户的 ID 在数据库中创建一条用户数据
。

## 获取用户信息

在完成用户的初始化后，我们希望在服务端获取到更多的用户信息，要完成这件事情我们需
要前面获取到的 openid、session_key、encryptedData 和 iv。**需要注意的是
，session_key 的有效时间是不固定的，每次执行 wx.login() 都有可能会生成新的
session_key，同时旧的 session_key 会失效，而解密用户信息需要的
session_key、encryptedData 和 iv 是一一对应的，因此我们要保证每次的流程都是先执
行 login，再去获取用户的 encryptedData 和 iv。**

我们要获取到用户信息，需要对密文进行对称解密，微信官方提供了示例代码可以直接使用
，以下是解密的流程：

```javascript
// 将 session_key、encryptedData 和 iv 以 BASE64 的编码方式转换成 Buffer；
// 解密秘钥，aeskey
const sessionKey = new Buffer(this.sessionKey, 'base64');
// 被加密的数据
const encryptedData = new Buffer(encryptedData, 'base64');
// 初始向量
const iv = new Buffer(iv, 'base64');
// 通过 crypto 的 createDecipheriv 方法创建 decipher 实例，使用 aes-128-cbc 算法
const decipher = crypto.createDecipheriv('aes-128-cbc', sessionKey, iv);
// 设置自动 padding 为 true，删除填充补位
decipher.setAutoPadding(true);
// 更新 decipher 实例，设置输出的编码格式为 utf8
const decoded = decipher.update(encryptedData, 'binary', 'utf8');
// 其余被解码的内容
decoded += decipher.final('utf8');
decoded = JSON.parse(decoded);
```

最终得到的 decoded 值就是最终的包含敏感数据的用户信息，比如国家、城市、性别等。

## 把用户的 ID 存放在 cookie 中方便后端鉴权

在这里我们用到的一些第三方的能力：

小程序端用到了 [weapp-cookie](https://github.com/charleslo1/weapp-cookie)，它的
作用是让小程序发送的请求中能带上 cookie（原生是不支持的）。

服务端使用了 [yar](https://github.com/hapijs/yar) 保存 session 数据，它的原理是
将保存的值进行加密，放到 cookie 中以维持前后端的会话。在这里我们需要设置 cookie
的过期时间，一般是 2 小时，防止另有企图的人拿到用户的 cookie 进行模拟请求。

我们可以在用户完成登录后，将用户的 UID 保存在 session 中，这样在需要鉴权的接口中
，就可以根据 session 中保存的 UID 进行判断是否可以继续进行后续的操作。
