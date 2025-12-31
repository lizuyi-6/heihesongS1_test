# 开发任务分解书

**项目**：直播间"切片"咨询师
**类型**：黑客松 MVP（48 小时）
**目标**：完成可演示的完整产品
**交付日期**：______

---

## 📋 任务概览

| 模块 | 任务数 | 预计工时 | 优先级 | 负责人 |
|------|--------|----------|--------|--------|
| 前端开发 | 7 | 12h | P0 | ____ |
| 后端开发 | 8 | 14h | P0 | ____ |
| AI 集成 | 4 | 8h | P0 | ____ |
| 数据准备 | 3 | 4h | P0 | ____ |
| 测试调试 | 3 | 6h | P0 | ____ |
| 演示准备 | 2 | 4h | P1 | ____ |
| **总计** | **27** | **48h** | - | - |

---

## 🎯 Phase 1：环境搭建与数据准备（4h）

### Task 1.1：项目初始化（1h）

**负责人**：______
**优先级**：P0
**依赖**：无

**前端**：
```bash
# 创建前端项目
npm create vite@latest livestream-ai -- --template react
cd livestream-ai
npm install

# 安装依赖
npm install hls.js socket.io-client tailwindcss
npm install -D @types/node
```

**后端**：
```bash
# 创建后端项目
mkdir livestream-ai-backend
cd livestream-ai-backend
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

# 安装依赖
pip install fastapi uvicorn python-socketio
pip install sentence-transformers chromadb
pip install Pillow opencv-python
pip install openai  # Qwen API 兼容
```

**交付物**：
- [ ] 前端项目可运行（npm run dev）
- [ ] 后端项目可运行（需要一个测试接口）
- [ ] README.md 说明如何启动

---

### Task 1.2：HLS 视频准备（2h）

**负责人**：______
**优先级**：P0
**依赖**：无

**步骤**：

1. **获取带货视频**
   - 找一个真实带货直播录像（.mp4 格式）
   - 时长：3-5 分钟（演示片段）
   - 内容：包含主播展示商品、说明价格

2. **转换视频为 HLS**
```bash
# 安装 ffmpeg
# Windows: 下载 https://ffmpeg.org/download.html
# Mac: brew install ffmpeg

# 转换命令
ffmpeg -i input.mp4 \
  -c:v libx264 -c:a aac \
  -f hls \
  -hls_time 10 \
  -hls_list_size 0 \
  -start_number 0 \
  output/stream.m3u8
```

3. **准备字幕文件**
   - 手动听写或用 ASR 工具生成
   - 格式：
```json
[
  { "time": 10.5, "text": "欢迎来到我的直播间" },
  { "time": 12.3, "text": "今天给大家带来一款超值口红" },
  { "time": 15.8, "text": "原价299，现在只要99元" },
  { "time": 20.2, "text": "这款口红是纯天然的，孕妇可用" }
]
```

**交付物**：
- [ ] HLS 视频文件（stream.m3u8 + .ts 文件）
- [ ] 字幕 JSON 文件（subtitles.json）
- [ ] 视频时长记录

---

### Task 1.3：API Key 申请（1h）

**负责人**：______
**优先级**：P0
**依赖**：无

**步骤**：

1. **申请 Qwen-Plus API**
   - 访问：https://dashscope.aliyuncs.com/
   - 注册账号
   - 创建 API Key
   - 免费额度：100 万 tokens

2. **配置环境变量**
```bash
# 后端 .env 文件
DASHSCOPE_API_KEY=your_api_key_here
```

3. **测试 API**
```python
# test_api.py
from openai import OpenAI

client = OpenAI(
    api_key=os.getenv("DASHSCOPE_API_KEY"),
    base_url="https://dashscope.aliyuncs.com/compatible-mode/v1"
)

response = client.chat.completions.create(
    model="qwen-plus",
    messages=[{"role": "user", "content": "你好"}]
)
print(response.choices[0].message.content)
```

**交付物**：
- [ ] API Key 配置到 .env
- [ ] API 测试通过
- [ ] API 调用示例代码

