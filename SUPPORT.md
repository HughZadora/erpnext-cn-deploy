# 支持与问题反馈

本仓库提供面向中文用户的 ERPNext/Frappe Docker 部署与运维资料，基于上游 [`frappe_docker`](https://github.com/frappe/frappe_docker)。它不是托管服务，也不提供对生产 ERPNext 实例的远程操作或响应时限承诺。

## 按问题类型选择入口

- **部署、备份或日常使用问题：**先阅读[ERPNext 求助指南](docs/business-operations/ERPNext求助指南_遇到问题怎么办.md)及对应的部署、备份和运维文档。
- **本仓库文档、示例或配置存在可复现缺陷：**使用[部署资料缺陷表单](https://github.com/HughZadora/erpnext-cn-deploy/issues/new?template=bug_report.yml)，说明相关文件、版本、复现步骤和预期结果。
- **文档或维护改进建议：**使用[建议表单](https://github.com/HughZadora/erpnext-cn-deploy/issues/new?template=feature_request.yml)。
- **上游软件问题：**Frappe 框架请到 [frappe](https://github.com/frappe/frappe/issues)，ERPNext 应用请到 [erpnext](https://github.com/frappe/erpnext/issues)，Docker 部署流程请到 [frappe_docker](https://github.com/frappe/frappe_docker/issues)。请遵守相应上游项目的支持与安全报告规则。
- **本仓库自身的安全漏洞：**按 [SECURITY.md](SECURITY.md) 使用私密报告渠道；不要在公开 Issue 中披露可利用细节。

提交公开问题时只提供经过脱敏的信息。不要附上密码、令牌、`.env` 文件、客户资料、数据库/站点备份、私有域名或私有 IP。仓库的 `./scripts/repository-check` 只验证仓库文件与语法，不会连接或验证你的生产实例，也不构成对生产操作的授权。
