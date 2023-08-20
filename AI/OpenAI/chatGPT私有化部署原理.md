
首先基于私有数据构建Bot的流程目前大致分为两种，一种是使用私有数据集进行finetune，从而得到理解私有数据的LLM大模型。另一种是利用文本的相似度搜索，构建出context，利用few shot learning的方式让LLM进行Q&A。前者的方案主要基于 [tatsu-lab/stanford_alpaca: Code and documentation to train Stanford's Alpaca models, and generate the data. (github.com)](https://github.com/tatsu-lab/stanford_alpaca)

基于上述的方案，衍生出了一系列产物，比如
[https://github.com/antimatter15/alpaca.cpp](https://t.co/tAzzVXrut3)
[https://github.com/tloen/alpaca-lora](https://t.co/jEMMZ1vtxg) 
[https://github.com/lm-sys/FastChat](https://t.co/uNzZ63uZeN) 
[https://github.com/nomic-ai/gpt4all 这些都是利用GPT3.5/GPT4作为数据打标师，从而一方面调优Meta的LLaMA，另一方面顺着这样的思路也能进行私有数据的finetune](https://t.co/sdoBHx5QJk)

另一种方案是利用这些已经对LLaMA finetune 好的模型进行构建。按照Langchain的文档
https://python.langchain.com/en/latest/use_cases/question_answering.html  
就可以快速构建，从文本Embedding->存储VectorDB->Similarity Search->Q&A with Context。这时候有人会问，Langchain 文档上这些Vectorstores 还是得用到OpenAIEmbeddings？那怎么私有化?

其实答案就藏在 [Llama-cpp — 🦜🔗 LangChain 0.0.136](https://python.langchain.com/en/latest/modules/models/text_embedding/examples/llamacpp.html) 这篇文档中，我们可以利用llama CPP进行构建Embedding，这也是近期引入的新功能  [Add embedding mode with arg flag. Currently working by StrikingLoo · Pull Request #282 · ggerganov/llama.cpp (github.com)](https://github.com/ggerganov/llama.cpp/pull/282)  
虽然目前是用llama.cpp构建Embedding，但是可以在Q&A使用的上述提到的finetune后的模型，通过这种方式组合起来达到私有化的目的。

近期一系列llama.cpp的优化，使得本地可以部署越来越大的参数模型，我自己体验下来，这些模型在英语交互上效果更好，在中文上还差了一些，所以如果需要使用中文交互，可以使用这几个finetune模型 [ymcui/Chinese-LLaMA-Alpaca: 中文LLaMA&Alpaca大语言模型+本地CPU部署 (Chinese LLaMA & Alpaca LLMs) (github.com)](https://github.com/ymcui/Chinese-LLaMA-Alpaca) 
[Facico/Chinese-Vicuna: Chinese-Vicuna: A Chinese Instruction-following LLaMA-based Model —— 一个中文低资源的llama+lora方案，结构参考alpaca (github.com)](https://github.com/Facico/Chinese-Vicuna)

这里分享一个简单的实现代码，通过 Loader读取数据进行分词并构建Embedding，接着根据问题进行similarity search，之后将匹配到的上下文和问题组合在一起丢给LLaMA进行回答。之后我会尝试使用ChatGLM搭建这一套流程并分享给大家

Creating a private data QA bot entirely using the open-source LLM project

```python
from langchain import PromptTemplate, LLMChain
from langchain.document_loaders import UnstructuredHTMLLoader
from langchain.embeddings import LlamaCppEmbeddings
from langchain.llms import LlamaCpp
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain.vectorstores.faiss import FAISS

loader = UnstructuredHTMLLoader("langchain/docs/_build/html/index.html")
embedding = LlamaCppEmbeddings(model_path="path/models/ggml-model-q4_0.bin")
llm = LlamaCpp(model_path="path/models/ggml-model-q4_0.bin")


def split_chunks(sources: list) -> list:
    chunks = []
    splitter = RecursiveCharacterTextSplitter(separator="", chunk_size=256, chunk_overlap=16)
    for chunk in splitter.split_documents(sources):
        chunks.append(chunk)
    return chunks


def generate_embedding(chunks: list):
    texts = [doc.page_content for doc in chunks]
    metadatas = [doc.metadata for doc in chunks]

    search_index = FAISS.from_texts(texts, embedding, metadatas=metadatas)

    return search_index


def similarity_search(
        query: str, index: FAISS
) -> (list, list):
    matched_docs = index.similarity_search(query, k=4)
    sources = []
    for doc in matched_docs:
        sources.append(
            {
                "page_content": doc.page_content,
                "metadata": doc.metadata,
            }
        )

    return matched_docs, sources


docs = loader.load()
chunks = split_chunks(docs)
embeddings = generate_embedding(chunks)

question = "What are the use cases of LangChain?"
matched_docs, sources = similarity_search(question, embeddings)

template = """
Please use the following context to answer questions.
Context: {context}
---
Question: {question}
Answer: Let's think step by step."""

context = "\n".join([doc.page_content for doc in matched_docs])
prompt = PromptTemplate(template=template, input_variables=["context", "question"]).partial(context=context)
llm_chain = LLMChain(prompt=prompt, llm=llm)

print(llm_chain.run(question))
```




## 一些样例 

[GPTBase](https://gptbase.ai/)

[SiteGPT – ChatGPT for every website](https://sitegpt.ai/)


[Databerry - Connect your data to ChatGPT and other Language Models](https://www.databerry.ai/)


https://app.copilothub.co/

