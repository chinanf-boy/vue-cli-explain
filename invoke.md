## invoke

`vue add 第二步/ vue invoke`

在已创建的项目中调用插件的生成器

比如 `vue invoke buefy`

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
    completeCbs: createCompleteCbs
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
    await installDeps(context, packageManager)
  }

// 经过 生成器运行, 因为是地址传输, 可以改变 createCompleteCbs数组
// 触发 完成后函数

  if (createCompleteCbs.length) {
    logWithSpinner('⚓', `Running completion hooks...`)
    for (const cb of createCompleteCbs) {
      await cb()
    }
  }

  stopSpinner()

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