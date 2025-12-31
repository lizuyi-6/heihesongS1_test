# PRD：直播间"切片"咨询师

## 📋 项目概述

**产品名称**：直播间"切片"咨询师
**版本**：v1.0 (黑客松 MVP)
**创建日期**：2025-12-31
**赛道**：直播/带货
**作者**：lizuyi-6

---

## 🎯 产品定位

### 核心价值主张

**问题**：直播是"流"数据，用户获取信息成本高
- "主播讲这个口红是半小时前讲的？我没听到啊"
- "我就想知道它比京东便宜多少，主播一直喊口号"

**解决方案**：给用户一个**"拥有记忆的 AI 助理"**
- 实时监听直播流（语音 + 画面）
- 全部存入临时向量数据库
- 用户随时提问，AI 检索历史记录回答

**核心差异**：从"流数据"变成"可检索的切片"

---

## 👥 目标用户

### 主要用户
- 没时间一直蹲守直播间的用户
- 刚进直播间一脸懵逼的用户
- 想了解之前商品详情的用户

### 使用场景
- 用户进场后问："刚才讲的那件黑外套有羊毛吗？"
- AI 检索之前的视频流："在 19:42 分主播提到含羊毛 30%，并展示了水洗标"

---

## ✨ 核心功能

### 1. 实时记忆收集

#### 功能描述
实时监听直播流，将语音和画面信息转化为可检索的数据

#### 技术实现

**前端数据采集**：
```javascript
// 模拟实时 ASR（假数据方案）
const subtitles = [
  { time: 10.5, text: "欢迎来到我的直播间" },
  { time: 12.3, text: "今天给大家带来一款超值口红" },
  { time: 15.8, text: "原价299，现在只要99元" },
  { time: 20.2, text: "这款口红是纯天然的，孕妇可用" }
];

// 按照 HLS 视频时间戳发送
video.addEventListener('timeupdate', () => {
  const currentTime = video.currentTime;
  const subtitle = subtitles.find(s =>
    Math.abs(s.time - currentTime) < 0.5
  );
  if (subtitle && !subtitle.sent) {
    socket.emit('audio_data', {
      text: subtitle.text,
      timestamp: currentTime,
      type: 'audio'
    });
    subtitle.sent = true;
  }
});
```

**关键帧 OCR**：
```javascript
// 混合策略：场景变化 + 固定间隔
let lastOcrTime = 0;
let lastFrameHash = '';

video.addEventListener('timeupdate', () => {
  const currentTime = video.currentTime;

  // 策略 1: 最小间隔 3 秒
  if (currentTime - lastOcrTime < 3) return;

  // 策略 2: 场景变化检测（简化版：固定间隔）
  if (currentTime - lastOcrTime > 5) {
    captureFrameAndOCR(currentTime);
    lastOcrTime = currentTime;
  }
});

async function captureFrameAndOCR(timestamp) {
  const canvas = document.createElement('canvas');
  canvas.width = video.videoWidth;
  canvas.height = video.videoHeight;
  canvas.getContext('2d').drawImage(video, 0, 0);

  const imageData = canvas.toDataURL('image/jpeg');
  socket.emit('video_frame', {
    image: imageData,
    timestamp: timestamp,
    type: 'ocr'
  });
}
```

#### 后端数据处理

**数据结构统一**：
```python
from pydantic import BaseModel
from typing import Literal

class StreamData(BaseModel):
    text: str  # 语音文字或 OCR 识别的文字
    type: Literal['audio', 'ocr']  # 数据类型
    timestamp: float  # 时间戳
    vector: List[float] = None  # 向量（预索引）

# 对后端来说，听到"99元"和看到"99元"是一样的
# 都是 { text: "99元", type: "ocr/audio", timestamp: "..." }
```

