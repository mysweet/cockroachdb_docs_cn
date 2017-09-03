# CockroachDB 比较

本页向你展示 CockroachDB 的关键特性与其他数据库比较如何。光标悬停在特性上会给出其含义，点击 CockroachDB 答案查看相关的文档。

<table class="comparison-chart">
  <tr>
    <th></th>
    <th>
      <select data-column="one">
        <option value="MySQL">MySQL</option>
        <option value="PostgreSQL">PostgreSQL</option>
        <option value="Oracle">Oracle</option>
        <option value="SQL Server">SQL Server</option>
        <option value="Cassandra">Cassandra</option>
        <option value="HBase">HBase</option>
        <option value="MongoDB" selected>MongoDB</option>
        <option value="DynamoDB">DynamoDB</option>
        <option value="Spanner">Spanner</option>
      </select>
    </th>
    <th class="comparison-chart__column-two">
      <select data-column="two">
        <option value="MySQL">MySQL</option>
        <option value="PostgreSQL" selected>PostgreSQL</option>
        <option value="Oracle">Oracle</option>
        <option value="SQL Server">SQL Server</option>
        <option value="Cassandra">Cassandra</option>
        <option value="HBase">HBase</option>
        <option value="MongoDB">MongoDB</option>
        <option value="DynamoDB">DynamoDB</option>
        <option value="Spanner">Spanner</option>
      </select>
    </th>
    <th>CockroachDB</th>
  </tr>
  <tr>
    <td class="comparison-chart__feature">
      自动缩放
      <a href="#" data-toggle="tooltip" title="自动并持续在一个集群的节点间再平衡数据。">
        <img src='../images/icon_info.svg' alt="tooltip icon">
      </a>
    </td>
    <td class="comparison-chart__column-one">
      <span class="support gray" data-dbs='["MySQL", "PostgreSQL", "Oracle", "SQL Server"]'>否</span>
      <span class="support" data-dbs='["Cassandra", "HBase", "MongoDB", "DynamoDB", "Spanner"]'>是</span>
    </td>
    <td class="comparison-chart__column-two">
      <span class="support gray" data-dbs='["MySQL", "PostgreSQL", "Oracle", "SQL Server"]'>否</span>
      <span class="support" data-dbs='["Cassandra", "HBase", "MongoDB", "DynamoDB", "Spanner"]'>是</span>
    </td>
    <td><a class="comparison-chart__link" href="frequently-asked-questions.md#how-does-cockroachdb-scale">是</a></td>
  </tr>
  <tr>
    <td class="comparison-chart__feature">
      自动故障转移
      <a href="#" data-toggle="tooltip" title="不被中断的数据可用性，从服务器重启到数据中心停电的大小规模故障。">        
        <img src='../images/icon_info.svg' alt="tooltip icon">
      </a>
    </td>
    <td class="comparison-chart__column-one">
      <span class="support" data-dbs='["MySQL", "PostgreSQL", "Oracle", "SQL Server"]'>可选</span>
      <span class="support" data-dbs='["Cassandra", "HBase", "MongoDB", "DynamoDB", "Spanner"]'>是</span>
    </td>
    <td class="comparison-chart__column-two">
      <span class="support" data-dbs='["MySQL", "PostgreSQL", "Oracle", "SQL Server"]'>可选</span>
      <span class="support" data-dbs='["Cassandra", "HBase", "MongoDB", "DynamoDB", "Spanner"]'>是</span>
    </td>
    <td><a class="comparison-chart__link" href="frequently-asked-questions.md#how-does-cockroachdb-survive-failures">是</a></td>
  </tr>
  <tr>
    <td class="comparison-chart__feature">
      自动修复
      <a href="#" data-toggle="tooltip" title="使用未被影响的副本作为源，在故障后自动修复消失的数据。">
        <img src='../images/icon_info.svg' alt="tooltip icon">
      </a>
    </td>
    <td class="comparison-chart__column-one">
      <span class="support gray" data-dbs='["MySQL", "PostgreSQL", "Oracle", "SQL Server"]'>否</span>
      <span class="support" data-dbs='["Cassandra", "HBase", "MongoDB", "DynamoDB", "Spanner"]'>是</span>
    </td>
    <td class="comparison-chart__column-two">
      <span class="support gray" data-dbs='["MySQL", "PostgreSQL", "Oracle", "SQL Server"]'>否</span>
      <span class="support" data-dbs='["Cassandra", "HBase", "MongoDB", "DynamoDB", "Spanner"]'>是</span>
    </td>
    <td><a class="comparison-chart__link" href="frequently-asked-questions.md#how-does-cockroachdb-survive-failures">是</a></td>
  </tr>
  <tr>
    <td class="comparison-chart__feature">
      强一致复制
      <a href="#" data-toggle="tooltip" title="一旦事务被提交，保证所有读都看到它。">
        <img src='../images/icon_info.svg' alt="tooltip icon">
      </a>
    </td>
    <td class="comparison-chart__column-one">
      <span class="support gray" data-dbs='["MySQL", "PostgreSQL", "HBase", "MongoDB"]'>否</span>
      <span class="support" data-dbs='["Oracle", "SQL Server", "Cassandra"]'>可选</span>
      <span class="support" data-dbs='["DynamoDB", "Spanner"]'>是</span>
    </td>
    <td class="comparison-chart__column-two">
      <span class="support gray" data-dbs='["MySQL", "PostgreSQL", "HBase", "MongoDB"]'>否</span>
      <span class="support" data-dbs='["Oracle", "SQL Server", "Cassandra"]'>可选</span>
      <span class="support" data-dbs='["DynamoDB", "Spanner"]'>是</span>
    </td>
    <td><a class="comparison-chart__link" href="frequently-asked-questions.md#how-is-cockroachdb-strongly-consistent">是</a></td>
  </tr>
  <tr>
    <td class="comparison-chart__feature">
      基于共识的复制
      <a href="#" data-toggle="tooltip" title="保证只要多数节点可用（即 3/5）就能取得进展。">
        <img src='../images/icon_info.svg' alt="tooltip icon">
      </a>
    </td>
    <td class="comparison-chart__column-one">
      <span class="support gray" data-dbs='["MySQL", "PostgreSQL", "Oracle", "SQL Server", "HBase", "MongoDB"]'>否</span>
      <span class="support" data-dbs='["Cassandra"]'>可选</span>
      <span class="support" data-dbs='["DynamoDB", "Spanner"]'>是</span>
    </td>
    <td class="comparison-chart__column-two">
      <span class="support gray" data-dbs='["MySQL", "PostgreSQL", "Oracle", "SQL Server", "HBase", "MongoDB"]'>否</span>
      <span class="support" data-dbs='["Cassandra"]'>可选</span>
      <span class="support" data-dbs='["DynamoDB", "Spanner"]'>是</span>
    </td>
    <td><a class="comparison-chart__link" href="frequently-asked-questions.md#how-is-cockroachdb-strongly-consistent">是</a></td>
  </tr>
  <tr>
    <td class="comparison-chart__feature">
      分布式事务
      <a href="#" data-toggle="tooltip" title="跨一个分布式集群正确提交的事务，无论是一个单一位置的几个节点，还是多个数据中心的很多节点。">
        <img src='../images/icon_info.svg' alt="tooltip icon">
      </a>
    </td>
    <td class="comparison-chart__column-one">
      <span class="support gray" data-dbs='["MySQL", "PostgreSQL", "Cassandra", "HBase", "MongoDB"]'>否</span>
      <span class="support" data-dbs='["Oracle", "SQL Server", "Spanner"]'>是</span>
      <span class="support gray" data-dbs='["DynamoDB"]'>否*</span>
    </td>
    <td class="comparison-chart__column-two">
      <span class="support gray" data-dbs='["MySQL", "PostgreSQL", "Cassandra", "HBase", "MongoDB", "DynamoDB"]'>否</span>
      <span class="support" data-dbs='["Oracle", "SQL Server", "Spanner"]'>是</span>
      <span class="support gray" data-dbs='["DynamoDB"]'>否*</span>
    </td>
    <td><a class="comparison-chart__link" href="frequently-asked-questions.md#does-cockroachdb-support-distributed-transactions">是</a></td>
  </tr>
  <tr>
    <td class="comparison-chart__feature">
      ACID 语义
      <a href="#" data-toggle="tooltip" title="保证每个事务提供原子性、一致性、隔离和耐久性。">
        <img src='../images/icon_info.svg' alt="tooltip icon">
      </a>
    </td>
    <td class="comparison-chart__column-one">
      <span class="support" data-dbs='["MySQL", "PostgreSQL", "Oracle", "SQL Server", "Spanner"]'>是</span>
      <span class="support gray" data-dbs='["Cassandra"]'>否</span>
      <span class="support" data-dbs='["HBase"]'>仅行</span>
      <span class="support" data-dbs='["DynamoDB"]'>仅行*</span>
      <span class="support" data-dbs='["MongoDB"]'>仅文档</span>
    </td>
    <td class="comparison-chart__column-two">
      <span class="support" data-dbs='["MySQL", "PostgreSQL", "Oracle", "SQL Server", "Spanner"]'>是</span>
      <span class="support gray" data-dbs='["Cassandra"]'>否</span>
      <span class="support" data-dbs='["HBase"]'>仅行</span>
      <span class="support" data-dbs='["DynamoDB"]'>仅行*</span>
      <span class="support" data-dbs='["MongoDB"]'>仅文档</span>
    </td>
    <td><a class="comparison-chart__link" href="frequently-asked-questions.md#do-transactions-in-cockroachdb-guarantee-acid-semantics">是</a></td>
  </tr>
  <tr>
    <td class="comparison-chart__feature">
      最终一致读
      <a href="#" data-toggle="tooltip" title="可选地允许从没有最新的写入数据的副本读取。">
        <img src='../images/icon_info.svg' alt="tooltip icon">
      </a>
    </td>
    <td class="comparison-chart__column-one">
      <span class="support" data-dbs='["MySQL", "PostgreSQL", "Oracle", "SQL Server", "Cassandra", "HBase", "MongoDB", "DynamoDB", "Spanner"]'>是</span>
    </td>
    <td class="comparison-chart__column-two">
      <span class="support" data-dbs='["MySQL", "PostgreSQL", "Oracle", "SQL Server", "Cassandra", "HBase", "MongoDB", "DynamoDB", "Spanner"]'>是</span>
    </td>
    <td><span class="gray comparison-chart__cockroach">否</span></td>
  </tr>
  <tr>
    <td class="comparison-chart__feature">
      SQL
      <a href="#" data-toggle="tooltip" title="开发者的端点基于 SQL 数据库查询语言标准。">
        <img src='../images/icon_info.svg' alt="tooltip icon">
      </a>
    </td>
    <td class="comparison-chart__column-one">
      <span class="support" data-dbs='["MySQL", "PostgreSQL", "Oracle", "SQL Server"]'>是</span>
      <span class="support gray" data-dbs='["Cassandra", "HBase", "MongoDB", "DynamoDB"]'>否</span>
      <span class="support" data-dbs='["Spanner"]'>只读</span>
    </td>
    <td class="comparison-chart__column-two">
      <span class="support" data-dbs='["MySQL", "PostgreSQL", "Oracle", "SQL Server"]'>是</span>
      <span class="support gray" data-dbs='["Cassandra", "HBase", "MongoDB", "DynamoDB"]'>否</span>
      <span class="support" data-dbs='["Spanner"]'>只读</span>
    </td>
    <td><a class="comparison-chart__link" href="frequently-asked-questions.md#why-is-cockroachdb-sql">是</a></td>
  </tr>
  <tr>
    <td class="comparison-chart__feature">
      开源
      <a href="#" data-toggle="tooltip" title="数据库的源代码对任何人用于任何目的可免费用于学习、修改和发布。">
        <img src='../images/icon_info.svg' alt="tooltip icon">
      </a>
    </td>
    <td class="comparison-chart__column-one">
      <span class="support" data-dbs='["MySQL", "PostgreSQL", "Cassandra", "HBase", "MongoDB"]'>是</span>
      <span class="support gray" data-dbs='["Oracle", "SQL Server", "DynamoDB", "Spanner"]'>否</span>
    </td>
    <td class="comparison-chart__column-two">
      <span class="support" data-dbs='["MySQL", "PostgreSQL", "Cassandra", "HBase", "MongoDB"]'>是</span>
      <span class="support gray" data-dbs='["Oracle", "SQL Server", "DynamoDB", "Spanner"]'>否</span>
    </td>
    <td><a class="comparison-chart__link" href="contribute-to-cockroachdb.md">是</a></td>
  </tr>
  <tr>
    <td class="comparison-chart__feature">
      商业版
      <a href="#" data-toggle="tooltip" title="数据库的企业或扩展版对付费客户可用。">
        <img src='../images/icon_info.svg' alt="tooltip icon">
      </a>
    </td>
    <td class="comparison-chart__column-one">
      <span class="support" data-dbs='["MySQL", "Cassandra", "HBase", "MongoDB"]'>可选</span>
      <span class="support gray" data-dbs='["PostgreSQL"]'>否</span>
      <span class="support" data-dbs='["Oracle", "SQL Server", "DynamoDB", "Spanner"]'>是</span>
    </td>
    <td class="comparison-chart__column-two">
      <span class="support" data-dbs='["MySQL", "Cassandra", "HBase", "MongoDB"]'>可选</span>
      <span class="support gray" data-dbs='["PostgreSQL"]'>否</span>
      <span class="support" data-dbs='["Oracle", "SQL Server", "DynamoDB", "Spanner"]'>是</span>
    </td>
    <td><span class="comparison-chart__cockroach">可选</span></td>
  </tr>
  <tr>
    <td class="comparison-chart__feature">
      支持
      <a href="#" data-toggle="tooltip" title='数据库使用和排错指南，或者是"有限" （免费，基于社区）或者是"完全" （付费，专门员工 24/7 支持。'>
        <img src='../images/icon_info.svg' alt="tooltip icon">
      </a>
    </td>
    <td class="comparison-chart__column-one">
      <span class="support" data-dbs='["MySQL", "PostgreSQL", "Oracle", "SQL Server", "Cassandra", "HBase", "MongoDB", "DynamoDB", "Spanner"]'>完全</span>
    </td>
    <td class="comparison-chart__column-two">
      <span class="support" data-dbs='["MySQL", "PostgreSQL", "Oracle", "SQL Server", "Cassandra", "HBase", "MongoDB", "DynamoDB", "Spanner"]'>完全</span>
    </td>
    <td><a class="comparison-chart__link" href="https://www.cockroachlabs.com/pricing/">完全</a></td>
  </tr>
</table>

\* 在 DynamoDB 中，分布式事务和 ACID 语义跨数据库中所有的数据，而不仅是每行，要求一个额外的<a href="https://aws.amazon.com/blogs/aws/dynamodb-transaction-library/">事务库</a>。