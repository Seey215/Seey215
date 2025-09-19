# Expert Consensus on Building MCP (with Code Examples)

Based on Anthropic's MCP practices and industry consensus, the following are executable code examples for each core consensus (based on Python language, simulating MCP server tool registration logic, core methods refer to `MCPBaseServer.register_tool` template, adapted for LLM agent calls):

## Basic premise: MCP server core class definition

First define the general MCP server base class, simulating the core capabilities of tool registration, parameter verification, and response processing. All subsequent examples are based on this class extension:
```python
from enum import Enum
from typing import Dict, List, Optional, Any

# Simulate the MCP protocol response format enumeration (refer to "concise/detailed" design in the web page)
class ResponseFormat(Enum):
    CONCISE = "concise"  # Concise mode: only return decision-critical information
    DETAILED = "detailed"  # Detailed mode: contains technical ID, supports subsequent tool calls

# MCP base server class: provides tool registration and execution entry
class MCPBaseServer:
    def __init__(self):
        self.registered_tools: Dict[str, Dict] = {}  # Store registered tools

    def register_tool(
        self,
        tool_id: str,          # Tool unique ID (must conform to namespace rules)
        tool_name: str,        # Tool name (human/agent understandable)
        tool_description: str, # Tool description (key: guide agents to call correctly)
        parameters: List[Dict],# Tool parameter definition (including required/type/example)
        execute_func: callable # Tool core execution logic (bound business logic)
    ):
        """MCP tool registration core method: connect tools to MCP protocol for agent calls"""
        self.registered_tools[tool_id] = {
            "tool_name": tool_name,
            "tool_description": tool_description,
            "parameters": parameters,
            "execute": execute_func
        }

    def call_tool(self, tool_id: str, params: Dict) -> Dict:
        """MCP tool call entry: parameter verification + execution + return formatted results"""
        if tool_id not in self.registered_tools:
            return {"error": "Tool not found", "tool_id": tool_id}
        
        tool = self.registered_tools[tool_id]
        # 1. Parameter verification (based on definition at registration)
        missing_params = [p["name"] for p in tool["parameters"] if p["required"] and p["name"] not in params]
        if missing_params:
            return {"error": f"Missing required parameters: {missing_params}", "suggestion": f"Please add parameters: {missing_params} (example: {tool['parameters'][0]['example']})"}
        
        # 2. Execute tool logic
        try:
            result = tool["execute"](**params)
            return {"success": True, "data": result}
        except Exception as e:
            return {"error": str(e), "suggestion": tool.get("error_suggestion", "Please check parameter format or contact developer")}
```

## Consensus 1: Tool design needs to adapt to agent "non-deterministic decision-making" and refuse to replicate traditional APIs

### Core logic

Tools need to **integrate multi-step tasks** to avoid agents stringing together multiple single tools (error-prone) and directly support complex decision scenarios.

### Positive example: integrated tool (solving customer duplicate charges)

```python
# Initialize MCP server
mcp_server = MCPBaseServer()

# 1. Register "solve customer duplicate charge" integrated tool (integrating 3-step logic)
def resolve_customer_charge_issue(customer_id: str, order_id: Optional[str] = None) -> Dict:
    """Core logic: 1. Check logs → 2. Analyze impact → 3. Generate solution"""
    # Step 1: Query duplicate charge logs for this customer
    logs = [{"log_id": "log_123", "time": "2025-09-10", "amount": 99.9, "status": "duplicate"}]
    # Step 2: Analyze if other customers are affected
    affected_customers = ["8765", "5432"] if "duplicate" in logs[0]["status"] else []
    # Step 3: Generate solution
    solution = f"Customer {customer_id} duplicate charge {logs[0]['amount']} yuan, affected customers {affected_customers}, suggest refund first and send apology email"
    return {
        "customer_id": customer_id,
        "duplicate_logs": logs,
        "affected_customers": affected_customers,
        "solution": solution
    }

# Register tool to MCP
mcp_server.register_tool(
    tool_id="payment.resolve_duplicate_charge",  # Namespace: service(payment)+function(resolve_duplicate_charge)
    tool_name="Resolve customer duplicate charge issue",
    tool_description="Function: One-click query customer duplicate charge logs, analyze affected customers, generate refund solutions;"
                    "Parameters: customer_id (required, customer ID, example: 9182), order_id (optional, order ID, example: ORD-202509);"
                    "Scenario: Call when customer reports 'single payment deducted multiple times'",
    parameters=[
        {"name": "customer_id", "type": "str", "required": True, "example": "9182"},
        {"name": "order_id", "type": "str", "required": False, "example": "ORD-202509"}
    ],
    execute_func=resolve_customer_charge_issue
)

# Agent call example (1 call to complete complex task)
call_result = mcp_server.call_tool("payment.resolve_duplicate_charge", {"customer_id": "9182"})
print(call_result["data"]["solution"])  # Output: Customer 9182 duplicate charge 99.9 yuan...
```