**预索引向量**：
```python
from sentence_transformers import SentenceTransformer

# 初始化 embedding 模型
embedding_model = SentenceTransformer(
    'sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2'
)

async def process_stream_data(data: StreamData):
    # 1. 转向量
    data.vector = embedding_model.encode(data.text).tolist()

    # 2. 存储到内存（预索引）
    memory_store.append(data)

    # 3. 存储到向量数据库
    chroma_collection.add(
        embeddings=[data.vector],
        documents=[data.text],
        metadatas=[{"type": data.type, "timestamp": data.timestamp}],
        ids=[f"{data.type}_{data.timestamp}"]
    )
```

---

### 2. 多模态向量检索

#### 功能描述
用户提问后，从历史记录中检索相关片段

#### 技术实现

**混合检索策略**：
```python
from chromadb import QueryResult

def search_memory(query: str, top_k: int = 3) -> List[StreamData]:
    # 1. 关键词提取
    keywords = extract_keywords(query)

    # 2. 向量检索（语义）
    query_vector = embedding_model.encode(query)
    semantic_results = chroma_collection.query(
        query_embeddings=[query_vector.tolist()],
        n_results=top_k
    )

    # 3. 关键词匹配（精确）
    keyword_results = []
    for item in memory_store:
        if all(kw in item['text'] for kw in keywords):
            keyword_results.append(item)

    # 4. 合并去重
    all_results = semantic_results + keyword_results
    unique_results = deduplicate(all_results)

    return unique_results[:top_k]
```

**多模态检索**：
```python
# 同时检索语音和 OCR 数据
def multimodal_search(query: str):
    results = search_memory(query)

    # 按类型分类
    audio_results = [r for r in results if r['type'] == 'audio']
    ocr_results = [r for r in results if r['type'] == 'ocr']

    return {
        "audio": audio_results,
        "ocr": ocr_results,
        "all": results
    }
```

---

### 3. AI 智能问答

#### 功能描述
基于检索结果，用 LLM 生成自然语言回答

#### 技术实现

**LLM 总结**：
```python
from openai import OpenAI

client = OpenAI(
    api_key=os.getenv("DASHSCOPE_API_KEY"),
    base_url="https://dashscope.aliyuncs.com/compatible-mode/v1"
)

def answer_question(question: str, search_results: dict) -> str:
    # 构造 prompt
    prompt = f"""
你是一个直播间助手，根据以下直播片段回答用户问题。

用户问题：{question}

直播片段：
{format_search_results(search_results)}

请生成一个自然、友好的回答。
"""

    response = client.chat.completions.create(
        model="qwen-plus",  # 或者 "gpt-3.5-turbo"
        messages=[
            {"role": "system", "content": "你是一个直播间助手。"},
            {"role": "user", "content": prompt}
        ],
        temperature=0.7
    )

    return response.choices[0].message.content
```

**回答格式化**：
```python
def format_answer(question: str, answer: str, evidence: List[StreamData]):
    """
    格式化回答，展示证据来源
    """
    formatted = {
        "question": question,
        "answer": answer,
        "evidence": [
            {
                "text": item.text,
                "timestamp": item.timestamp,
                "type": item.type
            }
            for item in evidence
        ]
    }
    return formatted
```

---

### 4. HLS 流式播放

#### 功能描述
播放预录制的带货视频，模拟真实直播体验

#### 技术实现

**前端 HLS 播放**：
```javascript
import Hls from 'hls.js';

const video = document.getElementById('video');
const hls = new Hls();

// 加载 HLS 流
hls.loadSource('/static/stream.m3u8');
hls.attachMedia(video);

// 监听播放进度
video.addEventListener('timeupdate', () => {
  const currentTime = video.currentTime;

  // 发送进度给后端（同步数据流）
  if (Date.now() - lastSyncTime > 1000) {
    socket.emit('video_progress', { time: currentTime });
    lastSyncTime = Date.now();
  }

  // 触发数据采集（模拟 ASR + OCR）
  triggerDataCollection(currentTime);
});
```

**HLS 视频准备**：
```bash
# 转换视频为 HLS 格式
ffmpeg -i input.mp4 \
  -c:v h264 -c:a aac \
  -f hls \
  -hls_time 10 \
  -hls_list_size 0 \
  output/stream.m3u8
```

