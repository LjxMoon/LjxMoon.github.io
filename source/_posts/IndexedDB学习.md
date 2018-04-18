---
title: IndexedDB学习笔记
tags: IndexedDB
---

# IndexedDB学习笔记

## 连接数据库

``` js
const request = IndexedDB.open('dbName', version)
```

连接数据库，`dbName` 是数据库名，`version` 是数据库版本号，版本号必须为整数。返回的是一个 `IDBOpenDBRquest` 对象，代表一个请求连接数据库的请求对象，可以对对象的 `onsuccess` 和 `onerror` 事件监听来定义连接成功或失败后执行的方法。

``` js
const request = IndexedDB.open('test', 1)
request.onsuccess = (e) => {
  console.log('连接数据库成功！')
}
```

连接数据库成功后执行的事件。

``` js
const request = IndexedDB.open('test', 1)
request.onerror = (e) => {
  console.log('连接数据库失败！')
}
```

连接数据库失败后执行的事件。

``` js
const request = IndexedDB.open('test', 2)
request.onupgradeneeded = (e) => {
  console.log('数据库当前版本为：' + e.newVersion)
}
```

数据库更新后执行的事件。在对indexedDB的对象仓库本身进行操作的时候，需要在数据库版本号更新时才会创建的 `ongradeneeded` 事件中进行操作。

## 建立对象仓库

``` javascript
const request = IndexedDB.open('test', 3)
request.onupgradeneeded = (e) => {
  const db = e.target.result
  const store = db.createObjectStore('User', {
 keyPath: 'userId',
 autoIncrement: false,
  })
  const idx = store.createIndex('userNameIndex', 'userName', {
 unique: false
  })
  console.log('创建对象仓库成功！')
}
```

`createObjectStore` 方法创建一个对象仓库，类似于关系型数据库的一个表，该方法接收两个参数：

* 第一个参数是该对象仓库的名字；
* 第二个参数是一个对象，其中 `keyPath` 是该对象仓库的主键，`autoIncrement` 为是否要求该主键自增，`false` 即为不允许主键自增，需要创建数据时自己传入。

`createIndex` 方法接收三个参数：

* 第一个参数为索引名；
* 第二个为数据对象的某个属性，这里使用 `userName` 属性来创建索引；
* 第三个参数为可选参数，值为一个对象。对象中的 `unique` 属性为 `true` ，代表索引值不可以相同，即两条数据的 `userName` 不可以相同，为 `false` 表示可以相同。

## 操作对象仓库中的数据

### 操作对象仓库中数据的方法

* **add()**：新增数据的方法，接受要新增的对象数据作为参数若参数对象的主键在对象仓库中已存在，会新增失败。
* **put()**：新增或更新数据的方法，接受要新增的对象数据作为参数，若参数对象的主键在对象仓库中已存在，会变为更新对应的数据对象。
* **get()**：获取数据的方法，接受要查询数据的主键作为参数。
* **delete()**：删除数据的方法，接受要查询数据的主键作为参数。

``` javascript
const request = indexedDB.open('test', 3)
request.onsuccess = (e) => {
  console.log('连接数据库成功！')
  const db = e.target.result
  const tx = db.transaction('User','readwrite')
  const store = tx.objectStore('User')
  const value = {
 'userId': 1,
 'userName': 'Luke',
 'age': 18
  }
  const req1 = store.put(value)
  const req2 = store.get(1)
  req2.onsuccess = function () {
 console.log(this.result)
  }
  const req3 = store.delete(1)
  req3.onsuccess = function () => {
 console.log('数据删除成功！')
  }
}
```

`transaction` 方法接收两个参数：

* 第一个参数是字符串或数组，是字符串时该参数是一个对象仓库的名字，是数组时该参数是由对象仓库名组成的数组，`transaction` 可以对参数中任何一个对象仓库进行操作；
* 第二个参数为对象仓库的事务模式，传入 `readonly` 表示只能对对象仓库进行读操作，无法进行写操作。传入 `readwrite` 时表示可以进行读写操作。

