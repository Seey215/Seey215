# 构建 MCP 的专家共识（含代码示例）

基于 Anthropic 的 MCP 实践与业内共识，以下为每项核心共识补充可落地的代码示例（以 Python 语言为基础，模拟 MCP 服务器工具注册逻辑，核心方法参考 `MCPBaseServer.register_tool` 模板，适配 LLM 智能体调用场景）。

## 基础前提：MCP 服务器核心类定义

先定义通用 MCP 服务器基类，模拟工具注册、参数校验、响应处理的核心能力，后续所有示例均基于此类扩展：
```python
from enum import Enum
from typing import Dict, List, Optional, Any

# 模拟MCP协议的响应格式枚举（参考网页中"concise/detailed"设计）
class ResponseFormat(Enum):
    CONCISE = "concise"  # 简洁模式：仅返回决策关键信息
    DETAILED = "detailed"  # 详细模式：含技术ID，支持后续工具调用

# MCP基础服务器类：提供工具注册、执行入口
class MCPBaseServer:
    def __init__(self):
        self.registered_tools: Dict[str, Dict] = {}  # 存储已注册工具

    def register_tool(
        self,
        tool_id: str,          # 工具唯一ID（需符合命名空间规则）
        tool_name: str,        # 工具名称（人类/智能体可理解）
        tool_description: str, # 工具描述（关键：指导智能体正确调用）
        parameters: List[Dict],# 工具参数定义（含必填/类型/示例）
        execute_func: callable # 工具核心执行逻辑（绑定业务逻辑）
    ):
        """MCP工具注册核心方法：将工具接入MCP协议，供智能体调用"""
        self.registered_tools[tool_id] = {
            "tool_name": tool_name,
            "tool_description": tool_description,
            "parameters": parameters,
            "execute": execute_func
        }

    def call_tool(self, tool_id: str, params: Dict) -> Dict:
        """MCP工具调用入口：校验参数+执行+返回格式化结果"""
        if tool_id not in self.registered_tools:
            return {"error": "Tool not found", "tool_id": tool_id}
        
        tool = self.registered_tools[tool_id]
        # 1. 参数校验（基于注册时的定义）
        missing_params = [p["name"] for p in tool["parameters"] if p["required"] and p["name"] not in params]
        if missing_params:
            return {"error": f"Missing required parameters: {missing_params}", "suggestion": f"请补充参数：{missing_params}（示例：{tool['parameters'][0]['example']}）"}
        
        # 2. 执行工具逻辑
        try:
            result = tool["execute"](**params)
            return {"success": True, "data": result}
        except Exception as e:
            return {"error": str(e), "suggestion": tool.get("error_suggestion", "请检查参数格式或联系开发者")}
```

## 共识 1：工具设计需适配智能体 "非确定性决策"，拒绝复刻传统 API

### 核心逻辑

工具需**整合多步骤任务**，避免智能体自行串联多个单一工具（易出错），直接支持复杂决策场景。

### 正例：整合式工具（解决客户重复收费问题）

```python
# 初始化MCP服务器
mcp_server = MCPBaseServer()

# 1. 注册"解决客户重复收费"整合工具（整合3步逻辑）
def resolve_customer_charge_issue(customer_id: str, order_id: Optional[str] = None) -> Dict:
    """核心逻辑：1.查日志→2.分析影响→3.生成方案"""
    # 步骤1：查询该客户重复收费日志
    logs = [{"log_id": "log_123", "time": "2025-09-10", "amount": 99.9, "status": "duplicate"}]
    # 步骤2：分析是否有其他客户受影响
    affected_customers = ["8765", "5432"] if "duplicate" in logs[0]["status"] else []
    # 步骤3：生成解决方案
    solution = f"客户{customer_id}重复收费{logs[0]['amount']}元，受影响客户{affected_customers}，建议优先退款并发送致歉邮件"
    return {
        "customer_id": customer_id,
        "duplicate_logs": logs,
        "affected_customers": affected_customers,
        "solution": solution
    }

# 注册工具到MCP
mcp_server.register_tool(
    tool_id="payment.resolve_duplicate_charge",  # 命名空间：服务(payment)+功能(resolve_duplicate_charge)
    tool_name="解决客户重复收费问题",
    tool_description="功能：一键查询客户重复收费日志、分析受影响客户、生成退款方案；"
                    "参数：customer_id（必填，客户ID，例：9182）、order_id（可选，订单ID，例：ORD-202509）；"
                    "场景：客户反馈"单次支付被扣款多次"时调用",
    parameters=[
        {"name": "customer_id", "type": "str", "required": True, "example": "9182"},
        {"name": "order_id", "type": "str", "required": False, "example": "ORD-202509"}
    ],
    execute_func=resolve_customer_charge_issue
)

# 智能体调用示例（1次调用完成复杂任务）
call_result = mcp_server.call_tool("payment.resolve_duplicate_charge", {"customer_id": "9182"})
print(call_result["data"]["solution"])  # 输出：客户9182重复收费99.9元...
```

