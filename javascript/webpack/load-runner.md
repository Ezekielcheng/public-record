### load-runner 分析

webpack 中使用 load-runner 模块进行 loader 处理

```js
runLoaders(
  {
    resource: this.resource,
    loaders: this.loaders,
    context: loaderContext,
    processResource: (loaderContext, resourcePath, callback) => {
      const resource = loaderContext.resource
      const scheme = getScheme(resource)
      hooks.readResource.for(scheme).callAsync(loaderContext, (err, result) => {
        if (err) return callback(err)
        if (typeof result !== 'string' && !result) {
          return callback(new UnhandledSchemeError(scheme, resource))
        }
        return callback(null, result)
      })
    },
  },
  (err, result) => {
    // Cleanup loaderContext to avoid leaking memory in ICs
    loaderContext._compilation =
      loaderContext._compiler =
      loaderContext._module =
      loaderContext.fs =
        undefined

    if (!result) {
      this.buildInfo.cacheable = false
      return processResult(
        err || new Error('No result from loader-runner processing'),
        null
      )
    }
    this.buildInfo.fileDependencies.addAll(result.fileDependencies)
    this.buildInfo.contextDependencies.addAll(result.contextDependencies)
    this.buildInfo.missingDependencies.addAll(result.missingDependencies)
    for (const loader of this.loaders) {
      this.buildInfo.buildDependencies.add(loader.loader)
    }
    this.buildInfo.cacheable = this.buildInfo.cacheable && result.cacheable
    processResult(err, result.result)
  }
)
```

进入 load-runner 模块中分析一下如何处理配置的 loader 以及文件的处理

#### 入口函数 runLoader

```js
exports.runLoaders = function runLoaders(options, callback) {
  // xxx 先忽略一些处理
  iteratePitchingLoaders(processOptions, loaderContext, function (err, result) {
    if (err) {
      return callback(err, {
        cacheable: requestCacheable,
        fileDependencies: fileDependencies,
        contextDependencies: contextDependencies,
        missingDependencies: missingDependencies,
      })
    }
    callback(null, {
      result: result,
      resourceBuffer: processOptions.resourceBuffer,
      cacheable: requestCacheable,
      fileDependencies: fileDependencies,
      contextDependencies: contextDependencies,
      missingDependencies: missingDependencies,
    })
  })
}
```

#### iteratePitchingLoaders 函数

```js
function iteratePitchingLoaders(options, loaderContext, callback) {
  // abort after last loader
  if (loaderContext.loaderIndex >= loaderContext.loaders.length)
    return processResource(options, loaderContext, callback)

  var currentLoaderObject = loaderContext.loaders[loaderContext.loaderIndex]

  // iterate
  if (currentLoaderObject.pitchExecuted) {
    loaderContext.loaderIndex++
    return iteratePitchingLoaders(options, loaderContext, callback)
  }

  // load loader module
  loadLoader(currentLoaderObject, function (err) {
    if (err) {
      loaderContext.cacheable(false)
      return callback(err)
    }
    var fn = currentLoaderObject.pitch
    currentLoaderObject.pitchExecuted = true
    if (!fn) return iteratePitchingLoaders(options, loaderContext, callback)

    runSyncOrAsync(
      fn,
      loaderContext,
      [
        loaderContext.remainingRequest,
        loaderContext.previousRequest,
        (currentLoaderObject.data = {}),
      ],
      function (err) {
        if (err) return callback(err)
        var args = Array.prototype.slice.call(arguments, 1)
        // Determine whether to continue the pitching process based on
        // argument values (as opposed to argument presence) in order
        // to support synchronous and asynchronous usages.
        var hasArg = args.some(function (value) {
          return value !== undefined
        })
        if (hasArg) {
          loaderContext.loaderIndex--
          iterateNormalLoaders(options, loaderContext, args, callback)
        } else {
          iteratePitchingLoaders(options, loaderContext, callback)
        }
      }
    )
  })
}
```

函数内部处理递归调用逻辑

1. 先加载 loader 的模块
   1.1 回调中获取 loader 模块中没有`pitch`时，会继续调用本身，`index++` 继续加载下一个模块
   1.2 当回调中有`pitch`时，直接调用`runSyncOrAsync`,在内部会条用`pitch`，**回调中判断流程是继续递归`normal`还是`pitch`**,
   1.2.1 如果执行结果不存在`undefined`，`index--` 加载 normalLoaders
   1.2.2 如果相反，加载 pitchLoaders

#### 加载 loader

1. 获取 loader 中导出的模块，写入`normal` `pitch` `raw`

```js
loader.normal = typeof module === 'function' ? module : module.default
loader.pitch = module.pitch
loader.raw = module.raw
```

其中 `normal` 是正常文件需要处理的函数 `pitch` 是用于阻断加载流程函数

继续上面的流程

1. 如果没有 pitch 函数导出，会调用 `processResource`,内部调用 `iterateNormalLoaders` ，此时的 index 从最后面的 loader 开始向前递归，
   1.1 如果 normal 函数没有继续往下，处理下一个 normal 函数
   1.2 如果有 normal 函数，调用参数处理函数`convertArgs`然后调用 `runSyncOrAsync` ，内部将 normal 操作的返回传入，当做递归 args，继续递归
   1.3 递归完成后，执行 webpack 调用传入的 callback
