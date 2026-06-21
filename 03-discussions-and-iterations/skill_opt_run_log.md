# Agent 技能描述自动编译与优化日志 (Skill Docstring Backpropagation)

本日志记录了通过自动反馈优化智能体工具定义（Tool Description）的完整实跑轨迹。
在大模型 Agent 系统中，工具的描述语（Docstring）是 AI 理解如何使用它的唯一依据。如果描述不精确，AI 会调用错误；本演练展示了系统如何通过捕获调用失败并自动改写工具描述语，直至 AI 100% 正确调用工具。

## 🛠️ 迭代 1
* **当前工具描述 (Tool Description)**: `Queries the database for user information.`
* **智能体传参 (Agent Invocation)**: `SELECT * FROM users WHERE registration_year = 2026 AND total_spent > 100`
* **工具执行反馈 (Sandbox Output)**: 
🔴 **失败**: `Query Error: Table not found. Did you mean 'user_spend_records'?`

* **优化器分析 (Critique)**: The agent passed a raw SQL query string, but the tool expects a structured query or specific parameters. The error indicates the table name is incorrect; the correct table is 'user_spend_records'. The tool description must explicitly state that the agent should not write raw SQL, and instead specify the required parameters (e.g., registration_year, total_spent) and the correct table name.

---

## 🛠️ 迭代 2
* **当前工具描述 (Tool Description)**: `Queries the database for user information. Do NOT pass raw SQL. Instead, provide the following parameters: registration_year (integer) and total_spent (float). The tool will query the 'user_spend_records' table. Example: registration_year=2026, total_spent=100.`
* **智能体传参 (Agent Invocation)**: `registration_year=2026, total_spent=100`
* **工具执行反馈 (Sandbox Output)**: 
🔴 **失败**: `SyntaxError: Invalid query 'registration_year=2026, total_spent=100'. Query MUST be a valid SQL statement starting with SELECT.`

* **优化器分析 (Critique)**: The agent passed the parameters as a string in the format 'registration_year=2026, total_spent=100', which is not a valid SQL query. The tool description incorrectly instructed the agent to 'provide the following parameters' without specifying that they must be passed as separate arguments to the function, not as a single string. The error indicates the tool expects a SQL SELECT statement. The description must be rewritten to clarify that the tool expects a SQL query string, and the agent should construct a proper SELECT statement using the given parameters.

---

## 🛠️ 迭代 3
* **当前工具描述 (Tool Description)**: `Queries the database for user information. You must pass a single string argument that is a valid SQL SELECT statement. Do NOT pass raw parameters like 'registration_year=2026, total_spent=100'. Instead, construct a query such as: "SELECT * FROM user_spend_records WHERE registration_year = 2026 AND total_spent = 100". The table is 'user_spend_records'. Ensure the query starts with SELECT and is properly formatted.`
* **智能体传参 (Agent Invocation)**: `SELECT * FROM user_spend_records WHERE registration_year = 2026 AND total_spent > 100`
* **工具执行反馈 (Sandbox Output)**: 
🔴 **失败**: `Query Error: Column not found. Table 'user_spend_records' schema: [id, username, signup_year, spend_amount].`

* **优化器分析 (Critique)**: The agent passed a valid SQL query but used column names 'registration_year' and 'total_spent' that do not exist in the table. The actual columns are 'signup_year' and 'spend_amount'. The tool description must explicitly list the correct column names and warn against guessing column names.

---

## 🛠️ 迭代 4
* **当前工具描述 (Tool Description)**: `Queries the database for user information. You must pass a single string argument that is a valid SQL SELECT statement. The table is 'user_spend_records' with columns: id (integer), username (text), signup_year (integer), spend_amount (float). Do NOT use column names like 'registration_year' or 'total_spent' — they do not exist. Construct a query such as: "SELECT * FROM user_spend_records WHERE signup_year = 2026 AND spend_amount > 100". Ensure the query starts with SELECT and is properly formatted.`
* **智能体传参 (Agent Invocation)**: `SELECT * FROM user_spend_records WHERE signup_year = 2026 AND spend_amount > 100`
* **工具执行反馈 (Sandbox Output)**: 
🟢 **成功**
```json
{
  "status": "success",
  "data": [
    {
      "username": "Alice",
      "spend_amount": 120
    },
    {
      "username": "Bob",
      "spend_amount": 250
    }
  ]
}
```

---
