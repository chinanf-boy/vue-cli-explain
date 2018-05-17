## create

`bin/vue.js`

``` js
program
  .command('create <app-name>')
  .description('创建一个由vue-cli-service支持的新项目“)
  .option('-p，--preset <presetName>'，'跳过提示并使用已保存或远程预设')
  .option('-d，--default'，'跳过提示并使用默认预设')
  .option('-i，--inlinePreset <json>'，'跳过提示并使用内嵌的JSON字符串作为预设')
  .option('-m，--packageManager <command>'，'安装时使用指定的npm客户dependencies')
  .option('-r，--registry <url>'，'安装依赖关系时使用指定的npm注册 (only for npm)')
  .option('-g，--git [message]'，'强制/跳过git初始化，可以指定初始commit message')
  .option('-f，--force'，'覆盖目标目录（如果存在的话）')
  .option('-c，--clone'，'获取远程预设时使用git clone')
  .option('-x，--proxy'，'创建项目时使用指定的代理')
  .action((name, cmd) => {
    require('../lib/create')(name, cleanArgs(cmd)) // <====
  })
```

- cleanArgs 

> 变好点, 过滤不正确命令

---

### 1. lib/create

``` js
const inquirer = require('inquirer')
const Creator = require('./Creator')
const clearConsole = require('./util/clearConsole')
const { error, stopSpinner } = require('@vue/cli-shared-utils')
const validateProjectName = require('validate-npm-package-name')

async function create (projectName, options) {
  if (options.proxy) {
    process.env.HTTP_PROXY = options.proxy // 代理
  }

  const inCurrent = projectName === '.'
  const name = inCurrent ? path.relative('../', process.cwd()) : projectName // 命令目录作为项目路径, 如果 '.'
  const targetDir = path.resolve(projectName || '.')

  const result = validateProjectName(name) // 验证
  if (!result.validForNewPackages) {
    console.error(chalk.red(`Invalid project name: "${projectName}"`))
    result.errors && result.errors.forEach(err => {
      console.error(chalk.red(err))
    })
    process.exit(1)
  }

```

