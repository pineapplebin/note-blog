# 微信小程序 云函数插入大量数据到数据库

## 背景

需要在云函数中，请求第三方接口数据，然后同步到云开发的数据库中，大概数量为1w条左右。

最开始时，请求到数量后使用

```js
for (const item of list) {
  db.collection('data').add(item)
}
```

的形式进行插入，但是耗时太长导致接口响应超时。同时`add`不支持批量导入。

## 方案

查找文档后发现，服务端API中有一个
[databaseMigrateImport](https://developers.weixin.qq.com/miniprogram/dev/wxcloud/reference-http-api/database/databaseMigrateImport.html)
的接口，可以指定云存储上一个*json*或*csv*文件进行批量导入数据到数据库中。

则方案设计为：
1. 云函数中请求第三方接口获取更新数据
2. 数据按格式写入到Buffer后，通过`cloud.uploadFile`上传*csv*文件
3. 调用`databaseMigrateImport`接口实现数据导入

## 实现

```js
// syncDataList
const cloud = require('wx-server-sdk')
const axios = require('axios').default

cloud.init({
  env: cloud.DYNAMIC_CURRENT_ENV,
})
const db = cloud.database()
const _ = db.command

exports.main = async () => {
  const res = await axios.request({ url: '' })  // 请求第三方接口

  // 上传文件
  const fileContent = ''
  const buffer = Buffer.from(fileContent, 'utf-8')

  await cloud.uploadFile({
    cloudPath: 'file.csv',
    fileContent: buffer,
  })

  await cloud.callFunction({
    name: 'data',
    data: {
      type: 'databaseMigrateImport',
      filePath: 'file.csv',
      collectionName: 'data',
    },
  })

  return true
}
```

```js
// databaseMigrateImport
exports.databaseMigrateImport = async (event) => {
  if (!event.accessToken) {
    const accessTokenRes = await cloud.callFunction({
      name: 'collections',
      data: {
        type: 'getAccessToken',
      },
    })
    event.accessToken = accessTokenRes.result
  }
  const url = `https://api.weixin.qq.com/tcb/databasemigrateimport?access_token=${event.accessToken}`

  const data = {
    env: cloud.getWXContext().ENV,
    file_path: event.filePath,
    collection_name: event.collectionName,
    file_type: 2,
    stop_on_error: true,
    conflict_mode: 2,
  }

  const res = await axios.request({
    url,
    method: 'POST',
    data,
  })

  if (res.data.errcode) {
    throw Error(res.data.errmsg)
  }
  return res.data
}
```
