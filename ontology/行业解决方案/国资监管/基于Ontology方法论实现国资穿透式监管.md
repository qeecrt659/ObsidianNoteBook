# 基于 Ontology 方法论实现国资穿透式监管

如果目标是 **国资委 / 央国企 / 地方国企的穿透式监管**，Ontology（本体）非常适合作为核心方法论。

它解决的本质问题不是“数据怎么展示”，而是：

> **把国资监管涉及的企业、人、股权、资金、项目、合同、供应商、投资、决策、资产、风险等全部变成一个统一的“业务世界模型”，然后沿着对象之间的关系不断穿透，最终定位到具体事项、具体资金、具体主体、具体责任人，并触发监管动作。**

这与当前国资监管的方向高度一致。2026 年 7 月国务院国资委已经明确提出，要“建强建好智能化监管系统、加强重点领域穿透监管、健全监管闭环”；2025 年智能监管业务模型中也已经出现“控股不控权穿透式监督模型”“应收应付穿透监管模型”等具体模型。

参考：
- 国务院国资委：https://wap.sasac.gov.cn/n2588020/n2588057/n35414555/n35414570/c35542876/content.html
- 国务院国资委智能监管业务模型：https://wap.sasac.gov.cn/n2588030/n16436141/c35013961/content.html

---

## 一、先理解：Ontology 在穿透式监管里到底做什么

传统监管系统一般是：

```text
财务系统 → 财务报表
采购系统 → 采购报表
合同系统 → 合同报表
司库系统 → 资金报表
投资系统 → 投资报表
三重一大 → 决策报表
```

问题是：

**每个系统自己都是对的，但它们之间没有真正连起来。**

例如监管人员看到：

> 某子公司向“华某科技有限公司”支付 3200 万元。

传统系统可能只能看到付款。

但监管真正想知道的是：

```text
为什么付3200万？
      ↓
对应哪个合同？
      ↓
合同来自哪个采购项目？
      ↓
谁中标？
      ↓
一共有几个投标人？
      ↓
这些投标人之间有没有关联？
      ↓
供应商股东是谁？
      ↓
和国企董监高有没有关系？
      ↓
这笔采购是谁决策的？
      ↓
有没有经过“三重一大”？
      ↓
有没有超授权？
      ↓
项目实际进度怎么样？
      ↓
是否已经验收？
      ↓
为什么已经付款？
```

**这整个链条，就是 Ontology。**

---

## 二、建议把国资监管 Ontology 设计成 8 类核心元素

不要简单理解成“知识图谱”。

完整的监管 Ontology 应该是：

> **Object + Property + Link + Event + Metric + Rule + Action + Policy**

也就是：

```text
对象
+
属性
+
关系
+
事件
+
指标
+
规则
+
监管动作
+
权限/制度
```

这比单纯的知识图谱多了非常关键的 **业务行为和监管闭环**。

---

## 三、第一步：建立“监管对象 Ontology”

这是整个系统的基础。

例如建立下面这些 Object Type。

### 1. 企业类对象

```text
Enterprise 企业
Subsidiary 子企业
JV 合资企业
OverseasCompany 境外企业
SPV 项目公司
Branch 分公司
```

企业属性：

```yaml
Enterprise:
  enterpriseId:
  name:
  unifiedSocialCreditCode:
  enterpriseLevel:
  registeredCapital:
  paidInCapital:
  legalRepresentative:
  establishmentDate:
  industry:
  mainBusiness:
  ownershipType:
  managementLevel:
  riskLevel:
```

---

## 四、第二步：建立“人”的 Ontology

例如：

```text
Person
Executive
Director
Supervisor
Employee
LegalRepresentative
BeneficialOwner
```

Person 可以有：

```yaml
Person:
  personId:
  name:
  position:
  company:
  department:
  appointmentDate:
  authorizationLevel:
```

以后一个人可能同时拥有很多身份：

