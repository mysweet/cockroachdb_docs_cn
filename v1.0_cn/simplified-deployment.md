# 简化的部署

部署和维护数据库永远是一项困难和昂贵的前景。简单性是我们最重要的设计目标之一。 CockroachDB 是自包含的，而且避开了外部依赖。没有明确的角色，如主、从、首要、次要，产生妨碍。反之，每个 CockroachDB 节点是对称和同等重要的，这意味着，在体系结构中没有单点故障。

-   没有外部依赖
-   使用闲话网络自组织
-   及其简单的配置而不需要“把手”
-   对称节点理想地适合基于容器的部署
-   每个节点提供了对集中管理控制台的访问

<img src='../images/2simplified-deployments.png' alt="CockroachDB is simple to deploy" style="width: 400px" />