---

## 🎨 Phase 2：前端开发（12h）

### Task 2.1：HLS 视频播放器（2h）

**负责人**：______
**优先级**：P0
**依赖**：Task 1.2

**需求**：
- 播放 HLS 视频
- 显示播放时间
- 暂停/播放控制

**实现**：
```jsx
// src/components/VideoPlayer.jsx
import { useEffect, useRef, useState } from 'react';
import Hls from 'hls.js';

export default function VideoPlayer() {
  const videoRef = useRef(null);
  const [currentTime, setCurrentTime] = useState(0);

  useEffect(() => {
    const video = videoRef.current;
    const hls = new Hls();

    hls.loadSource('/output/stream.m3u8');
    hls.attachMedia(video);

    return () => {
      hls.destroy();
    };
  }, []);

  const handleTimeUpdate = () => {
    setCurrentTime(videoRef.current.currentTime);
  };

  return (
    <div className="video-container">
      <video
        ref={videoRef}
        onTimeUpdate={handleTimeUpdate}
        controls
        width="800"
        height="450"
      />
      <div>播放时间: {currentTime.toFixed(1)}s</div>
    </div>
  );
}
```

**交付物**：
- [ ] 视频可以播放
- [ ] 显示当前播放时间
- [ ] 样式美观（用 Tailwind CSS）

---

### Task 2.2：WebSocket 客户端（2h）

**负责人**：______
**优先级**：P0
**依赖**：Task 3.1（后端 WebSocket）

**需求**：
- 连接后端 WebSocket
- 发送模拟 ASR 数据
- 发送视频帧（OCR）
- 接收 AI 回答

**实现**：
```jsx
// src/utils/websocket.js
import { io } from 'socket.io-client';

export const socket = io('http://localhost:8000', {
  transports: ['websocket']
});

export const sendAudioData = (text, timestamp) => {
  socket.emit('audio_data', {
    text,
    timestamp,
    type: 'audio'
  });
};

export const sendVideoFrame = (imageData, timestamp) => {
  socket.emit('video_frame', {
    image: imageData,
    timestamp,
    type: 'ocr'
  });
};

export const sendQuestion = (question) => {
  socket.emit('user_question', { question });
};

export const onAnswer = (callback) => {
  socket.on('ai_answer', callback);
};
```

**交付物**：
- [ ] WebSocket 连接成功
- [ ] 可以发送数据
- [ ] 可以接收回答
- [ ] 错误处理

---

### Task 2.3：数据采集器（3h）

**负责人**：______
**优先级**：P0
**依赖**：Task 2.1, Task 1.2

**需求**：
- 根据视频时间戳发送字幕数据
- 定时截图并发送（模拟 OCR）
- 控制台日志显示

**实现**：
```jsx
// src/components/DataCollector.jsx
import { useEffect, useRef } from 'react';
import { sendAudioData, sendVideoFrame } from '../utils/websocket';
import subtitles from '../data/subtitles.json';

export default function DataCollector({ videoRef }) {
  const lastSentIndex = useRef(-1);
  const lastOcrTime = useRef(0);

  useEffect(() => {
    const interval = setInterval(() => {
      if (!videoRef.current) return;

      const currentTime = videoRef.current.currentTime;

      // 1. 发送字幕数据（模拟 ASR）
      const subtitleIndex = subtitles.findIndex(
        (s, i) =>
          i > lastSentIndex.current &&
          Math.abs(s.time - currentTime) < 0.5
      );

      if (subtitleIndex !== -1) {
        const subtitle = subtitles[subtitleIndex];
        sendAudioData(subtitle.text, subtitle.time);
        console.log(`[${subtitle.time.toFixed(1)}] 发送语音数据:`, subtitle.text);
        lastSentIndex.current = subtitleIndex;
      }

      // 2. 发送视频帧（模拟 OCR）
      if (currentTime - lastOcrTime.current > 5) {
        captureFrame(currentTime);
        lastOcrTime.current = currentTime;
      }
    }, 100);

    return () => clearInterval(interval);
  }, [videoRef]);

  const captureFrame = (currentTime) => {
    const video = videoRef.current;
    const canvas = document.createElement('canvas');
    canvas.width = video.videoWidth;
    canvas.height = video.videoHeight;
    const ctx = canvas.getContext('2d');
    ctx.drawImage(video, 0, 0);

    const imageData = canvas.toDataURL('image/jpeg', 0.7);
    sendVideoFrame(imageData, currentTime);
    console.log(`[${currentTime.toFixed(1)}] 发送 OCR 帧`);
  };

  return null; // 不渲染任何 UI
}
```