- [inquirer](https://github.com/SBoudrias/Inquirer.js#examples)

> 终端-选项选择

- [rimraf](https://github.com/isaacs/rimraf)

> 删除

- [validateProjectName](https://github.com/npm/validate-npm-package-name)

> 给定的字符串是一个可接受的npm包名称？

``` js
  if (fs.existsSync(targetDir)) { // 同步验证存在目录
    if (options.force) { // 
      rimraf.sync(targetDir) // 直接删掉目录下的东西
    } else {
      await clearConsole() // 刷一刷文字 或 提示更新
      if (inCurrent) {
        const { ok } = await inquirer.prompt([ // 选择项
          {
            name: 'ok',
            type: 'confirm',
            message: `Generate project in current directory?`
          }
        ])
        if (!ok) {
          return
        }
      } else {
        const { action } = await inquirer.prompt([
          {
            name: 'action',
            type: 'list',
            message: `Target directory ${chalk.cyan(targetDir)} already exists. Pick an action:`,
            choices: [
              { name: 'Overwrite', value: 'overwrite' },
              { name: 'Merge', value: 'merge' },
              { name: 'Cancel', value: false }
            ]
          }
        ])
        if (!action) {
          return
        } else if (action === 'overwrite') {
          rimraf.sync(targetDir)
        }
      }
    }
  }
    // ⬆️对 目录是否存在东西 进行-用户选择处理

```

> 下面我们开始, 对`要什么`配置的定义

``` js
  const promptModules = [
    'babel',
    'typescript',
    'pwa',
    'router',
    'vuex',
    'cssPreprocessors',
    'linter',
    'unit',
    'e2e'
  ].map(file => require(`./promptModules/${file}`)) // 返回 带有命令解析-参数 的函数 (cli) =>{}

  const creator = new Creator(name, targetDir, promptModules)
  await creator.create(options)
}

```


``` js
module.exports = (...args) => {
  create(...args).catch(err => {
    stopSpinner(false) // do not persist
    error(err)
    process.exit(1)
  }) // 增加 错误信息❌
}

```


- [Creator]()

- [creator.create]()

### 2. Creator constructor

``` js
module.exports = class Creator {
  constructor (name, context, promptModules) {
    this.name = name // 项目名称
    this.context = process.env.VUE_CLI_CONTEXT = context // 目标目录
    const { presetPrompt, featurePrompt } = this.resolveIntroPrompts() // <==== ⚠️
    this.presetPrompt = presetPrompt
    this.featurePrompt = featurePrompt
    this.outroPrompts = this.resolveOutroPrompts() // <=== ⚠️
    this.injectedPrompts = [] // 
    this.promptCompleteCbs = []
    this.createCompleteCbs = []

    this.run = this.run.bind(this)

    const promptAPI = new PromptModuleAPI(this) // < ==== 🧠
    promptModules.forEach(m => m(promptAPI)) // < ==== 🧠
  }
```

- ⚠️ 

其实给用户的选项还没有完成 比如用 `vuex | ts/js | ...`

> `presetPrompt`: 默认的, `featurePrompt`: 用户可以自定义

``` bash
? Please pick a preset: (Use arrow keys)
❯ default (babel, eslint) 
  Manually select features 

# 当然在这里还没有开始运行
```



---

- 🧠 

`PromptModuleAPI` 提供接口 用来 改变 

- **this.featurePrompt.choices** `push` 

> 特性选项

```
? Check the features needed for your project: (Press <space> to select, <a> to toggle all, <i> to invert s
election)
❯◯ TypeScript
 ◯ Progressive Web App (PWA) Support
 ◯ Router
 ◯ Vuex
 ◯ CSS Pre-processors
 ◯ Linter / Formatter
 ◯ Unit Testing
 ◯ E2E Testing
```

- **this.injectedPrompts** `push/find`

> 各个「vuex|ts|...」相关特性配置-选项

> 比如 `CSS Pre-processors`

```
? Pick a CSS pre-processor (PostCSS, Autoprefixer and CSS Modules are supported by default): (Use arrow ke
ys)
❯ SCSS/SASS
  LESS
  Stylus
```

- **this.promptCompleteCbs** `push`

> 选项完成后✅-运行函数

一般是选择性添加到 `pcakage.json` 中的 安装列表

---

现在我们有了, 自定义的完整配置, 供给用户选择

---

### 3. Creator create

> 只说-主线路

现在我们开始, `preset` 几种选择 { 用户|默认-preset|内置|什么都不选}的可用性

``` js
  async create (cliOptions = {}) {
    const isTestOrDebug = process.env.VUE_CLI_TEST || process.env.VUE_CLI_DEBUG
    const { run, name, context, createCompleteCbs } = this

    let preset
    if (cliOptions.preset) {
      // vue create foo --preset bar
      // 用户
      preset = await this.resolvePreset(cliOptions.preset, cliOptions.clone)
    } else if (cliOptions.default) {
      // vue create foo --default
      // 默认-preset
      preset = defaults.presets.default
    } else if (cliOptions.inlinePreset) {
      // vue create foo --inlinePreset {...}
      // 内置
      try {
        preset = JSON.parse(cliOptions.inlinePreset)
      } catch (e) {
        error(`CLI inline preset is not valid JSON: ${cliOptions.inlinePreset}`)
        process.exit(1)
      }
    } else {
      // 什么都不选
      // 也就是进入, 命令行提供-项目组成-选择
      preset = await this.promptAndResolvePreset()
    }
```

``` js
    // clone before mutating
    preset = cloneDeep(preset)
    // 注入 core service
    preset.plugins['@vue/cli-service'] = Object.assign({
      projectName: name
    }, preset)

    const packageManager = ( // 用什么下载？？
      cliOptions.packageManager ||
      loadOptions().packageManager ||
      (hasYarn() ? 'yarn' : 'npm')
    )
```

信息输出

``` js
    await clearConsole()
    logWithSpinner(`✨`, `Creating project in ${chalk.yellow(context)}.`)

```

`package.json`构建

``` js
    // get latest CLI version
    const { latest } = await getVersions()
    // generate package.json with plugin dependencies
    const pkg = {
      name,
      version: '0.1.0',
      private: true,
      devDependencies: {}
    }
    const deps = Object.keys(preset.plugins)
    deps.forEach(dep => {
      pkg.devDependencies[dep] = preset.plugins[dep].version ||
        (/^@vue/.test(dep) ? `^${latest}` : `latest`)
    }) // 组合
    // write package.json
    await writeFileTree(context, {
      'package.json': JSON.stringify(pkg, null, 2)
    })
```

初始化git

``` js
    // intilaize git repository before installing deps
    // so that vue-cli-service can setup git hooks.
    const shouldInitGit = await this.shouldInitGit(cliOptions)
    if (shouldInitGit) {
      logWithSpinner(`🗃`, `Initializing git repository...`)
      await run('git init')
    }
```

下载 命令插件

``` js
    // install plugins
    stopSpinner()
    log(`⚙  Installing CLI plugins. This might take a while...`)
    log()
    if (isTestOrDebug) {
      // in development, avoid installation process
      await setupDevProject(context) // 如果测试或者调试, 阻止下载
    } else {
      await installDeps(context, packageManager, cliOptions.registry)
    }
```

生成对应插件文件

``` js
    // run generator
    log()
    log(`🚀  Invoking generators...`)
    const plugins = this.resolvePlugins(preset.plugins)
    const generator = new Generator(context, {
      pkg, // 传入 package
      plugins,
      completeCbs: createCompleteCbs
    })
    await generator.generate({
      extractConfigFiles: preset.useConfigFiles
    })
```

最终下载开发库

``` js
    // install additional deps (injected by generators)
    log(`📦  Installing additional dependencies...`)
    log()
    if (!isTestOrDebug) {
      // 为什么 不需要其他传值
      // 目标目录, yarn/npm, 下载网址
    // 因为所有的信息都放在了 package.json
    // $ yarn 就自己下载了
      await installDeps(context, packageManager, cliOptions.registry)
    }
```

完成下载后, 对应添加 `package.json` 安装列表

``` js
    // run complete cbs if any (injected by generators)
    log()
    logWithSpinner('⚓', `Running completion hooks...`)
    for (const cb of createCompleteCbs) {
      await cb()
    }

```

第一个提交

``` js
    // commit initial state
    if (shouldInitGit) {
      await run('git add -A')
      if (isTestOrDebug) {
        await run('git', ['config', 'user.name', 'test'])
        await run('git', ['config', 'user.email', 'test@test.com'])
      }
      const msg = typeof cliOptions.git === 'string' ? cliOptions.git : 'init'
      await run('git', ['commit', '-m', msg])
    }
```

完成显示

``` js
    // log instructions
    stopSpinner()
    log()
    log(`🎉  Successfully created project ${chalk.yellow(name)}.`)
    log(
      `👉  Get started with the following commands:\n\n` +
      (this.context === process.cwd() ? `` : chalk.cyan(` ${chalk.gray('$')} cd ${name}\n`)) +
      chalk.cyan(` ${chalk.gray('$')} ${packageManager === 'yarn' ? 'yarn serve' : 'npm run serve'}`)
    )
    log()

```

统一下, 错误输出

``` js
generator.printExitLogs()
```

