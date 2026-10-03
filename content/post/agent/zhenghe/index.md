---
description: ""
title: "第一阶段整合代码"
draft: false
date: "2026-08-15T08:43:51+08:00"
slug: "zhenghe"
categories:
 - 
tags:
 - 
image: ""
---

```
from openai import OpenAI
from dotenv import load_dotenv, find_dotenv
import json,os,ast
import urllib.request
from pathlib import Path
import openmeteo_requests
load_dotenv(override=True, dotenv_path=find_dotenv())

class LLMClient():
    def chat(self,message,tools=None):
        raise NotImplemented("子类必须实现 chat()")

class RealLLM(LLMClient): #client初始化

    def __init__(self,provide="deepseek"):
        cfg={
            "deepseek":{"base_url":"https://api.deepseek.com","key":"DEEPSEEK_API_KEY","model":"deepseek-v4-flash"},
            "kimi": {"base_url": "https://api.moonshot.cn/v1", "key": "MOONSHOT_API_KEY", "model": "kimi-k2.6"},
        }[provide]
        self.client = OpenAI(api_key=os.getenv(cfg["key"]), base_url=cfg["base_url"])
        print(f"实际使用的 provider={provide}, key 已加载={'是' if os.getenv(cfg['key']) else '否'}")
       # print(repr(os.getenv(cfg["key"])))
        self.model=cfg["model"]
        print(provide)
    def chat(self,messages,tools=None):
        return self.client.chat.completions.create(model=self.model,messages=messages,tools=tools)

class StreamLLM(LLMClient):
    def __init__(self,provide="deepseek"):
        cfg={
            "deepseek":{"base_url":"https://api.deepseek.com","key":"DEEPSEEK_API_KEY","model":"deepseek-v4-flash"},
            "kimi": {"base_url": "https://api.moonshot.cn/v1", "key": "MOONSHOT_API_KEY", "model": "kimi-k2.6"},
        }[provide]
        self.client = OpenAI(api_key=os.getenv(cfg["key"]), base_url=cfg["base_url"])
        self.model=cfg["model"]
        print(provide)
    def chat(self, messages,**kwargs):
        return self.client.chat.completions.create(model=self.model,messages=messages,stream=True, reasoning_effort=kwargs.get("reasoning_effort","low"),extra_body={"thinking": {"type": "enabled"}},max_tokens=kwargs.get("max_tokens",500))

def route_model(question): #模型硬路由

    """从用户输入里找模型名,找不到用默认"""
    for name in ["kimi", "deepseek", "qwen", "gpt"]:
        if name in question.lower():
            return name
    return "deepseek"   # 默认



def run_tool(name,arg):#调用工具
    try:
        return TOOLS_FUNC[name](**arg)
    except Exception as e:
        return(f"参数出错了:{e}")

def fc_loop(question,llm,max_rounds=3):#fc主函数
   messages=[{"role":"user","content":question}]
   tool_records = []   
   for _ in range(max_rounds):
        rep=llm.chat(messages,tools=TOOLS_SCHEMA)
        msg=rep.choices[0].message
        if not msg.tool_calls:
            return {
                "answer": msg.content,     # 最终答案
                "tool_records": tool_records,   # [] = 没调工具(模型猜的!)
            }
            #return msg.content
        messages.append(msg)
        for tc in msg.tool_calls:
            try:
                arg=json.loads(tc.function.arguments)
            except Exception as e:
                messages.append({"role": "tool", "tool_call_id": tc.id, "content": f"参数解析失败:{e}"})
                continue
            result=run_tool(tc.function.name,arg)
            print(arg)
            tool_records.append({"tool": tc.function.name, "args": arg, "result": result})
            messages.append({"role":"tool","tool_call_id":tc.id,"content":result})
          #  print(msg)
   return {"answer": "达到最大轮数", "tool_records": tool_records}

def get_weather(latitude:float,longitude:float)->str:#查询天气函数实现
    openmeteo = openmeteo_requests.Client()
    url = (f"https://api.open-meteo.com/v1/forecast?latitude={latitude}"
           f"&longitude={longitude}"
           f"&current=temperature_2m,relative_humidity_2m,wind_speed_10m,wind_direction_10m"
           f"&daily=weather_code,temperature_2m_max,temperature_2m_min,sunrise,sunset,precipitation_probability_max"
           f"&timezone=Asia%2FShanghai")
    data = json.loads(urllib.request.urlopen(url).read())
    WEATHER_CODE_DESC = {
    0: "晴朗的天空",
    1: "主要晴朗",
    2: "局部多云",
    3: "阴天",
    45: "雾气",
    48: "霜雾沉积",
    51: "毛毛雨：轻度",
    53: "毛毛雨：中度",
    55: "毛毛雨：密集",
    56: "冻毛毛雨：轻微",
    57: "冻毛毛雨：强度高",
    61: "降雨：轻度",
    63: "降雨：中度",
    65: "降雨：强雨",
    66: "冻雨：轻度",
    67: "冻雨：强烈",
    71: "降雪量：轻度",
    73: "降雪量：中度",
    75: "降雪量：重度",
    77: "雪粒",
    80: "阵雨：轻度",
    81: "阵雨：中度",
    82: "阵雨：猛烈",
    85: "雪阵阵：轻微",
    86: "雪阵阵：猛烈",
    95: "雷暴：轻度或中度",
    96: "雷暴伴轻微冰雹",
    99: "雷暴伴强烈冰雹",
    }
    cur = data["current"]
    today = {k: v[0] for k, v in data["daily"].items()}
    return json.dumps({
        "天气": WEATHER_CODE_DESC.get(today["weather_code"], f"未知天气码{today['weather_code']}"),
        "当前温度_C": cur["temperature_2m"],
        "湿度_%": cur["relative_humidity_2m"],
        "今日最高_C": today["temperature_2m_max"],
        "今日最低_C": today["temperature_2m_min"],
        "日出": today["sunrise"][11:16],
        "日落": today["sunset"][11:16],
        "降雨概率_%": today["precipitation_probability_max"],
    }, ensure_ascii=False)

def calculator(expr: str) -> str:#四则运算函数实现
    return str(ast.literal_eval(expr))

TOOLS_FUNC = {"get_weather": get_weather, "calculator": calculator}   

with open("D:\\Desktop\\桌面文件\\AgentLearn\\agent\\tools.json", encoding="utf-8") as f:
   TOOLS_SCHEMA = json.load(f) 

def get_response(messages,max_tokens=500):#流式回复
    streamllm=StreamLLM()
    response = streamllm.chat(messages,max_tokens=max_tokens)
    reasoning_content, content = "", ""
    for chunk in response:
        delta = chunk.choices[0].delta
        if delta.reasoning_content:
            reasoning_content += delta.reasoning_content
        if delta.content:
            content += delta.content
    if content:
        messages.append({"role": "assistant", "content": content})
    else:
        content = reasoning_content
    return content

def ask(quextion,system_prompt="你是擅长多种角度对比方案的资深决策专家",max_tokens=500):
    msgs=[
        {"role": "system", "content": system_prompt},
        {"role": "user", "content": quextion}
    ]
    response = get_response(msgs, max_tokens)
    return response

def tot_solve(question,n=3):#tot实现
    schemes=ask(
        f"请针对以下问题给出{n}种不同解决思路，编号列出：\n{question}\n",
        system_prompt="你是擅长发散思维的规划专家",
        max_tokens=600,
    )
    scores = ask(
        f"请对以下 {n} 个方案根据实行难度，风险评估逐一打分(1-10分),并各用一句话说明理由:\n{schemes}",
        system_prompt="你是严格的评审专家,打分要客观",
        max_tokens=500,
    )
    best = ask(
        f"综合以下评分,选出最优方案,并给出具体实施步骤:\n{scores}",
        system_prompt="你是决策专家,直接给最终选择",
        max_tokens=600,
    )
    return schemes, scores, best

def cot_prompt(question,system_prompt="你是一个逻辑缜密，思维严谨的推理助手"):#cot实现函数
    u=f"请一步步思考，再给出答案。\n问题:{question}"
    return [{"role": "system", "content": system_prompt}, {"role": "user", "content": u}]

DEFAULT_EXAMPLES = [
    {"question": "我明天下午3点约了张医生复诊",
     "answer": "时间:明天下午3点;人物:张医生;事项:复诊"},
    {"question": "周五晚上和李总在望江楼吃饭",
     "answer": "时间:周五晚上;人物:李总;事项:吃饭"},
]

def few_shot_prompt(question,examples=DEFAULT_EXAMPLES,system_prompt="你是擅长信息抽取的助手") :#fewshort实现
    msgs=[{"role":"system","content":system_prompt}]
    for ex in examples:
        msgs.append({"role":"user","content":ex["question"]})
        msgs.append({"role":"assistant","content":ex["answer"]})
    msgs.append({"role":"user","content":question})
    return msgs

def react_prompt(question):#react启动
    return fc_loop(question, RealLLM(route_model(question))) 

BUILDERS = {"cot": cot_prompt, "few_shot": few_shot_prompt, "react": react_prompt}

def auto_select(question):#思维选择器
    content=f"判断下面用户的问题适合哪种回答，只输出四个英文字符串中的一个输出格式：cot_prompt或few_shot_prompt或react_prompt或tot_solve)\n规则:注重结果中间思考过程的任务，逻辑推理，数学解题过程，公式推导输出,日常对话cot_prompt,格式固定模板化输出的，格式化输出文本，文本提取特定词汇格式化输出的输出,Few_shot_prompt,需要依赖其他工具的输出react_prompt,开放性有多种选择回答的问题，思维发散问题，思维风暴，多种选择多种路径实现，多选择对比找最优解决方案输出tot_solve\n问题:{question}"
    msgs = get_response([{"role": "user", "content": content}], max_tokens=50)
    print(f"[调试] 原始: {msgs!r}")
    text = msgs.lower()
    for mode in ["tot", "react", "few_shot", "cot"]:   
        if mode in text:
            return mode
    print("[警告] 未识别到模式名,兜底 cot")
    return "cot"

def run_mode(mode,question,**kw):
    if mode =="tot":
        schemes, scores, best = tot_solve(question, n=kw.get("n", 3))
        return f"【方案】\n{schemes}\n\n【评审】\n{scores}\n\n【决策】\n{best}"
    if mode =="react":
        out = react_prompt(question) 
        if out["tool_records"]:
        # 展示调用记录
            print("工具调用记录:", out["tool_records"])
        else:
         # ⚠️ 模型没调工具,可能是在猜(幻觉风险)
            print("警告:模型未调用任何工具,回答可能为模型推断")
        return out["answer"]
         #return fc_loop(question, RealLLM("deepseek")) 
    if mode not in BUILDERS:
          raise ValueError(f"未知模式:{mode},可选 {list(BUILDERS) + ['tot']}")
    msgs = BUILDERS[mode](question,**kw)
    return get_response(msgs,**kw)    

if __name__=="__main__":
    while True:
        user_input = input("User: ")
        if user_input.lower() in ["exit", "quit"]:
            break
        mode = auto_select(user_input) 
        print(f"[使用模式] {mode}")
        result = run_mode(mode, user_input)     # ② 按模式调度(统一返回字符串)
        print("Agent导师:", result)     
```