### Negative example: split tools (agent needs to string them together)

```python
# 1. Register "query charge logs" tool
def get_payment_logs(customer_id: str) -> List[Dict]:
    return [{"log_id": "log_123", "time": "2025-09-10", "amount": 99.9, "status": "duplicate"}]

mcp_server.register_tool(
    tool_id="payment.get_logs",
    tool_name="Query customer charge logs",
    tool_description="Query customer charge logs",  # Vague description
    parameters=[{"name": "customer_id", "type": "str", "required": True, "example": "9182"}],
    execute_func=get_payment_logs
)

# 2. Register "analyze affected customers" tool
def analyze_impact(log_id: str) -> List[str]:
    return ["8765", "5432"]  # Need to call get_payment_logs first to get log_id

mcp_server.register_tool(
    tool_id="payment.analyze_impact",
    tool_name="Analyze affected customers",
    parameters=[{"name": "log_id", "type": "str", "required": True, "example": "log_123"}],
    execute_func=analyze_impact
)

# 3. Register "generate solution" tool
def generate_solution(customer_id: str, affected: List[str]) -> str:
    return f"Customer {customer_id} needs refund, affected customers {affected}"

mcp_server.register_tool(
    tool_id="payment.generate_solution",
    tool_name="Generate refund solution",
    parameters=[{"name": "customer_id", "type": "str", "required": True}, {"name": "affected", "type": "list", "required": True}],
    execute_func=generate_solution
)

# Agent needs 3 calls (easy to miss steps/parameter errors)
log_result = mcp_server.call_tool("payment.get_logs", {"customer_id": "9182"})
impact_result = mcp_server.call_tool("payment.analyze_impact", {"log_id": log_result["data"][0]["log_id"]})
solution_result = mcp_server.call_tool("payment.generate_solution", {"customer_id": "9182", "affected": impact_result["data"]})
```

## Consensus 2: Tool functions need "high cohesion" and focus on high-value workflows

### Core logic

Tools need to integrate "all process-dependent information for agents to complete a type of task" to avoid agents calling multiple times to splice fragmented data.

### Positive example: highly cohesive "get customer panoramic information" tool

```python
def get_customer_context(customer_id: str, response_format: ResponseFormat = ResponseFormat.CONCISE) -> Dict:
    """Integrate customer basic information, recent transactions, service records"""
    base_info = {"name": "Sarah Chen", "email": "sarah@acme.com", "member_level": "VIP"}
    recent_trans = [{"order_id": "ORD-202508", "amount": 199.9, "date": "2025-08-25"}]
    service_notes = ["2025-08-30: Feedback on delayed logistics, compensation coupon issued"]
   
    if response_format == ResponseFormat.DETAILED:
        # Detailed mode: contains user_id and other technical fields, supports subsequent calls (such as sending emails)
        return {
            "customer_id": customer_id,
            "user_id": "usr_789",  # Technical ID
            "base_info": base_info,
            "recent_transactions": recent_trans,
            "service_notes": service_notes
        }
    # Concise mode: only return decision-critical information
    return {
        "customer_name": base_info["name"],
        "member_level": base_info["member_level"],
        "last_transaction": recent_trans[0] if recent_trans else None,
        "latest_service_issue": service_notes[0] if service_notes else None
    }

mcp_server.register_tool(
    tool_id="customer.get_context",
    tool_name="Get customer panoramic information",
    tool_description="Function: Get customer basic information, recent transactions, service records at once, support concise/detailed two response modes;"
                    "Parameters: customer_id (required, example: 9182), response_format (optional, default concise, enumeration values: concise/detailed);"
                    "Scenario: Call when customer cancels request, retention plan formulation",
    parameters=[
        {"name": "customer_id", "type": "str", "required": True, "example": "9182"},
        {"name": "response_format", "type": "enum", "required": False, "enum_values": [e.value for e in ResponseFormat], "default": "concise"}
    ],
    execute_func=get_customer_context
)

# Agent 1 call to get all required information (no need to splice multiple times)
context = mcp_server.call_tool("customer.get_context", {"customer_id": "9182", "response_format": "concise"})
print(context["data"]["latest_service_issue"])  # Output: 2025-08-30: Feedback on delayed logistics...
```

