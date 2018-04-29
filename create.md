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

> 终端选择

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
    this.name = name
    this.context = process.env.VUE_CLI_CONTEXT = context
    const { presetPrompt, featurePrompt } = this.resolveIntroPrompts() // <==== ⚠️
    this.presetPrompt = presetPrompt
    this.featurePrompt = featurePrompt
    this.outroPrompts = this.resolveOutroPrompts() // <=== ⚠️
    this.injectedPrompts = []
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

> Creator

- **this.featurePrompt.choices** `push` 

> 特性选项

- **this.injectedPrompts** `push/find`

> 相关特性配置-选项

- **this.promptCompleteCbs**`push`

> 选项完成后✅-运行函数

---

现在我们有了, 自定义的完整配置, 供给用户选择

---

### 3. Creator create