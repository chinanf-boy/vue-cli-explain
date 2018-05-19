## inspect

[options] [paths...] 使用vue-cli-service检查项目中的webpack配置

`vue inspect`

``` js
program
  .command('inspect [paths...]')
  .option('--mode <mode>')
  .option('--rule <ruleName>', '检查特定的模块规则')
  .option('--plugin <pluginName>', '检查特定的模块插件')
  .option('--rules', '列表所有模块名')
  .option('--plugins', '列表所有插件名')
  .option('-v --verbose', '在输出中显示完整的函数定义')
  .description('使用vue-cli-service检查项目中的webpack配置')
  .action((paths, cmd) => {
    require('../lib/inspect')(paths, cleanArgs(cmd))
  })
```

### inspect

`vue-cli/packages/@vue/cli/lib/inspect.js`

1. require

``` js
const fs = require('fs')
const path = require('path')
const execa = require('execa')
const resolve = require('resolve')

```

2. main

`vue inspect` 本质上是调用 `node vue-cli-service` 的服务的

``` js

module.exports = function inspect (paths, args) {
  const cwd = process.cwd()
  let servicePath
  try {
    servicePath = resolve.sync('@vue/cli-service', { basedir: cwd }) // 找到命令路径下模块
  } catch (e) {
    const { error } = require('@vue/cli-shared-utils')
    error(`Failed to locate @vue/cli-service. Make sure you are in the right directory.`)
    process.exit(1)
  }
  const binPath = path.resolve(servicePath, '../../bin/vue-cli-service.js') // 找到 模块命令文件 文件
  if (fs.existsSync(binPath)) { // 
    execa('node', [
      binPath, // 子进程调用
      'inspect',
      ...(args.mode ? ['--mode', args.mode] : []),
      ...(args.rule ? ['--rule', args.rule] : []),
      ...(args.plugin ? ['--plugin', args.plugin] : []),
      ...(args.rules ? ['--rules'] : []),
      ...(args.plugins ? ['--plugins'] : []),
      ...(args.verbose ? ['--verbose'] : []),
      ...paths
    ], { cwd, stdio: 'inherit' })
  }
}

```

然后没了, 至于 `@vue/cli-service` 的 细节我们 ~~[放到后面,vue serve / vue build](./vue-cli-service.md)~~ 再说