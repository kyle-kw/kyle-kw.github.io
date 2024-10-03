+++
title = 'OpenAI代理'
date = 2024-10-02T19:58:46+08:00
tags = ["OpenAI", "Claude", "Proxy"]
+++


由于前段时间Openai将大陆和港澳地区的api服务封禁了，属于双向奔赴了。

解决方式有很多，基本思路是搭建一个代理服务，让代理“代替”请求。

下面就介绍一种比较简单免费的方式，个人使用量不大，可以很好的转发。

### 腾讯云函数

> 云函数（Serverless Cloud Function，SCF）是腾讯云为企业和开发者们提供的无服务器执行环境，帮助您在无需购买和管理服务器的情况下运行代码。您只需使用平台支持的语言编写核心代码并设置代码运行的条件，即可在腾讯云基础设施上弹性、安全地运行代码。SCF 是实时文件处理和数据处理等场景下理想的计算平台。

官方文档： [云函数](https://cloud.tencent.com/document/product/583)

之前折腾了[Cloudflare Workers](https://developers.cloudflare.com/workers)，Cloudflare家的云函数，个人也是免费，而且可以自定义域名。
但是有一个致命的问题，函数运行的服务器是按照访问访问的ip来决定的，无法手动指定（我暂时没有找到），由于代理肯定是国内访问的，那么workers出口就是距离最近的香港
（可以访问https://<you-domain>/cdn-cgi/trace查看当前的三字码）
而香港又被禁止访问api。所以就放弃了这个方案。

下面使用腾讯云函数，它可以指定函数的运行地区，且可以固定ip，这对api访问很友好。

### 使用云函数

具体步骤可以参考： [openai-api-proxy](https://github.com/easychen/openai-api-proxy/blob/master/FUNC.md)。

其中也有一个[app.js](https://github.com/easychen/openai-api-proxy/blob/master/app.js)文件，使用的腾讯云的敏感词检查，对于我来说有点多余。我对其进行了一些修改，并兼容了Claude的模型。

由于腾讯云函数进行了升级，上面步骤中的截图有一些和实际UI不同，注意甄别。还有截止到目前官方云函数无法简单的设置自定义域名。

下面是我修改之后的`app.js`，根据上面的步骤，将函数部署之后就可以使用了。注意：1. 可以选择部署的地区。 2. 可以固定出口ip。 3. 将公网请求打开，会给一个外网域名，后续可以根据这个域名代理Openai和Claude的api。

```javascript
const express = require('express')
const fetch = require('cross-fetch')
var multer = require('multer');
const cors = require('cors');
const bodyParser = require('body-parser')
const { createParser } = require('eventsource-parser');

const app = express()
var forms = multer({limits: { fieldSize: 10*1024*1024 }});
app.use(forms.array()); 
app.use(cors());
app.use(bodyParser.json({limit : '50mb' }));  
app.use(bodyParser.urlencoded({ extended: true }));

app.all(`*`, async (req, res) => {
  let proxy_type = 'openai';
  const openai_key = req.headers.authorization?.split(' ')[1];
  const claude_key = req.headers['x-api-key'];
  if (openai_key){
    proxy_type = 'openai';
  } else if (claude_key) {
    proxy_type = 'claude'
  } else {
    return res.status(403).send('Forbidden');
  }

  const options = {
      method: req.method,
      timeout: process.env.TIMEOUT || 30000,
      headers: {
        'Content-Type': 'application/json; charset=utf-8',
      },
  };
  if(req.originalUrl) {
    req.url = req.originalUrl;
  }
  let url;
  if (proxy_type === 'claude'){
    options.headers["x-api-key"] = claude_key;
    if ("anthropic-beta" in req.headers){
      options.headers["anthropic-beta"] = req.headers["anthropic-beta"];
    }
    if ("anthropic-version" in req.headers){
      options.headers["anthropic-version"] = req.headers["anthropic-version"];
    }
    url = `https://api.anthropic.com${req.url}`;

  } else {
    options.headers["Authorization"] = `Bearer ${openai_key}`;
    url = `https://api.openai.com${req.url}`;
  }

  const { moderation, moderation_level, ...restBody } = req.body;

  if( req.method.toLocaleLowerCase() === 'post' && req.body ) {
    options.body = JSON.stringify(restBody);
  }

  try {
    const response = await myFetch(url, options);

    if (req.body.stream) {
      if( response.ok ) {
        res.writeHead(200, {
          'Content-Type': 'text/event-stream',
          'Cache-Control': 'no-cache',
          'Connection': 'keep-alive',
        });
        const parser = createParser((event) => {
            if (event.type === 'event') {
                let one_chunk = ''
                if (event.event){
                  one_chunk = 'event: ' + event.event + '\n';
                }
                one_chunk += 'data: ' + event.data + '\n\n';
                res.write(one_chunk);
                if (event.data === '[DONE]' || event.event === 'message_stop'){
                  res.end();
                }
            }
        });

        if (!response.body.getReader) {
          const body = response.body;
          if (!body.on || !body.read) {
            throw new error('unsupported "fetch" implementation');
          }
          body.on("readable", () => {
            let chunk;
            while (null !== (chunk = body.read())) {
              parser.feed(chunk.toString());
            }
            
          });
        } else {
          const reader = response.body.getReader();
          const decoder = new TextDecoder()
          try {
            while (true) {
              const { done, value } = await reader.read();
              if (done) {
                break;
              }
              parser.feed(decoder.decode(value));
            }
          } finally {
            reader.releaseLock();
          }          
        }
        
      } else {
        const body = await response.text();
        res.status(response.status).send(body);
      }
      
    } else {
      const content_type = response.headers.get("Content-Type");
      if (content_type.includes("application/json")) {
        const data = await response.json();
        res.json(data);

      } else if (content_type.includes("audio")) {
        const audioBlob = await response.blob();
        res.setHeader('Content-Type', 'audio/mpeg');
        const audioStream = audioBlob.stream();
        audioStream.pipe(res);

      } else {
        res.status(500).send("Content-Type: " + content_type);
      }
    }

  } catch (error) {
    res.status(500).json({"error":error.toString()});  
  }

})


async function myFetch(url, options) {
  const {timeout, ...fetchOptions} = options;
  const controller = new AbortController();
  const timeoutId = setTimeout(() => controller.abort(), timeout||30000)
  const res = await fetch(url, {...fetchOptions, signal: controller.signal});
  clearTimeout(timeoutId);
  return res;
}

app.use(function(err, req, res, next) {
  console.error(err)
  res.status(500).send('Internal Serverless Error')
})

const port = process.env.PORT || 9000;
app.listen(port, () => {
  console.log(`Server start on http://localhost:${port}`);
})
```
