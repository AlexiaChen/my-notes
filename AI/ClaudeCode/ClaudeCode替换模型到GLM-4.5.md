
主要是ClaudeCode太贵了，特别是Max订阅，而且最近他们修改机制了，用不起了，国产的模型便宜，而且GLM-4.5 达到了SOTA水平。所以决定替换。

先下载CC，当然，在Linux下你用nvm来管理node的版本

```bash
npm install -g @anthropic-ai/claude-code
claude --version
```


然后去智普的平台 https://www.bigmodel.cn

申请API Key

在  `~/.bashrc` 中写入:

```bash
export ANTHROPIC_BASE_URL=https://open.bigmodel.cn/api/anthropic
export ANTHROPIC_AUTH_TOKEN=你刚刚从bigmodel获取到的API_KEY
```

```bash
source ~/.bashrc 
```

然后就可以在项目的根目录运行claude命令，用TUI的方式来交互。