### Negative example: split "single information" tools

```python
# 1. Register "get customer basic information" tool
def get_customer_basic(customer_id: str) -> Dict:
    return {"name": "Sarah Chen", "email": "sarah@acme.com"}

mcp_server.register_tool(
    tool_id="customer.get_basic",
    tool_name="Get customer basic information",
    parameters=[{"name": "customer_id", "type": "str", "required": True, "example": "9182"}],
    execute_func=get_customer_basic
)

# 2. Register "get customer transactions" tool
def list_customer_trans(customer_id: str) -> List[Dict]:
    return [{"order_id": "ORD-202508", "amount": 199.9, "date": "2025-08-25"}]

mcp_server.register_tool(
    tool_id="customer.list_trans",
    tool_name="Get customer transaction records",
    parameters=[{"name": "customer_id", "type": "str", "required": True, "example": "9182"}],
    execute_func=list_customer_trans
)

# 3. Register "get service records" tool
def get_service_notes(customer_id: str) -> List[str]:
    return ["2025-08-30: Feedback on delayed logistics, compensation coupon issued"]

mcp_server.register_tool(
    tool_id="customer.get_notes",
    tool_name="Get customer service records",
    parameters=[{"name": "customer_id", "type": "str", "required": True, "example": "9182"}],
    execute_func=get_service_notes
)

# Agent needs 3 calls (waste tokens and easy to miss certain information)
basic = mcp_server.call_tool("customer.get_basic", {"customer_id": "9182"})
trans = mcp_server.call_tool("customer.list_trans", {"customer_id": "9182"})
notes = mcp_server.call_tool("customer.get_notes", {"customer_id": "9182"})
```

## Consensus 3: Namespace needs to "clearly distinguish resources/services" to eliminate tool ambiguity

### Core logic

Tool ID needs to be named according to "**service domain + resource type + operation**" (such as `asana.projects.search`) to help agents quickly locate among hundreds of tools.

### Positive example: standard namespace tools

```python
# 1. Asana service: project-related tools
def asana_search_projects(project_name: str, page: int = 1, page_size: int = 10) -> List[Dict]:
    return [{"project_id": "asana_prj_1", "name": "Acme Product Development", "status": "In Progress"}]

mcp_server.register_tool(
    tool_id="asana.projects.search",  # Namespace: service(asana)+resource(projects)+operation(search)
    tool_name="Search Asana projects",
    tool_description="Function: Search projects in Asana by project name, support pagination;"
                    "Parameters: project_name (required, example: Acme Product Development), page (optional, default 1), page_size (optional, default 10)",
    parameters=[
        {"name": "project_name", "type": "str", "required": True, "example": "Acme Product Development"},
        {"name": "page", "type": "int", "required": False, "default": 1},
        {"name": "page_size", "type": "int", "required": False, "default": 10}
    ],
    execute_func=asana_search_projects
)

# 2. Asana service: user-related tools
def asana_get_user_tasks(user_id: str) -> List[Dict]:
    return [{"task_id": "asana_task_1", "title": "Complete requirements document", "due_date": "2025-09-15"}]

mcp_server.register_tool(
    tool_id="asana.users.get_tasks",  # Namespace: service(asana)+resource(users)+operation(get_tasks)
    tool_name="Get Asana user tasks",
    parameters=[{"name": "user_id", "type": "str", "required": True, "example": "asana_usr_123"}],
    execute_func=asana_get_user_tasks
)

# Agent can quickly identify the purpose through tool ID (such as "asana.projects.search" clearly indicates Asana project search)
prj_result = mcp_server.call_tool("asana.projects.search", {"project_name": "Acme Product Development"})
```