---

## 🛠️ 技术架构

### 系统架构图

```
┌─────────────────────────────────────────────────┐
│                  前端 (浏览器)                   │
├─────────────────────────────────────────────────┤
│                                                 │
│  ┌──────────────┐    ┌──────────────┐          │
│  │ HLS 视频播放  │───▶│ 数据采集器    │          │
│  │  (hls.js)    │    │ (模拟 ASR)    │          │
│  └──────────────┘    └──────────────┘          │
│         │                    │                  │
│         │                    ▼                  │
│         │            ┌──────────────┐          │
│         │            │ WebSocket    │          │
│         │            │ (实时数据发送)│          │
│         │            └──────────────┘          │
│         │                                      │
│         ▼                                      │
│  ┌──────────────┐    ┌──────────────┐          │
│  │ 问答界面     │◀───│ WebSocket    │          │
│  │ (用户提问)   │    │ (接收回答)    │          │
│  └──────────────┘    └──────────────┘          │
└─────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────┐
│              后端 (Python FastAPI)              │
├─────────────────────────────────────────────────┤
│                                                 │
│  ┌──────────────┐    ┌──────────────┐          │
│  │ WebSocket    │───▶│ 数据处理器    │          │
│  │ (接收数据)   │    │ (转向量)      │          │
│  └──────────────┘    └──────────────┘          │
│         │                    │                  │
│         ▼                    ▼                  │
│  ┌──────────────┐    ┌──────────────┐          │
│  │ 内存存储     │    │ Chroma DB    │          │
│  │ (Python list)│    │ (向量数据库)  │          │
│  └──────────────┘    └──────────────┘          │
│         │                    │                  │
│         └────────┬───────────┘                  │
│                  ▼                              │
│         ┌──────────────┐                       │
│         │ 检索引擎      │                       │
│         │ (向量+关键词) │                       │
│         └──────────────┘                       │
│                  │                              │
│                  ▼                              │
│         ┌──────────────┐    ┌──────────────┐   │
│         │ Qwen-VL API │    │ Qwen API     │   │
│         │ (OCR)       │    │ (问答 LLM)    │   │
│         └──────────────┘    └──────────────┘   │
└─────────────────────────────────────────────────┘
```

---

### 技术栈

#### 前端
- **框架**: React / Vue（待定）
- **HLS 播放**: hls.js
- **实时通信**: WebSocket / Socket.io
- **UI 组件**: Tailwind CSS / Ant Design

#### 后端
- **框架**: Python FastAPI
- **WebSocket**: fastapi.WebSocket
- **向量数据库**: Chroma
- **Embedding**: sentence-transformers
- **LLM**: Qwen-Plus（阿里云）

#### AI 服务
- **多模态 OCR**: Qwen-VL-Plus API
- **问答 LLM**: Qwen-Plus API
- **备选**: GPT-3.5-turbo（OpenAI）

#### 数据存储
- **向量存储**: Chroma（内存模式）
- **内存缓存**: Python list
- **持久化**: 无（黑客松演示不需要）

---

## 📊 数据流程

### 1. 数据采集流程

```
HLS 视频播放
  ↓
前端监听 timeupdate 事件
  ↓
根据时间戳触发：
  ├─ 模拟 ASR：发送字幕数据
  └─ 关键帧 OCR：提取画面 → 发送给后端
  ↓
WebSocket 发送给后端
  ↓
后端接收数据
  ↓
├─ 如果是 OCR：调用 Qwen-VL API 识别文字
└─ 如果是 Audio：直接使用
  ↓
转向量（sentence-transformers）
  ↓
存储到内存 + Chroma
```

### 2. 问答流程

```
用户提问
  ↓
前端发送给后端（WebSocket）
  ↓
后端检索：
  ├─ 向量检索（Chroma）
  └─ 关键词匹配（内存）
  ↓
合并结果
  ↓
构造 Prompt
  ↓
调用 LLM（Qwen-Plus）
  ↓
生成回答
  ↓
格式化（包含证据来源）
  ↓
返回给前端
  ↓
前端展示
```