**交付物**：
- [ ] 字幕数据按时间戳发送
- [ ] 视频帧每 5 秒发送一次
- [ ] 控制台显示发送日志
- [ ] 与视频播放同步

---

### Task 2.4：问答界面（2h）

**负责人**：______
**优先级**：P0
**依赖**：Task 2.2

**需求**：
- 输入框（用户提问）
- 历史记录展示
- AI 回答展示
- 证据来源展示

**实现**：
```jsx
// src/components/ChatPanel.jsx
import { useState, useEffect } from 'react';
import { sendQuestion, onAnswer } from '../utils/websocket';

export default function ChatPanel() {
  const [messages, setMessages] = useState([]);
  const [input, setInput] = useState('');

  useEffect(() => {
    onAnswer((data) => {
      setMessages((prev) => [...prev, { role: 'ai', ...data }]);
    });
  }, []);

  const handleSend = () => {
    if (!input.trim()) return;

    setMessages((prev) => [...prev, { role: 'user', content: input }]);
    sendQuestion(input);
    setInput('');
  };

  return (
    <div className="chat-panel">
      <div className="messages">
        {messages.map((msg, i) => (
          <div key={i} className={`message ${msg.role}`}>
            {msg.role === 'user' ? (
              <p>{msg.content}</p>
            ) : (
              <div>
                <p>{msg.answer}</p>
                {msg.evidence && (
                  <div className="evidence">
                    <h4>证据来源：</h4>
                    {msg.evidence.map((ev, j) => (
                      <div key={j}>
                        [{ev.timestamp.toFixed(1)}s] {ev.text}
                      </div>
                    ))}
                  </div>
                )}
              </div>
            )}
          </div>
        ))}
      </div>

      <div className="input-area">
        <input
          type="text"
          value={input}
          onChange={(e) => setInput(e.target.value)}
          onKeyPress={(e) => e.key === 'Enter' && handleSend()}
          placeholder="问我关于直播的问题..."
        />
        <button onClick={handleSend}>发送</button>
      </div>
    </div>
  );
}
```

**交付物**：
- [ ] 可以输入问题
- [ ] 显示 AI 回答
- [ ] 显示证据来源
- [ ] 美观的 UI

---

### Task 2.5：主界面整合（2h）

**负责人**：______
**优先级**：P0
**依赖**：Task 2.1, Task 2.3, Task 2.4

**需求**：
- 整合所有组件
- 响应式布局
- 演示模式优化

**实现**：
```jsx
// src/App.jsx
import VideoPlayer from './components/VideoPlayer';
import DataCollector from './components/DataCollector';
import ChatPanel from './components/ChatPanel';
import { useRef } from 'react';

function App() {
  const videoRef = useRef(null);

  return (
    <div className="app">
      <header>
        <h1>直播间"切片"咨询师</h1>
        <p>实时记忆 · 智能问答 · 多模态检索</p>
      </header>

      <main>
        <div className="video-section">
          <VideoPlayer ref={videoRef} />
          <DataCollector videoRef={videoRef} />
        </div>

        <div className="chat-section">
          <ChatPanel />
        </div>
      </main>

      <footer>
        <p>技术栈：HLS + FastAPI + Chroma + Qwen-Plus</p>
      </footer>
    </div>
  );
}

export default App;
```

**交付物**：
- [ ] 完整的界面
- [ ] 响应式布局
- [ ] 演示模式（全屏）

---

### Task 2.6：前端样式优化（1h）