### Negative example: ambiguous named tools

```python
# 1. Ambiguous tool ID: no distinction of resource type
def search_asana(keyword: str) -> List[Dict]:
    return [{"id": "asana_1", "name": "Acme Product Development", "type": "project"}]  # Cannot determine if it's project/task/user

mcp_server.register_tool(
    tool_id="asana.search",  # Only contains service(asana), no resource type
    tool_name="Search Asana content",
    tool_description="Search content in Asana",  # Vague description
    parameters=[{"name": "keyword", "type": "str", "required": True, "example": "Acme"}],
    execute_func=search_asana
)

# 2. Chaotic tool ID: no service domain
def get_tasks(user_id: str) -> List[Dict]:
    return [{"task_id": "task_1", "title": "Complete requirements document"}]  # Cannot determine if it's Asana/Jira/other tool's tasks

mcp_server.register_tool(
    tool_id="get_tasks",  # No service domain, no resource type
    tool_name="Get tasks",
    parameters=[{"name": "user_id", "type": "str", "required": True, "example": "123"}],
    execute_func=get_tasks
)

# Agent is easy to confuse when calling (such as "asana.search" cannot determine if it's searching projects or tasks, "get_tasks" cannot determine which platform's tasks)
confused_result = mcp_server.call_tool("asana.search", {"keyword": "Acme"})
```

## Consensus 4: Tool responses need "high signal, low redundancy" to reduce agent screening costs

### Core logic

Responses only return information "required for agent's downstream decisions" (such as `name`/`status`), eliminating meaningless technical identifiers (such as `uuid`/`mime_type`).

### Positive example: high signal response tools

```python
def get_file_info(file_id: str, response_format: ResponseFormat = ResponseFormat.CONCISE) -> Dict:
    # Underlying data (contains technical fields, but only returns key information)
    raw_data = {
        "file_id": file_id,
        "uuid": "f-uuid-123e4567-e89b-12d3-a456-426614174000",  # Redundant technical field
        "name": "Acme Project Plan.pdf",
        "file_type": "PDF",
        "size_mb": 2.5,
        "last_edited": "2025-09-01",
        "server_path": "/data/files/123",  # Redundant technical field
        "thumbnail_url_256px": "https://cdn/acme-256.jpg"  # Redundant technical field
    }

    if response_format == ResponseFormat.CONCISE:
        # Only return key information for agent decision-making (such as determining whether to download/share)
        return {
            "file_name": raw_data["name"],
            "file_type": raw_data["file_type"],
            "size_mb": raw_data["size_mb"],
            "last_edited": raw_data["last_edited"]
        }
    # Detailed mode only supplements necessary technical IDs (such as file_id for subsequent download tool calls)
    return {
        "file_id": raw_data["file_id"],
        "file_name": raw_data["name"],
        "file_type": raw_data["file_type"],
        "size_mb": raw_data["size_mb"],
        "last_edited": raw_data["last_edited"]
    }

mcp_server.register_tool(
    tool_id="file.get_info",
    tool_name="Get file information",
    tool_description="Function: Return key file information such as name, type, size, etc., eliminating redundant technical fields;"
                    "Parameters: file_id (required, example: file_456), response_format (optional, default concise)",
    parameters=[
        {"name": "file_id", "type": "str", "required": True, "example": "file_456"},
        {"name": "response_format", "type": "enum", "required": False, "default": "concise", "enum_values": [e.value for e in ResponseFormat]}
    ],
    execute_func=get_file_info
)

# Response example (no redundant fields, agent can use directly)
file_info = mcp_server.call_tool("file.get_info", {"file_id": "file_456"})
print(file_info["data"])
# Output: {"file_name": "Acme Project Plan.pdf", "file_type": "PDF", "size_mb": 2.5, "last_edited": "2025-09-01"}
```

### Negative example: redundant response tools