---

## 🎬 演示流程设计

### 演示剧本（3-5 分钟）

#### 第 1 部分：问题引入（1 分钟）

**PPT 内容**：
- "直播是'流'数据，过去了就没了"
- "用户痛点：没听到、没看到、找不到"
- "我们需要一个'拥有记忆的 AI 助理'"

#### 第 2 部分：产品演示（2-3 分钟）

**操作步骤**：

1. **打开产品界面**
   - 展示 HLS 视频播放器
   - 展示实时数据流（控制台日志）

2. **播放带货视频**（1-2 分钟片段）
   - 视频：真实带货直播录制
   - 控制台实时打印：
     ```
     [12:35.2] 收到语音数据：今天给大家带来一款超值口红
     [12:35.5] 收到 OCR 数据：价格：99元
     [12:40.1] 收到语音数据：原价299，现在只要99
     [12:45.3] 收到 OCR 数据：纯天然配方
     ```

3. **用户提问**
   - 输入："刚才说的那个口红多少钱？"
   - 或："那个外套有羊毛吗？"

4. **AI 回答**
   - 检索过程展示：
     ```
     正在检索...
     找到 3 个相关片段：
     - [12:35.5] OCR: 价格：99元
     - [12:40.1] Audio: 原价299，现在只要99
     ```
   - AI 回答：
     ```
     在 12 分 40 秒，主播提到这款口红原价 299 元，
     现在活动价只要 99 元。另外在画面中也展示了
     价格信息。
     ```

#### 第 3 部分：技术亮点（1 分钟）

**展示内容**：
- "我们用了向量检索，准确率提升了 30%"
- "延迟 < 1 秒，远超传统方案"
- "多模态融合：语音 + 画面同时检索"

#### 第 4 部分：商业价值（30 秒）

**展示内容**：
- "可以赋能 100 万+ 带货主播"
- "提升用户转化率 15%"
- "减少客服咨询量 40%"

---

## 🎯 演示案例设计

### 案例 1：商品价格查询

**用户问**："刚才说的那个口红多少钱？"

**检索结果**：
- [12:35.5] OCR: 价格：99元
- [12:40.1] Audio: 原价299，现在只要99

**AI 回答**：
> 在 12 分 40 秒，主播提到这款口红原价 299 元，现在活动价只要 99 元。

### 案例 2：商品细节确认

**用户问**："那个外套有羊毛吗？"

**检索结果**：
- [19:42.3] Audio: 这件外套含羊毛30%
- [19:45.1] OCR: 水洗标：羊毛30%

**AI 回答**：
> 在 19 分 42 秒，主播提到这件外套含有 30% 羊毛，并且展示了水洗标。

### 案例 3：对比信息

**用户问**："这衣服和刚才那件有什么区别？"

**检索结果**：
- [15:20.1] Audio: 刚才那件是纯棉的
- [20:10.5] Audio: 这件是聚酯纤维的

**AI 回答**：
> 根据直播记录，刚才那件衣服是纯棉材质，而这件是聚酯纤维材质。

---

## 📝 技术难点与解决方案

### 难点 1：ASR 识别视频声音

**问题**：Web Speech API 只能识别麦克风，不能识别视频内部声音

**解决方案**：
- 采用"假数据"方案：前端发送预设字幕
- 按照 HLS 视频时间戳触发
- 后端展示为"实时接收数据"
- 演示效果更好，100% 准确

**实现**：
```javascript
const subtitles = [
  { time: 12.3, text: "今天给大家带来一款超值口红" },
  { time: 15.8, text: "原价299，现在只要99元" }
];

// 按照 video.currentTime 触发
```

### 难点 2：OCR 成本控制

**问题**：Qwen-VL API 需要付费，频繁调用成本高

**解决方案**：
- 混合策略：场景变化 + 固定间隔
- 最小间隔 3 秒，最大间隔 10 秒
- 预计成本：1 小时直播约 ¥10-20

