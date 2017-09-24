# [react-native-storage](https://github.com/sunnylqm/react-native-storage/blob/master/README-CHN.md) 分析

安装

```
npm install react-native-storage --save
```

一个用于 ReactNative 或者 Web App 的异步的、持久化的 Key-Value 存储系统，其实就是在 `AsyncStorage` 和 `localStorage` 外封装了一层，也支持实现自己的 Storage 系统

配置什么的直接看文档即可，但对于 `key - id` 存储的描述有点模糊，这里通过源码窥探个究竟

## 数据保存

key 和 id 都不能带 `_` 符号，不带 id 的情况下，直接使用提供的存储系统存储数据即可，带 id 的情况下则有些不同

```javascript

save(params) {
  const { key, id, data, rawData, expires = this.defaultExpires } = params;
  //..
  let dataToSave = { rawData: data };
  //...
  let now = new Date().getTime();
  if(expires !== null) {
    dataToSave.expires = now + expires;
  }
  dataToSave = JSON.stringify(dataToSave);
  if(id === undefined) {
    if(this.enableCache) {
      const cacheData = JSON.parse(dataToSave);
      this.cache[key] = cacheData;
    }
    return this.setItem(key, dataToSave);
  }else {
    //..
    return this._mapPromise.then(() => this._saveToMap({
      key,
      id,
      data: dataToSave
    }));
  }
}
```

### key - id 的形式存储

上面的 `save` 方法中，如果带 `id` 的形式存储，会使用到 `this._mapPromise` 和 `this._saveToMap`

`this._saveToMap` 是一个 `Promise` 对象，在 Storage 对象初始化的时候初始化，并记录在 `this._m` 变量

```javascript
this._mapPromise = this.getItem('map').then(map => {
  this._m = this._checkMap(map && JSON.parse(map) || {});
  // delete this._mapPromise;
});
```

原来 Storage 会在存储系统存储一个 `map` 为 key 的对象，这个对象有记录库的版本号，版本升级的时候会重置，初始化格式如下

```javascript
_initMap() {
  return {
    innerVersion: this._innerVersion, //版本号
    index: 0, //
    __keys__: {}
  };
}
```

再来分析 `this._saveToMap` 方法，关于 key - id 的形式存储的数据，新的 id 为 `key + '_' + id`

```javascript

_saveToMap(params) {
  let { key, id, data } = params
  , newId = this._getId(key, id) //key + '_' + id
  , m = this._m;
  if(m[newId] !== undefined) {
    //..更新缓存数据
    return this.setItem('map_' + m[newId], data);
  }
  if(m[m.index] !== undefined){ //m.index 默认为 0
    //loop over, delete old data
    let oldId = m[m.index];
    let splitOldId = oldId.split('_')
    delete m[oldId];
    this._removeIdInKey(splitOldId[0], splitOldId[1])
    if(this.enableCache) {
      delete this.cache[oldId];
    }
  }
  m[newId] = m.index;
  m[m.index] = newId;

  m.__keys__[key] = m.__keys__[key] || [];
  m.__keys__[key].push(id);

  //...
  let currentIndex = m.index;
  if(++m.index === this._SIZE) {
    m.index = 0;
  }
  this.setItem('map_' + currentIndex, data);
  this.setItem('map', JSON.stringify(m));
}
```

上面的代码配合例子应该容易理解一点，在初始化的情况下，如下进行两次次数据保存

```javascript
storage.save({
    key: 'test',
    id: `testId0`,
    data: {
      text: 'Hello',
    },
  });

storage.save({
      key: 'test',
      id: `testId1`,
      data: {
        text: 'World',
      },
    });
```

第一次存储后，`map` 的数据为

```javascript
{
  innerVersion: ...,  //版本号不变
  index: 1,
  test_testId0: 0,
  0: `test_testId0`,
  __keys__: {
    test: [`testId0`],
  }
}
```

而存储在 `AsyncStorage` 中的 key - value 为 `'map_0' - {rawData: 'Hello', expires: xxx}`

第二次存储后，`map` 的数据为

```javascript
{
  innerVersion: ...,  //版本号不变
  index: 2,
  test_testId0: 0,
  0: `test_testId0`,
  test_testId1: 1,
  1: `test_testId1`,
  __keys__: {
    test: [`testId0`,`testId1`],
  }
}
```

而存储在 `AsyncStorage` 中的 key - value 为 `'map_1' - {rawData: 'Wrold', expires: xxxx}`

## 数据查找

当需要查询某个 key 的 id 值的时候，就可以通过 `this.m.__keys__` 中查找，具体方法为 `getIdsForKey(key)`

load 用来读出数据，如果不指定 `id` 直接根据 key 在内存或者存储系统查找即可，比较有意思的是，如果加载的数据不存在，还可以通过指定的 `sync` 对象的 key 函数进行获取，这个还是比较有用的