```python
def get_file_raw_data(file_id: str) -> Dict:
    # Return all underlying data, containing a lot of useless fields for agents
    return {
        "file_id": file_id,
        "uuid": "f-uuid-123e4567-e89b-12d3-a456-426614174000",
        "name": "Acme Project Plan.pdf",
        "mime_type": "application/pdf",  # Redundant (agent only needs "PDF")
        "size_bytes": 2621440,  # Redundant (agent needs to manually convert to MB)
        "server_path": "/data/files/123",
        "thumbnail_url_256px": "https://cdn/acme-256.jpg",
        "thumbnail_url_512px": "https://cdn/acme-512.jpg",
        "upload_server_ip": "10.0.0.1"  # Completely irrelevant
    }

mcp_server.register_tool(
    tool_id="file.get_raw",
    tool_name="Get file raw data",
    parameters=[{"name": "file_id", "type": "str", "required": True, "example": "file_456"}],
    execute_func=get_file_raw_data
)

# Response example (redundant fields account for more than 50%, agent needs to manually filter)
raw_data = mcp_server.call_tool("file.get_raw", {"file_id": "file_456"})
print(raw_data["data"]["mime_type"])  # Agent needs to map to "PDF" by itself
print(raw_data["data"]["size_bytes"] / 1024 / 1024)  # Agent needs to manually convert to MB
```

## Consensus 5: Token efficiency needs "active control" to avoid context overflow

### Core logic

Control response scale through **pagination, filtering, truncation**, when exceeding token limits, prompt agents to "narrow query scope" to avoid exhausting agent context windows.

### Positive example: token efficiency optimization tools

```python
def search_logs(
    keyword: str,
    time_range: str,  # Format: "YYYY-MM-DD to YYYY-MM-DD"
    page: int = 1,
    page_size: int = 50,  # Default pagination: avoid returning too much data at once
    max_tokens: int = 2000  # Maximum token limit (about 500 logs)
) -> Dict:
    # Simulate massive logs (1000 entries)
    all_logs = [{"log_id": f"log_{i}", "content": f"[{time_range}] {keyword} log content{i}"} for i in range(1000)]
   
    # 1. Pagination calculation
    start_idx = (page - 1) * page_size
    end_idx = start_idx + page_size
    paginated_logs = all_logs[start_idx:end_idx]
   
    # 2. Token estimation (simplified: about 40 tokens per log)
    total_tokens = len(paginated_logs) * 40
    truncated = False
    if total_tokens > max_tokens:
        # Truncate and prompt
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
        "suggestion": "If logs are incomplete, you can narrow time_range (example: 2025-09-10 to 2025-09-10) or reduce page_size" if truncated else None
    }

mcp_server.register_tool(
    tool_id="logs.search",
    tool_name="Search system logs",
    tool_description="Function: Search logs by keywords and time range, support pagination and token truncation to avoid context overflow;"
                    "Parameters: keyword (required, example: payment failed), time_range (required, example: 2025-09-01 to 2025-09-10),"
                    "page (optional, default 1), page_size (optional, default 50), max_tokens (optional, default 2000)",
    parameters=[
        {"name": "keyword", "type": "str", "required": True, "example": "payment failed"},
        {"name": "time_range", "type": "str", "required": True, "example": "2025-09-01 to 2025-09-10"},
        {"name": "page", "type": "int", "required": False, "default": 1},
        {"name": "page_size", "type": "int", "required": False, "default": 50},
        {"name": "max_tokens", "type": "int", "required": False, "default": 2000}
    ],
    execute_func=search_logs
)

# Call example: time_range too wide causing token limit exceeded, tool actively truncates and prompts
logs_result = mcp_server.call_tool(
    "logs.search",
    {"keyword": "payment failed", "time_range": "2025-01-01 to 2025-09-10", "page_size": 100}
)
print(logs_result["data"]["truncated"])  # Output: True
print(logs_result["data"]["suggestion"])  # Output: If logs are incomplete, you can narrow time_range...
```

### Negative example: no token control tools

```python
def get_all_logs(keyword: str) -> Dict:
    # No pagination/truncation: return all matching logs (possibly 1000+ entries, exceeding agent context)
    all_logs = [{"log_id": f"log_{i}", "content": f"Log content{i}: {keyword}"} for i in range(1000)]
    return {
        "logs": all_logs,
        "count": len(all_logs)
    }

mcp_server.register_tool(
    tool_id="logs.get_all",
    tool_name="Get all matching logs",
    tool_description="Get all logs containing keywords",  # No risk scale prompt
    parameters=[{"name": "keyword", "type": "str", "required": True, "example": "payment failed"}],
    execute_func=get_all_logs
)

# Call example: return 1000 logs (about 40000 tokens), directly exceeding Claude Code's default 25000 token limit
all_logs_result = mcp_server.call_tool("logs.get_all", {"keyword": "payment failed"})
print(len(all_logs_result["data"]["logs"]))  # Output: 1000 (agent context overflow, task failed)
```

