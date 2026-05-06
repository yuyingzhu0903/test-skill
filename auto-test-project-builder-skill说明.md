# AutoTestProjectBuilder Skill — 完整说明

> Skill 文件位置：`~/.claude/skills/AutoTestProjectBuilder/SKILL.md`
> 永久固化，所有会话自动生效，团队成员共享同一套 skills 目录即可使用。

---

## 1. Skill 功能定位

基于用户提供的任意前后端业务项目，自动执行：

**静态代码分析 → 生成完整可运行测试工程**

完全对齐 `factory-hold-test` 标准架构，支持：
- E2E 自动化全链路覆盖
- Harness 调度 / CI/CD 流水线接入
- 测试数据 YAML 驱动管理
- 主流程 + 异常 + 边界用例自动生成
- 日志 + Allure 报告自动输出

---

## 2. 用户只需提供三个输入

| 输入 | 说明 | 示例 |
|------|------|------|
| 后端项目路径 | Java/Spring Boot 源码目录 | `/path/to/my-service` |
| 前端项目路径 | Vue/React 源码目录（可用 Postman collection 替代） | `/path/to/my-web` |
| 目标测试工程名 | 生成的测试工程目录名 | `my-module-test` |

---

## 3. Skill 执行流程图

```
START（接收三个输入）
    │
    ▼
PHASE 1：分析后端
    扫描 Controller / FeignClient / DTO / Enum / Mapper
    提取：接口列表、字段约束、枚举值、DB表结构
    输出：analysis/backend_api_inventory.md
    │
    ▼
PHASE 2：分析前端
    扫描 .vue/.tsx、api/*.js、constants/
    补充：前端必填项、联动规则、入参格式
    │
    ▼
PHASE 3：合并 API Contract
    后端必填 + 前端必填 + 枚举值 → 每个接口完整契约
    │
    ▼
PHASE 4：生成目录结构
    config/ common/ data/ service/ testcases/ reports/ analysis/
    │
    ▼
PHASE 5：生成基础设施代码
    http_client + auth_token + db_client + assertions + DataFactory
    日志规则：每次执行生成 reports/{timestamp}.log（一次执行一个日志文件）
    │
    ▼
PHASE 6：生成 Service 层
    每个业务模块 → 一个 *_service.py，方法一一对应接口，无断言
    │
    ▼
PHASE 7：生成 YAML 测试数据
    正向 / 异常 / 边界 三类数据，枚举值全部来自源码分析
    │
    ▼
PHASE 8：生成测试用例
    主流程串联 + 状态机守卫 + 负向用例 + API+DB 双重验证
    │
    ▼
PHASE 9：生成配置 & 文档
    config.yaml / .env.sit / CLAUDE.md / run_commands.md
    │
    ▼
PHASE 10：验证 & 输出报告
    pytest --collect-only 验证无 import 错误
    输出：模块数、接口数、用例数、TODO 清单
    提交到 git 分支 test/{project_name}-init
    │
    ▼
DONE：完整可运行测试工程
```

---

## 4. 生成的标准测试工程目录结构

```
{project_name}/
├── conftest.py               # session 级 fixtures：settings/auth/http/db/services
├── pytest.ini                # markers、日志、addopts
├── requirements.txt          # pytest + requests + allure + pymysql + dotenv + yaml
├── CLAUDE.md                 # 工程规范（E2E优先、YAML驱动、无硬编码）
├── .env.sit                  # SIT 环境变量（DB、超时、base_url）
├── .env.ontest               # OnTest 环境变量
│
├── config/
│   ├── config.yaml           # 多服务 URL 路由（${env}模板） + OAuth 凭证
│   └── settings.py           # 模板解析 + get_host(service) 方法
│
├── common/
│   ├── http_client.py        # Bearer 自动注入 + 3次重试 + Allure step + 慢请求检测
│   ├── auth_token.py         # OAuth token 缓存管理（60s buffer）
│   ├── db_client.py          # 连接池（DBUtils+PyMySQL）+ fetch_one/fetch_all/execute
│   └── assertions.py         # assert_success / assert_field_eq / SoftAssert / assert_db_row
│
├── data/
│   ├── factory.py            # DataFactory（时间戳+随机后缀，无需清库）
│   └── test_data/
│       └── {module}_cases.yaml   # 每模块一个，三段式：happy_path / error_cases / boundary
│
├── service/
│   └── {module}_service.py   # 每模块一个，方法=接口，只调用不断言
│
├── testcases/
│   ├── conftest.py           # 状态 fixture 链（state1 → state2 → final_state）
│   └── test_{module}_lifecycle.py  # 主流程 + 状态守卫 + DB 双验证
│
├── run_test/
│   └── run_commands.md       # 常用执行命令速查
│
├── reports/                  # 运行产物
│   ├── allure-results/       # Allure 原始数据
│   ├── allure-html/          # Allure HTML 报告（按时间戳隔离）
│   └── html/                 # pytest-html 报告
│
└── analysis/                 # 分析产物（只读参考）
    ├── backend_api_inventory.md   # 全量接口文档（从源码提取）
    └── module_interface_spec.md   # 每模块接口规格（含枚举值、约束）
```