### 反例：拆分式工具（需智能体自行串联）

```python
# 1. 注册"查询收费日志"工具
def get_payment_logs(customer_id: str) -> List[Dict]:
    return [{"log_id": "log_123", "time": "2025-09-10", "amount": 99.9, "status": "duplicate"}]

mcp_server.register_tool(
    tool_id="payment.get_logs",
    tool_name="查询客户收费日志",
    tool_description="查询客户收费日志",  # 描述模糊
    parameters=[{"name": "customer_id", "type": "str", "required": True, "example": "9182"}],
    execute_func=get_payment_logs
)

# 2. 注册"分析受影响客户"工具
def analyze_impact(log_id: str) -> List[str]:
    return ["8765", "5432"]  # 需先调用get_payment_logs获取log_id

mcp_server.register_tool(
    tool_id="payment.analyze_impact",
    tool_name="分析受影响客户",
    parameters=[{"name": "log_id", "type": "str", "required": True, "example": "log_123"}],
    execute_func=analyze_impact
)

# 3. 注册"生成方案"工具
def generate_solution(customer_id: str, affected: List[str]) -> str:
    return f"客户{customer_id}需退款，受影响客户{affected}"

mcp_server.register_tool(
    tool_id="payment.generate_solution",
    tool_name="生成退款方案",
    parameters=[{"name": "customer_id", "type": "str", "required": True}, {"name": "affected", "type": "list", "required": True}],
    execute_func=generate_solution
)

# 智能体需3次调用（易遗漏步骤/参数错误）
log_result = mcp_server.call_tool("payment.get_logs", {"customer_id": "9182"})
impact_result = mcp_server.call_tool("payment.analyze_impact", {"log_id": log_result["data"][0]["log_id"]})
solution_result = mcp_server.call_tool("payment.generate_solution", {"customer_id": "9182", "affected": impact_result["data"]})
```

## 共识 2：工具功能需 "高内聚"，聚焦高价值 workflows

### 核心逻辑

工具需整合 "智能体完成某类任务的全流程依赖信息"，避免智能体多次调用拼接碎片数据。

### 正例：高内聚的 "获取客户全景信息" 工具

```python
def get_customer_context(customer_id: str, response_format: ResponseFormat = ResponseFormat.CONCISE) -> Dict:
    """整合客户基本信息、近期交易、服务记录"""
    base_info = {"name": "Sarah Chen", "email": "sarah@acme.com", "member_level": "VIP"}
    recent_trans = [{"order_id": "ORD-202508", "amount": 199.9, "date": "2025-08-25"}]
    service_notes = ["2025-08-30：反馈物流延迟，已补偿优惠券"]
   
    if response_format == ResponseFormat.DETAILED:
        # 详细模式：含user_id等技术字段，支持后续调用（如发送邮件）
        return {
            "customer_id": customer_id,
            "user_id": "usr_789",  # 技术ID
            "base_info": base_info,
            "recent_transactions": recent_trans,
            "service_notes": service_notes
        }
    # 简洁模式：仅返回决策关键信息
    return {
        "customer_name": base_info["name"],
        "member_level": base_info["member_level"],
        "last_transaction": recent_trans[0] if recent_trans else None,
        "latest_service_issue": service_notes[0] if service_notes else None
    }

mcp_server.register_tool(
    tool_id="customer.get_context",
    tool_name="获取客户全景信息",
    tool_description="功能：一次性获取客户基本信息、近期交易、服务记录，支持简洁/详细两种响应模式；"
                    "参数：customer_id（必填，例：9182）、response_format（可选，默认concise，枚举值：concise/detailed）；"
                    "场景：客户取消请求、挽留方案制定时调用",
    parameters=[
        {"name": "customer_id", "type": "str", "required": True, "example": "9182"},
        {"name": "response_format", "type": "enum", "required": False, "enum_values": [e.value for e in ResponseFormat], "default": "concise"}
    ],
    execute_func=get_customer_context
)

# 智能体1次调用获取所有所需信息（无需多次拼接）
context = mcp_server.call_tool("customer.get_context", {"customer_id": "9182", "response_format": "concise"})
print(context["data"]["latest_service_issue"])  # 输出：2025-08-30：反馈物流延迟...
```

