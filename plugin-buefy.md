### buefy

Lightweight UI components for Vue.js based on Bulma 

基于 移动端UI Bulma , 给 Vue 提供组件的 buefy

---

## vue-cli plugin

plugin: https://github.com/buefy/vue-cli-plugin-buefy

> Vue CLI 3.x plugin to add buefy to your Vue Project 

如何初始化一个带有 buefy 项目

这就是 `vue-cli-plugin-buefy` 所做得

> 🧠, 接下来有两个概念, `项目` 和 `插件` 是 两个不同的项目

## vue-cli-plugin-buefy

### prompts

> `vue-cli-plugin-buefy/generator/prompts.js`

插件配置

``` js
module.exports = [{
  name: `addStyle`,
  type: `list`,
  message: `Add Buefy style?`,
  choices: ['none', 'css', 'scss'],
  default: 0
},
{
  name: `materialDesignIcons`,
  type: `confirm`,
  message: 'Include Material Design Icons?',
  default: false
},
{
  name: `fontAwesomeIcons`,
  type: 'confirm',
  message: 'Include Font Awesome Icons?',
  default: false
}]

```

### generator

> `vue-cli-plugin-buefy/generator/index.js`

生成器代码

``` js
// api 控制内部 generator 接口
module.exports = (api, options) => {
  const pkg = {  // 要放入 pkg
    dependencies: {
      'buefy': '^0.6.3'
    }
  }

  // extend package
  api.extendPackage(pkg) // 要生成器加入 pkg

  let buefyLines = `\nimport Buefy from 'buefy'`
  if (options.addStyle === 'css') {
    buefyLines += `\nimport 'buefy/lib/buefy.css'`
  } else if (options.addStyle === 'scss') {


    api.render('./templates/style') // 从 插件中获得, 并等待 ejs.render 的调用

    buefyLines += `\nimport './assets/scss/app.scss'`
  }

  // use Buefy
  buefyLines += `\n\nVue.use(Buefy)`
  // 从这些 import 什么的可以看出, 是要放入 main.js 中的

```

### 完成下载后