**负责人**：______
**优先级**：P1
**依赖**：Task 2.5

**需求**：
- 使用 Tailwind CSS
- 统一配色方案
- 演示友好的大字体

**交付物**：
- [ ] 全局样式配置
- [ ] 组件样式
- [ ] 演示模式样式

---

## 🔧 Phase 3：后端开发（14h）

### Task 3.1：FastAPI + WebSocket（2h）

**负责人**：______
**优先级**：P0
**依赖**：Task 1.1

**需求**：
- FastAPI 基础框架
- WebSocket 连接管理
- CORS 配置

**实现**：
```python
# main.py
from fastapi import FastAPI, WebSocket, WebSocketDisconnect
from fastapi.middleware.cors import CORSMiddleware
from socketio import AsyncServer
import socketio

# FastAPI app
app = FastAPI(title="直播间切片咨询师 API")

# CORS
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Socket.IO
sio = AsyncServer(async_mode='asgi', cors_allowed_origins='*')
socket_app = socketio.ASGIApp(sio, app)

# 连接管理
connected_clients = []

@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    await websocket.accept()
    connected_clients.append(websocket)
    try:
        while True:
            data = await websocket.receive_json()
            await handle_websocket_message(websocket, data)
    except WebSocketDisconnect:
        connected_clients.remove(websocket)

async def handle_websocket_message(websocket: WebSocket, data: dict):
    """处理 WebSocket 消息"""
    message_type = data.get('type')

    if message_type == 'audio_data':
        await process_audio_data(websocket, data)
    elif message_type == 'video_frame':
        await process_video_frame(websocket, data)
    elif message_type == 'user_question':
        await process_user_question(websocket, data)

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

**交付物**：
- [ ] FastAPI 服务器可运行
- [ ] WebSocket 连接成功
- [ ] 基础路由配置

---

### Task 3.2：数据模型定义（1h）

**负责人**：______
**优先级**：P0
**依赖**：Task 3.1

**需求**：
- Pydantic 模型
- 数据结构定义

**实现**：
```python
# models.py
from pydantic import BaseModel
from typing import List, Literal, Optional

class StreamData(BaseModel):
    """流数据模型"""
    text: str
    type: Literal['audio', 'ocr']
    timestamp: float
    vector: Optional[List[float]] = None

class QuestionRequest(BaseModel):
    """用户提问"""
    question: str

class AnswerResponse(BaseModel):
    """AI 回答"""
    question: str
    answer: str
    evidence: List[StreamData]
```

**交付物**：
- [ ] 所有数据模型定义
- [ ] 类型检查通过

---

### Task 3.3：Embedding 模型集成（2h）

**负责人**：______
**优先级**：P0
**依赖**：Task 3.2

**需求**：
- 加载 sentence-transformers
- 文本转向量
- 预索引

**实现**：
```python
# embeddings.py
from sentence_transformers import SentenceTransformer
from typing import List
import numpy as np

# 初始化模型
model = SentenceTransformer(
    'sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2'
)

def encode_text(text: str) -> List[float]:
    """将文本转向量"""
    embedding = model.encode(text)
    return embedding.tolist()

def encode_texts(texts: List[str]) -> List[List[float]]:
    """批量转向量"""
    embeddings = model.encode(texts)
    return embeddings.tolist()

# 测试
if __name__ == "__main__":
    result = encode_text("这是一个测试")
    print(f"向量维度: {len(result)}")
    print(f"前5个值: {result[:5]}")
```

**交付物**：
- [ ] 模型加载成功
- [ ] 转向量功能正常
- [ ] 性能测试（100 条文本 < 1 秒）

---

### Task 3.4：Chroma 向量数据库（2h）

**负责人**：______
**优先级**：P0
**依赖**：Task 3.3

**需求**：
- 初始化 Chroma
- 存储向量
- 检索功能

**实现**：
```python
# vector_store.py
import chromadb
from chromadb.config import Settings
from typing import List, Dict, Any

# 初始化 Chroma
chroma_client = chroma.Client(Settings(
    anonymized_telemetry=False
))

