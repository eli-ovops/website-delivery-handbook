# 卓建官网开发方法论最终版 Manifest

- 生成日期：2026-07-19
- 依据版本：不可变 tag `v1.3.5`
- commit：`add4e5d0b0ba04778a500ef4281255c65c723120`
- tree：`6c593e879bb71468c76e53f677931687887ba8a4`
- 内容边界：本地方法论文档与可复用 Skill；不含业务源码、数据库、媒体、env 值或 private 内容。

## 建议阅读顺序

1. [00_给同事的使用说明.md](00_给同事的使用说明.md)：第一次接手先读。
2. [04_交付清单与检查表.md](04_交付清单与检查表.md)：快速掌握完整门禁。
3. [01_卓建官网从Payload安装到V1.3.5_复盘与开发偏好.md](01_卓建官网从Payload安装到V1.3.5_复盘与开发偏好.md)：理解全过程、用户偏好、返工和安全经验。
4. [02_前端架构与开发规范.md](02_前端架构与开发规范.md)：前端、后端和架构人员精读。
5. [03_版本演进与开发日志.md](03_版本演进与开发日志.md)：按版本追溯决策和永久规则。
6. [payload-client-website-development/SKILL.md](payload-client-website-development/SKILL.md)：迁移到新客户项目时使用。

## 根文档

| 文件 | 用途 | 行数 | 字节数 |
| --- | --- | ---: | ---: |
| `00_给同事的使用说明.md` | 角色化阅读路线、10 分钟上手、交付顺序、常见误区 | 192 | 16,058 |
| `01_卓建官网从Payload安装到V1.3.5_复盘与开发偏好.md` | 旧版 905 行复盘的完整升级版；覆盖 V1.3.5 和两轮安全审计 | 796 | 61,503 |
| `02_前端架构与开发规范.md` | 当前架构与下个项目推荐规范、数据流、路由、测试与 checklist | 403 | 27,870 |
| `03_版本演进与开发日志.md` | v1.0 至 v1.3.5 的目标、返工、门禁、结果与永久规则 | 328 | 22,224 |
| `04_交付清单与检查表.md` | 可打印的一页式全流程检查表 | 178 | 9,280 |
| `MANIFEST.md` | 文件索引、阅读顺序、版本与边界 | 70 | 5,750 |

统计口径：包内共 32 个文件，其中 31 个 Markdown 和 1 个 `agents/openai.yaml`；根文档合计 1,967 行，包内文件合计 180,012 字节。后续若修改本 Manifest，自身行数和总字节会随之变化，应以交付时重新运行的文件统计为准。

## Skill 包

### 入口与界面元数据

- [payload-client-website-development/SKILL.md](payload-client-website-development/SKILL.md)：决策流程、阶段路由、输出要求和 reference 选择规则。
- `payload-client-website-development/agents/openai.yaml`：Codex/Skill 界面元数据和默认提示。

### References

1. [01_需求与页面字段消费矩阵.md](payload-client-website-development/references/01_需求与页面字段消费矩阵.md)
2. [02_内容模型_草稿发布_排序资源位.md](payload-client-website-development/references/02_内容模型_草稿发布_排序资源位.md)
3. [03_双模式后台_角色与权限矩阵.md](payload-client-website-development/references/03_双模式后台_角色与权限矩阵.md)
4. [04_素材_富文本_关系_招聘私有附件.md](payload-client-website-development/references/04_素材_富文本_关系_招聘私有附件.md)
5. [05_独立Next前端_BFF_适配层.md](payload-client-website-development/references/05_独立Next前端_BFF_适配层.md)
6. [06_双语_provenance_snapshot_静态媒体.md](payload-client-website-development/references/06_双语_provenance_snapshot_静态媒体.md)
7. [07_SQLite_迁移_运行资产_备份回滚.md](payload-client-website-development/references/07_SQLite_迁移_运行资产_备份回滚.md)
8. [08_安全边界与公开写接口.md](payload-client-website-development/references/08_安全边界与公开写接口.md)
9. [09_测试验收与证据分层.md](payload-client-website-development/references/09_测试验收与证据分层.md)
10. [10_故障模式_止损与反屎山.md](payload-client-website-development/references/10_故障模式_止损与反屎山.md)
11. [11_甲方运营后台偏好与交付原则.md](payload-client-website-development/references/11_甲方运营后台偏好与交付原则.md)
12. [12_卓建脱敏案例.md](payload-client-website-development/references/12_卓建脱敏案例.md)

### Templates

- [需求访谈问卷.md](payload-client-website-development/templates/需求访谈问卷.md)
- [页面字段消费矩阵.md](payload-client-website-development/templates/页面字段消费矩阵.md)
- [角色权限矩阵.md](payload-client-website-development/templates/角色权限矩阵.md)
- [Collection_Global设计表.md](payload-client-website-development/templates/Collection_Global设计表.md)
- [媒体与私有附件清单.md](payload-client-website-development/templates/媒体与私有附件清单.md)
- [迁移与运行资产清单.md](payload-client-website-development/templates/迁移与运行资产清单.md)
- [双语provenance清单.md](payload-client-website-development/templates/双语provenance清单.md)
- [测试验收矩阵.md](payload-client-website-development/templates/测试验收矩阵.md)
- [发布回滚清单.md](payload-client-website-development/templates/发布回滚清单.md)
- [独立审计清单.md](payload-client-website-development/templates/独立审计清单.md)
- [开发日志模板.md](payload-client-website-development/templates/开发日志模板.md)
- [交接包模板.md](payload-client-website-development/templates/交接包模板.md)

## 使用边界

根文档描述卓建项目事实；Skill references 提炼通用方法；templates 可复制后填写。新项目不得直接继承卓建的角色、栏目、路由、数据库、时区、双语 fallback 或发布目录。生产实际 release、symlink、env 布局和权限必须在当次工作中只读核对，旧文档不能替代现场事实。
