## serve

` vue serve `

在零配置下以开发模式提供.js或.vue文件

``` js
const loadCommand = require('../lib/util/loadCommand')

program
  .command('serve [entry]')
  .description('在零配置下以开发模式提供.js或.vue文件')
  .option('-o, --open', 'Open browser')
  .action((entry, cmd) => {
    loadCommand('serve', '@vue/cli-service-global').serve(entry, cleanArgs(cmd))
  })
```

- loadCommand

[主要就是请求](./readme.md#loadcommand)


---

### [@vue/cli-service-global](./cli-service-global.md)
