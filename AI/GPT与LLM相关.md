
一个开源的多模态LLM Demo 

[Prismer - a Hugging Face Space by lorenmt](https://huggingface.co/spaces/lorenmt/prismer)


gpt观看一个youtube视频 [Chat YouTube](https://chatyoutube.com/)


Peo集成了new bing chatGPT all in one等 https://poe.com/ 

chatGPT prompt的样例集合 [Awesome ChatGPT Prompts | This repo includes ChatGPT prompt curation to use ChatGPT better.](https://prompts.chat/)  

[如何用 ChatGPT 构建你的专属知识问答机器人 - Frank 的个人博客 (frankzhao.cn)](https://blog.frankzhao.cn/build_gpt_bot_for_doc/)




[GPT4All Chat](https://gpt4all.io/index.html)

[HuggingChat (huggingface.co)](https://huggingface.co/chat/privacy)

[peter! 🥷 在 Twitter: "🪄 INTRODUCING: Chat with any Github Repo. 1) Scrapes @github repo 🤖 2) 🗂️ Embeds codebase using @LangChainAI, stores embeddings in @activeloopai 3) Chat with the codebase w/ @streamlit 👨‍💻 Demo, Code explained &amp; Github Link↓ https://t.co/1sBw0C3QuT" / Twitter](https://twitter.com/pwang_szn/status/1650801868568772608)


[How ChatGPT works: a deep dive | Dan Hollick 🇿🇦 (typefully.com)](https://typefully.com/DanHollick/how-chatgpt-works-a-deep-dive-yA3ppZC)

[Finchat - ChatGPT for Finance | FinChat.io](https://finchat.io/)

[How ChatGPT Works Technically | ChatGPT Architecture - YouTube](https://www.youtube.com/watch?v=bSvTVREwSNw&feature=youtu.be)

[Transformers from Scratch (e2eml.school)](https://e2eml.school/transformers.html)

[rl-for-llms.md (github.com)](https://gist.github.com/yoavg/6bff0fecd65950898eba1bb321cfbd81)



现在的很多AI项目都用word embedding做向量搜索来检索文本或知识库，但这里面有很大的误区： 1. 词向量模型往往训练的是词语在训练语料上的接近性，而非语义上的同义性。这有助于召回，但对准确性造成较大影响。 2. query的句向量和text chunk的句向量仍然表示的是接近性，它往往无法作为问答对来使用。
3. 常见的影响包括：在向量空间中，可能地名、人名等名词会被混杂在一起，失去准确性。例如搜A地出B地的内容，搜A人物出B人物的内。一个query的中心词可能被忽略，转而匹配了整句话的泛化词表征…… 因此，当下的很多AI项目在涉及内容检索的步骤时都非常拉跨…… 解法还需要回到古典的NLP和搜索技术上

还有就是，取决于你prompt怎么写的，通常这类问答都要求GPT如果没搜到就说没有找到，但你也可以让它：文档找不到就给我编一个！

以下是一个缓解办法，[LangChainの新機能Contextual Compression Retrieverを試す｜mah_lab / 西見 公宏｜note](https://note.com/mahlab/n/n7d72e83904cc) 


[A Very Gentle Introduction to Large Language Models without the Hype | by Mark Riedl | Apr, 2023 | Medium](https://mark-riedl.medium.com/a-very-gentle-introduction-to-large-language-models-without-the-hype-5f67941fa59e)

上下文压缩的文本查询优化 [Harrison Chase 在 Twitter: "⭐️Contextual Compression⭐️ We introduce multiple new methods in @LangChainAI to compress retrieved documents w.r.t. the query before passing to an LLM for generation Inspired by @willpienaar at the "LLMs in production" conference Blog: https://t.co/bs7psooUry 🧵More details:" / Twitter](https://twitter.com/hwchase17/status/1649428295467905025) 

https://www.promptingguide.ai/

[LangChain Tutorial in Python - Crash Course - Python Engineer (python-engineer.com)](https://www.python-engineer.com/posts/langchain-crash-course/)

[Replit - How to train your own Large Language Models](https://blog.replit.com/llm-training)
[如何训练你自己的大型语言模型 - 宝玉 - 博客园 (cnblogs.com)](https://www.cnblogs.com/dotey/p/17336142.html) 

[Let's build GPT: from scratch, in code, spelled out. - YouTube](https://www.youtube.com/watch?v=kCc8FmEb1nY)

[AutoGPT - a Hugging Face Space by aliabid94](https://huggingface.co/spaces/aliabid94/AutoGPT)

[Implementing a sales & support agent with LangChain | by Tomaz Bratanic | Apr, 2023 | Towards Data Science](https://towardsdatascience.com/implementing-a-sales-support-agent-with-langchain-63c4761193e7)

[Logan.GPT 在 Twitter: "Can someone pls explain what @LangChainAI does? 😅 Asking for a friend of course." / Twitter](https://twitter.com/OfficialLoganK/status/1648709651205173248)

[Building LLM applications for production (huyenchip.com)](https://huyenchip.com/2023/04/11/llm-engineering.html)

[Build an Ecommerce Chatbot with Redis, LangChain, and OpenAI | Redis](https://redis.com/blog/build-ecommerce-chatbot-with-redis/)

[Summary of the tokenizers (huggingface.co)](https://huggingface.co/docs/transformers/tokenizer_summary)

[Chunking Strategies for LLM applications | Pinecone](https://www.pinecone.io/learn/chunking-strategies/)

[Building an Image Recognition App in Javascript using Pinecone, Hugging Face, and Vercel | Pinecone](https://www.pinecone.io/learn/pinecone-vision-app/)