collection = chroma_client.create_collection(
    name="livestream_memory",
    metadata={"hnsw:space": "cosine"}
)

def add_stream_data(data: StreamData):
    """添加流数据到向量数据库"""
    collection.add(
        embeddings=[data.vector],
        documents=[data.text],
        metadatas=[{
            "type": data.type,
            "timestamp": data.timestamp
        }],
        ids=[f"{data.type}_{data.timestamp}"]
    )

def search_similar(query_vector: List[float], top_k: int = 3):
    """向量检索"""
    results = collection.query(
        query_embeddings=[query_vector],
        n_results=top_k
    )
    return results

# 测试
if __name__ == "__main__":
    # 测试添加
    test_data = StreamData(
        text="这是一个测试",
        type="audio",
        timestamp=10.5,
        vector=[0.1] * 384  # 384 是向量维度
    )
    add_stream_data(test_data)
    print("数据添加成功")

    # 测试检索
    results = search_similar([0.1] * 384)
    print(f"检索结果: {results}")
```

**交付物**：
- [ ] Chroma 初始化成功
- [ ] 可以添加数据
- [ ] 可以检索数据
- [ ] 性能测试（检索 < 100ms）

---

### Task 3.5：数据处理流程（2h）

**负责人**：______
**优先级**：P0
**依赖**：Task 3.3, Task 3.4

**需求**：
- 接收音频数据 → 转向量 → 存储
- 接收视频帧 → OCR → 转向量 → 存储
- 内存缓存

**实现**：
```python
# processor.py
from embeddings import encode_text
from vector_store import add_stream_data
from models import StreamData
from typing import List
import base64
from PIL import Image
import io

# 内存存储
memory_store: List[StreamData] = []

async def process_audio_data(websocket: WebSocket, data: dict):
    """处理音频数据"""
    text = data.get('text')
    timestamp = data.get('timestamp')

    # 1. 转向量
    vector = encode_text(text)

    # 2. 构造数据模型
    stream_data = StreamData(
        text=text,
        type='audio',
        timestamp=timestamp,
        vector=vector
    )

    # 3. 存储到内存
    memory_store.append(stream_data)

    # 4. 存储到向量数据库
    add_stream_data(stream_data)

    # 5. 日志
    print(f"[{timestamp}] 收到语音: {text}")

async def process_video_frame(websocket: WebSocket, data: dict):
    """处理视频帧（OCR）"""
    image_data = data.get('image')
    timestamp = data.get('timestamp')

    # 1. 解码图片
    image_bytes = base64.b64decode(image_data.split(',')[1])
    image = Image.open(io.BytesIO(image_bytes))

    # 2. OCR 识别（调用 Qwen-VL）
    text = await ocr_with_qwen(image)

    if not text:
        return  # 没有识别到文字

    # 3. 转向量
    vector = encode_text(text)

    # 4. 构造数据模型
    stream_data = StreamData(
        text=text,
        type='ocr',
        timestamp=timestamp,
        vector=vector
    )

    # 5. 存储
    memory_store.append(stream_data)
    add_stream_data(stream_data)

    # 6. 日志
    print(f"[{timestamp}] OCR 识别: {text}")
```

**交付物**：
- [ ] 音频数据处理正常
- [ ] 视频帧处理正常（OCR 集成后）
- [ ] 内存存储正常
- [ ] 控制台日志清晰

---

### Task 3.6：OCR 集成（2h）

**负责人**：______
**优先级**：P0
**依赖**：Task 1.3, Task 3.5

**需求**：
- 调用 Qwen-VL API
- 识别图片中的文字
- 错误处理

**实现**：
```python
# ocr.py
from openai import OpenAI
import os
from PIL import Image
import base64
import io

client = OpenAI(
    api_key=os.getenv("DASHSCOPE_API_KEY"),
    base_url="https://dashscope.aliyuncs.com/compatible-mode/v1"
)