```text
张某
 ├─ A集团 副总经理
 ├─ B公司 董事
 ├─ C公司 法定代表人
 └─ 某投资基金 投资人
```

这一步对发现关联交易非常重要。

---

## 五、第三步：把监管的核心业务对象全部 Ontology 化

第一版至少建立下面 15 个核心 Object Type：

| Object Type | 含义 |
|---|---|
| Enterprise | 企业 |
| Person | 人员 |
| Organization | 组织 |
| Supplier | 供应商 |
| Customer | 客户 |
| Project | 项目 |
| Investment | 投资 |
| Tender | 招投标 |
| Contract | 合同 |
| BankAccount | 银行账户 |
| Payment | 资金交易 |
| Invoice | 发票 |
| Asset | 资产 |
| Decision | 三重一大/经营决策 |
| RiskEvent | 风险事件 |

然后逐步增加：

```text
Debt
Guarantee
Loan
Fund
Equity
BoardMeeting
Approval
Budget
PurchaseOrder
Bid
Bidder
Acceptance
Litigation
CreditEvent
AuditFinding
RectificationTask
...
```

---

## 六、真正关键的是 Link Type

Ontology 的威力不是 Object，而是：

> **Object 之间建立什么关系。**

例如：

```text
Enterprise
    ──owns────→ Enterprise

Enterprise
    ──employs─→ Person

Person
    ──directorOf→ Enterprise

Enterprise
    ──owns────→ BankAccount

Contract
    ──supplier→ Supplier

Contract
    ──belongsTo→ Project

Payment
    ──forContract→ Contract

Payment
    ──fromAccount→ BankAccount

Payment
    ──toAccount──→ BankAccount

Tender
    ──awardedTo──→ Supplier

Tender
    ──creates────→ Contract

Decision
    ──approves───→ Investment

Decision
    ──approves───→ Project

Decision
    ──approves───→ Contract
```

到这里，数据开始真正变成一个：

## 国资监管关系网络

---

## 七、这样“穿透”就自然产生了

比如监管人员打开：

### 中国XX集团

首先看到：

```text
中国XX集团
│
├── A公司
│   ├── A1公司
│   └── A2公司
│
├── B公司
│   ├── B1公司
│   └── B2公司
│
└── C公司
```

点 A1 公司：

```text
A1公司
│
├── 股东
├── 董事
├── 高管
├── 银行账户
├── 投资项目
├── 采购
├── 合同
├── 供应商
├── 资产
├── 债务
├── 担保
├── 三重一大事项
└── 风险事件
```

再点击某个项目：

```text
项目A
│
├── 立项
├── 可研
├── 投资决策
├── 三重一大
├── 招投标
├── 合同
├── 供应商
├── 发票
├── 付款
├── 项目进度
└── 验收
```

这才是真正意义上的：

> **一层一层穿透。**

---

## 八、一个典型的“采购穿透监管”案例

假设：

```text
某国企
   ↓
采购项目A
   ↓
中标企业：华某科技
   ↓
合同：3200万元
   ↓
已经付款：3100万元
```

系统沿 Ontology 自动穿透。

首先：

```text
采购项目A
     ↓
投标人
 ┌────┼────┐
甲公司 乙公司 华某科技
```

继续穿透：

```text
甲公司 ──股东──→ 王某
乙公司 ──股东──→ 王某
```

发现：

> 两个投标人实际由同一自然人控制。

系统产生：

```text
风险：疑似关联投标
风险等级：高
```

继续：

```text
华某科技
   ↓
股东
   ↓
李某
```

而：

```text
李某
   ↓
关联关系
   ↓
国企采购负责人
```

出现第二个风险。

再继续：

```text
合同3200万
     ↓
付款3100万
     ↓
项目实际完成度
     ↓
35%
```

产生第三个风险：

> **项目进度 35%，付款进度 97%，付款进度明显异常。**

传统 BI 要做三张甚至十张报表。

Ontology 是：

> **一个业务网络。**

