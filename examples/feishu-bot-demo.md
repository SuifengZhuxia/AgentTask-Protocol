# 飞书机器人极简实现示例

基于 AgentTask Protocol 实现的飞书群AI代理机器人，零成本搭建自动化任务接单/交付/验收系统

## 核心流程
1. 发布者私聊机器人发送任务需求 → 机器人自动按协议格式化 → 发布到群内
2. 执行者私聊机器人发送「接单 + 任务ID」→ 机器人更新任务状态
3. 执行者私聊机器人提交交付内容 → 机器人按协议结构化存储
4. 机器人自动调用AI接口完成验收 → 群内播报验收结果（不泄露交付内容）
5. 发布者私下完成点对点结算

## 前置准备
1. 飞书开放平台创建自建应用，开启机器人能力
2. 开通飞书应用「接收单聊消息」「接收群聊消息」权限
3. 申请豆包/通义千问等大模型API接口
4. 飞书群内添加机器人

## 核心消息格式示例

### 1. 任务发布群消息格式
```text
【任务发布 T_20240520_001】
📌 标题：整理 3 个周口本地免费公园信息
📋 要求：需包含公园地址、开放时间、核心特色，结构化输出
💰 报酬：20 元
⏰ 截止：2024-05-20 18:00
📤 交付方式：私聊机器人提交，不公开
🔄 状态：待接单
基于 AgentTask Protocol 发布
```

### 2. 私聊接单响应格式
```text
接单 T_20240520_001

✅ 已成功接单【T_20240520_001】
请在截止时间前，私聊发送：交付 T_20240520_001 <交付内容>
任务要求：需包含公园地址、开放时间、核心特色，结构化输出
```

### 3. 验收结果群播报格式
```text
【任务验收 T_20240520_001】
👤 交付人：xxx
✅ 验收结果：通过
💬 评语：内容完整，符合要求
🔄 任务状态：已完成
基于 AgentTask Protocol 自动验收
```

## 极简Python代码核心逻辑
```python
# 核心依赖：飞书开放平台SDK + 大模型API
from larksuiteoapi import Config, DOMAIN_FEISHU
from larksuiteoapi.event import handle_event
from larksuiteoapi.service.im.v1.event import MessageReceiveEvent
import json
import requests

# 1. 飞书应用配置
app_config = Config.new_internal_app_config(
    app_id="你的飞书APP_ID",
    app_secret="你的飞书APP_SECRET",
    domain=DOMAIN_FEISHU
)

# 2. 大模型API配置
AI_API_KEY = "你的大模型API_KEY"
AI_API_URL = "https://api.doubao.com/chat/completions"

# 3. 任务ID生成函数
def generate_task_id():
    from datetime import datetime
    return f"T_{datetime.now().strftime('%Y%m%d')}_{datetime.now().strftime('%H%M%S')}"

# 4. 消息接收处理函数
def handle_message(ctx, event: MessageReceiveEvent):
    message = event.event.message
    content = json.loads(message.content)["text"]
    chat_type = message.chat_type
    sender_id = event.event.sender.sender_id.user_id

    # 私聊处理：发布任务/接单/交付
    if chat_type == "p2p":
        # 发布任务
        if "任务：" in content:
            # 调用AI格式化任务为协议标准格式
            task_json = format_task_by_ai(content)
            # 发送到群内
            send_task_to_group(task_json)
            # 回复发布者
            reply_message(sender_id, f"✅ 任务已发布，ID：{task_json['taskId']}")
        
        # 接单处理
        elif "接单" in content:
            task_id = content.split(" ")[1]
            update_task_status(task_id, "assigned")
            reply_message(sender_id, f"✅ 已成功接单【{task_id}】")
        
        # 交付处理
        elif "交付" in content:
            task_id = content.split(" ")[1]
            submit_content = content.split(" ", 2)[2]
            # 结构化交付内容
            submit_json = format_submit(task_id, sender_id, submit_content)
            # 自动验收
            review_result = auto_review(task_id, submit_json)
            # 群内播报结果
            send_review_to_group(task_id, review_result)
            # 回复双方
            reply_message(sender_id, f"✅ 交付已提交，验收结果：{review_result['result']}")

# 5. AI格式化任务函数
def format_task_by_ai(user_content):
    prompt = f"""
    请将以下用户需求，按AgentTask Protocol v1.0.0的Task格式，输出标准JSON，必填字段不能缺失：
    用户需求：{user_content}
    只输出JSON，不要其他内容
    """
    # 调用大模型API
    response = requests.post(AI_API_URL, headers={"Authorization": f"Bearer {AI_API_KEY}"}, json={
        "model": "doubao-pro",
        "messages": [{"role": "user", "content": prompt}]
    })
    return json.loads(response.json()["choices"][0]["message"]["content"])

# 6. 自动验收函数
def auto_review(task_id, submit_json):
    # 获取任务要求
    task_requirement = get_task_requirement(task_id)
    prompt = f"""
    请按以下任务要求，验收交付内容，输出AgentTask Protocol标准的Review JSON格式：
    任务要求：{task_requirement}
    交付内容：{submit_json['content']}
    只输出JSON，不要其他内容
    """
    # 调用大模型API
    response = requests.post(AI_API_URL, headers={"Authorization": f"Bearer {AI_API_KEY}"}, json={
        "model": "doubao-pro",
        "messages": [{"role": "user", "content": prompt}]
    })
    return json.loads(response.json()["choices"][0]["message"]["content"])

# 主函数
if __name__ == "__main__":
    handle_event(app_config, MessageReceiveEvent, handle_message)
```
## 注意事项
- 本示例仅为核心逻辑演示，完整实现需补齐异常处理、数据存储、权限控制
- 交付内容仅在私聊处理，禁止在群内公开，保障私密性
- 所有任务与交付数据需按协议规范存储，保证可追溯
- 资金结算完全点对点，机器人不碰任何资金，规避合规风险