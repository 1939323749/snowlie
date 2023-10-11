---
title: "让台灯告诉你可以洗衣服了)"
date: 2023-10-11T22:46:22+08:00
draft: false
---

# 让台灯告诉你可以洗衣服了)

这学期发现宿舍楼的洗衣机换成了u净, 但因为每层楼只有一台, 经常走过去发现已经被别人占用了, 于是就想着能不能做个东西来提醒我洗衣机是否空闲, 于是就有了这个项目.

## 获得洗衣机状态

u净在大学中还是比较常见的, 于是上GitHub简单搜索发现了这个项目:[U-Clean-Status-Checker-Android](https://github.com/CarlWtrs/U-Clean-Status-Checker-Android), 查看源代码发现这几行是用于获取状态的:

```kotlin
suspend fun getStatus(postData: String): Pair<Boolean, String> = withContext(Dispatchers.IO) {
    val url = "https://phoenix.ujing.online/api/v1/wechat/devices/scanWasherCode"
    val headers = mapOf(
        "Accept" to "application/json, text/plain, */*",
        "Authorization" to "Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJhcHBVc2VySWQiOiJvZ3lSVDF1M0dlZU9OV2N5SGdHekZYM3RoLVVNIiwiZXhwIjoxNzAyMjgzNTg4LCJpYXQiOjE2OTQyNDgzODgsImlkIjozMDE5NDgzMiwibmFtZSI6IjE5ODc2NTc2NzY4In0.N83KdLj5-3DuyaY4-n9lsocUpq71QwnCvB4Ox7FL1D0",
        "Accept-Language" to "zh-CN,zh-Hans;q=0.9",
        "Accept-Encoding" to "gzip, deflate, br",
        "Content-Type" to "application/json; charset=utf-8",
        "x-app-code" to "BCI",
        "Origin" to "https://wx.zhinengxiyifang.cn",
        "User-Agent" to "Mozilla/5.0 (iPhone; CPU iPhone OS 15_6 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) Mobile/15E148 MicroMessenger/8.0.33(0x18002121) NetType/WIFI Language/zh_CN",
        "Referer" to "https://wx.zhinengxiyifang.cn/",
        "x-app-version" to "0.1.40",
        "Connection" to "keep-alive"
    )

    val data = mapOf("qrCode" to postData)
    val body =
        Json.encodeToString(data).toRequestBody("application/json; charset=utf-8".toMediaType())

    val request = Request.Builder()
        .url(url)
        .headers(headers.toHeaders())
        .post(body)
        .build()

    val responseText = OkHttpClient().newCall(request).execute().body!!.string()

    val jsonObject = JSONObject(responseText)
    val createOrderEnabled =
        jsonObject.getJSONObject("data").getJSONObject("result").getBoolean("createOrderEnabled")
    val reason = jsonObject.getJSONObject("data").getJSONObject("result").getString("reason")
    return@withContext Pair(createOrderEnabled, reason)
}
```

简单分析一下, 发现只需要发送一个POST请求, 参数为`qrCode`, 值为洗衣机的二维码, 就可以获取到洗衣机的状态. 走到洗衣机前, 扫描二维码, 抓包得到一个这样的url:

```plain
http://app.littleswan.com/u_download.html?type=Ujing&uuid=0000000000000A1234567202208220003498
```

先把这个函数改写为curl命令进行简单的分析:

```bash
curl -X POST "https://phoenix.ujing.online/api/v1/wechat/devices/scanWasherCode" \
-H "Accept: application/json, text/plain, */*" \
-H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJhcHBVc2VySWQiOiJvZ3lSVDF1M0dlZU9OV2N5SGdHekZYM3RoLVVNIiwiZXhwIjoxNzAyMjgzNTg4LCJpYXQiOjE2OTQyNDgzODgsImlkIjozMDE5NDgzMiwibmFtZSI6IjE5ODc2NTc2NzY4In0.N83KdLj5-3DuyaY4-n9lsocUpq71QwnCvB4Ox7FL1D0" \
-H "Accept-Language: zh-CN,zh-Hans;q=0.9" \
-H "Accept-Encoding: gzip, deflate, br" \
-H "Content-Type: application/json; charset=utf-8" \
-H "x-app-code: BCI" \
-H "Origin: https://wx.zhinengxiyifang.cn" \
-H "User-Agent: Mozilla/5.0 (iPhone; CPU iPhone OS 15_6 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) Mobile/15E148 MicroMessenger/8.0.33(0x18002121) NetType/WIFI Language/zh_CN" \
-H "Referer: https://wx.zhinengxiyifang.cn/" \
-H "x-app-version: 0.1.40" \
-H "Connection: keep-alive" \
-d '{"qrCode": "http://app.littleswan.com/u_download.html?type=Ujing&uuid=0000000000000A1234567202208220003498"}' | jq
```

得到:

```json
{
  "code": 0,
  "message": "",
  "data": {
    "service": "washer",
    "path": "washer/home.js",
    "result": {
      "orderId": 0,
      "deviceId": "...",
      "macAddress": "",
      "isSlotMachine": false,
      "moduleType": 7,
      "status": 1,
      "mobile": "",
      "createOrderEnabled": false,
      "reason": "此设备正在运行中，请使用其他设备",
      "needSync": false,
      "needSyncBLE": false,
      "deviceTypeId": 9
    }
  }
}
```

其中`createOrderEnabled`字段为`false`表示正在运行, 为`true`表示空闲.

## 通知

因为我的桌子上有一个米家的台灯, 于是就想着用它来通知我洗衣机的状态.

简单的搜索了一下, 发现有一个叫做`miio`的python库, 用它来可以很简单地控制台灯.

先下载`miio`库和`miiocli`

```bash
pip install python-miio
```

根据文档, 先获取设备的ip和token, 运行:

```bash
miiocli cloud
```

然后输入米家的账号和密码, 就可以获取到设备的ip和token了.

```plain
miiocli cloud
Username: username
Password: passwd
== 台灯 (Device online ) ==
    Model: yeelink.light.lamp4
    Token: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
    IP: 192.168.2.241 (mac: 12:34:56:78:90:AB)
    DID: 476680402
    Locale: cn
```

记住这里的`Token`和`IP`, 用于后面的程序.

然后就可以用`miio`库来控制台灯了, 例如:

```python
from miio import Yeelight
import time
import requests

dev = Yeelight(ip="192.168.2.241",token= "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx")

headers = {
    'Accept': 'application/json, text/plain, */*',
    'Authorization': 'Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJhcHBVc2VySWQiOiJvZ3lSVDF1M0dlZU9OV2N5SGdHekZYM3RoLVVNIiwiZXhwIjoxNzAyMjgzNTg4LCJpYXQiOjE2OTQyNDgzODgsImlkIjozMDE5NDgzMiwibmFtZSI6IjE5ODc2NTc2NzY4In0.N83KdLj5-3DuyaY4-n9lsocUpq71QwnCvB4Ox7FL1D0',
    'Accept-Language': 'zh-CN,zh-Hans;q=0.9',
    'Content-Type': 'application/json; charset=utf-8',
    'x-app-code': 'BCI',
    'Origin': 'https://wx.zhinengxiyifang.cn',
    'User-Agent': 'Mozilla/5.0 (iPhone; CPU iPhone OS 15_6 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) Mobile/15E148 MicroMessenger/8.0.33(0x18002121) NetType/WIFI Language/zh_CN',
    'Referer': 'https://wx.zhinengxiyifang.cn/',
    'x-app-version': '0.1.40',
    'Connection': 'keep-alive',
}

json_data = {
    'qrCode': 'http://app.littleswan.com/u_download.html?type=Ujing&uuid=0000000000000A1234567202208220003498',
}

dev.off()

while True:
    response = requests.post('https://phoenix.ujing.online/api/v1/wechat/devices/scanWasherCode', headers=headers, json=json_data)
    isWasherAvailable = response.json()['data']['result']['createOrderEnabled']
    if isWasherAvailable:
        dev.on()
        break
    time.sleep(30)
```

好了, 灯亮了就去洗衣服吧)