---

## 九、资金穿透尤其适合用 Ontology

例如从一笔资金出发：

```text
Payment #P202608010001
金额：5000万
```

向前穿：

```text
付款
 ↓
账户
 ↓
付款企业
 ↓
所属集团
```

向后穿：

```text
付款
 ↓
收款账户
 ↓
供应商
 ↓
供应商股东
 ↓
实际控制人
```

横向穿：

```text
付款
 ↓
合同
 ↓
采购
 ↓
项目
 ↓
预算
```

向上穿：

```text
项目
 ↓
投资决策
 ↓
三重一大
 ↓
董事会
 ↓
审批人
```

最终得到：

```text
集团
↓
企业
↓
项目
↓
采购
↓
合同
↓
付款
↓
银行账户
↓
供应商
↓
供应商股东
↓
实际控制人
```

这就是典型的 **Ontology-based 穿透监管**。

---

## 十、第四个关键：把“监管规则”也做进 Ontology

只有关系图还不是监管平台。

还必须有：

## Risk Rule

例如定义：

### Rule-001：超合同付款

```text
SUM(Payment.amount)
>
Contract.amount
```

触发：

```text
RiskEvent:
    type = 超合同付款
    severity = HIGH
```

### Rule-002：项目进度与付款异常

```text
项目进度 < 50%
AND
付款比例 > 80%
```

触发：

> 项目资金支付异常。

### Rule-003：关联投标

如果：

```text
Bidder A
     ↓
BeneficialOwner
     ↓
Person X

Bidder B
     ↓
BeneficialOwner
     ↓
Person X
```

系统自动产生：

```text
RiskEvent:
  疑似关联投标
```

### Rule-004：控股不控权

例如：

```text
集团持股 60%
```

但：

```text
董事会控制权不足
经营管理权不足
财务控制权不足
关键岗位任免权不足
```

系统判断：

> **可能存在“控股不控权”风险。**

Ontology 在这里特别适合，因为判断“控制权”不能只看一张股权表，而要同时看：

```text
股权
+
董事会席位
+
表决权
+
经营管理权
+
财务权
+
人员任免权
+
协议控制
```

---

## 十一、第五个关键：监管指标也 Ontology 化

比如：

```text
Enterprise
 ├─ Revenue
 ├─ Profit
 ├─ DebtRatio
 ├─ CashFlow
 ├─ ROE
 ├─ InvestmentReturn
 ├─ OverdueReceivables
 └─ RiskScore
```

但指标不是孤立的。

比如：

> 集团负债率突然升高。

点击：

```text
负债率 78%
       ↓
哪些公司贡献？
       ↓
A公司
       ↓
哪些债务？
       ↓
银行贷款
       ↓
哪个项目？
       ↓
房地产项目X
```

从一个 KPI：

> **一直穿透到底层业务事实。**

---

## 十二、第六个关键：把监管动作 Action 也放进 Ontology

很多国内监管平台的问题是：

> 能发现问题，但是解决问题还是线下打电话、Excel、发通知。

Ontology 应该定义 Action Type。

例如：

```text
CreateInvestigation
发起核查

RequestExplanation
要求企业说明

RequestEvidence
要求补充材料

CreateRectification
下发整改

EscalateRisk
升级风险

AssignInvestigator
指定核查人员

CloseRisk
风险销号
```

于是：

```text
RiskEvent
   ↓
发起核查
   ↓
责任企业
   ↓
提交说明
   ↓
上传证据
   ↓
监管人员复核
   ↓
整改
   ↓
验收
   ↓
销号
```

这才形成：

## 监管闭环

---

## 十三、建议整个国资 Ontology 分成 8 个 Domain

不建议建一个几百张表的大一统模型，而是：

```text
                  国资监管 Ontology
                         │
        ┌────────────────┼────────────────┐
        │                │                │
     企业域            人员域           财务域
        │                │                │
     股权域            投资域           资金域
        │                │                │
     采购域 ──────── 合同域 ───────── 项目域
                         │
                     风险监管域
```

