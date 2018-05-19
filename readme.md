# Vue-cli

「 desc 」

[![explain](http://llever.com/explain.svg)](https://github.com/chinanf-boy/Source-Explain)
    
Explanation

> "version": "3.0.0-beta.9",


[github source](https://github.com/vuejs/vue-cli)

~~[english](./README.en.md)~~

---

## 使用

``` bash
vue create my-project
```

---

本目录

---

## package.json

请不要用单一库, 来看待 `vue-cli`

我们下载时使用 `npm install -g @vue/cli`

> @vue 什么东西, 你可以将 `@vue - 组织` | `cli - 其中之一产品`

构成 多个库`packages`的形式, 是使用[`lerna.json`- 一个用于管理多个包的JavaScript项目的工具。](https://github.com/lerna/lerna)

## @vue/cli

`packages/@vue/cli/package.json`

``` js
  "name": "@vue/cli", // 上传 npm 名字
  "bin": {
    "vue": "bin/vue.js" // 命令
  },
```

### bin/vue.js


## 1. node-版本-要求

代码 11-17

``` js
if (!semver.satisfies(process.version, requiredVersion)) {
  console.log(chalk.red(
    `You are using Node ${process.version}, but this version of vue-cli ` +
    `requires Node ${requiredVersion}.\nPlease upgrade your Node version.`
  ))
  process.exit(1)
}
```

- 1.1 [semver](https://github.com/npm/node-semver)

npm的语义版本

---

## 2. 调试模式

代码 19-27

``` js
// enter debug mode when creating test repo
if (
  slash(process.cwd()).indexOf('/packages/test') > 0 && (
    fs.existsSync(path.resolve(process.cwd(), '../@vue')) ||
    fs.existsSync(path.resolve(process.cwd(), '../../@vue'))
  )
) {
  process.env.VUE_CLI_DEBUG = true
}
```

- 2.1 [slash](https://github.com/sindresorhus/slash)

统一 `window` 和 `unix` 斜号

---

## 3. 命令解析

代码 29-34

``` js
const program = require('commander')
const loadCommand = require('../lib/util/loadCommand')

program
  .version(require('../package').version)
  .usage('<command> [options]')
```

---

> 命令缩影

- [X] __vue create__ ` [options] <app-name> 创建一个由vue-cli-service支持的新项目`
- [x] __vue add__ ` <plugin> [pluginOptions] 安装插件并在已创建的项目中调用其生成器` ⬇️
- [x] __vue invoke__ ` <plugin> [pluginOptions] 在已创建的项目中调用插件的生成器`   ⬆️
- [x] __vue inspect__ ` [options] [paths...] 使用vue-cli-service检查项目中的webpack配置`
- [x] __vue serve__ ` [options] [entry] 在零配置下以开发模式提供.js或.vue文件`
- [x] __vue build__ ` [options] [entry] 在生产模式下使用零配置构建.js或.vue文件`
- [ ] __vue init__ ` <template> <app-name> 从远程模板（传统API，需要@vue）生成项目`

---

### 3.1 vue create

代码 36-50

<details>


``` js
program
  .command('create <app-name>')
  .description('create a new project powered by vue-cli-service')
  .option('-p, --preset <presetName>', 'Skip prompts and use saved or remote preset')
  .option('-d, --default', 'Skip prompts and use default preset')
  .option('-i, --inlinePreset <json>', 'Skip prompts and use inline JSON string as preset')
  .option('-m, --packageManager <command>', 'Use specified npm client when installing dependencies')
  .option('-r, --registry <url>', 'Use specified npm registry when installing dependencies (only for npm)')
  .option('-g, --git [message]', 'Force / skip git intialization, optionally specify initial commit message')
  .option('-f, --force', 'Overwrite target directory if it exists')
  .option('-c, --clone', 'Use git clone when fetching remote preset')
  .option('-x, --proxy', 'Use specified proxy when creating project')
  .action((name, cmd) => {
    require('../lib/create')(name, cleanArgs(cmd))
  })

```

</details>

[ >>> create -explain ](./create.md)

---

### 3.2 vue add

代码 52-58

<details>

``` js
program
  .command('add <plugin> [pluginOptions]')
  .allowUnknownOption()
  .description('install a plugin and invoke its generator in an already created project')
  .action((plugin) => {
    require('../lib/add')(plugin, minimist(process.argv.slice(3)))
  })
```

- [minimist](https://github.com/substack/minimist)

解析参数选项


</details>

[ >>> add-explain ](./add.md)

---

### 3.3 vue invoke

代码 60-66

<details>

``` js
program
  .command('invoke <plugin> [pluginOptions]')
  .allowUnknownOption()
  .description('invoke the generator of a plugin in an already created project')
  .action((plugin) => {
    require('../lib/invoke')(plugin, minimist(process.argv.slice(3)))
  })
```


</details>

[ >>> invoke-explain ](./invoke.md)

---

### 3.4 vue inspect

代码 68-75

<details>

``` js
program
  .command('inspect [paths...]')
  .option('--mode <mode>')
  .option('--rule <ruleName>', 'inspect a specific module rule')
  .option('--plugin <pluginName>', 'inspect a specific plugin')
  .option('--rules', 'list all module rule names')
  .option('--plugins', 'list all plugin names')
  .option('-v --verbose', 'Show full function definitions in output')
  .description('inspect the webpack config in a project with vue-cli-service')
  .action((paths, cmd) => {
    require('../lib/inspect')(paths, cleanArgs(cmd))
  })
```

- [cleanArgs](#cleanargs)

清理没有命令

</details>

[ >>> inspect-explain ](./inspect.md)

---

### 3.5 vue serve

代码 

<details>

``` js
program
  .command('serve [entry]')
  .description('serve a .js or .vue file in development mode with zero config')
  .option('-o, --open', 'Open browser')
  .action((entry, cmd) => {
    loadCommand('serve', '@vue/cli-service-global').serve(entry, cleanArgs(cmd))
  })
```

- [loadCommand](#loadcommand)



</details>

[ >>> serve-explain ](./serve.md)

---

### 3.6 vue build

代码 85-93

<details>

``` js
program
  .command('build [entry]')
  .option('-t, --target <target>', 'Build target (app | lib | wc | wc-async, default: app)')
  .option('-n, --name <name>', 'name for lib or web-component mode (default: entry filename)')
  .option('-d, --dest <dir>', 'output directory (default: dist)')
  .description('build a .js or .vue file in production mode with zero config')
  .action((entry, cmd) => {
    loadCommand('build', '@vue/cli-service-global').build(entry, cleanArgs(cmd))
  })
```


</details>

[ >>> build-explain ](./build.md)

---

### 3.7  vue init

代码 

<details>

``` js
program
  .command('init <template> <app-name>')
  .description('generate a project from a remote template (legacy API, requires @vue/cli-init)')
  .option('-c, --clone', 'Use git clone when fetching remote template')
  .action(() => {
    loadCommand('init', '@vue/cli-init')
  })
```


</details>

[ >>> init-explain ](./init.md)

---

### 3.8 命令溢出

代码 103-110

<details>

``` js
// output help information on unknown commands
program
  .arguments('<command>')
  .action((cmd) => {
    program.outputHelp()
    console.log(`  ` + chalk.red(`Unknown command ${chalk.yellow(cmd)}.`))
    console.log()
  })
```


</details>

---

### 3.9 命令帮助

代码 112-119

<details>

``` js
// add some useful info on help
program.on('--help', () => {
  console.log()
  console.log(`  Run ${chalk.cyan(`vue <command> --help`)} for detailed usage of given command.`)
  console.log()
})

program.commands.forEach(c => c.on('--help', () => console.log()))

```


</details>

---

### 3.10 命令错误

代码 121-161

<details>

``` js
// enhance common error messages
const enhanceErrorMessages = (methodName, log) => {
  program.Command.prototype[methodName] = function (...args) {
    if (methodName === 'unknownOption' && this._allowUnknownOption) {
      return
    }
    this.outputHelp()
    console.log(`  ` + chalk.red(log(...args)))
    console.log()
    process.exit(1)
  }
}

enhanceErrorMessages('missingArgument', argName => {
  return `Missing required argument ${chalk.yellow(`<${argName}>`)}.`
})

enhanceErrorMessages('unknownOption', optionName => {
  return `Unknown option ${chalk.yellow(optionName)}.`
})

enhanceErrorMessages('optionMissingArgument', (option, flag) => {
  return `Missing required argument for option ${chalk.yellow(option.flags)}` + (
    flag ? `, got ${chalk.yellow(flag)}` : ``
  )
})

program.parse(process.argv)

if (!process.argv.slice(2).length) {
  program.outputHelp()
}

```

</details>

#### cleanArgs

``` js
//命令将Command对象本身作为选项传递，
//仅将实际选项提取到新对象中。
function cleanArgs (cmd) {
  const args = {}
  cmd.options.forEach(o => {
    const key = o.long.replace(/^--/, '')
    //如果一个选项不存在并且Command有一个同名的方法
     //它不应该被复制
    if (typeof cmd[key] !== 'function') {
      args[key] = cmd[key]
    }
  })
  return args
}

```


#### loadCommand

`vue-cli/packages/@vue/cli/lib/util/loadCommand.js`

主要就是请求

``` js
// 
module.exports = function loadCommand (commandName, moduleName) {
  const isNotFoundError = err => {
    return err.message.match(/Cannot find module/)
  }
  try {
    return require(moduleName) // 直接请求
  } catch (err) {
    if (isNotFoundError(err)) {
      try {
        return require('import-global')(moduleName) // 全局请求
      } catch (err2) {
        if (isNotFoundError(err2)) {
          const chalk = require('chalk')
          const { hasYarn } = require('@vue/cli-shared-utils')
          const installCommand = hasYarn() ? `yarn global add` : `npm install -g`
          console.log()
          console.log(
            // 叫用户自己 全局下载下载
            `  Command ${chalk.cyan(`vue ${commandName}`)} requires a global addon to be installed.\n` +
            `  Please run ${chalk.cyan(`${installCommand} ${moduleName}`)} and try again.` 
          )
          console.log()
          process.exit(1)
        } else {
          throw err2
        }
      }
    } else {
      throw err
    }
  }
}

```