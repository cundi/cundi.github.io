

auth_chain_a（强烈推荐）：对首个包的认证部分进行使用Encrypt-then-MAC模式以真正免疫认证包的CCA攻击，预防各种探测和重防攻击，数据流自带RC4加密，同时此协议支持单端口多用户，不同用户之间无法解密数据，每次加密密钥均不相同，具体设置方法参见breakwa11的博客。
使用此插件的服务器与客户机的UTC时间差不能超过24小时，即只需要年份日期正确即可，针对UDP部分也有加密及长度混淆。
使用此插件建议加密使用 none。此插件不能兼容原协议，支持服务端自定义参数，参数为10进制整数，表示最大客户端同时使用数，最小值支持直接设置为 1，此插件能实时响应实际的客户端数量（你的客户端至少有一个连接没有断开才能保证你占用了一个客户端数，否则设置为1时其它客户端一连接别的就一定连不上）。
推荐使用 auth_chain_a 插件，在以上插件里混淆能力较高，而抗检测能力最高，即使多人使用也难以识别封锁。同时如果要发布公开代理，以上auth插件均可严格限制使用客户端数（要注意的是若为auth_sha1_v4_compatible，那么用户只要使用原协议就没有限制效果），而auth_chain_a协议的限制最为精确。
—— 以上内容引用自： ShadowsocksR Wiki

```json
{
  "server":"0.0.0.0",
  "server_ipv6":"::",
  "local_address":"127.0.0.1",
  "local_port":1080,
  "port_password":{
      "443":{"protocol":"auth_chain_a", "password":"123456", "obfs":"tls1.2_ticket_auth", "obfs_param":""},
      "4392":{"protocol":"auth_aes128_md5", "password":"123456", "obfs":"tls1.2_ticket_auth", "obfs_param":""}
  },
  "timeout":400,
  "method":"chacha20",
  "protocol": "origin",
  "protocol_param": "",
  "obfs": "plain",
  "obfs_param": "",
  "redirect": "",
  "dns_ipv6": true,
  "fast_open": true,
  "workers": 1
}
```
