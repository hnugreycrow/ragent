# Elasticsearch 中文关键词检索

本目录为本地开发提供 Elasticsearch 8.18.8 和同版本 IK 分词插件。需要已运行的 Docker Desktop（Linux 容器）；容器内存上限为 2 GiB，JVM 堆为 1 GiB。

在项目根目录执行：

```powershell
docker compose -f resources/docker/elasticsearch/elasticsearch-stack.compose.yaml up -d --build
docker compose -f resources/docker/elasticsearch/elasticsearch-stack.compose.yaml ps
docker compose -f resources/docker/elasticsearch/elasticsearch-stack.compose.yaml logs --tail 100 elasticsearch
```

首次构建需要下载官方 ES 镜像和 IK 插件。IK 固化在镜像内，后续重启无需重新安装。修改版本时须同时修改 Compose 的镜像标签和 ES_VERSION 构建参数。

本配置关闭认证且仅发布到 127.0.0.1:9200，适合后端直接运行于宿主机的开发环境。生产部署需要另外配置认证和 TLS；若后端也运行于容器中，需要配置共享 Docker 网络及对应的服务地址。

## 验证服务与分词

```powershell
Invoke-RestMethod 'http://127.0.0.1:9200'
docker exec ragent-es bin/elasticsearch-plugin list

$analysisBody = @{
    analyzer = 'ik_smart'
    text = '在全校范围内跨专业、跨学院选修课程'
} | ConvertTo-Json
$result = Invoke-RestMethod -Uri 'http://127.0.0.1:9200/_analyze' -Method Post -ContentType 'application/json; charset=utf-8' -Body ([System.Text.Encoding]::UTF8.GetBytes($analysisBody))
$result.tokens | Select-Object token
```

## 项目接入

`bootstrap/src/main/resources/application.yaml` 已开启以下配置，修改后需重启后端：

```yaml
rag:
  keyword:
    type: es
    es:
      uris: http://127.0.0.1:9200
      index: rag_keyword_store
      analyzer: ik_max_word
      search-analyzer: ik_smart
  search:
    channels:
      vector:
        enabled: true
      keyword:
        enabled: true
```

开启配置不会自动回填存量文档。ES 就绪后重启后端，后端启动时会自动确保共享索引存在，无需手动建索引。再对一份文档重新执行入库/索引流程，确认日志出现 `ES 关键词索引写入成功`。

```powershell
Invoke-RestMethod 'http://127.0.0.1:9200/rag_keyword_store/_count'
```

索引创建前此请求返回 404；写入成功后 count 应大于零。问答日志应出现 `KeywordSearch` 且关键词召回条数非零。如果出现 `关键词索引同步失败，已跳过`，需检查 ES 连接和 IK 插件；向量写入成功并不代表关键词同步成功。

## 停止服务

```powershell
docker compose -f resources/docker/elasticsearch/elasticsearch-stack.compose.yaml down
```

数据保留在命名卷中。不要添加 `-v`，除非明确需要删除索引数据。
