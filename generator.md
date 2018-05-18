## Generator

被 `vue invoke/vue create` 使用的 生成器

`vue-cli/packages/@vue/cli/lib/Generator.js`

---

###  create

被 `vue create` 使用

<details>

`create.js`


``` js
// 初始化
    const generator = new Generator(context, {
      pkg, // 传入 package
      plugins,
      completeCbs: createCompleteCbs
    })
// 运行
    
    await generator.generate({
      extractConfigFiles: preset.useConfigFiles
    })
```

</details>

### invoke

被 `vue invoke` 使用

<details>

`invoke.js`

``` js
// 初始化
 const generator = new Generator(context, {
    pkg,
    plugins: [plugin],
    files: await readFiles(context),
    completeCbs: createCompleteCbs
  })
```

``` js
// 运行
  await generator.generate({
    extractConfigFiles: true,
    checkExisting: true
  })
```
</details>

---

### Generator

`vue-cli/packages/@vue/cli/lib/Generator.js`

### constructor

1. 初始化

``` js
  constructor (context, {
    pkg = {},
    plugins = [],
    completeCbs = [],
    files = {}
  } = {}) {
    this.context = context
    this.plugins = plugins
    this.originalPkg = pkg
    this.pkg = Object.assign({}, pkg)
    this.completeCbs = completeCbs

    // for conflict resolution
    this.depSources = {}
    // virtual file tree
    this.files = files
    this.fileMiddlewares = []
    this.postProcessFilesCbs = []
    // exit messages
    this.exitLogs = []

    const cliService = plugins.find(p => p.id === '@vue/cli-service')
    const rootOptions = cliService && cliService.options
    // apply generators from plugins
    plugins.forEach(({ id, apply, options }) => {
// id : 插件名, apply: 插件/generator 代码, options: 插件自定义配置
     
      const api = new GeneratorAPI(id, this, options, rootOptions || {})
      // 运行 插件/generator 代码
      apply(api, options, rootOptions)
    })
  }
```

- `GeneratorAPI`

提供给插件, 能控制生成器内部的接口API

- `apply(api, options, rootOptions)`

在这里, 将接口和配置, 还有主配置给予 `插件/generator 代码`

这样说可能还有点混乱, 我们取个例子, 说明下吧

[https://github.com/buefy/vue-cli-plugin-buefy](./plugin-buefy.md#generator)

### generate

2. 生成

``` js
  async generate ({
    extractConfigFiles = false,
    checkExisting = false
  } = {}) {
    // 在应用插件进行比较之前保存文件系统
    const initialFiles = Object.assign({}, this.files)
    // 从package.json中提取配置到专用文件中。
    this.extractConfigFiles(extractConfigFiles, checkExisting)
    // 等待文件解析
    await this.resolveFiles()
    // set package.json
    this.sortPkg()
    this.files['package.json'] = JSON.stringify(this.pkg, null, 2)
    // write/update file tree to disk
    await writeFileTree(this.context, this.files, initialFiles)
  }
```


### extractConfigFiles

整理插件中定义的 有关诸如 `babel` 之类的配置

<details>

``` js
// checkExisting 为 true , 代表是需要config合并以及写入
  extractConfigFiles (extractAll, checkExisting) {
    // 经过 插件 调用 generator api , generator 的 pkg
    // 需要重新修整
    const extract = key => {
      if (
        configTransforms[key] &&
        this.pkg[key] &&
        // 如果该字段存在于原始package.json中，则不提取
        !this.originalPkg[key]
      ) {
        const value = this.pkg[key]
        // value 定义的 相关 babel eslint 之类的
        // package.json 的配置 会和默认配置 组合

        const transform = configTransforms[key]
// js 和 json , yaml 三种书写格式 转换总是要注意的
        const res = transform( // 合并 插件参数 与项目默认参数
          value,
          checkExisting,
          this.context
        )
        const { content, filename } = res // 重新写好的文件名和内容
        this.files[filename] = content
        delete this.pkg[key]
      }
    }
    if (extractAll) {
      for (const key in this.pkg) {
        extract(key)
      }
    } else if (!process.env.VUE_CLI_TEST) {
      // by default, always extract vue.config.js
      extract('vue')
    }
  }
```

- `configTransforms[key]` 

这个 `boolean` 的 true , 其实也就那几个

``` js
// configTransforms
// 只需合并这几个原配置的参数
module.exports = {
  vue: makeJSTransform('vue.config.js'),
  babel: makeJSONTransform('.babelrc'),
  postcss: makeMutliExtensionJSONTransform('.postcssrc', true),
  eslintConfig: makeMutliExtensionJSONTransform('.eslintrc', true),
  jest: makeJSTransform('jest.config.js')
}

// 从 make js transform
// 可以看出 js json 的格式 其实也是需要注意⚠️的点
```
</details>

### resolveFiles

将插件中需要放入项目的文件 `ejs` 解析并写入

<details>

``` js
  async resolveFiles () {
    const files = this.files
    for (const middleware of this.fileMiddlewares) {
      await middleware(files, ejs.render)
    }
    // normalize paths // 净化 window/linux/mac
    Object.keys(files).forEach(file => {
      const normalized = slash(file)
      if (file !== normalized) {
        files[normalized] = files[file]
        delete files[file]
      }
    })
    // 在 文件中间件 解析后, 对所有文件的钩子
    //
    for (const postProcess of this.postProcessFilesCbs) {
      await postProcess(files)
    }
    debug('vue:cli-files')(this.files)
  }
```

</details>

### sortPkg

放入 默认 package.json 配置

<details>

``` js
  sortPkg () {
    // ensure package.json keys has readable order
    this.pkg.dependencies = sortObject(this.pkg.dependencies)
    this.pkg.devDependencies = sortObject(this.pkg.devDependencies)
    this.pkg.scripts = sortObject(this.pkg.scripts, [
      'serve',
      'build',
      'test',
      'e2e',
      'lint',
      'deploy'
    ])
    this.pkg = sortObject(this.pkg, [
      'name',
      'version',
      'private',
      'scripts',
      'dependencies',
      'devDependencies',
      'vue',
      'babel',
      'eslintConfig',
      'prettier',
      'postcss',
      'browserslist',
      'jest'
    ])

    debug('vue:cli-pkg')(this.pkg)
  }
```

</details>


### writeFileTree

`vue-cli/packages/@vue/cli/lib/util/writeFileTree.js`

对 现有文件 写入

> (this.context, this.files, initialFiles)

<details>

``` js
function deleteRemovedFiles (directory, newFiles, previousFiles) {
  // get all files that are not in the new filesystem and are still existing
  const filesToDelete = Object.keys(previousFiles)
    .filter(filename => !newFiles[filename])

  // delete each of these files
  return Promise.all(filesToDelete.map(filename => {
    return fs.unlink(path.join(directory, filename))
  }))
}

module.exports = async function writeFileTree (dir, files, previousFiles) {
  if (process.env.VUE_CLI_SKIP_WRITE) {
    return
  }
  if (previousFiles) {
    await deleteRemovedFiles(dir, files, previousFiles)
  }
  return Promise.all(Object.keys(files).map(async (name) => {
    const filePath = path.join(dir, name)
    await fs.ensureDir(path.dirname(filePath))
    await fs.writeFile(filePath, files[name])
  }))
}

```

</details>
