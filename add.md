
## app

`vue add [pluginName]`

安装插件 并在已经创建的项目中调用其生成器

> 不如 `vue add buefy`

``` js
const chalk = require('chalk')
const invoke = require('./invoke')
const { loadOptions } = require('./options')
const { installPackage } = require('./util/installDeps') // 下载
const { resolveModule } = require('./util/module')
const {
  log, // 记录
  error, // 错误
  hasYarn, // 有 yarn 
  stopSpinner, // 停止 命令行转圈圈
  resolvePluginId // 请求 插件 @vue/cli-plugin-buefy / vue-cli-plugin-buefy
} = require('@vue/cli-shared-utils')

async function add (pluginName, options = {}, context = process.cwd()) {
    // 第一步 下载 插件
  const packageName = resolvePluginId(pluginName)

  log()
  log(`📦  Installing ${chalk.cyan(packageName)}...`)
  log()

  const packageManager = loadOptions().packageManager || (hasYarn() ? 'yarn' : 'npm')
  await installPackage(context, packageManager, null, packageName)

  stopSpinner()

  log()
  log(`${chalk.green('✔')}  Successfully installed plugin: ${chalk.cyan(packageName)}`)
  log()

// 第二步 调用 插件/generator 模版
  const generatorPath = resolveModule(`${packageName}/generator`, context)
  if (generatorPath) { // 有
    invoke(pluginName, options, context)
  } else { // 没有
    log(`Plugin ${packageName} does not have a generator to invoke`)
  }
}

module.exports = (...args) => {
  return add(...args).catch(err => {
    error(err)
    if (!process.env.VUE_CLI_TEST) {
      process.exit(1)
    }
  })
}

```

#### [invoke](./invoke.md)

> 模版触发
