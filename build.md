## build

`vue build`

在生产模式下使用零配置构建.js或.vue文件

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

- loadCommand

[主要就是请求](./readme.md#loadcommand)


---

### [@vue/cli-service-global](./cli-service-global.md)