---

## 5. 生成的 YAML 测试数据结构（三段式）

```yaml
# data/test_data/{module}_cases.yaml

# ── 正向用例 ──────────────────────────────────
happy_path:
  basic_create:
    materialCode: "X01-89030009"       # 来自源码枚举/配置
    materialStatus: "SOP"              # 来自 MaterialStatusEnum.java
    reportFaultQty: 1
    dutyIsMyDepartment: false
    # ... 所有必填字段（后端+前端）

# ── 异常用例 ──────────────────────────────────
error_cases:
  missing_required_materialCode:
    # 缺少 materialCode，其余字段正常
    expected_error_code: 400

  invalid_enum_materialStatus:
    materialStatus: "INVALID_VALUE"
    expected_error_code: 400

# ── 边界用例 ──────────────────────────────────
boundary:
  min_reportFaultQty:
    reportFaultQty: 1                  # 最小值
  max_reportFaultQty:
    reportFaultQty: 9999               # 最大值
  boundary_minus_1:
    reportFaultQty: 0                  # 边界-1（应报错）
```

---

## 6. 生成的测试用例结构

```python
# testcases/test_{module}_lifecycle.py

@allure.epic("{系统名称}")
@allure.feature("{模块名称}")
class Test{Module}Lifecycle:

    @allure.story("主流程-创建")
    @pytest.mark.smoke
    @pytest.mark.p0
    def test_create_{entity}(self, {module}_service, test_data, db_client):
        case = test_data["happy_path"]["basic_create"]
        payload = {k: v for k, v in case.items() if not k.startswith("expected")}

        # 1. 调用接口
        resp = {module}_service.create(payload)

        # 2. 验证 API 响应
        assert_success(resp)
        entity_id = assert_field_not_none(resp, "data.aid")

        # 3. 验证 DB（双重验证）
        if db_client:
            row = db_client.fetch_one(
                "SELECT status, {key_field} FROM {main_table} WHERE aid = %s",
                (entity_id,)
            )
            assert_db_row(row, {
                "{key_field}": case["{keyField}"],
                "status": "CREATED"
            })

    @allure.story("主流程-提交")
    @pytest.mark.p0
    def test_submit_{entity}(self, created_{entity}, {module}_service, db_client):
        # created_{entity} fixture 已自动完成创建步骤
        entity_id = created_{entity}
        resp = {module}_service.submit({"aid": entity_id})
        assert_success(resp)
        # DB 验证 ...
```

---

## 7. 不可违反的质量规则

| 规则 | 说明 |
|------|------|
| **YAML 驱动** | 测试数据全部在 YAML 文件，代码中禁止硬编码业务数据 |
| **无 DTO 模型** | 测试工程只用 plain dict，禁止生成 dataclass/DTO 类 |
| **英文方法名** | 所有 Python 方法统一 snake_case 英文命名 |
| **每次执行一个日志** | `reports/{timestamp}.log`，禁止每个测试用例单独建日志 |
| **双重验证** | 每个生命周期用例必须同时验证 API 响应 + DB 状态 |
| **DataFactory 唯一性** | 所有业务标识通过时间戳+随机后缀生成，无需清库 |
| **无硬编码凭证** | DB密码/Token 等全部写入 `.env.sit`，代码中禁止出现 |
| **Service 层无断言** | Service 只做 HTTP 调用，断言全部在 testcases 层 |

---

## 8. 团队调用说明

### 方式一：自然语言触发（推荐）

