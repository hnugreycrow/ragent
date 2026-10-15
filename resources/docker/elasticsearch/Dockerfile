# 与 Spring Boot 3.5.7 管理的 Elasticsearch 客户端版本保持一致。
ARG ES_VERSION=8.18.8
FROM docker.elastic.co/elasticsearch/elasticsearch:${ES_VERSION}
ARG ES_VERSION

# IK 必须与 Elasticsearch 使用完全相同的版本，构建时安装以便重启时离线使用。
RUN bin/elasticsearch-plugin install --batch \
    https://get.infini.cloud/elasticsearch/analysis-ik/${ES_VERSION}