### Domain 1：企业与股权

```text
Enterprise
Ownership
Shareholder
BeneficialOwner
Board
Executive
```

### Domain 2：投资

```text
Investment
Project
InvestmentPlan
Approval
PostInvestment
Exit
```

### Domain 3：采购

```text
Procurement
Tender
Bid
Bidder
Supplier
Award
```

### Domain 4：合同

```text
Contract
ContractParty
ContractChange
Acceptance
```

### Domain 5：资金

```text
BankAccount
Payment
Collection
Loan
Guarantee
Debt
```

### Domain 6：财务

```text
Budget
Revenue
Expense
Receivable
Payable
FinancialStatement
Metric
```

### Domain 7：决策

```text
Decision
Meeting
Proposal
Vote
Approval
ThreeMajorOneLarge
```

### Domain 8：风险监管

```text
Risk
RiskRule
Alert
Investigation
Finding
Rectification
Evidence
```

---

## 十四、国资委和国企不要建设两套完全不同的 Ontology

这是架构设计上非常关键的一点。

更推荐：

## 联邦式 Ontology

大致：

```text
                 国资监管标准 Ontology
                         │
            ┌────────────┼────────────┐
            ↓            ↓            ↓
         央企A         央企B        地方国企C
       Ontology       Ontology        Ontology
            │            │             │
          ERP          ERP           ERP
          司库         财务          采购
          合同         投资          合同
          项目         OA            司库
```

国资委定义：

```text
标准 Object Type
标准 Link Type
标准监管指标
标准风险分类
标准数据语义
标准接口
标准监管模型
```

企业可以：

> **继承 + 扩展。**

例如国资委规定：

```text
Contract
Payment
Supplier
Enterprise
Investment
```

是标准对象。

石油企业可以扩展：

```text
OilField
Well
Pipeline
```

电力企业可以扩展：

```text
PowerPlant
GeneratingUnit
Grid
```

航空企业可以扩展：

```text
Aircraft
Route
Airport
```

但是它们最终仍然可以连接：

```text
Project
Contract
Payment
Supplier
Investment
Risk
```

这使监管既能够：

> **统一监管**

又不会：

> **要求所有央企使用完全一样的业务模型。**

---

## 十五、国资委真正需要的是“监管 Ontology”，而不是复制企业 ERP 数据

国资委没有必要把企业：

```text
SAP
用友
金蝶
司库
采购
OA
MES
CRM
```

所有原始数据全部复制过来。

企业内部：

```text
原始业务系统
       ↓
企业数据平台
       ↓
企业 Ontology
```

国资委：

```text
监管 Ontology
       ↑
监管对象
监管指标
重点交易
风险事件
必要明细
```

遇到风险再：

## 按需向下穿透

例如平时看到：

```text
集团
→ 二级企业
→ 风险指标
```

发现异常：

```text
点击风险
→ 三级企业
→ 项目
→ 合同
→ 付款
→ 银行流水
```

这会比“把所有数据全部集中到国资委”现实得多。

---

## 十六、AI 在这个体系中才真正有价值

如果下面已经建立 Ontology：

```text
企业
+
人
+
项目
+
合同
+
资金
+
供应商
+
决策
+
风险
```

那么监管人员可以直接问：

> “过去 12 个月哪些三级以下企业存在重大异常资金支付？”

AI 实际不是直接去查数据库。

而是：

```text
用户问题
    ↓
AI Agent
    ↓
理解 Ontology
    ↓
Enterprise
Payment
Contract
Project
Supplier
Risk
    ↓
图查询 + SQL + 规则模型
    ↓
返回结果
```

继续问：

> 为什么 XX 公司风险这么高？

AI：

```text
XX公司
│
├─ 3笔重大超合同付款
├─ 2个异常投资项目
├─ 1个控股不控权企业
└─ 4家供应商存在关联关系
```