## Consensus 6: Tool descriptions need to be "clear and unambiguous" to reduce agent misuse rate

### Core logic

Tool descriptions need to clearly specify "**function + parameter meaning + examples + scenarios**", especially parameters need to be marked with type, requiredness, and examples to avoid agents passing wrong values.

### Positive example: clearly described tools

```python
def send_customer_email(
    customer_id: str,
    email_type: str,  # Enumeration: refund_confirm/retention_offer
    template_id: Optional[str] = None
) -> Dict:
    """Send customer email: automatically match templates based on type (when no template_id)"""
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
    tool_name="Send customer email",
    tool_description="Function: Send emails to specified customers, support refund confirmation and retention offer types, customizable templates;"
                    "Parameter description:"
                    "- customer_id (required, unique customer ID, example: 9182);"
                    "- email_type (required, email type, only supports: refund_confirm (refund confirmation), retention_offer (retention offer));"
                    "- template_id (optional, template ID, example: tpl_custom_01, use default template if not filled);"
                    "Call scenario: Send confirmation email after customer refund, send retention offer when customer cancels;"
                    "Error prompt: If email_type is not in enumeration, need to re-pass refund_confirm or retention_offer",
    parameters=[
        {"name": "customer_id", "type": "str", "required": True, "example": "9182"},
        {"name": "email_type", "type": "str", "required": True, "example": "retention_offer", "enum_values": ["refund_confirm", "retention_offer"]},
        {"name": "template_id", "type": "str", "required": False, "example": "tpl_custom_01"}
    ],
    execute_func=send_customer_email,
    error_suggestion="Please check if email_type is 'refund_confirm' or 'retention_offer', or add template_id"
)

# Agent can call correctly according to description (unambiguous)
email_result = mcp_server.call_tool(
    "email.send_customer",
    {"customer_id": "9182", "email_type": "retention_offer"}
)
print(email_result["data"]["status"])  # Output: sent
```

### Negative example: vague description tools

```python
def send_email(customer_id: str, type: str, template: Optional[str] = None) -> Dict:
    """Send email"""  # Vague description
    if type not in ["refund", "retention"]:
        raise ValueError("Type error")
    return {"status": "sent"}

mcp_server.register_tool(
    tool_id="email.send",
    tool_name="Send email",
    tool_description="Send email to customer, parameters include type and template",  # No parameter meaning/examples
    parameters=[
        {"name": "customer_id", "type": "str", "required": True},  # No example
        {"name": "type", "type": "str", "required": True},  # No enumeration value description
        {"name": "template", "type": "str", "required": False}  # No example
    ],
    execute_func=send_email
)

# Agent is easy to misuse (ambiguous parameters)
# Error 1: type passed "retention" (actually needs "retention")
error1 = mcp_server.call_tool("email.send", {"customer_id": "9182", "type": "retention"})
print(error1["error"])  # Output: Type error (no correction suggestion)

# Error 2: don't know template parameter format (pass "retention template" instead of ID)
error2 = mcp_server.call_tool("email.send", {"customer_id": "9182", "type": "retention", "template": "retention template"})
```

## Consensus 9: Error handling needs to "provide actionable suggestions" to guide agents to correct

### Core logic

Error responses need to clearly specify "**error reason + specific correction steps**" rather than just returning error codes or vague prompts to help agents quickly adjust call parameters.

### Positive example: actionable error prompt tools

