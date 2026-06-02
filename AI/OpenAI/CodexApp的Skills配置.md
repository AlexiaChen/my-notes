
根据 OpenAI 官方开发者文档关于 Agent Skills 的存储机制，对 Codex 系统的全局和系统层级结构进行了核实。 

在官方规范中，针对 Repo 级别之上的全局配置，划分为了 USER（用户级全局） 和 ADMIN/SYSTEM（系统级全局） 两个层级。你提到的 `.zip` 包需要解压后放入对应的目录中。以下是 Windows 和 Linux 系统的准确存放位置： 

## 🛠️ 1. 用户级全局（User Level / Global）

- 适用场景：你个人开发的、或者从社区下载的自定义 Skill，希望在当前登录用户的所有项目中都能直接调用。
- Windows 路径：  
    `C:\Users\<你的用户名>\.agents\skills\` 或 `C:\Users\<你的用户名>\.codex\skills\`  
    _(快捷方式：在资源管理器地址栏输入 `%USERPROFILE%\.agents\skills`)_
- Linux 路径：  
    `/home/<你的用户名>/.agents/skills/` 或 `~/.codex/skills/` 

---

## 🏛️ 2. 系统/管理员级全局（Admin / System Level）

- 适用场景：由系统管理员配置、或者需要对整台机器/容器内的所有用户强制生效的底座级自动化脚本与工具。
- Windows 路径：  
    `C:\ProgramData\codex\skills\`
- Linux 路径：  
    `/etc/codex/skills/`

---

## ⚠️ 核心确认：文件解压结构规范

Codex 无法直接读取一个裸露的 `skills.zip` 压缩包。你必须新建一个以该 Skill 命名的文件夹，并将 Zip 包里的内容解压进去，使其保持以下多层探测结构： [8]

```text
# 以系统级（ADMIN）路径为例：
/etc/codex/skills/  (或 Windows 的 ProgramData)
└── my-custom-tool/             <-- 必须：解压后创建的 Skill 唯一标识文件夹
    ├── SKILL.md                <-- 核心：第一级目录下必须有该描述文件
    ├── requirements.txt        <-- 可选：Python 依赖项
    └── run.py                  <-- 可选：执行的具体脚本
```

如果你希望通过一条命令直接在终端中完成对 Zip 包的远程下载、解压和全局归档，可以告诉我你当前所处的系统环境（Windows PowerShell 还是 Linux Bash），我可以为你编写直接可用的命令行脚本。

  
