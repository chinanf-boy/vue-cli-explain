## invoke

`vue add 第二步/ vue invoke`

在已创建的项目中调用插件的生成器

比如 `vue invoke buefy`

---

说一说思路先, 在[上一步](./add.md)下载完 `插件`后

我们来到了调用的环节, 此处需要插件作者配合编写

1. 我本身要有一个存储

比如 [Generator](./generator.md) 先作为一个存储-存在

既然是存储, 自然有一系列的 

``` js
function Generator(options){
  this.pkg = options.pkg // package.json
  this.context = options.pwd // process.cwd()
  // ....等等
}
```

> `Generator == 存储`

2. 拿到插件作者的插件定义

我们有了, 存储的位置, 我们开始把插件作者定义的内容放进去

- [插件/generator](#plugingenerator)

> 一个是 代码本身的内容 `code`

- [插件/prompts](#pluginprompts)

> 一个是 代码需要的选项配置 `config`

3. 给予插件作者权力

我们有`Generator`, 也有了插件作者输入的内容, 但这些内容, 可以说一点用都没用

因为我们没有放出我们`Generator-接口`,那么插件作者的代码充其量只是漫长的 `string` 而已

那么什么为`接口`

比如我们要改变 `Generator` 中的 `this.pkg`, 我们需要 

``` js
let generator = new Generator(options) // 实例化存储

// 将实例的 generator 参数带入
function API(generator){   // ==> 一个接口总部原型
  this.generator = generator
}

API.prototype.setPkg = function(obj){ // 给插件作者的接口
  this.generator.pkg = obj // 
}

let useAPI = new API(generator) // 实例化接口总部

```

那么插件作者方面, 被给予的权力

``` js
// 插件/generator 代码
// api 就是被实例化 具有改变 generator 的 接口总部
module.exports = (api, options) =>{
  api.setPkg(//....)
}

```

[vue-cli-接口使用 buefy插件](./plugin-buefy.md)

[vue-cli-接口定义 GeneratorAPI](./generator.md#constructor)

---

## invoke.js

`vue-cli/packages/@vue/cli/lib/invoke.js`

### require

``` js
const Generator = require('./Generator') 
const { loadOptions } = require('./options') // 单次运行的全局选项
const { installDeps } = require('./util/installDeps')
// 下载
const { loadModule } = require('./util/module')
// 确认 模块真伪, 并返回文件内容
const {
  log, // 简写 输出
  error, // 错误输出
  hasYarn, // 确认 yarn
  hasGit, // 确认 git
  logWithSpinner, // 开始转
  stopSpinner, // 停止转
  resolvePluginId // 搜寻 简短插件名-🆔, 函数内部加前缀, 
// 是否存在   
// 1. vue-cli-plugin-🆔, @vue/cli-plugin-🆔   @bar/vue-cli-plugin-🆔 直接返回
// 2. @vue/🆔, @bar/🆔
// 变 上面 
// 3。 🆔
// 变 `vue-cli-plugin-${🆔}`



} = require('@vue/cli-shared-utils')

```

### readFiles

文件获取,返回 {文件名:文件路径,...}

``` js
async function readFiles (context) { // 搜寻 文件配上 匹配器-globby 
  const files = await globby(['**'], {
    cwd: context,
    onlyFiles: true,
    gitignore: true,
    ignore: ['**/node_modules/**']
  })
  const res = {}
  for (const file of files) {
    const name = path.resolve(context, file)
    res[file] = isBinary.sync(name)
      ? fs.readFileSync(name)
      : fs.readFileSync(name, 'utf-8')
  }
  return res // {fileName:filePath,...}
}

```

### 1. invoke

____1.0____ 主函数

``` js
async function invoke (pluginName, options = {}, context = process.cwd()) {
  delete options._
  const pkgPath = path.resolve(context, 'package.json')  // 命令行下
  const isTestOrDebug = process.env.VUE_CLI_TEST || process.env.VUE_CLI_DEBUG
  // 如果测试和调试, 就不会真下载

  if (!fs.existsSync(pkgPath)) { // 纠错
    throw new Error(`package.json not found in ${chalk.yellow(context)}`)
  }

  const pkg = require(pkgPath)

  // attempt to locate the plugin in package.json
  const findPlugin = deps => {
    if (!deps) return
    let name
    // official
    if (deps[(name = `@vue/cli-plugin-${pluginName}`)]) {
      return name
    }
    // full id, scoped short, or default short
    if (deps[(name = resolvePluginId(pluginName))]) {
      return name
    }
  }

  const id = findPlugin(pkg.devDependencies) || findPlugin(pkg.dependencies)
  if (!id) {
    throw new Error(
      `Cannot resolve plugin ${chalk.yellow(pluginName)} from package.json. ` +
        `Did you forget to install it?`
    )
  }

```

### pluginGenerator

__1.1__ 生成内容载入

``` js
  const pluginGenerator = loadModule(`${id}/generator`, context)
  if (!pluginGenerator) {
    throw new Error(`Plugin ${id} does not have a generator.`)
  }

```

### pluginPrompts

__1.2__ 个性化, 插件-选项选择

``` js
  // resolve options if no command line options are passed, and the plugin
  // contains a prompt module.
  if (!Object.keys(options).length) {
      // 如果没有其他选项, 如果项目包含了 prompts, 命令行-选项选择触发
    const pluginPrompts = loadModule(`${id}/prompts`, context)
    if (pluginPrompts) {
      options = await inquirer.prompt(pluginPrompts)
    }
  }

```

- [prompts](./plugin-buefy.md#prompts)

### generator

__1.3__ 生成器运行

``` js
  const plugin = {
    id,
    apply: pluginGenerator,
    options
  }

  const createCompleteCbs = []
  const generator = new Generator(context, {
    pkg,
    plugins: [plugin],
    files: await readFiles(context),
    completeCbs: createCompleteCbs // 地址传输
  })

  log()
  logWithSpinner('🚀', `Invoking generator for ${id}...`)
  await generator.generate({
    extractConfigFiles: true,
    checkExisting: true
  })

```

- [Generator](./generator.md)

> 生成器

- [readFiles](#readfiles)



---

### installDeps

__1.4__ 查看运行生成器后, 相关下载依赖是否发生变化

``` js
  const newDeps = generator.pkg.dependencies
  const newDevDeps = generator.pkg.devDependencies
  const depsChanged =
    JSON.stringify(newDeps) !== JSON.stringify(pkg.dependencies) ||
    JSON.stringify(newDevDeps) !== JSON.stringify(pkg.devDependencies)

  if (!isTestOrDebug && depsChanged) {
    logWithSpinner('📦', `Installing additional dependencies...`)
    const packageManager =
      loadOptions().packageManager || (hasYarn() ? 'yarn' : 'npm')
    await installDeps(context, packageManager) // 直接 命令输入 yarn/npm 就可以下载
  }

```

### createCompleteCbs

``` js
// 经过 生成器运行, 因为是 地址传输, 可以改变 createCompleteCbs数组
// 触发 完成后函数

  if (createCompleteCbs.length) {
    logWithSpinner('⚓', `Running completion hooks...`)
    for (const cb of createCompleteCbs) {
      await cb()
    }
  }

  stopSpinner() // 停止转圈圈

```

### git

__1.5__ 运行git帮一下

``` js
  log()
  log(`   Successfully invoked generator for plugin: ${chalk.cyan(id)}`)
  if (!process.env.VUE_CLI_TEST && hasGit()) {
      // 子进程 运行下
      // git 有关索引中文件的信息
    const { stdout } = await execa('git', [
      'ls-files',
      '--exclude-standard',
      '--modified',
      '--others'
    ])
    if (stdout.trim()) {
      log(`   The following files have been updated / added:\n`)
      log(
        chalk.red(
          stdout
            .split(/\r?\n/g)
            .map(line => `     ${line}`)
            .join('\n')
        )
      )
      log()
    }
  }
  log(
    `   You should review these changes with ${chalk.cyan(
      `git diff`
    )} and commit them.`
  )
  log()

```

### generator

__1.6__ `generator` 最后信息输出

``` js
  generator.printExitLogs()
}

```

#### printExitLogs

`vue-cli/packages/@vue/cli/lib/Generator.js`

错误统计

``` js
// 这个 错误信息 统计的 好像没有在哪看到使用
// 应该是待续功能
  printExitLogs () {
    if (this.exitLogs.length) {
      this.exitLogs.forEach(({ id, msg, type }) => {
        const shortId = toShortPluginId(id)
        const logFn = logTypes[type]
        if (!logFn) {
          logger.error(`Invalid api.exitLog type '${type}'.`, shortId)
        } else {
          logFn(msg, msg && shortId)
        }
      })
      logger.log()
    }
  }
```

### exports

__1.7__ 导出

``` js
module.exports = (...args) => {
  return invoke(...args).catch(err => {
    error(err)
    if (!process.env.VUE_CLI_TEST) {
      process.exit(1)
    }
  })
}

```