```python
def create_meeting(
    attendee_emails: List[str],
    start_time: str,  # Format: "YYYY-MM-DD HH:MM" (24-hour format)
    duration_min: int  # Range: 15-180
) -> Dict:
    # Parameter verification: clear error reason
    if not all("@" in email for email in attendee_emails):
        raise ValueError("Some attendee email formats are invalid")
    if len(start_time.split(" ")) != 2 or len(start_time.split(":")[1]) != 2:
        raise ValueError("Start time format error")
    if duration_min < 15 or duration_min > 180:
        raise ValueError("Meeting duration must be between 15-180 minutes")
   
    return {"meeting_id": "meet_789", "status": "created", "start_time": start_time}

mcp_server.register_tool(
    tool_id="calendar.create_meeting",
    tool_name="Create meeting",
    tool_description="Function: Create calendar meeting, specify attendees, time, duration;"
                    "Parameters: attendee_emails (required, email list, example: ['jane@acme.com']),"
                    "start_time (required, format: YYYY-MM-DD HH:MM, example: 2025-09-15 14:30),"
                    "duration_min (required, 15-180 minutes, example: 60)",
    parameters=[
        {"name": "attendee_emails", "type": "list", "required": True, "example": ["jane@acme.com", "john@acme.com"]},
        {"name": "start_time", "type": "str", "required": True, "example": "2025-09-15 14:30"},
        {"name": "duration_min", "type": "int", "required": True, "example": 60}
    ],
    execute_func=create_meeting,
    error_suggestion=lambda e: {
        "邮箱错误": "Please check if attendee emails contain '@' (example: ['jane@acme.com'])",
        "时间格式错误": "Please pass in the format 'YYYY-MM-DD HH:MM' (example: 2025-09-15 14:30)",
        "时长错误": "Meeting duration needs to be set to 15-180 minutes (example: 30/60)"
    }.get(str(e).split("：")[0], "Please check if parameter format conforms to example")
)

# Error call example: time format error, return actionable suggestion
error_result = mcp_server.call_tool(
    "calendar.create_meeting",
    {"attendee_emails": ["jane@acme.com"], "start_time": "2025-09-15 2点半", "duration_min": 60}
)
print(error_result["error"])  # Output: Start time format error
print(error_result["suggestion"])  # Output: Please pass in the format 'YYYY-MM-DD HH:MM' (example: 2025-09-15 14:30)
```

### Negative example: vague error prompt tools

```python
def create_meeting_v2(attendees: List[str], time: str, duration: int) -> Dict:
    if not all("@" in a for a in attendees):
        raise ValueError("E001")  # Error code with no meaning
    if len(time.split(" ")) != 2:
        raise ValueError("E002")
    if duration < 15 or duration > 180:
        raise ValueError("E003")
    return {"meeting_id": "meet_789"}

mcp_server.register_tool(
    tool_id="calendar.create_meeting_v2",
    tool_name="Create meeting V2",
    parameters=[
        {"name": "attendees", "type": "list", "required": True},
        {"name": "time", "type": "str", "required": True},
        {"name": "duration", "type": "int", "required": True}
    ],
    execute_func=create_meeting_v2
)

# Error call example: only return error code, agent cannot correct
error_result_v2 = mcp_server.call_tool(
    "calendar.create_meeting_v2",
    {"attendees": ["jane.acme.com"], "time": "2025-09-15", "duration": 200}
)
print(error_result_v2["error"])  # Output: E001 (agent cannot understand meaning)
print(error_result_v2.get("suggestion"))  # Output: None (no correction suggestion)
```

## Summary: Core principles of MCP tool code design

1.  **Naming conventions**: Tool ID named according to "service + resource + operation" (such as `payment.resolve_duplicate_charge`);

2.  **Clear description**: Clearly specify "function + parameter meaning + examples + scenarios" to avoid ambiguity;

3.  **Concise response**: Only return key information for agent decision-making, eliminating redundant technical fields;

4.  **Efficiency control**: Limit token consumption through pagination and truncation, prompt to narrow query scope when exceeding limits;

5.  **Error friendly**: Error responses need to contain "reason + actionable suggestions" rather than error codes;

6.  **High cohesion**: Integrate multi-step logic to avoid agents calling multiple times to splice data.

The above code examples are all based on Anthropic's MCP practice logic and can be directly adapted to LLM agents such as Claude to reduce call error rates and improve task completion efficiency.

## Reference documents

*   [Writing effective tools for AI agents—using AI agents \ Anthropic](https://www.anthropic.com/engineering/writing-tools-for-agents)

*   [豆包对话](https://www.doubao.com/thread/w1590f5b1360ca747)

> (Note: Some content in the document may be AI-generated)