```
帮我为 li-xxx 出入库系统构建测试工程：

后端源码：/Users/me/projects/li-xxx/li-xxx-service
前端源码：/Users/me/projects/li-xxx/li-xxx-web
目标路径：/Users/me/projects/li-xxx-test

要求覆盖：出库单创建、审核、发货、回单 全流程
```

### 方式二：明确调用 Skill

```
/AutoTestProjectBuilder
后端：/path/to/backend
前端：/path/to/frontend
工程名：my-module-test
```

### 方式三：只有后端（无前端）

```
根据后端代码生成测试工程：
后端：/path/to/backend
工程名：my-module-test
（无前端，用 Postman collection 替代：/path/to/collection.json）
```

---

## 9. 执行完成后的输出报告

```
╔══════════════════════════════════════════════════════════════╗
║          AutoTestProjectBuilder — Generation Complete         ║
╠══════════════════════════════════════════════════════════════╣
║ Project: li-xxx-test                                       ║
║ Location: /Users/me/projects/li-xxx-test                   ║
╠══════════════════════════════════════════════════════════════╣
║ Analysis Results:                                             ║
║   Modules discovered:        5                                ║
║   API endpoints extracted:   23                               ║
║   Enum values mapped:        47                               ║
╠══════════════════════════════════════════════════════════════╣
║ Generated Files:                                              ║
║   Infrastructure files:      6  (http_client, auth, db...)    ║
║   Service files:             5  (one per module)              ║
║   Test case files:           5  (one per module)              ║
║   Test data YAML files:      5  (one per module)              ║
║   Config files:              6  (config.yaml, .env.sit...)    ║
╠══════════════════════════════════════════════════════════════╣
║ Test Coverage:                                                ║
║   Total test cases:          38                               ║
║   Happy path cases:          15                               ║
║   Negative cases:            16                               ║
║   Boundary cases:            7                                ║
╠══════════════════════════════════════════════════════════════╣
║ TODO Items (requires human input):                            ║
║   ⚠  DB credentials in .env.sit                              ║
║   ⚠  Auth service URL in config.yaml                         ║
║   ⚠  serviceId/appId in service_auth                         ║
║   ⚠  Fields marked "# TODO: query DB" in YAML files         ║
╠══════════════════════════════════════════════════════════════╣
║ To run immediately:                                           ║
║   cd /Users/me/projects/li-xxx-test                        ║
║   pip install -r requirements.txt                             ║
║   pytest -m smoke --env=sit -v                                ║
╚══════════════════════════════════════════════════════════════╝
```

---

## 10. 常用执行命令（自动生成到 run_commands.md）

```bash
# 全量执行
pytest --env=sit --alluredir=reports/allure-results

# 冒烟测试（快速验证）
pytest -m smoke --env=sit -v

# P0 核心用例
pytest -m p0 --env=sit -v

# 指定模块
pytest testcases/test_{module}_lifecycle.py --env=sit -v

# 无 VPN（跳过 DB 验证）
pytest --env=sit --no-db -v

# 生成 Allure HTML 报告
allure generate reports/allure-results \
  -o reports/allure-html/$(date +%Y%m%d_%H%M%S) --clean

# 打开 Allure 报告
allure open reports/allure-html/{timestamp}
```

---

## 11. 与 factory-hold-test 架构对照

| 组件 | factory-hold-test | AutoTestProjectBuilder 生成 |
|------|-------------------|---------------------------|
| HTTP 客户端 | `common/http_client.py` | 完全一致，自动注入 Bearer token |
| 认证管理 | `common/auth_token.py` | 完全一致，class 级别 token 缓存 |
| 数据库客户端 | `common/db_client.py` | 完全一致，DBUtils 连接池 |
| 断言工具 | `common/assertions.py` | 完全一致，SoftAssert + DB 断言 |
| 数据工厂 | `data/factory.py` | 同模式，方法名按新项目实体命名 |
| 配置管理 | `config/settings.py` | 完全一致，${env} 模板解析 |
| Service 层 | `service/hold_service.py` | 每模块生成，方法名按接口命名 |
| 测试数据 | `data/test_data/*.yaml` | 每模块生成，三段式结构 |
| Fixture 链 | `testcases/conftest.py` | 按状态机自动生成 staged→submitted→audited |
| 测试用例 | `testcases/test_*.py` | 每模块生成，含 Allure 标注 |

---

*Generated by AutoTestProjectBuilder Skill · factory-hold-test standard architecture*
