# my-notes
A collection of my notes

```bash
# - **路径末尾的 `/` 很重要**：`/source/` 表示同步目录内容，`/source` 表示同步目录本身
rsync -av --delete --progress --exclude='.git/' /home/user/source/ /home/user/backup/
```