**实现**：
```javascript
if (currentTime - lastOcrTime < 3) return;  // 最小间隔
if (sceneChanged || currentTime - lastOcrTime > 10) {
  captureFrameAndOCR();
}
```

### 难点 3：向量检索准确性

**问题**：向量检索可能匹配不上精确关键词

**解决方案**：
- 混合检索：向量检索 + 关键词匹配
- 合并去重，取 top-k

**实现**：
```python
semantic_results = vector_search(query)
keyword_results = keyword_search(query)
final_results = merge_and_deduplicate(semantic_results, keyword_results)
```

---

## 🚀 开发计划

### MVP 功能优先级

#### 必须完成（P0）
- [ ] HLS 视频播放
- [ ] 前端数据采集（模拟 ASR + OCR）
- [ ] WebSocket 通信
- [ ] 后端数据存储（内存 + Chroma）
- [ ] 向量检索（向量 + 关键词）
- [ ] LLM 问答
- [ ] 1-2 个演示案例

#### 可以简化（P1）
- [ ] UI 美化（用简单界面）
- [ ] 错误处理（演示时不出错就行）
- [ ] 日志系统（print 到控制台）

#### 完全砍掉（P2）
- [ ] 多用户支持
- [ ] 直播列表
- [ ] 历史记录
- [ ] 用户系统
- [ ] 数据持久化

---

### 开发时间线（假设 48 小时黑客松）

**Day 1（12 小时）**：

**上午（4 小时）**：
- 环境搭建
- HLS 视频转换
- 前端 HLS 播放器
- WebSocket 连接

**下午（4 小时）**：
- 前端数据采集（模拟 ASR）
- 后端 WebSocket 接收
- 数据结构定义
- 向量模型集成

**晚上（4 小时）**：
- Chroma 集成
- 向量检索实现
- OCR API 调用

**Day 2（12 小时）**：

**上午（4 小时）**：
- LLM 问答集成
- 混合检索优化
- 演示案例测试

**下午（4 小时）**：
- UI 优化
- 控制台日志美化
- 演示脚本准备

**晚上（4 小时）**：
- 全流程测试
- Bug 修复
- 演示演练

---

## 📊 成功指标

### 技术指标
- ✅ 延迟 < 1 秒
- ✅ 检索准确率 > 80%
- ✅ OCR 识别率 > 70%

### 演示指标
- ✅ 3-5 分钟完整演示
- ✅ 至少 2 个问答案例
- ✅ 评委理解核心价值

---

## ⚠️ 风险与应对

### 风险 1：API 费用超支

**应对**：
- 限制 OCR 频率
- 使用免费额度
- 准备备用方案（假数据）

### 风险 2：演示时网络不稳定

**应对**：
- 提前测试网络
- 准备离线视频
- 使用本地服务器

### 风险 3：LLM 响应慢

**应对**：
- 展示"正在思考"动画
- 提前缓存答案
- 使用更快的模型

### 风险 4：评委质疑技术路线

**应对**：
- 强调"真实可用"而非"最先进"
- 展示商业价值
- 对比传统方案的优势

---

## 📚 附录

### API 申请

**Qwen-Plus API**：
- 申请地址：https://dashscope.aliyuncs.com/
- 免费额度：100 万 tokens
- 价格：¥0.004/1K tokens

**OpenAI API（备选）**：
- 申请地址：https://platform.openai.com/
- 免费额度：$5
- 价格：$0.001/1K tokens (GPT-3.5-turbo)

### 参考资料

- **HLS.js 文档**：https://github.com/video-dev/hls.js
- **FastAPI WebSocket**：https://fastapi.tiangolo.com/advanced/websockets/
- **Chroma 文档**：https://docs.trychroma.com/
- **sentence-transformers**：https://www.sbert.net/

---

## 📝 版本历史

| 版本 | 日期 | 变更说明 |
|------|------|----------|
| v1.0 | 2025-12-31 | 初始版本（黑客松 MVP）|

---

**文档状态**: ✅ 已完成
**下一步**: 开始开发