> 在pith递归过程中执行的runSyncOrAsync方法，会传递3个参数，`remainingRequest`,`previousRequest`和`data`
> 在某些情况 loader 只关心 request 后面的元数据(metadata)，并且忽略前一个 loader 的结果，pitch可以共享前面的一些数据，其中 bundle-loader中就使用了这种方式

```js
function processResource(options, loaderContext, callback) {
  // set loader index to last loader
  loaderContext.loaderIndex = loaderContext.loaders.length - 1

  var resourcePath = loaderContext.resourcePath
  if (resourcePath) {
    options.processResource(
      loaderContext,
      resourcePath,
      function (err, buffer) {
        if (err) return callback(err)
        options.resourceBuffer = buffer
        iterateNormalLoaders(options, loaderContext, [buffer], callback)
      }
    )
  } else {
    iterateNormalLoaders(options, loaderContext, [null], callback)
  }
}

function iterateNormalLoaders(options, loaderContext, args, callback) {
  if (loaderContext.loaderIndex < 0) return callback(null, args)

  var currentLoaderObject = loaderContext.loaders[loaderContext.loaderIndex]

  // iterate
  if (currentLoaderObject.normalExecuted) {
    loaderContext.loaderIndex--
    return iterateNormalLoaders(options, loaderContext, args, callback)
  }

  var fn = currentLoaderObject.normal
  currentLoaderObject.normalExecuted = true
  if (!fn) {
    return iterateNormalLoaders(options, loaderContext, args, callback)
  }

  convertArgs(args, currentLoaderObject.raw)

  runSyncOrAsync(fn, loaderContext, args, function (err) {
    if (err) return callback(err)

    var args = Array.prototype.slice.call(arguments, 1)
    iterateNormalLoaders(options, loaderContext, args, callback)
  })
}
```

#### runSyncOrAsync

```js
function runSyncOrAsync(fn, context, args, callback) {
  var isSync = true
  var isDone = false
  var isError = false // internal error
  var reportedError = false
  context.async = function async() {
    if (isDone) {
      if (reportedError) return // ignore
      throw new Error('async(): The callback was already called.')
    }
    isSync = false
    return innerCallback
  }
  var innerCallback = (context.callback = function () {
    if (isDone) {
      if (reportedError) return // ignore
      throw new Error('callback(): The callback was already called.')
    }
    isDone = true
    isSync = false
    try {
      callback.apply(null, arguments)
    } catch (e) {
      isError = true
      throw e
    }
  })
  try {
    var result = (function LOADER_EXECUTION() {
      return fn.apply(context, args)
    })()
    if (isSync) {
      isDone = true
      if (result === undefined) return callback()
      if (
        result &&
        typeof result === 'object' &&
        typeof result.then === 'function'
      ) {
        return result.then(function (r) {
          callback(null, r)
        }, callback)
      }
      return callback(null, result)
    }
  } catch (e) {
    if (isError) throw e
    if (isDone) {
      // loader is already "done", so we cannot use the callback function
      // for better debugging we print the error on the console
      if (typeof e === 'object' && e.stack) console.error(e.stack)
      else console.error(e)
      return
    }
    isDone = true
    reportedError = true
    callback(e)
  }
}
```

`runSyncOrAsync`函数兼容 loader 不论是同步还是异步执行，进行一层包装

1. loaderContext 是从 webpack 生成并传入的
2. 内部定义了 2 个方法，async 和 callback
   分析不同情况 loader 时的调用,借用官方 demo
3. 几种情况loader  
  3.1 同步 loader `isSync` 为true,如果返回是Promise对象，Promise执行完成后调用callback，否则直接调用callback
  ```js
    module.exports = function (content, map, meta) {
      return someSyncOperation(content)
    }
    //  或者使用定义的callback
    module.exports = function (content, map, meta) {
      return this.callback(null, content)
    }
  ```
  3.2 异步loader，调用loader里面async时，isSync置为false，避免进入同步判断里，进入自定义loader的流程，执行完异步后，将处理好的参数传入回调 `callback.apply(null, arguments)`
  ```js
    module.exports = function(content, map, meta) {
      var callback = this.async();
      someAsyncOperation(content, function(err, result) {
        if (err) return callback(err);
        callback(null, result, map, meta);
      });
    };
    module.exports = function(content, map, meta) {
      var callback = this.async();
      someAsyncOperation(content, function(err, result, sourceMaps, meta) {
        if (err) return callback(err);
        callback(null, result, sourceMaps, meta);
      });
    };
  ```

所以在pith执行内部如果有返回值，则直接进入normal的递归，执行normal
这里就是官方解释的执行流程

1. 没有 pitch 或者 pitch 执行无返回值

```txt
|- a-loader `pitch`
  |- b-loader `pitch`
    |- c-loader `pitch`
      |- requested module is picked up as a dependency
    |- c-loader normal execution
  |- b-loader normal execution
|- a-loader normal execution
```

2. 存在 pitch 且有返回值

```txt
|- a-loader `pitch`
  |- b-loader `pitch` returns a module
|- a-loader normal execution

```

#### 参考资料
[*loader api*](https://www.webpackjs.com/api/loaders/)  
[*bundle-loader*](https://github.com/webpack-contrib/bundle-loader/blob/master/index.js)