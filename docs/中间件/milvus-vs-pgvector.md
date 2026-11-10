

## 前言

在做RAG（检索增强生成）应用或推荐系统时，很多小伙伴都会遇到同一个灵魂拷问：**向量数据库到底该怎么选？**

市面上选项很多，但讨论最激烈、也是让大家最纠结的，往往就是“二选一”——是选择嵌入在PostgreSQL里的pgvector，还是选择专门的向量数据库Milvus？

这两个工具表面上看都在做同一件事——向量相似度搜索，但本质上代表了两种完全不同的系统设计理念。

一个选择把向量检索能力融入现有业务系统，一个选择把向量检索能力做深做精。

今天这篇文章就专门跟大家一起聊聊这个话题，希望对你会有所帮助。


## 一、两者到底有什么不同？

先给个结论性的对比：

| 对比维度 | pgvector | Milvus |
|---------|----------|--------|
| **本质** | PostgreSQL的一个扩展插件 | 专为向量打造的独立分布式数据库 |
| **设计理念** | 在关系型数据库里“顺便”做向量检索 | 把向量检索做到极致 |
| **适合数据量** | ≤500万条 | 千万级到百亿级 |
| **运维复杂度** | ★☆☆☆☆（极低） | ★★★★☆（较高） |
| **典型场景** | 已有PostgreSQL的中小项目 | 大规模、高性能AI应用 |

如果用一个比喻来理解：pgvector就像在你的家庭小厨房里加了一台空气炸锅，偶尔炸个薯条完全够用，还不占地方。

Milvus则像一个专业的中央厨房，能同时处理几百桌订单，但你需要单独租场地、雇人、维护设备。


## 二、它们到底是怎么做向量检索的？

在对比之前，我们先搞懂一个核心问题：**向量检索的本质是什么？**

向量检索就是在一堆高维空间里的点中，找到离目标点最近的K个点。

如果暴力计算（把所有点都算一遍），数据量一大就慢如蜗牛。

所以，必须用**索引**来加速——就像书的目录，让你不用翻完整本书就能找到内容。

pgvector和Milvus都采用了两种主流索引：**IVF（倒排文件索引）**和**HNSW（分层导航小世界）**。

但它们的实现方式和优化方向完全不同。

### 2.1 pgvector的索引原理

pgvector直接在PostgreSQL的存储引擎之上增加了一种新的数据类型`vector`，并利用PostgreSQL的索引接口实现了IVFFlat和HNSW索引。

**IVFFlat原理图**：


