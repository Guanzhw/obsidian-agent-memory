---
id: "{{uuid}}"
type: memory
status: draft
created_at: "{{iso_datetime}}"
updated_at: "{{iso_datetime}}"
recorded_at: "{{iso_datetime}}"
sources: []
scope: "{{明确适用范围}}"
evidence: unverified
maintenance: shared
---

# {{偏好、环境或可复用经验}}

## 主张

只记录有复用价值的信息。区分 explicit、observed、inferred 与 unverified。

## 证据

用户明确表达或工具观测的来源和具体定位；模型推测保留在 draft。

## 作用域与时间

适用设备、项目、任务类型或阶段。未知 valid_from 不填写。

## 复核与失效条件

何时需要复核，什么证据会替代它。review_after 不表示到期自动为假。

## 相关内容

链接知识页/项目。不要复制知识库中已维护的技术正文。