### 使用索引值查询数据

``` javascript
const index = store.index('userNameIndex')
index.get('DrMoon').onsuccess = function () {
  console.log(this.result)
}
```

使用创建对象仓库时定义的索引来获取数据对象：

* 首先要使用 `index` 方法定义一个索引查询对象，该方法接收一个索引项作为参数，这个索引项需是之前创建对象索引时采用的索引名，即 `createIndex` 方法的第一个参数；
* 然后就可以使用刚才定义的索引查询对象的 `get` 方法来查询数据对象。

### 使用游标来查询数据

使用 `get` 方法获取数据需要知道数据的主键值，且只能获取单个主键值的数据，若想查询范围数值内的主键值的数据，可以使用对象仓库的 `openCursor` 方法，该方法接收两个参数：

* 第一个参数定义要查询的目标数据对象的主键值范围，首先需要使用 `IDBKeyRange` 对象的方法来定义查询的主键值范围：
  * **IDBKeyRange.bound(1, 10, false, false)** ：
    `bound` 方法可以定义查询的主键值范围，该方法接收四个参数：
    * 第一个参数为要查询的主键值范围的最小值；
    * 第二个参数为要查询的主键值范围的最大值；
    * 第三个参数表示范围是否包含范围的最小值，即要查询的主键值范围是否包含第一个参数，为 `true` 表示不包含，`false` 表示包含，默认为 `false` ；
    * 第四个参数表示范围是否包含范围的最大值，即要查询的主键值范围是否包含第二个参数，为 `true` 表示不包含，`false` 表示包含，默认为 `false` 。
  * **IDBKeyRange.only(1)** ：
    `only` 方法可以定义只包含单个主键值的范围，该方法接收一个参数：
      * 该参数即为要查询的主键值
  * **IDBKeyRange.lowerBound(1, false)**：
    `lowerBound` 方法可以定义小于某个主键值的主键值范围，该方法接收两个参数：
    * 第一个参数定义主键值的最小值，要查询的主键值都要大于这个主键值；
    * 第二个参数定义主键值范围是否包含范围的最小值，即要查询的主键值范围是否包含第一个参数，为 `true` 表示不包含，`false` 表示包含，默认为 `false` 。
  * **IDBKeyRange.upperBound(10, false)**：
    `upperBound` 方法可以定义小于某个主键值的主键值范围，该方法接收两个参数：
    * 第一个参数定义主键值的最大值，要查询的主键值都要小于这个主键值；
    * 第二个参数定义主键值范围是否包含范围的最大值，即要查询的主键值范围是否包含第一个参数，为 `true` 表示不包含，`false` 表示包含，默认为 `false` 。
* 第二个参数时查询数据对象时游标的读取方向：
  * **next**：游标读取的数据按主键值升序排列，主键值相等的数据都被读取。参数默认为该值。
  * **nextunique**：游标读取的数据按主键值升序排列，主键值相等的数据只读取第一条。
  * **prev**：游标读取的数据按主键值降序排列，主键值相等的数据都被读取。
  * **prevunique**：游标读取的数据按主键值降序排列，主键值相等的数据只读取第一条。

``` javascript
    const range = IDBKeyRange.bound(1, 10)
    let data = []
    let req = store.openCursor(range, 'next')
    req.onsuccess = function (e) {
      let cursor = this.result
      if (cursor) {
        data.push(cursor.value)
        cursor.continue()
      } else {
        console.log(data)
      }
    }
```

`openCursor` 方法根据接收的参数对对象仓库进行查询，每查询到一条数据回将该条数据的结果推入 `data` 数组，然后使用 `continue` 方法进行下一次查询，直到查询结束后将 `data` 数组打印出来。这样便完成了对要求的主键值范围内的数据对象的查询。

若 `openCursor` 方法接收到的第一个参数为空或者 `null` ，该方法会将对象仓库中的所有数据查询出来。