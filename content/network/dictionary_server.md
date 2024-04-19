---
title: "如何创建一个简易查词服务器"
date: 2024-04-19T17:30:22+08:00
draft: false
---

# 如何创建一个查词简易服务器

TLDR: 本文通过`express`和`orama`实现了一个简易的查词服务器.

在这篇文章中, 我们将使用`express`和`orama`来创建一个简单的查词服务器. 该服务器将接受一个单词, 并可以根据需要定制返回的查询结果.

## 1. 安装依赖

`express`是一个基于Node.js的Web应用程序框架, 它可以帮助我们快速地搭建一个Web服务器. 使用其可以简单搭建我们需要的服务器. `orama`是一个全文搜索引擎,其大小只有2KB, 但是功能强大. 我们将使用`orama`来实现查词功能.

首先, 我们需要nodejs环境(nodejs版本需要大于等于16.0). 然后我们可以通过以下命令安装`express`和`orama`:

```bash
npm install express @orama/orama
```

## 2. 数据准备

既然要做一个查词服务器, 那么我们需要一个内容准确、丰富的数据来源. 这里我使用了来自有道的数据, 来源: [kajweb/dict](https://github.com/kajweb/dict)

下载你需要的数据: 四六级/考研/高考/托福/雅思等等. 然后解压文件. 以下以`KaoYan_3.json`为例:

这里要注意: 得到的json文件并不是标准的json格式, 其文件内容每一行可以解析为一个对象, 我们需要对其进行处理或者在读取时进行处理.

## 3. 代码实现

首先, 我们需要先初始化一个`npm`项目

```bash
npm init -y
```

创建一个`index.js`文件, 先测试数据读取, 在其中编写以下代码:

```javascript
import fs from 'fs';
const dataRaw = fs.readFileSync('./KaoYan_3.json', 'utf-8');

const data = dataRaw.split('\n').map((line) => {
  try {
    return JSON.parse(line);
  } catch (e) {
    return null;
  }
}).filter((item) => item !== null);

console.log(data.length);
```

```bash
$ node index.js
3728
```

数据读取成功

接下来, 创建`orama`索引, 确定我们需要的字段

```javascript
import { create } from '@orama/orama';

const db = await create({
  schema: {
    headWord: 'string',     // 单词
    content: {
        usphone: 'string',  // 美式发音
        trans: 'string[]',  // 翻译
        sentence: {
            en: 'string[]', // 英文例句
            cn: 'string[]', // 例句的中文翻译
        }
    }
  },
});
```

在运行时可能会提示
> SyntaxError: await is only valid in async functions and the top level bodies of modules

这时候我们可以在`package.json`中添加`"type": "module"`来解决这个问题

```json
{
  ...
  "type": "module",
  "main": "index.js",
  ...
}
```

接下来, 我们需要将数据导入到`orama`中:

```javascript
...
import { insertMultiple } from '@orama/orama';
...

await insertMultiple(db, data.map((element) => {
    let sentences = element.content.word.content.sentence;
    if(sentences!=undefined) {
        sentences = sentences.sentences.map((item) => {
            return {
                en: item.sContent,
                cn: item.sCn,
            }
        });
    } else {
        sentences = [];
    }
    return {
        headWord: element.headWord,
        content: {
            usphone: element.content.word.content.usphone,
            trans: element.content.word.content.trans.map((item) => item.tranCn),
            sentence: {
                en: sentences.map((item) => item.en),
                cn: sentences.map((item) => item.cn)
            },
        }
    }
}));
```

这里由于`orama`无法创建对象数组的索引, 所以我们需要将其转换为字符串数组.

接下来, 我们需要创建一个`express`服务器, 并实现查询功能:

```javascript
...
import express from 'express';
...

const app = express();
app.get('/', async (req, res) => {
    const { term, offset } = req.query;
    const result = await search(db, {
        term,
        offset: offset ? parseInt(offset) : 0,
    }).then((result) => {
        return {
            formatted: result.elapsed.formatted,
            result: result.hits.map((item) => item.document),
            count: result.count,
            offset: offset ? parseInt(offset) : 0,
        };
    });
    res.json(result);
});

console.log('Server is starting...');

app.listen(3000, () => {
    console.log('Server is running on port 3000');
});

```

最后, 我们可以通过以下命令启动服务器:

```bash
node index.js
```

## 4. 测试

我们直接通过`curl`或者浏览器访问`http://localhost:3000/?term=hello`即可查询到`hello`的结果.

```bash
$curl "http://localhost:3000/?term=hello" | jq   
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   281  100   281    0     0  22670      0 --:--:-- --:--:-- --:--:-- 31222
{
  "formatted": "2ms",
  "result": [
    {
      "headWord": "operator",
      "content": {
        "usphone": "'ɑpəretɚ",
        "trans": [
          "操作人员；接线员"
        ],
        "sentence": {
          "en": [
            "Hello, operator? Could you put me through to Room 31?"
          ],
          "cn": [
            "喂，接线员吗？请帮我接31号房间。"
          ]
        }
      }
    }
  ],
  "count": 1,
  "offset": 0
}
```

可以看到我们成功查询到了`hello`的结果. 出现在了`operator`的例句中.

## 5. 总结

在本文中, 我们使用了`express`和`orama`来创建了一个简单的查词服务器. 通过这个服务器, 我们可以根据需要定制返回的查询结果. 你可以根据自己的需求来定制这个服务器, 比如添加更多的数据源, 添加更多的查询条件等等.

## 6. 参考

官方文档:[express](https://expressjs.com/), [orama](https://docs.askorama.ai/open-source/)

### 代码

```javascript
import { create, insertMultiple, search } from "@orama/orama";
import { readFileSync } from 'fs';
import express from 'express';

const db = await create({
  schema: {
    headWord: 'string',
    content: {
        usphone: 'string',
        trans: 'string[]',
        sentence: {
            en: 'string[]',
            cn: 'string[]',
        }
    }
  },
});

console.log('Database created');

let rawData = readFileSync('KaoYan_3.json','utf-8');

const data = rawData.split('\n').map((line) => {
    try {
      return JSON.parse(line);
    } catch (e) {
      return null;
    }
  }).filter((item) => item !== null);

await insertMultiple(db, data.map((element) => {
    let sentences = element.content.word.content.sentence;
    if(sentences!=undefined) {
        sentences = sentences.sentences.map((item) => {
            return {
                en: item.sContent,
                cn: item.sCn,
            }
        });
    } else {
        sentences = [];
    }
    return {
        headWord: element.headWord,
        content: {
            usphone: element.content.word.content.usphone,
            trans: element.content.word.content.trans.map((item) => item.tranCn),
            sentence: {
                en: sentences.map((item) => item.en),
                cn: sentences.map((item) => item.cn)
            },
        }
    }
}));

console.log('Data inserted');

const app = express();
app.get('/', async (req, res) => {
    const { term, offset } = req.query;
    const result = await search(db, {
        term,
        offset: offset ? parseInt(offset) : 0,
    }).then((result) => {
        return {
            formatted: result.elapsed.formatted,
            result: result.hits.map((item) => item.document),
            count: result.count,
            offset: offset ? parseInt(offset) : 0,
        };
    });
    res.json(result);
});

console.log('Server is starting...');

app.listen(3000, () => {
    console.log('Server is running on port 3000');
});
```

PS: `orama`也支持中文的索引,但效果很差,10条记录的导入时间就达到了2s,查询时间约为130ms,所以这里只是简单的实现了英文的索引.