### 反例：拆分的 "单一信息" 工具

```python
# 1. 注册"获取客户基本信息"工具
def get_customer_basic(customer_id: str) -> Dict:
    return {"name": "Sarah Chen", "email": "sarah@acme.com"}

mcp_server.register_tool(
    tool_id="customer.get_basic",
    tool_name="获取客户基本信息",
    parameters=[{"name": "customer_id", "type": "str", "required": True, "example": "9182"}],
    execute_func=get_customer_basic
)

# 2. 注册"获取客户交易"工具
def list_customer_trans(customer_id: str) -> List[Dict]:
    return [{"order_id": "ORD-202508", "amount": 199.9, "date": "2025-08-25"}]

mcp_server.register_tool(
    tool_id="customer.list_trans",
    tool_name="获取客户交易记录",
    parameters=[{"name": "customer_id", "type": "str", "required": True, "example": "9182"}],
    execute_func=list_customer_trans
)

# 3. 注册"获取服务记录"工具
def get_service_notes(customer_id: str) -> List[str]:
    return ["2025-08-30：反馈物流延迟，已补偿优惠券"]

mcp_server.register_tool(
    tool_id="customer.get_notes",
    tool_name="获取客户服务记录",
    parameters=[{"name": "customer_id", "type": "str", "required": True, "example": "9182"}],
    execute_func=get_service_notes
)

# 智能体需3次调用（浪费token，且易遗漏某类信息）
basic = mcp_server.call_tool("customer.get_basic", {"customer_id": "9182"})
trans = mcp_server.call_tool("customer.list_trans", {"customer_id": "9182"})
notes = mcp_server.call_tool("customer.get_notes", {"customer_id": "9182"})
```

## 共识 3：命名空间需 "清晰区分资源 / 服务"，消除工具歧义

### 核心逻辑

工具 ID 需按 "**服务领域 + 资源类型 + 操作**" 命名（如`asana.projects.search`），帮助智能体在数百个工具中快速定位。

### 正例：规范命名空间的工具

```python
# 1. Asana服务：项目相关工具
def asana_search_projects(project_name: str, page: int = 1, page_size: int = 10) -> List[Dict]:
    return [{"project_id": "asana_prj_1", "name": "Acme产品开发", "status": "进行中"}]

mcp_server.register_tool(
    tool_id="asana.projects.search",  # 命名空间：服务(asana)+资源(projects)+操作(search)
    tool_name="搜索Asana项目",
    tool_description="功能：按项目名称搜索Asana中的项目，支持分页；"
                    "参数：project_name（必填，例：Acme产品开发）、page（可选，默认1）、page_size（可选，默认10）",
    parameters=[
        {"name": "project_name", "type": "str", "required": True, "example": "Acme产品开发"},
        {"name": "page", "type": "int", "required": False, "default": 1},
        {"name": "page_size", "type": "int", "required": False, "default": 10}
    ],
    execute_func=asana_search_projects
)

# 2. Asana服务：用户相关工具
def asana_get_user_tasks(user_id: str) -> List[Dict]:
    return [{"task_id": "asana_task_1", "title": "完成需求文档", "due_date": "2025-09-15"}]

mcp_server.register_tool(
    tool_id="asana.users.get_tasks",  # 命名空间：服务(asana)+资源(users)+操作(get_tasks)
    tool_name="获取Asana用户任务",
    parameters=[{"name": "user_id", "type": "str", "required": True, "example": "asana_usr_123"}],
    execute_func=asana_get_user_tasks
)

# 智能体可通过工具ID快速识别用途（如"asana.projects.search"明确是Asana的项目搜索）
prj_result = mcp_server.call_tool("asana.projects.search", {"project_name": "Acme产品开发"})
```

### 反例：模糊命名的工具