![pgvector的索引原理 示意图1](https://web-1259428203.cos.ap-chengdu.myqcloud.com/2a5c4d77-ad0a-4173-a364-41619e71efbd.png)


![pgvector的索引原理 示意图2](https://web-1259428203.cos.ap-chengdu.myqcloud.com/29b8f459-7207-4985-a356-933629407912.png)


**HNSW原理图**：


![pgvector的索引原理 示意图3](https://web-1259428203.cos.ap-chengdu.myqcloud.com/22b59646-7cdc-4a03-adef-645aa216555b.png)


![pgvector的索引原理 示意图4](https://web-1259428203.cos.ap-chengdu.myqcloud.com/bfcec674-c843-4441-98d3-b94fa38b744e.png)


pgvector把这两种索引算法“塞进”了PostgreSQL的B-tree索引框架中。

好处是：你创建索引的语法和普通B-tree几乎一样，PostgreSQL的查询优化器能自动决定是否使用向量索引。

但缺点也很明显——pgvector不能利用多核并行扫描，也无法使用GPU加速，因为PostgreSQL本身不支持这些特性。

### 2.2 Milvus的索引原理

Milvus是为向量检索从头设计的系统，它的索引层是一个独立的、高度优化的模块。

**Milvus整体架构**：


![Milvus的索引原理 示意图5](https://web-1259428203.cos.ap-chengdu.myqcloud.com/b015d64e-3e45-4ab2-9ae8-baaa49a94f11.png)


Milvus的索引节点可以并行构建索引，查询节点可以并发执行搜索。它支持10+种索引算法，包括：

- **HNSW**：基于图的索引，查询快，内存占用大
- **IVF_FLAT**：聚类+全精度，召回率高
- **IVF_PQ**：乘积量化，内存压缩8-16倍
- **GPU索引**：利用CUDA加速，延迟可降至亚毫秒级
- **DiskANN**：磁盘索引，支持百亿级数据

**Milvus的HNSW索引查找过程（多线程并行）**：


![Milvus的索引原理 示意图6](https://web-1259428203.cos.ap-chengdu.myqcloud.com/5f5f77a5-2353-4074-8a8a-971c744019a1.png)


正是这种“存储计算分离”和“并行执行”的架构，让Milvus在处理千万级以上向量时，性能远超pgvector。


## 三、核心功能深度对比

### 3.1 向量类型与索引

| 功能 | pgvector | Milvus |
|------|----------|--------|
| 稠密向量 | ✅ | ✅ |
| 稀疏向量 | 基础支持 | ✅ 原生支持 |
| 二值向量 | ❌ | ✅ |
| 动态Schema | ❌ | ✅ |
| 索引类型 | HNSW, IVFFlat | 10+种（含GPU索引） |
| 量化压缩 | ❌ | ✅ PQ/GPQ/INT8/1-bit |
| 磁盘索引 | ❌ | ✅ DiskANN |

### 3.2 混合检索能力

pgvector的最大优势是混合检索非常自然——用SQL一条语句搞定：

```sql
SELECT * FROM products 
WHERE category = 'electronics' 
  AND price < 1000
ORDER BY embedding <=> query_vec 
LIMIT 10;
```

Milvus也支持标量过滤，但过滤条件需要写在表达式里，不如SQL直观：

```python
results = collection.search(
    data=[query_vec],
    anns_field="embedding",
    param={"metric_type": "IP"},
    limit=10,
    expr="category == 'electronics' && price < 1000"
)
```

### 3.3 事务与一致性

pgvector继承了PostgreSQL的ACID事务，适合需要强一致性的金融、订单等场景。Milvus提供最终一致性，更注重高吞吐和低延迟。

### 3.4 硬件加速

Milvus支持GPU索引（GPU IVF、GPU HNSW），利用CUDA加速，查询延迟可以降低到亚毫秒级。pgvector没有GPU支持。


## 四、性能实测

根据多家机构的基准测试，两者在不同规模下的性能表现差异明显：

**百万级向量测试**（128维）：
| 指标 | pgvector | Milvus |
|------|----------|--------|
| QPS | 1,200 | 3,500 |
| 延迟 | 8-12ms | 3-5ms |
| 精确度 | 98.7% | 99.2% |

**千万级向量测试**（768维BERT向量，4节点集群）：
| 指标 | Milvus | pgvector |
|------|--------|----------|
| 写入吞吐量 | ~12万向量/秒 | ~1.5万向量/秒 |
| 查询延迟（缓存） | ~20ms | ~120ms |
| 磁盘占用 | 1.5倍原始大小 | 2.2倍原始大小 |

**结论很清晰**：在小规模数据（≤500万）下，pgvector的性能完全够用。

但数据量达到千万级以上时，Milvus在写入吞吐量和查询延迟上的优势开始变得非常明显。


## 五、运维复杂度

这是两者差异最大的维度。

**pgvector部署**（一行SQL）：

```sql
CREATE EXTENSION vector;
```

备份用`pg_dump`，高可用用`repmgr`或Patroni，全部复用PostgreSQL生态，不需要学习任何新工具。

内存占用方面，100万条以下数据，pgvector可控制在2GB以内。

**Milvus部署**（需要Docker Compose或K8s）：

```yaml
# docker-compose.yml 片段
services:
  etcd:
    image: quay.io/coreos/etcd:v3.5.5
  minio:
    image: minio/minio:latest
  standalone:
    image: milvusdb/milvus:v2.6.0
    depends_on:
      - etcd
      - minio
```

即便是单机版，也需要同时运行etcd、MinIO和Milvus三个容器。

生产环境集群还需要配置Pulsar或Kafka。

不过，Milvus 2.6版本做了大量简化工作，例如内置Woodpecker消息队列，降低了对Kafka的依赖。

**一句话总结**：如果你只有一台2核4G的云服务器，pgvector是最务实的方案；如果你有专门的机器或K8s集群，可以考虑Milvus。


## 六、代码实战

### 6.1 pgvector完整示例

```sql
-- 1. 安装扩展
CREATE EXTENSION vector;

-- 2. 创建带向量列的表
CREATE TABLE documents (
    id SERIAL PRIMARY KEY,
    content TEXT,
    embedding VECTOR(1536),      -- 1536维嵌入
    category TEXT,
    created_at TIMESTAMP DEFAULT NOW()
);

-- 3. 创建HNSW索引（加速检索）
CREATE INDEX ON documents USING hnsw (embedding vector_cosine_ops);

-- 4. 插入向量数据（假设已有embedding数组）
INSERT INTO documents (content, embedding, category) 
VALUES 
    ('PostgreSQL向量扩展介绍', '[0.12, -0.34, ...]', '技术'),
    ('Milvus分布式向量数据库', '[0.45, -0.12, ...]', '技术');

-- 5. 执行向量相似度检索
SELECT content, 1 - (embedding <=> '[0.11, -0.33, ...]') AS similarity
FROM documents 
WHERE category = '技术'
ORDER BY embedding <=> '[0.11, -0.33, ...]' 
LIMIT 5;
```

### 6.2 Milvus完整示例

技术栈使用的Java + Spring AI Alibaba。

**pom.xml依赖**：

```xml
<dependency>
    <groupId>com.alibaba.cloud.ai</groupId>
    <artifactId>spring-ai-alibaba-starter-milvus-store</artifactId>
    <version>1.0.0</version>
</dependency>
```

**application.yml配置**：

```yaml
spring:
  ai:
    vectorstore:
      milvus:
        host: localhost
        port: 19530
        collection-name: documents
        embedding-dimension: 1536
```

**Java代码**：

```java
@Configuration
public class MilvusConfig {
    @Bean
    public VectorStore vectorStore(EmbeddingModel embeddingModel) {
        return new MilvusVectorStore(MilvusVectorStoreConfig.builder()
                .withHost("localhost")
                .withPort(19530)
                .withCollectionName("documents")
                .withEmbeddingDimension(1536)
                .build(), embeddingModel);
    }
}

@Service
public class DocumentService {
    @Autowired
    private VectorStore vectorStore;
    
    public List<Document> search(String query, int topK) {
        // 内部自动完成向量化 + 检索
        return vectorStore.similaritySearch(
            SearchRequest.query(query).withTopK(topK)
        );
    }
}
```


## 七、优缺点

### pgvector：轻量、简单、够用

**优点**：
- 部署极简，一行SQL即可启用
- 复用PostgreSQL全套运维体系（备份、高可用、监控）
- 内存占用低（<100万条可控制在2GB）
- 支持ACID事务，数据一致性有保障
- 学习成本几乎为零，直接用SQL
- 混合检索最自然（SQL标量+向量）

**局限**：
- 数据量超过500万后性能明显下降
- 索引类型有限（无PQ等量化压缩）
- 无内置GPU加速
- 分布式扩展困难，依赖PostgreSQL原生分片方案
- 查询节点单线程执行，无法并行

**适用场景**：数据量<500万、已有PostgreSQL基础设施的中小项目、对运维简单性要求极高的团队、需要强事务一致性的场景。

### Milvus：专业、强大、可扩展

**优点**：
- 原生分布式架构，可水平扩展至百亿级向量
- 索引类型丰富（10+种），支持GPU加速
- 写入吞吐量高，延迟低（3-5ms）
- 2.6版本大幅优化内存和成本（INT8压缩）
- 支持多种向量类型（稠密/稀疏/二值）
- 支持动态Schema，灵活适应业务变化

**局限**：
- 运维复杂度高，需要管理多个组件（etcd、MinIO等）
- 资源门槛较高（默认8GB+内存）
- 学习曲线陡峭
- 与关系型数据的混合查询需要应用层实现
- 不提供ACID事务（最终一致性）

**适用场景**：数据量>500万、对查询性能和扩展性要求高的AI应用，如RAG、推荐系统、图像检索、多模态搜索等。


## 八、如何选择？


![八、如何选择？ 示意图7](https://web-1259428203.cos.ap-chengdu.myqcloud.com/61d70baf-fef8-426b-80ee-a4e0a81d0599.png)


## 总结

回到最初的问题：Milvus和pgvector，哪个更好？

答案很简单：**看你的数据规模和业务场景。**

- 如果你的业务数据量在**百万级以下**，或者你**已经在用PostgreSQL**，希望保持架构简洁，那pgvector就是最务实的选择。一个扩展、几行SQL，就能把向量检索能力接入现有系统，无需额外维护。
- 如果你的数据量达到**千万甚至亿级以上**，对**查询延迟**和**写入吞吐量**有极致要求，且团队有能力维护分布式系统，那Milvus才是正确的答案。

我个人的建议是：**从pgvector起步，用最简单的方案先跑通业务**。

等数据量真的涨起来、性能瓶颈真正出现时，再考虑迁移到Milvus也不迟。

过早引入复杂的分布式系统，只会增加不必要的运维成本。

<!-- article-related-reading:start -->
## 相关阅读

- [Mybatis-Plus和Mybatis-Flex，哪个更好？](./mybatis-plus-vs-mybatis-flex.html)
- [Maven和Gradle，哪个更好？](./maven-vs-gradle.html)
- [Nacos 和 Apollo，哪个更好？](./nacos-vs-apollo.html)

<!-- article-related-reading:end -->