```javascript
load(params) {
  let { key, id, autoSync = true, syncInBackground = true, syncParams } = params;
  return this._mapPromise.then(() => new Promise((resolve, reject) => {
    if(id === undefined) {
      return resolve(this._lookupGlobalItem({
        key, resolve, reject, autoSync, syncInBackground, syncParams
      }));
    }
    return resolve(this._lookUpInMap({
      key, id, resolve, reject, autoSync, syncInBackground, syncParams
    }));
  }));
}
```

### 通过 key 进行数据查找和 Sycn 对象

现在 cache 中进行查找，如果没有才通过 `this.getItem(key)` 方法从存储系统中查找，最后都会调用 `_loadGlobalItem` 方法，`_loadGlobalItem` 方法，检查到数据结果为空或者已经过期，如果 `autoSync` 为 `true`，会调用 `sycn` 对象的 `key` 属性所指的函数进行数据同步，如果 `syncInBackground` 为 `true`，会返回过期数据，而在后台进行数据更新

```javascript

_lookupGlobalItem(params) {
  let ret;
  let { key } = params;
  //省略内存查找
  return this.getItem(key).then(ret => this._loadGlobalItem({ ret, ...params }));
}

_loadGlobalItem(params) {
  let { key, ret, autoSync, syncInBackground, syncParams } = params;
  if(ret === null || ret === undefined) {
    if(autoSync && this.sync[key]) {
      return new Promise((resolve, reject) => this.sync[key]({ resolve, reject, syncParams }));
    }
    return Promise.reject(new NotFoundError(JSON.stringify(params)));
  }
  if (typeof ret === 'string') {
    ret = JSON.parse(ret);
    //保存在内存
  }
  let now = new Date().getTime();
  if(ret.expires < now) { //数据过期
    if (autoSync && this.sync[key]){
      if(syncInBackground) {  //syncInBackground 为 true，会返回过期数据，而在后台进行数据更新
        this.sync[key]({ syncParams });
        return Promise.resolve(ret.rawData);
      }
      return new Promise((resolve, reject) => this.sync[key]({ resolve, reject, syncParams }));
    }
    return Promise.reject(new ExpiredError(JSON.stringify(params)));
  }
  return Promise.resolve(ret.rawData);
}
```

### 通过 key-id 查找

自然地也是先在内存中查找，通过存储的流程也不难推断出查找的过程，保存在存储系统的 key 为 `map_${index}`，index 的值就需要通过 `this._m` 中查找，最后调用 `this._loadMapItem` 方法，这个方法和 `this._loadGlobalItem` 类似，数据过期可以通过 `sync` 对象去获取

```javascript

_lookUpInMap(params) {
  let m = this._m,
    ret;
  let { key, id } = params;
  let newId = this._getId(key, id); //key_id
  if(this.enableCache && this.cache[newId]) {
    ret = this.cache[newId];
    return this._loadMapItem( {ret, ...params } );
  }
  if(m[newId] !== undefined) {
    return this.getItem('map_' + m[newId]).then( ret => this._loadMapItem( {ret, ...params } ) );
  }
  return this._noItemFound( {ret, ...params } );
}
```

```javascript
_loadMapItem(params) {
  let { ret, key, id, autoSync, batched, syncInBackground, syncParams } = params;
  if(ret === null || ret === undefined) {
    return this._noItemFound(params); //可过 autoSync 为 true，则通过 sync 对象去获取
  }
  if(typeof ret === 'string'){
    ret = JSON.parse(ret);
    const { key, id } = params;
    const newId = this._getId(key, id);
    if (this.enableCache) {
      this.cache[newId] = ret;
    }
  }
  let now = new Date().getTime();
  if(ret.expires < now) {
    if(autoSync && this.sync[key]) {
      if(syncInBackground) {
        this.sync[key]({ id, syncParams });
        return Promise.resolve(ret.rawData);
      }
      return new Promise((resolve, reject) => this.sync[key]({ id, resolve, reject, syncParams }));
    }
    if(batched) {
      return Promise.resolve({ syncId: id });
    }
    return Promise.reject(new ExpiredError(JSON.stringify(params)));
  }
  return Promise.resolve(ret.rawData);
}
```

## 小结

总的来说，key-id 形式的存储可以提供了在相同的 key 值的情况下，额外提供了一个新的标识来区分数据源，但并没有想到有什么有效的应用场景？而最大缓存数量也只是对 key-id 这种存储方式生效；对于 key 的形式来说，可以说是没上限的，也缺少 LRU 特性！缓存的数据如果不自己清理，存储大小很可能会随着版本迭代而日益增加；`sync` 对象就很有用，可以在数据过期或者不存在的时候进行数据的获取，但是全局只能配置一个 `sync` 对象，不能每次查询的时候动态配置