```python
# 1. 模糊工具ID：未区分资源类型
def search_asana(keyword: str) -> List[Dict]:
    return [{"id": "asana_1", "name": "Acme产品开发", "type": "project"}]  # 无法确定是项目/任务/用户

mcp_server.register_tool(
    tool_id="asana.search",  # 仅含服务(asana)，无资源类型
    tool_name="搜索Asana内容",
    tool_description="搜索Asana中的内容",  # 描述模糊
    parameters=[{"name": "keyword", "type": "str", "required": True, "example": "Acme"}],
    execute_func=search_asana
)

# 2. 混乱工具ID：无服务领域
def get_tasks(user_id: str) -> List[Dict]:
    return [{"task_id": "task_1", "title": "完成需求文档"}]  # 无法确定是Asana/Jira/其他工具的任务

mcp_server.register_tool(
    tool_id="get_tasks",  # 无服务领域，无资源类型
    tool_name="获取任务",
    parameters=[{"name": "user_id", "type": "str", "required": True, "example": "123"}],
    execute_func=get_tasks
)

# 智能体调用时易混淆（如"asana.search"无法确定搜索项目还是任务，"get_tasks"无法确定是哪个平台的任务）
confused_result = mcp_server.call_tool("asana.search", {"keyword": "Acme"})
```

## 共识 4：工具响应需 "高信号、低冗余"，减少智能体筛选成本

### 核心逻辑

响应仅返回智能体 "下游决策所需信息"（如`name`/`status`），剔除无意义的技术标识符（如`uuid`/`mime_type`）。

### 正例：高信号响应工具

```python
def get_file_info(file_id: str, response_format: ResponseFormat = ResponseFormat.CONCISE) -> Dict:
    # 底层数据（含技术字段，但仅返回关键信息）
    raw_data = {
        "file_id": file_id,
        "uuid": "f-uuid-123e4567-e89b-12d3-a456-426614174000",  # 冗余技术字段
        "name": "Acme项目计划.pdf",
        "file_type": "PDF",
        "size_mb": 2.5,
        "last_edited": "2025-09-01",
        "server_path": "/data/files/123",  # 冗余技术字段
        "thumbnail_url_256px": "https://cdn/acme-256.jpg"  # 冗余技术字段
    }

    if response_format == ResponseFormat.CONCISE:
        # 仅返回智能体决策关键信息（如判断是否下载/分享）
        return {
            "file_name": raw_data["name"],
            "file_type": raw_data["file_type"],
            "size_mb": raw_data["size_mb"],
            "last_edited": raw_data["last_edited"]
        }
    # 详细模式仅补充必要技术ID（如file_id用于后续下载工具调用）
    return {
        "file_id": raw_data["file_id"],
        "file_name": raw_data["name"],
        "file_type": raw_data["file_type"],
        "size_mb": raw_data["size_mb"],
        "last_edited": raw_data["last_edited"]
    }

mcp_server.register_tool(
    tool_id="file.get_info",
    tool_name="获取文件信息",
    tool_description="功能：返回文件名称、类型、大小等关键信息，剔除冗余技术字段；"
                    "参数：file_id（必填，例：file_456）、response_format（可选，默认concise）",
    parameters=[
        {"name": "file_id", "type": "str", "required": True, "example": "file_456"},
        {"name": "response_format", "type": "enum", "required": False, "default": "concise", "enum_values": [e.value for e in ResponseFormat]}
    ],
    execute_func=get_file_info
)

# 响应示例（无冗余字段，智能体可直接用）
file_info = mcp_server.call_tool("file.get_info", {"file_id": "file_456"})
print(file_info["data"])
# 输出：{"file_name": "Acme项目计划.pdf", "file_type": "PDF", "size_mb": 2.5, "last_edited": "2025-09-01"}
```

### 反例：冗余响应工具

```python
def get_file_raw_data(file_id: str) -> Dict:
    # 返回所有底层数据，含大量智能体无用字段
    return {
        "file_id": file_id,
        "uuid": "f-uuid-123e4567-e89b-12d3-a456-426614174000",
        "name": "Acme项目计划.pdf",
        "mime_type": "application/pdf",  # 冗余（智能体只需"PDF"）
        "size_bytes": 2621440,  # 冗余（智能体需手动转MB）
        "server_path": "/data/files/123",
        "thumbnail_url_256px": "https://cdn/acme-256.jpg",
        "thumbnail_url_512px": "https://cdn/acme-512.jpg",
        "upload_server_ip": "10.0.0.1"  # 完全无关
    }

mcp_server.register_tool(
    tool_id="file.get_raw",
    tool_name="获取文件原始数据",
    parameters=[{"name": "file_id", "type": "str", "required": True, "example": "file_456"}],
    execute_func=get_file_raw_data
)

# 响应示例（冗余字段占比超50%，智能体需手动筛选）
raw_data = mcp_server.call_tool("file.get_raw", {"file_id": "file_456"})
print(raw_data["data"]["mime_type"])  # 智能体需自行映射为"PDF"
print(raw_data["data"]["size_bytes"] / 1024 / 1024)  # 智能体需手动转MB
```

