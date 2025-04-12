
[X 上的 熠辉 Indie：“如何免费拥有一个满血版DeepSeek - R1智能客服？看完这个视频你会get到： 1. 从template0选择优秀模板，并用cursor + agent模式轻松修改成自己官网 2. 去火山引擎获得一个超稳满血版DeepSeek-R1 3. 如何创建一个Dify + DeepSeek-R1智能客服？ 4. 用Cursor将DeepSeek-R1的智能客服接入自己的网站 https://t.co/BiSyDNodcE” / X](https://x.com/yihui_indie/status/1892930985047531763)

[zhayujie/chatgpt-on-wechat: 基于大模型搭建的聊天机器人，同时支持 微信公众号、企业微信应用、飞书、钉钉 等接入，可选择GPT3.5/GPT-4o/GPT-o1/ DeepSeek/Claude/文心一言/讯飞星火/通义千问/ Gemini/GLM-4/Claude/Kimi/LinkAI，能处理文本、语音和图片，访问操作系统和互联网，支持基于自有知识库进行定制企业智能客服。](https://github.com/zhayujie/chatgpt-on-wechat)

[cs-lazy-tools/ChatGPT-On-CS: 基于大模型的智能对话客服工具，支持微信、拼多多、千牛、哔哩哔哩、抖音企业号、抖音、抖店、微博聊天、小红书专业号运营、小红书、知乎等平台接入，可选择 GPT3.5/GPT4.0/ 懒人百宝箱 （后续会支持更多平台），能处理文本、语音和图片，通过插件访问操作系统和互联网等外部资源，支持基于自有知识库定制企业 AI 应用。](https://github.com/cs-lazy-tools/ChatGPT-On-CS)


[Nutlope/roomGPT: Upload a photo of your room to generate your dream room with AI.](https://github.com/Nutlope/roomGPT)

[arc53/DocsGPT: Chatbot for documentation, that allows you to chat with your data. Privately deployable, provides AI knowledge sharing and integrates knowledge into your AI workflow](https://github.com/arc53/DocsGPT)

[lvwzhen/law-cn-ai: ⚖️ AI 法律助手](https://github.com/lvwzhen/law-cn-ai)


[langgenius/dify: Dify is an open-source LLM app development platform. Dify's intuitive interface combines AI workflow, RAG pipeline, agent capabilities, model management, observability features and more, letting you quickly go from prototype to production.](https://github.com/langgenius/dify)

[unclecode/crawl4ai: 🚀🤖 Crawl4AI: Open-source LLM Friendly Web Crawler & Scraper. Don't be shy, join here: https://discord.gg/mEkkMXFG](https://github.com/unclecode/crawl4ai)

[Home - Crawl4AI Documentation (v0.5.x)](https://docs.crawl4ai.com/)



[! crawl4ai The requested image's platform (linux/arm64/v8) does not match the detected host platform (linux/amd64/v3) and no specific platform was requested 0.0s · Issue #240 · unclecode/crawl4ai](https://github.com/unclecode/crawl4ai/issues/240#issuecomment-2507773599)


[目前国内可用Docker镜像源汇总（截至2025年1月） - 知乎](https://zhuanlan.zhihu.com/p/18057716675)

```bash
# Secured Instance 

docker run -p 11235:11235 -e CRAWL4AI_API_TOKEN=your_secret_token unclecode/crawl4ai:all

# Unsecured Instance 
docker run -p 11235:11235 unclecode/crawl4ai:all

# AMD64 
docker pull unclecode/crawl4ai:basic-amd64
docker pull unclecode/crawl4ai:all-amd64
docker run -p 11235:11235 -e CRAWL4AI_API_TOKEN=landuitest unclecode/crawl4ai:basic-amd64

python -m venv myaienv

source myaienv/bin/activate
pip install crawl4ai -i https://pypi.tuna.tsinghua.edu.cn/simple/
```

```python
import requests

# Setup headers if token is being used
api_token = "your_secret_token"  # Same token set in CRAWL4AI_API_TOKEN
headers = {"Authorization": f"Bearer {api_token}"} if api_token else {}

# Making authenticated requests
response = requests.post(
    "http://localhost:11235/crawl",
    headers=headers,
    json={
        "urls": "https://example.com",
        "priority": 10
    }
)

# Checking task status
task_id = response.json()["task_id"]
status = requests.get(
    f"http://localhost:11235/task/{task_id}",
    headers=headers
)
```

[Command Line Interface - Crawl4AI Documentation (v0.5.x)](https://docs.crawl4ai.com/core/cli/)

[Markdown Generation - Crawl4AI Documentation (v0.5.x)](https://docs.crawl4ai.com/core/markdown-generation/)