- [onCreateComplete](#oncreatecomplete)

``` js
  api.onCreateComplete(() => {
    // inject to main.js
    const fs = require('fs')

    // 可以获得 项目的文件路径
    const mainPath = api.resolve('./src/main.js')

    // get content
    let contentMain = fs.readFileSync(mainPath, { encoding: 'utf-8' })
    const lines = contentMain.split(/\r?\n/g).reverse()

    // inject import
    const lastImportIndex = lines.findIndex((line) => line.match(/^import/))
    lines[lastImportIndex] += buefyLines
// 注入
    // modify app
    contentMain = lines.reverse().join('\n')
    fs.writeFileSync(mainPath, contentMain, { encoding: 'utf-8' })
// 重写


// 再一次, 我们又拿到了, 项目中的`public/index.html`
// 注入
// 重写
    if (options.materialDesignIcons || options.fontAwesomeIcons) {
      const indexPath = api.resolve('./public/index.html')
      let contentIndex = fs.readFileSync(indexPath, { encoding: 'utf8' })

      const lines = contentIndex.split(/\r?\n/g).reverse()
      const lastLink = lines.findIndex((line) => line.match(/^\s*<link/))

      if (options.materialDesignIcons) {
        lines[lastLink] += `\n<link rel="stylesheet" href="//cdn.materialdesignicons.com/2.0.46/css/materialdesignicons.min.css">`
      }
      if (options.fontAwesomeIcons) {
        lines[lastLink] += `\n<script defer src="https://use.fontawesome.com/releases/v5.0.8/js/all.js" integrity="sha384-SlE991lGASHoBfWbelyBPLsUlwY1GwNDJo3jSJO04KZ33K2bwfV9YBauFfnzvynJ" crossorigin="anonymous"></script>`
      }

      contentIndex = lines.reverse().join('\n')
      fs.writeFileSync(indexPath, contentIndex, { encoding: 'utf8' })
    }
  })
}

// Done
```

那么, 如果我们使用这个插件, 我们的 `package.json main.js index.html style`

---

下面我们说说我们用到的 接口

---

### api

本次使用的接口函数

`vue-cli/packages/@vue/cli/lib/GeneratorAPI.js`

1. extendPackage

> 加入 项目 package.json 行列

<details>

``` js
 /**
   * Extend the package.json of the project.
   * Nested fields are deep-merged unless `{ merge: false }` is passed.
   * Also resolves dependency conflicts between plugins.
   * Tool configuration fields may be extracted into standalone files before
   * files are written to disk.
   *
   * @param {object | () => object} fields - Fields to merge.
   */
  extendPackage (fields) {
    const pkg = this.generator.pkg
    const toMerge = isFunction(fields) ? fields(pkg) : fields
    for (const key in toMerge) {
      const value = toMerge[key]
      const existing = pkg[key]
      if (isObject(value) && (key === 'dependencies' || key === 'devDependencies')) {
        // use special version resolution merge
        pkg[key] = mergeDeps(
          this.id,
          existing || {},
          value,
          this.generator.depSources
        )
      } else if (!(key in pkg)) {
        pkg[key] = value
      } else if (Array.isArray(value) && Array.isArray(existing)) {
        pkg[key] = existing.concat(value)
      } else if (isObject(value) && isObject(existing)) {
        pkg[key] = merge(existing, value)
      } else {
        pkg[key] = value
      }
    }
  }
```

</details>

2. render

> 使用 `ejs.render` 渲染 模版, 并保存在 生成器的files

<details>

``` js
/**
   * Render template files into the virtual files tree object.
   *
   * @param {string | object | FileMiddleware} source -
   *   Can be one of:
   *   - relative path to a directory;
   *   - Object hash of { sourceTemplate: targetFile } mappings;
   *   - a custom file middleware function.
   * @param {object} [additionalData] - additional data available to templates.
   * @param {object} [ejsOptions] - options for ejs.
   */
  render (source, additionalData = {}, ejsOptions = {}) {
    const baseDir = extractCallDir() // 取出被调用时的 目录 也就是插件/generator的目录
    if (isString(source)) {
      source = path.resolve(baseDir, source) // 插件的目录/文件
      this._injectFileMiddleware(async (files) => { // files 生产器的缓存
        const data = this._resolveData(additionalData)
        const _files = await globby(['**/*'], { cwd: source }) // 从插件的目标-文件
        for (const rawPath of _files) {

          // 这里需要注意的是 globby 匹配 , 这个 rawPath 是去掉 source的
          // 意思就是 
          // 如果 source == '~/desktop/vue-create-project/node_modules/buefy/generator/ **
          // rawPath == '**' 只带有后面的目录与文件名

          let filename = path.basename(rawPath)
          // _* => .*
          if (filename.charAt(0) === '_') {
            filename = `.${filename.slice(1)}`
          }
          const targetPath = path.join(path.dirname(rawPath), filename)
          const sourcePath = path.resolve(source, rawPath)
          const content = renderFile(sourcePath, data, ejsOptions)
          // only set file if it's not all whitespace, or is a Buffer (binary files)
          if (Buffer.isBuffer(content) || /[^\s]/.test(content)) {
            files[targetPath] = content
          }
        }
      })
    } else if (isObject(source)) {
      this._injectFileMiddleware(files => {
        const data = this._resolveData(additionalData)
        for (const targetPath in source) {
          const sourcePath = path.resolve(baseDir, source[targetPath])
          const content = renderFile(sourcePath, data, ejsOptions)
          if (Buffer.isBuffer(content) || content.trim()) {
            files[targetPath] = content
          }
        }
      })
    } else if (isFunction(source)) {
      this._injectFileMiddleware(source)
    }
  }
```

- _injectFileMiddleware

files 增加

``` js
  /**
   * Inject a file processing middleware.
   *
   * @private
   * @param {FileMiddleware} middleware - A middleware function that receives the
   *   virtual files tree object, and an ejs render function. Can be async.
   */
  _injectFileMiddleware (middleware) {
    this.generator.fileMiddlewares.push(middleware)
  }
```

### extractCallDir

插件中 `api.render` 调用的目录

``` js
function extractCallDir () {
  // extract api.render() callsite file location using error stack
  const obj = {}
  Error.captureStackTrace(obj)
  const callSite = obj.stack.split('\n')[3]
  const fileName = callSite.match(/\s\((.*):\d+:\d+\)$/)[1]
  return path.dirname(fileName)
}
```

### renderFile

用 `ejs.render` 解析模版文件

``` js
function renderFile (name, data, ejsOptions) {
  if (isBinary.sync(name)) {
    return fs.readFileSync(name) // return buffer
  }
  const template = fs.readFileSync(name, 'utf-8')

  // custom template inheritance via yaml front matter.
  // ---
  // extend: 'source-file'
  // replace: !!js/regexp /some-regex/
  // OR
  // replace:
  //   - !!js/regexp /foo/
  //   - !!js/regexp /bar/
  // ---
  const parsed = yaml.loadFront(template)
  const content = parsed.__content
  let finalTemplate = content.trim() + `\n`
  if (parsed.extend) {
    const extendPath = path.isAbsolute(parsed.extend)
      ? parsed.extend
      : resolve.sync(parsed.extend, { basedir: path.dirname(name) })
    finalTemplate = fs.readFileSync(extendPath, 'utf-8')
    if (parsed.replace) {
      if (Array.isArray(parsed.replace)) {
        const replaceMatch = content.match(replaceBlockRE)
        if (replaceMatch) {
          const replaces = replaceMatch.map(m => {
            return m.replace(replaceBlockRE, '$1').trim()
          })
          parsed.replace.forEach((r, i) => {
            finalTemplate = finalTemplate.replace(r, replaces[i])
          })
        }
      } else {
        finalTemplate = finalTemplate.replace(parsed.replace, content.trim())
      }
    }
  }

  return ejs.render(finalTemplate, data, ejsOptions)
}
```
</details>


#### onCreateComplete

3. 增加完成勾子

<details>

``` js
  /**
   * Push a callback to be called when the files have been written to disk.
   *
   * @param {function} cb
   */
  onCreateComplete (cb) {
    this.generator.completeCbs.push(cb)
  }
```

运行

[invoke.js](./invoke.md#createcompletecbs)

</details>

4. resolve

返回项目目录文件-真实路径

<details>

``` js
  resolve (_path) {
    return path.resolve(this.generator.context, _path)
  }
```

</details>