## 共识 5：token 效率需 "主动控制"，避免上下文溢出

### 核心逻辑

通过**分页、过滤、截断**控制响应规模，超过 token 限制时提示智能体 "缩小查询范围"，避免耗尽智能体上下文窗口。

### 正例：token 效率优化工具

```python
def search_logs(
    keyword: str,
    time_range: str,  # 格式："YYYY-MM-DD to YYYY-MM-DD"
    page: int = 1,
    page_size: int = 50,  # 默认分页：避免一次性返回过多数据
    max_tokens: int = 2000  # 最大token限制（约500条日志）
) -> Dict:
    # 模拟海量日志（1000条）
    all_logs = [{"log_id": f"log_{i}", "content": f"[{time_range}] {keyword} 日志内容{i}"} for i in range(1000)]
   
    # 1. 分页计算
    start_idx = (page - 1) * page_size
    end_idx = start_idx + page_size
    paginated_logs = all_logs[start_idx:end_idx]
   
    # 2. Token估算（简化：每条日志约40 token）
    total_tokens = len(paginated_logs) * 40
    truncated = False
    if total_tokens > max_tokens:
        # 截断并提示
        max_logs = max_tokens // 40
        paginated_logs = paginated_logs[:max_logs]
        truncated = True
   
    return {
        "logs": paginated_logs,
        "pagination": {
            "current_page": page,
            "page_size": page_size,
            "total_pages": (len(all_logs) + page_size - 1) // page_size
        },
        "truncated": truncated,
        "suggestion": "若日志不完整，可缩小time_range（例：2025-09-10 to 2025-09-10）或减小page_size" if truncated else None
    }

mcp_server.register_tool(
    tool_id="logs.search",
    tool_name="搜索系统日志",
    tool_description="功能：按关键词和时间范围搜索日志，支持分页和token截断，避免上下文溢出；"
                    "参数：keyword（必填，例：支付失败）、time_range（必填，例：2025-09-01 to 2025-09-10）、"
                    "page（可选，默认1）、page_size（可选，默认50）、max_tokens（可选，默认2000）",
    parameters=[
        {"name": "keyword", "type": "str", "required": True, "example": "支付失败"},
        {"name": "time_range", "type": "str", "required": True, "example": "2025-09-01 to 2025-09-10"},
        {"name": "page", "type": "int", "required": False, "default": 1},
        {"name": "page_size", "type": "int", "required": False, "default": 50},
        {"name": "max_tokens", "type": "int", "required": False, "default": 2000}
    ],
    execute_func=search_logs
)

# 调用示例：time_range过宽导致token超限制，工具主动截断并提示
logs_result = mcp_server.call_tool(
    "logs.search",
    {"keyword": "支付失败", "time_range": "2025-01-01 to 2025-09-10", "page_size": 100}
)
print(logs_result["data"]["truncated"])  # 输出：True
print(logs_result["data"]["suggestion"])  # 输出：若日志不完整，可缩小time_range...
```

### 反例：无 token 控制工具

```python
def get_all_logs(keyword: str) -> Dict:
    # 无分页/截断：返回所有匹配日志（可能1000+条，超智能体上下文）
    all_logs = [{"log_id": f"log_{i}", "content": f"日志内容{i}：{keyword}"} for i in range(1000)]
    return {
        "logs": all_logs,
        "count": len(all_logs)
    }

mcp_server.register_tool(
    tool_id="logs.get_all",
    tool_name="获取所有匹配日志",
    tool_description="获取所有含关键词的日志",  # 未提示规模风险
    parameters=[{"name": "keyword", "type": "str", "required": True, "example": "支付失败"}],
    execute_func=get_all_logs
)

# 调用示例：返回1000条日志（约40000 token），直接超出Claude Code默认25000 token限制
all_logs_result = mcp_server.call_tool("logs.get_all", {"keyword": "支付失败"})
print(len(all_logs_result["data"]["logs"]))  # 输出：1000（智能体上下文溢出，任务失败）
```

