
## 容器安装

```bash
# Elasticsearch Docker 镜像默认以 `uid:1000` 和 `gid:1000` 运行。这是为了安全考虑，避免容以 root 用户运行https://github.com/elastic/elasticsearch-docker/issues/21)
chown 1000:1000 /mnt/rasp_data/es/
# 把持久化数据挂载出来
docker run -d --name elasticsearch -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" -v /mnt/rasp_data/es/:/usr/share/elasticsearch/data elasticsearch:6.8.0
```

