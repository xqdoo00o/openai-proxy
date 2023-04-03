# OpenAI-Proxy
## Cloudflare Worker
创建Cloudflare Worker服务->HTTP路由器，快速编辑->配置如下
```
// 是否拒绝所有无 Origin 请求
const ALLOW_NO_ORIGIN = false;
// 允许的Origin列表
const ALLOWED_ORIGIN = [/^https?:\/\/.*\.example\.com$/, /^https?:\/\/test\.com$/];
const validateOrigin = (req) => {
  const origin = req.headers.get('Origin')
  if (origin) {
    for (let i = 0; i < ALLOWED_ORIGIN.length; i++) {
      if (ALLOWED_ORIGIN[i].exec(origin)) {
        return true
      }
    }
  }
  return ALLOW_NO_ORIGIN
}
export default {
  async fetch(request) {
    if (validateOrigin(request)) {
      const requestUrl = new URL(request.url)
      requestUrl.host = "api.openai.com"
      request = new Request(requestUrl, request)
      //如需前端发送API Key，注释掉下一行
      request.headers.set('Authorization', 'Bearer sk-your-token')
      const response = await fetch(request)
      const corsResponse = new Response(response.body, response)
      corsResponse.headers.set('Access-Control-Allow-Origin', '*')
      corsResponse.headers.set('Cache-Control', 'no-store')
      //gzip压缩api响应，打开域名面板->速度->优化，关闭Brotli后才会生效
      corsResponse.headers.set('Content-Encoding', 'gzip')
      return corsResponse
    } else {
      return new Response('[CloudFlare Workers] REQUEST NOT ALLOWED', {status: 403});
    }
  }
}
```
默认配置允许源域名是*.example.com或test.com的请求正常反代，修改`ALLOWED_ORIGIN`更改允许源域名列表

如修改`ALLOW_NO_ORIGIN`为true，则不检查源域名，所有请求都正常反代。

注意：默认worker域名国内被干扰，点击触发器->添加自定义域，添加CF里正常使用的域名访问worker