再问：

> 给我分析这 4 家供应商。

继续沿 Ontology 穿透。

这时 AI 才不是：

> **聊天机器人。**

而是：

## 监管 Agent

---

## 十七、最终平台应该长什么样

最终不是传统：

> **国资监管数据大屏**

而应该是：

```text
┌──────────────────────────────────────┐
│           国资智能穿透监管平台          │
├──────────────────────────────────────┤
│                                      │
│  搜索 / AI Copilot / 监管驾驶舱       │
│                                      │
├──────────────────────────────────────┤
│ 企业 │投资│资金│采购│合同│项目│风险│决策 │
├──────────────────────────────────────┤
│             Ontology Runtime          │
│                                      │
│ Object │ Link │ Metric │ Rule │ Action│
├──────────────────────────────────────┤
│             监管模型 / AI             │
│                                      │
│ 风险识别 │ 异常检测 │ 图分析 │ AI Agent │
├──────────────────────────────────────┤
│                数据平台               │
├──────────────────────────────────────┤
│ ERP │司库│采购│合同│OA│项目│工商│司法│银行│
└──────────────────────────────────────┘
```

所以真正的平台核心实际上不是：

**“穿透式监管系统”**

四个字。

而应该是：

## 国资监管 Ontology Operating System

---

## 十八、如果真的准备开发，不要一次做全部

第一版建议只选 **3 个穿透场景**。

### P0-1：企业股权穿透

做到：

```text
集团
→ 子企业
→ 股权
→ 董监高
→ 实际控制关系
→ 参股企业
→ 风险企业
```

### P0-2：合同—资金穿透

做到：

```text
项目
→ 采购
→ 合同
→ 发票
→ 验收
→ 付款
→ 银行账户
→ 供应商
```

### P0-3：供应商风险穿透

做到：

```text
Supplier
→ 股东
→ 实控人
→ 高管
→ 其他供应商
→ 国企人员
→ 历史投标
→ 历史合同
→ 历史付款
→ 司法/信用
```

这三个做完，**Ontology 穿透监管的价值基本就能完整展示出来**。

然后 P1 再扩：

```text
投资穿透
三重一大
债务
担保
资产
应收应付
境外企业
重大项目
```

---

## 十九、这套方法最重要的思想

可以把传统监管和 Ontology 监管的区别概括成：

| 传统国资监管 | Ontology 穿透监管 |
|---|---|
| 看报表 | 看业务对象 |
| 看指标 | 指标穿透到事实 |
| 数据按系统组织 | 数据按现实世界组织 |
| 企业表 | 企业对象 |
| 合同表 | Contract Object |
| 付款表 | Payment Object |
| 靠字段关联 | Link |
| 固定查询 | 动态图遍历 |
| 人工发现风险 | Rule / Model 自动发现 |
| 风险列表 | Risk Object |
| 线下整改 | Action 闭环 |
| AI 查文档 | AI 操作监管 Ontology |

**最核心的一句话：**

> **不要把穿透式监管理解成“把数据钻取做得更深”，而应该把它理解成“构建一个可计算、可追溯、可行动的国资监管数字世界”。Ontology 就是这个数字世界的语义和行为模型。**

如果正在做类似 Palantir 的 Ontology 平台，那么这个场景非常匹配：

- **Ontology Management**：负责定义 Enterprise、Contract、Payment、Supplier、Risk 等模型；
- **Ontology Runtime**：负责把实际国企数据实例化并建立关系；
- **Rule / AI 层**：负责发现异常；
- **Action 层**：完成核查、整改、销号。

这会比单独做一套“穿透监管应用”更有平台价值。

## 相关笔记

- [[企业本体方法论_对外完整版_v2|企业本体方法论]]
- [[中国穿透式监管需求分析]]
- [[DRP平台建设-ljf-updated|DRP 平台建设]]
- [[drp_architecture_detailed|DRP 技术架构]]