## 共识 6：工具描述需 "清晰无歧义"，降低智能体误用率

### 核心逻辑

工具描述需明确 "**功能 + 参数含义 + 示例 + 场景**"，尤其参数需标注类型、必填性、示例，避免智能体传错值。

### 正例：清晰描述的工具

```python
def send_customer_email(
    customer_id: str,
    email_type: str,  # 枚举：refund_confirm/retention_offer
    template_id: Optional[str] = None
) -> Dict:
    """发送客户邮件：根据类型自动匹配模板（无template_id时）"""
    template_map = {
        "refund_confirm": "tpl_refund_01",
        "retention_offer": "tpl_retention_01"
    }
    used_template = template_id or template_map[email_type]
    return {
        "status": "sent",
        "customer_id": customer_id,
        "email_type": email_type,
        "used_template": used_template
    }

mcp_server.register_tool(
    tool_id="email.send_customer",
    tool_name="发送客户邮件",
    tool_description="功能：向指定客户发送邮件，支持退款确认、挽留方案两种类型，可自定义模板；"
                    "参数说明："
                    "- customer_id（必填，客户唯一ID，例：9182）；"
                    "- email_type（必填，邮件类型，仅支持：refund_confirm（退款确认）、retention_offer（挽留方案））；"
                    "- template_id（可选，模板ID，例：tpl_custom_01，不填则用默认模板）；"
                    "调用场景：客户退款后发送确认邮件、客户取消时发送挽留方案；"
                    "错误提示：若email_type不在枚举内，需重新传入refund_confirm或retention_offer",
    parameters=[
        {"name": "customer_id", "type": "str", "required": True, "example": "9182"},
        {"name": "email_type", "type": "str", "required": True, "example": "retention_offer", "enum_values": ["refund_confirm", "retention_offer"]},
        {"name": "template_id", "type": "str", "required": False, "example": "tpl_custom_01"}
    ],
    execute_func=send_customer_email,
    error_suggestion="请检查email_type是否为'refund_confirm'或'retention_offer'，或补充template_id"
)

# 智能体可按描述正确调用（无歧义）
email_result = mcp_server.call_tool(
    "email.send_customer",
    {"customer_id": "9182", "email_type": "retention_offer"}
)
print(email_result["data"]["status"])  # 输出：sent
```

### 反例：模糊描述的工具

```python
def send_email(customer_id: str, type: str, template: Optional[str] = None) -> Dict:
    """发送邮件"""  # 描述模糊
    if type not in ["refund", "retention"]:
        raise ValueError("类型错误")
    return {"status": "sent"}

mcp_server.register_tool(
    tool_id="email.send",
    tool_name="发送邮件",
    tool_description="给客户发邮件，参数含类型和模板",  # 未说明参数含义/示例
    parameters=[
        {"name": "customer_id", "type": "str", "required": True},  # 无示例
        {"name": "type", "type": "str", "required": True},  # 无枚举值说明
        {"name": "template", "type": "str", "required": False}  # 无示例
    ],
    execute_func=send_email
)

# 智能体易误用（参数模糊）
# 错误1：type传"挽留"（实际需传"retention"）
error1 = mcp_server.call_tool("email.send", {"customer_id": "9182", "type": "挽留"})
print(error1["error"])  # 输出：类型错误（无修正建议）

# 错误2：不知template参数格式（传"挽留模板"而非ID）
error2 = mcp_server.call_tool("email.send", {"customer_id": "9182", "type": "retention", "template": "挽留模板"})
```

## 共识 9：错误处理需 "提供可行动建议"，引导智能体修正

### 核心逻辑

错误响应需明确 "**错误原因 + 具体修正步骤**"，而非仅返回错误码或模糊提示，帮助智能体快速调整调用参数。

### 正例：可行动错误提示工具