async def ocr_with_qwen(image: Image.Image) -> str:
    """使用 Qwen-VL 进行 OCR"""
    try:
        # 转换为 base64
        buffered = io.BytesIO()
        image.save(buffered, format="JPEG")
        img_base64 = base64.b64encode(buffered.getvalue()).decode()

        # 调用 API
        response = client.chat.completions.create(
            model="qwen-vl-plus",
            messages=[{
                "role": "user",
                "content": [
                    {"type": "image_url", "image_url": {"url": f"data:image/jpeg;base64,{img_base64}"}},
                    {"type": "text", "text": "请识别图片中的所有文字，只输出文字内容，不要解释。"}
                ]
            }]
        )

        text = response.choices[0].message.content.strip()
        return text

    except Exception as e:
        print(f"OCR 错误: {e}")
        return ""
```

**交付物**：
- [ ] OCR 功能正常
- [ ] 错误处理完善
- [ ] 性能测试（单次 OCR < 3 秒）

---

### Task 3.7：检索引擎（2h）

**负责人**：______
**优先级**：P0
**依赖**：Task 3.4, Task 3.5

**需求**：
- 向量检索
- 关键词匹配
- 混合检索

**实现**：
```python
# retriever.py
from embeddings import encode_text
from vector_store import search_similar
from processor import memory_store

def extract_keywords(text: str) -> List[str]:
    """简单的关键词提取"""
    import re
    # 移除标点符号，分词
    words = re.findall(r'\w+', text)
    return words

def keyword_search(query: str, top_k: int = 3) -> List[StreamData]:
    """关键词搜索"""
    keywords = extract_keywords(query)

    results = []
    for item in memory_store:
        # 检查是否包含所有关键词
        if all(kw in item['text'] for kw in keywords):
            results.append(item)

    return results[:top_k]

def hybrid_search(query: str, top_k: int = 3) -> List[StreamData]:
    """混合检索"""
    # 1. 向量检索
    query_vector = encode_text(query)
    semantic_results = search_similar(query_vector, top_k)

    # 2. 关键词搜索
    keyword_results = keyword_search(query, top_k)

    # 3. 合并去重
    all_results = semantic_results + keyword_results
    seen = set()
    unique_results = []

    for item in all_results:
        if item['timestamp'] not in seen:
            unique_results.append(item)
            seen.add(item['timestamp'])

    return unique_results[:top_k]
```

**交付物**：
- [ ] 向量检索正常
- [ ] 关键词搜索正常
- [ ] 混合检索正常
- [ ] 性能测试（检索 < 200ms）

---

### Task 3.8：LLM 问答（3h）

**负责人**：______
**优先级**：P0
**依赖**：Task 3.7

**需求**：
- 检索相关片段
- 构造 Prompt
- 调用 Qwen-Plus
- 格式化回答

**实现**：
```python
# qa.py
from openai import OpenAI
import os
from retriever import hybrid_search
from models import AnswerResponse

client = OpenAI(
    api_key=os.getenv("DASHSCOPE_API_KEY"),
    base_url="https://dashscope.aliyuncs.com/compatible-mode/v1"
)

async def answer_question(question: str) -> AnswerResponse:
    """回答用户问题"""

    # 1. 检索
    search_results = hybrid_search(question, top_k=3)

    if not search_results:
        return AnswerResponse(
            question=question,
            answer="抱歉，我没有找到相关信息。",
            evidence=[]
        )

    # 2. 构造 Prompt
    context = "\n".join([
        f"- [{item['timestamp']:.1f}s] {item['text']}"
        for item in search_results
    ])

    prompt = f"""
你是一个直播间助手，根据以下直播片段回答用户问题。

用户问题：{question}

直播片段：
{context}

请生成一个自然、友好的回答。直接给出答案，不要重复问题。
"""

    # 3. 调用 LLM
    try:
        response = client.chat.completions.create(
            model="qwen-plus",
            messages=[
                {"role": "system", "content": "你是一个直播间助手。"},
                {"role": "user", "content": prompt}
            ],
            temperature=0.7
        )

        answer = response.choices[0].message.content.strip()

        # 4. 格式化
        return AnswerResponse(
            question=question,
            answer=answer,
            evidence=search_results
        )

    except Exception as e:
        print(f"LLM 错误: {e}")
        return AnswerResponse(
            question=question,
            answer=f"抱歉，回答问题时出错了: {str(e)}",
            evidence=search_results
        )

