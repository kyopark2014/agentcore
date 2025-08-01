# Memory의 사용

## AgentCore의 Memory

[How AgentCore Memory Works](https://github.com/awslabs/amazon-bedrock-agentcore-samples/tree/main/01-tutorials/04-AgentCore-memory#how-agentcore-memory-works)와 같이 AgentCore에는 short/long term memory를 가지고 있습니다. short term memory는 sliding windows형태로 근래 k개의 메시지를 저장하였다가 prompt로 활용합니다. long term memory는 아래와 같이 vector store 형태로 운용됩니다. 또한 long term memory는 Semantic, Summary, User Preference, Custom과 같은 형태의 strategy를 가지고 있고 namespace를 이용해 구분합니다. 

<img width="600" alt="image" src="https://github.com/user-attachments/assets/4920d36a-884d-4668-8eb4-f9665fe300d0" />


Memory와 관련된 용어는 아래와 같습니다.

- actorId: Memory의 event와 관련된 사용자 또는 agent/user 형태로 된 계정으로서 "/agent-support-123/customer-456"의 형태를 가집니다.
- sessionId: 대화 세션과 같은 이벤트들의 그룹
  
MemoryClient의 list_memories을 활용하여 메모리 항목을 확인합니다.

```python
global memory_id
from bedrock_agentcore.memory import MemoryClient
memory_client = MemoryClient(region_name="us-west-2")

memories = memory_client.list_memories()
logger.info(f"memories: {memories}")
for memory in memories:
    logger.info(f"Memory Arn: {memory.get('arn')}")
    memory_id = memory.get('id')
    logger.info(f"Memory ID: {memory_id}")
    logger.info("--------------------------------------------------------------------")
```

이때의 조회결과는 아래와 같습니다. 여기서 memory의 id가 대화 내용을 저장하거나 읽어올 때에 활용됩니다.

```java
[
   {
      "arn":"arn:aws:bedrock-agentcore:us-west-2:262976740991:memory/LangGraph-VMvQCK89aW",
      "id":"LangGraph-VMvQCK89aW",
      "status":"ACTIVE",
      "createdAt":datetime.datetime(2025,7,20,23,23,51,557000,"tzinfo=tzlocal())",
      "updatedAt":datetime.datetime(2025,7,20,23,23,51,557000,"tzinfo=tzlocal())",
      "memoryId":"LangGraph-VMvQCK89aW"
   }
]
```

메모리가 없다면 아래와 같이 생성합니다.

```python
if len(memories) == 0:
    result = client.create_memory(
        name=user_id,
        description="LangGraph Memory",
        event_expiry_days=7, # 7 - 365 days
        # memory_execution_role_arn=memory_execution_role_arn
    )
    logger.info(f"result: {result}")
```

메모리 생성시의 결과입니다.

```java
{
   "arn":"arn:aws:bedrock-agentcore:us-west-2:262976740991:memory/agentcore_strands-HuPfmNFtZM",
   "id":"LangGraph-HuPfmNFtZM",
   "name":"LangGraph",
   "description":"LangGraph Memory",
   "eventExpiryDuration":7,
   "status":"ACTIVE",
   "createdAt":datetime.datetime(2025,7,21,12,56,28,854000,"tzinfo=tzlocal())",
   "updatedAt":datetime.datetime(2025,7,21,12,56,28,854000,"tzinfo=tzlocal())",
   "strategies":[
      
   ],
   "memoryId":"agentcore_strands-HuPfmNFtZM",
   "memoryStrategies":[
      
   ]
}
```

대화 내용은 아래와 같이 저장할 수 있습니다.

```python
memory_result = memory_client.create_event(
    memory_id=memory.get("id"),
    actor_id=user_id, 
    session_id=user_id, 
    messages=[
        (query, "USER"),
        (result, "ASSISTANT")
    ]
)
logger.info(f"result of save conversation to memory: {memory_result}")
```

저장시 아래와 같을 결과로 리턴합니다.

```java
{
   "memoryId":"LangGraph-VMvQCK89aW",
   "actorId":"LangGraph",
   "sessionId":"LangGraph",
   "eventId":"0000001753022422000#5f585a3e",
   "eventTimestamp":datetime.datetime(2025,7,20,23,40,22,"tzinfo=tzlocal())",
   "branch":{
      "name":"main"
   }
}
```

저장된 대화 내용을 아래와 같이 조회할 수 있습니다.

```python
conversations = memory_client.list_events(
    memory_id=memory.get("id"),
    actor_id=user_id,
    session_id=user_id,
    max_results=5,
)
logger.info(f"conversations: {conversations}")
```

이때의 조회 결과는 아래와 같습니다. 이를 LangGraph에서 활용하기 위해서는 적절한 memory 포맷으로 변경이 필요합니다.

```java
[
   {
      "memoryId":"LangGraph-VMvQCK89aW",
      "actorId":"LangGraph",
      "sessionId":"LangGraph",
      "eventId":"0000001753022526000#c8b6ecfa",
      "eventTimestamp":datetime.datetime(2025,7,20,23,42,6,"tzinfo=tzlocal())",
      "payload":[
         {
            "conversational":{
               "content":{
                  "text":"너의 이름은?"
               },
               "role":"USER"
            }
         },
         {
            "conversational":{
               "content":{
                  "text":"안녕하세요! 제 이름은 서연입니다 😊 \n저는 여러분의 질문에 친근하고 상세하게 답변해드리는 AI 도우미예요. \n무엇을 도와드릴까요?"
               },
               "role":"ASSISTANT"
            }
         }
      ],
      "branch":{
         "name":"main"
      }
   },
   {
      "memoryId":"LangGraph-VMvQCK89aW",
      "actorId":"LangGraph",
      "sessionId":"LangGraph",
      "eventId":"0000001753022422000#5f585a3e",
      "eventTimestamp":datetime.datetime(2025,7,20,23,40,22,"tzinfo=tzlocal())",
      "payload":[
         {
            "conversational":{
               "content":{
                  "text":"안녕"
               },
               "role":"USER"
            }
         },
         {
            "conversational":{
               "content":{
                  "text":"안녕하세요! 저는 서연이에요. 현재 시각은 2025년 7월 20일 23시 40분이네요. 무엇을 도와드릴까요? 날씨 정보, 주식 정보, 도서 검색 등 다양한 정보를 알려드릴 수 있어요. 또한 파일 시스템 관리나 AWS 서비스 작업도 도와드릴 수 있답니다. 어떤 것이 궁금하신가요?"
               },
               "role":"ASSISTANT"
            }
         }
      ],
      "branch":{
         "name":"main"
      }
   }
]
```

## 메모리 ID의 처리

Memory의 모든 동작은 Memory ID가 반드시 필요한데, [list_memories](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/bedrock-agentcore-control/client/list_memories.html)로 조회하면 description등은 보여주지 않고 id만 제공하여 어떤 대화가 mapping되어 있는지 알 수 없습니다. 따라서, Memory ID 생성 시점에 config 형태로 저장해서, 대화와 mapping이 필요합니다. 대화는 sessionId와 actorId를 필수로 가지고 있어야 하므로, config에서 이 정보를 Memory ID와 mapping 할 수 있어야 합니다.

## Reference

[Amazon Bedrock AgentCore Memory](https://github.com/awslabs/amazon-bedrock-agentcore-samples/tree/main/01-tutorials/04-AgentCore-memory)

[tool - agent_core_memory.py](https://github.com/strands-agents/tools/blob/main/src/strands_tools/agent_core_memory.py)

[boto3 - create_memory](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/bedrock-agentcore-control/client/create_memory.html)

[boto3 - get_memory_record](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/bedrock-agentcore/client/get_memory_record.html)

[boto3 - retrieve_memory_records](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/bedrock-agentcore/client/retrieve_memory_records.html)

[Bedrock AgentCore Memory SDK](https://github.com/aws/bedrock-agentcore-sdk-python/tree/main/src/bedrock_agentcore/memory)

[LangGraph with AgentCore Memory Tool (Short term memory)](https://github.com/awslabs/amazon-bedrock-agentcore-samples/blob/main/01-tutorials/04-AgentCore-memory/01-short-term-memory/01-single-agent/with-langgraph-agent/personal-fitness-coach.ipynb)