```python
def create_meeting(
    attendee_emails: List[str],
    start_time: str,  # 格式："YYYY-MM-DD HH:MM"（24小时制）
    duration_min: int  # 范围：15-180
) -> Dict:
    # 参数校验：明确错误原因
    if not all("@" in email for email in attendee_emails):
        raise ValueError("部分参会者邮箱格式无效")
    if len(start_time.split(" ")) != 2 or len(start_time.split(":")[1]) != 2:
        raise ValueError("开始时间格式错误")
    if duration_min < 15 or duration_min > 180:
        raise ValueError("会议时长需在15-180分钟之间")
   
    return {"meeting_id": "meet_789", "status": "created", "start_time": start_time}

mcp_server.register_tool(
    tool_id="calendar.create_meeting",
    tool_name="创建会议",
    tool_description="功能：创建日历会议，指定参会者、时间、时长；"
                    "参数：attendee_emails（必填，邮箱列表，例：['jane@acme.com']）、"
                    "start_time（必填，格式：YYYY-MM-DD HH:MM，例：2025-09-15 14:30）、"
                    "duration_min（必填，15-180分钟，例：60）",
    parameters=[
        {"name": "attendee_emails", "type": "list", "required": True, "example": ["jane@acme.com", "john@acme.com"]},
        {"name": "start_time", "type": "str", "required": True, "example": "2025-09-15 14:30"},
        {"name": "duration_min", "type": "int", "required": True, "example": 60}
    ],
    execute_func=create_meeting,
    error_suggestion=lambda e: {
        "邮箱错误": "请检查参会者邮箱是否含'@'（例：['jane@acme.com']）",
        "时间格式错误": "请按'YYYY-MM-DD HH:MM'格式传入（例：2025-09-15 14:30）",
        "时长错误": "会议时长需设置为15-180分钟（例：30/60）"
    }.get(str(e).split("：")[0], "请检查参数格式是否符合示例")
)

# 错误调用示例：时间格式错误，返回可行动建议
error_result = mcp_server.call_tool(
    "calendar.create_meeting",
    {"attendee_emails": ["jane@acme.com"], "start_time": "2025-09-15 2点半", "duration_min": 60}
)
print(error_result["error"])  # 输出：开始时间格式错误
print(error_result["suggestion"])  # 输出：请按'YYYY-MM-DD HH:MM'格式传入（例：2025-09-15 14:30）
```

### 反例：模糊错误提示工具

```python
def create_meeting_v2(attendees: List[str], time: str, duration: int) -> Dict:
    if not all("@" in a for a in attendees):
        raise ValueError("E001")  # 错误码无含义
    if len(time.split(" ")) != 2:
        raise ValueError("E002")
    if duration < 15 or duration > 180:
        raise ValueError("E003")
    return {"meeting_id": "meet_789"}

mcp_server.register_tool(
    tool_id="calendar.create_meeting_v2",
    tool_name="创建会议V2",
    parameters=[
        {"name": "attendees", "type": "list", "required": True},
        {"name": "time", "type": "str", "required": True},
        {"name": "duration", "type": "int", "required": True}
    ],
    execute_func=create_meeting_v2
)

# 错误调用示例：仅返回错误码，智能体无法修正
error_result_v2 = mcp_server.call_tool(
    "calendar.create_meeting_v2",
    {"attendees": ["jane.acme.com"], "time": "2025-09-15", "duration": 200}
)
print(error_result_v2["error"])  # 输出：E001（智能体无法理解含义）
print(error_result_v2.get("suggestion"))  # 输出：None（无修正建议）
```

## 总结：MCP 工具代码设计的核心准则

1.  **命名规范**：工具 ID 按 "服务 + 资源 + 操作" 命名（如`payment.resolve_duplicate_charge`）；

2.  **描述清晰**：明确 "功能 + 参数含义 + 示例 + 场景"，避免歧义；

3.  **响应精简**：仅返回智能体决策关键信息，剔除冗余技术字段；

4.  **效率控制**：通过分页、截断限制 token 消耗，超限时提示缩小查询范围；

5.  **错误友好**：错误响应需含 "原因 + 可行动建议"，而非错误码；

6.  **高内聚**：整合多步骤逻辑，避免智能体多次调用拼接数据。

以上代码示例均基于 Anthropic 的 MCP 实践逻辑，可直接适配 Claude 等 LLM 智能体，降低调用错误率并提升任务完成效率。

## 参考文档

*   [Writing effective tools for AI agents—using AI agents \ Anthropic](https://www.anthropic.com/engineering/writing-tools-for-agents)

*   [豆包对话](https://www.doubao.com/thread/w1590f5b1360ca747)

> （注：文档部分内容可能由 AI 生成）