# 处理用户提问
async def process_user_question(websocket: WebSocket, data: dict):
    """处理用户提问"""
    question = data.get('question')

    print(f"用户提问: {question}")

    # 生成回答
    response = await answer_question(question)

    # 发送给前端
    await websocket.send_json({
        "type": "ai_answer",
        "data": response.dict()
    })

    print(f"AI 回答: {response.answer}")
```

**交付物**：
- [ ] 问答功能正常
- [ ] 回答质量测试
- [ ] 性能测试（端到端 < 2 秒）

---

## 🧪 Phase 4：测试与调试（6h）

### Task 4.1：单元测试（2h）

**负责人**：______
**优先级**：P1
**依赖**：所有开发任务

**测试项**：
- [ ] Embedding 模型测试
- [ ] 向量检索测试
- [ ] OCR 功能测试
- [ ] LLM 问答测试

**交付物**：
- [ ] 测试报告
- [ ] Bug 列表

---

### Task 4.2：集成测试（2h）

**负责人**：______
**优先级**：P0
**依赖**：Task 4.1

**测试流程**：
1. 启动后端
2. 启动前端
3. 播放视频
4. 观察数据流
5. 提问测试

**交付物**：
- [ ] 端到端测试通过
- [ ] Bug 修复

---

### Task 4.3：性能优化（2h）

**负责人**：______
**优先级**：P1
**依赖**：Task 4.2

**优化项**：
- [ ] 向量检索速度（< 200ms）
- [ ] OCR 并发控制
- [ ] LLM 调用缓存
- [ ] WebSocket 消息队列

**交付物**：
- [ ] 性能测试报告
- [ ] 优化后性能指标

---

## 🎬 Phase 5：演示准备（4h）

### Task 5.1：演示数据准备（2h）

**负责人**：______
**优先级**：P0
**依赖**：所有开发任务

**任务**：
1. 准备 3 个问答案例
2. 测试每个案例
3. 准备备用方案

**交付物**：
- [ ] 案例测试通过
- [ ] 演示脚本

---

### Task 5.2：演示演练（2h）

**负责人**：______
**优先级**：P0
**依赖**：Task 5.1

**任务**：
1. 完整演示流程（3-5 分钟）
2. 时间控制
3. 备用方案测试

**交付物**：
- [ ] 演示视频录制
- [ ] 演讲稿

---

## 📊 进度跟踪

### 每日站会（建议）

**每天上午 10:00，15 分钟**

**每个成员汇报**：
1. 昨天完成了什么
2. 今天计划做什么
3. 遇到什么阻碍

### 任务看板

使用 GitHub Projects 或 Trello：

```
[Backlog] → [To Do] → [In Progress] → [Done]
```

---

## ✅ 验收标准

### 技术验收
- [ ] HLS 视频播放流畅
- [ ] 数据采集实时同步（< 100ms 延迟）
- [ ] 向量检索准确率 > 80%
- [ ] OCR 识别率 > 70%
- [ ] LLM 回答质量 > 4/5

### 演示验收
- [ ] 3-5 分钟完整演示
- [ ] 至少 2 个问答案例
- [ ] 无重大 Bug
- [ ] 评委能理解核心价值

---

## 🚨 风险与应对

| 风险 | 概率 | 影响 | 应对 |
|------|------|------|------|
| OCR 成本超支 | 中 | 中 | 限制 OCR 频率，使用假数据 |
| LLM 响应慢 | 中 | 中 | 展示加载动画，使用缓存 |
| 演示时网络问题 | 高 | 高 | 准备离线视频，本地服务器 |
| API 达到限额 | 低 | 中 | 准备备用 API Key |

---

## 📞 联系方式

**技术负责人**：______
**前端负责人**：______
**后端负责人**：______

---

**文档版本**: v1.0
**最后更新**: 2025-12-31
**预计完成**: ____
