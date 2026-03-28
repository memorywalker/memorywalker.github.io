---
title: Rust开发一个最简单的RAG
date: 2026-03-28T21:11:00
categories:
  - AI
tags:
  - AI
  - rust
---
## Rust开发一个最简单的RAG

由于之前本机电脑运行LM studio的效果比Ollama好很多，就来试试使用LM Studio提供的OpenAI兼容API来实现简单Agent功能
现在用的比较多的库是Python的LangChain，但是为了让我学过的rust不会生疏，还是得多用起来
Rust中对AI相关的支持库还是挺多的，比如Rig，今天想从最简单的方式去尝试开发，不用Rig库，这样也知道其中的细节流程

### RAG运行步骤

1. 参考数据准备，包括数据清洗，分割
2. 对分割好的Chuck数据片段向量编码（嵌入）
3. 把数据片段和它的向量值存入向量数据库，供以后增强检索
4. 用户查询文本向量化后，在向量数据库中检索出k个和这个向量最近邻的相关数据
5. 将查询到的相关数据重排后和用户的查询数据一起作为上下文提供给大模型
6. 大模型根据额外的上下文知识，进行推理给出最终结果到用户

下面就按上面的基本步骤来实现最简单的RAG

cargo.toml需要添加以下依赖

```toml
[dependencies]
tokio = { version = "1.0", features = ["full"] }
reqwest = { version = "0.11", features = ["json"] }
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
anyhow = "1.0"
dotenv = "0.15"
lancedb = { version = "0.22.3", features = ["polars"] }
polars = ">=0.37,<0.40.0"
polars-arrow = ">=0.37,<0.40.0"
arrow-array = "56.2.0"
arrow-json = "56.2.0"
arrow-schema = "56.2.0"
futures = "0.3"
uuid = { version = "1.0", features = ["v4"] }
```

### 文本分割

`src/ingest.rs` 中对数据清洗，长文本分割为文本片段，并去调用嵌入模型获取嵌入向量。我这里只是最简单的按长度进行文本分割。

```rust
use anyhow::Result;
use crate::{embedding, vectordb::Record, vectordb};
use uuid::Uuid;
// 文本分割
fn split_text(text: &str, chunk_size: usize) -> Vec<String> {
    let mut chunks = Vec::new();
    let mut start = 0;
    while start < text.len() {
        let end = usize::min(start + chunk_size, text.len());
        chunks.push(text[start..end].to_string());
        start = end;    
    }
    chunks
}
// 把分割后的文本向量化后，存储到向量数据库中
pub async fn ingest_text(text: &str) -> Result<()> {
    let chunks = split_text(text, 300);
    let mut records = Vec::new();
    for chunk in chunks {
        println!("处理文本块: {}", chunk);
        let embedding = embedding::embed(&chunk).await?;
        records.push(Record {
            id: Uuid::new_v4().to_string(),
            text: chunk,
            vector: embedding,
        });
    }
    if !records.is_empty() {
        let embedding_dim = records[0].vector.len() as i32;
        vectordb::insert_records(records, embedding_dim).await?;
    }    
    Ok(())
}
```
### 数据嵌入向量化

`src/embedding.rs` 中使用reqwest库直接访问LM Studio提供的API接口，将输入文本通过文本嵌入模型获得对应的嵌入向量的值，这个值就是f32类型的一维数组。

```rust
use anyhow::Result;
use reqwest::Client;
use serde_json::json;
use std::env;

pub async fn embed(text: &str) -> Result<Vec<f32>> {
    let api_url = env::var("EMBEDDING_API")?;
    let model = env::var("EMBEDDING_MODEL")?;

    let client = Client::new();
    let request_body = json!({
        "model": model,
        "input": text
    });

    let response = client.post(&api_url)
        .json(&request_body)
        .send()
        .await?
        .json::<serde_json::Value>()
        .await?;

    let arr = response["data"][0]["embedding"].as_array().unwrap();

    Ok(arr.iter()
        .map(|v| v.as_f64().unwrap() as f32)
        .collect())
}
```

### 量数据库存储和检索

向量数据库有很多，AI推荐的是[Qdrant](https://qdrant.tech/)，但是这个需要Docker环境在windows使用有点麻烦，我选择了[LanceDB](https://lancedb.com/)，这是个使用rust实现的开源向量数据库。它支持本地数据文件存储，不需要运行任何服务，和SQLite有点像。虽然这个库是rust实现的内核，但是对rust支持挺一般的。我主要参考了官方的指南的这个代码 https://github.com/lancedb/docs/blob/main/tests/rs/quickstart.rs

`src/vectordb.rs` 这个是目前整个工程中最长的代码了，虽然也就100多行，主要是我让AI帮我生成代码，始终编译有问题，走了弯路，最后还是参考官方代码正常实现了。

Lancedb需要使用`arrow_array`的数据结构来往LanceDB中存储数据，因此需要实现`records_to_reader()`方法来把文本和对应的向量数据转换成`arrow_array`的RecordBatch。schema是用来告诉数据库这个表的结构是什么样的。具体这个库的使用有很多细节，包括建立索引，查询选择不同的算法，在官方指南有详细介绍算法的实现，这里我只是用了最简单的方法。

```rust
use anyhow::{anyhow, Result, Context};
use arrow_array::types::Float32Type;
use arrow_array::{Array, FixedSizeListArray, Float32Array, LargeStringArray, RecordBatch, RecordBatchIterator};
use arrow_schema::{DataType, Field, Schema};
use lancedb::query::{ExecutableQuery, QueryBase, Select};
use lancedb::{connect, table::Table, Connection};
use serde::{Serialize, Deserialize};
use std::sync::OnceLock;
use std::sync::Arc;
use futures::TryStreamExt;

static DB: OnceLock<Connection> = OnceLock::new();
// 初始化数据库
pub async fn init() -> Result<()> {
    let db = connect("data").execute().await?;
    DB.set(db).map_err(|_| anyhow!("Database already initialized"))?;
    Ok(())
}
// 一个文本片段结构，主要包括文本内容和它对应的向量值
#[derive(Serialize, Deserialize, Clone, Debug)]
pub struct Record {
    pub id: String,
    pub text: String,
    pub vector: Vec<f32>,
}
// 告诉数据库这个表的结构，例如第一列id，数据类型是字串
fn create_schema(vector_dim: i32) -> Arc<Schema> {
    Arc::new(Schema::new(vec![
        Field::new("id", DataType::LargeUtf8, false),
        Field::new("text", DataType::LargeUtf8, false),
        Field::new(
            "vector",
            DataType::FixedSizeList(
                Arc::new(Field::new("item", DataType::Float32, true)),
                vector_dim,
            ),
            false,
        ),
    ]))
}

type BatchIter = RecordBatchIterator<
    std::vec::IntoIter<std::result::Result<RecordBatch, arrow_schema::ArrowError>>,
>;
// 将多个文本数据转换为arrow_array的结构
fn records_to_reader(schema: Arc<Schema>, rows: &[Record]) -> BatchIter {
    let ids = LargeStringArray::from_iter_values(rows.iter().map(|row| row.id.as_str()));
    let texts = LargeStringArray::from_iter_values(rows.iter().map(|row| row.text.as_str()));
    let vectors = FixedSizeListArray::from_iter_primitive::<Float32Type, _, _>(
        rows.iter()
            .map(|row| Some(row.vector.iter().copied().map(Some).collect::<Vec<_>>())),
        rows.first().map(|r| r.vector.len() as i32).unwrap_or(0),
    );

    let batch = RecordBatch::try_new(
        schema.clone(),
        vec![Arc::new(ids), Arc::new(texts), Arc::new(vectors)],
    )
    .unwrap();
    RecordBatchIterator::new(vec![Ok(batch)].into_iter(), schema)
}
// 插入一条记录
pub async fn insert_records(records: Vec<Record>, vector_dim: i32) -> anyhow::Result<()> {
    let db = DB.get().unwrap();
    let schema = create_schema(vector_dim);
    let table = match db.open_table("docs").execute().await {
        Ok(table) => table,
        Err(_) => {// 只有表没有创建的时候才执行Create
            db.create_table("docs", records_to_reader(schema.clone(), &records))
                .execute()
                .await?
        }
    };
    // 添加一条数据到数据表中
    table
        .add(records_to_reader(schema.clone(), &records))
        .execute()
        .await?;    
    Ok(())
}
// 从数据库中检索和输入的向量最邻近的n个数据
pub async fn search(query_vector: Vec<f32>, limit: usize) -> anyhow::Result<Vec<Record>> {
    let db = DB.get().ok_or(anyhow::anyhow!("Database not initialized"))?;

    let table: Table = db.open_table("docs").execute().await.unwrap();
    let mut results = table
        .query()
        .nearest_to(query_vector)// 这里可以有不同的算法
        .unwrap()
        // .select(Select::Columns(vec![
        //     "id".to_string(),
        //     "text".to_string(),
        // ]))
        .limit(limit)
        .execute()
        .await
        .unwrap();

    let mut records = Vec::new();
    // 使用 try_next() 遍历流中的每个 RecordBatch
    while let Some(batch) = results.try_next().await? {
        // 从 batch 中提取列
        let ids = batch
            .column(0)
            .as_any()
            .downcast_ref::<LargeStringArray>()
            .context("Column 0 is not a StringArray")?;
        let texts = batch
            .column(1)
            .as_any()
            .downcast_ref::<LargeStringArray>()
            .context("Column 1 is not a StringArray")?;
        let vectors = batch
            .column(2)
            .as_any()
            .downcast_ref::<FixedSizeListArray>()
            .context("Column 2 is not a FixedSizeListArray")?;

        for i in 0..batch.num_rows() {
            let id = ids.value(i).to_string();
            let text = texts.value(i).to_string();

            // 提取向量：从 FixedSizeListArray 中取出第 i 个元素，转换为 Float32Array
            let vector_arc = vectors.value(i);
            let vec_array = vector_arc
                .as_any()
                .downcast_ref::<Float32Array>()
                .context("Failed to downcast vector element to Float32Array")?;
            let vector = vec_array.values().to_vec();
            records.push(Record { id, text, vector });
        }
    }
    println!("查询到 {} 条相关记录", records.len());
    for rec in records.iter() {
        println!("记录ID: {}, 文本: {}, 向量前5维: {:?}", rec.id, rec.text, &rec.vector[..5.min(rec.vector.len())]);        
    }
    Ok(records)
}
```

### 实现RAG流程

`src/rag.rs` 中按RAG的流程逐步调用

```rust
use anyhow::Result;
use crate::{embedding, vectordb, llm};

pub async fn ask(question: &str) -> Result<String> {
    // 1. 获取问题的向量表示
    let embedding = embedding::embed(question).await?;
    // 2. 在向量数据库中查询相关内容，取最接近的3个
    let docs = vectordb::search(embedding, 3).await?;
    // 3. 将查询到的内容拼接成上下文
    let context = docs.into_iter().map(|r| r.text.clone()).collect::<Vec<_>>().join("\n---\n");
    // 4. 构建提示词并调用LLM生成回答
    let prompt = format!("你是一个专业助手，请基于上下文回答问题: \n\n上下文: \n{}\n\n问题: {}", context, question);
    // 5. 返回LLM的回答
    let response = llm::chat(&prompt).await?;
    Ok(response)
}
```
### 调用LLM获取返回结果

`src/llm.rs`负责接收提示词，使用配置的大语言模型进行推理，并获取最终的结果返回。这里主要是调整提示词，用来在不同的使用场景获取更好的效果。

```rust
use anyhow::Result;
use reqwest::Client;
use serde_json::json;
use std::env;

pub async fn chat(prompt: &str) -> Result<String> {
    let api_url = env::var("LLM_API")?;
    let model = env::var("MODEL")?;

    let client = Client::new();
    let request_body = json!({
        "model": model,
        "messages": [
            {"role": "system", "content": "You are a helpful assistant."},
            {"role": "user", "content": prompt}
        ]
    });

    let response = client.post(&api_url)
        .json(&request_body)
        .send()
        .await?
        .json::<serde_json::Value>()
        .await?;
    Ok(response["choices"][0]["message"]["content"].as_str().unwrap().to_string())
}
```

### Agent应用

RAG只是基于大模型的一种应用，我们可以根据不同的目的开发不同的Agent满足需求。增加了一个Agent层用来管理多个不同的Agent。`src/agent.rs`目前只有一个rag的功能的agent，它把用户的输入传给rag模块，获取返回的结果。

```rust
use anyhow::Result;
use crate::rag;

pub async fn run(input: &str) -> Result<String> {
    println!("用户输入: {}", input);
    let response = rag::ask(input).await?;
    Ok(response)
}
```

### 应用程序总入口

`src/main.rs` 从终端获取用户输入，并将输入给Agent，并将Agent返回结果显示在终端。这里输入了三段背景知识资料。对于复杂系统会把pdf文件转成文本，进行分割，存储到向量数据库中，作为额外的知识库。

```rust
mod llm;
mod embedding;
mod vectordb;
mod ingest;
mod rag;
mod agent;

use std::io::{self, Write};
use anyhow::Result;

#[tokio::main]
async fn main() -> Result<()> {
    dotenv::dotenv().ok();
    println!("Agent start!");
    vectordb::init().await?;
    // 这里测试输入3段背景知识资料
    ingest::ingest_text("Rust is a systems programming language focused on safety, speed, and concurrency. It was designed to be a safe alternative to C and C++, with a strong emphasis on memory safety and zero-cost abstractions. Rust achieves memory safety without a garbage collector, using a system of ownership with rules that the compiler checks at compile time. This allows developers to write efficient and safe code, making Rust a popular choice for performance-critical applications such as game development, operating systems, and web servers.").await?;
    ingest::ingest_text("tokio is an asynchronous runtime for Rust that provides the building blocks needed for writing asynchronous applications. It includes a multi-threaded, work-stealing scheduler, a powerful timer system, and support for asynchronous I/O. Tokio allows developers to write high-performance, scalable applications that can handle many concurrent tasks without blocking the main thread. It is widely used in web servers, network applications, and other scenarios where high concurrency is required.").await?;
    ingest::ingest_text("memorywalker is from China and he love studing").await?;
    
    loop {
        print!("\n> ");
        io::stdout().flush().unwrap();

        let mut input = String::new();
        std::io::stdin().read_line(&mut input)?;
        let response = agent::run(input.trim()).await?;
        println!("\n{}", response);  
    }
    Ok(())
}

```

### 环境配置

项目的根目录下新建`.env`文件，其中内容为环境变量配置值，用来在程序中获取API和模型配置信息

```ini
LLM_API=http://localhost:1234/v1/chat/completions
EMBEDDING_API=http://localhost:1234/v1/embeddings
EMBEDDING_MODEL=text-embedding-nomic-embed-text-v1.5
MODEL=qwen/qwen3.5-9b
```

另外还要配置LM Studio，在它的开发者界面中打开服务运行，并同时加载千问3.5-9b模型和文本嵌入模型

![LMStudio](uploads/ai/lmstuido_server.png)
### 最终运行效果

因为我运行了多次这个程序，导致背景知识三段话被重复插入到了数据库中，当我询问`tell me something about memorywalker`时，向量数据库只返回了和memorywalker相关3条记录，rust的记录没有一条返回，的确找到了相关的背景知识。虽然3条记录的内容相同，但是id是不同的，这是因为我重复运行程序，main函数的测试数据被存储了3次。
另外看模型的思考过程中，它发现背景知识中`memorywalker is from China and he love studing`有语法错误，它做了一些纠结后，最后还是以一个专业助手的角度把语法错误改正，并给出了英文结论`Based on the provided context, memorywalker is from China and he loves studying.`。
当我再次问模型`Is he a good guy?`，模型改为了用中文思考，并用中文给出了回答。

```
Agent start!
处理文本块: Rust is a systems programming language focused on safety, speed, and concurrency. It was designed to be a safe alternative to C and C++, with a strong emphasis on memory safety and zero-cost abstractions. Rust achieves memory safety without a garbage collector, using a system of ownership with rules
处理文本块:  that the compiler checks at compile time. This allows developers to write efficient and safe code, making Rust a popular choice for performance-critical applications such as game development, operating systems, and web servers.
处理文本块: tokio is an asynchronous runtime for Rust that provides the building blocks needed for writing asynchronous applications. It includes a multi-threaded, work-stealing scheduler, a powerful timer system, and support for asynchronous I/O. Tokio allows developers to write high-performance, scalable appl
处理文本块: ications that can handle many concurrent tasks without blocking the main thread. It is widely used in web servers, network applications, and other scenarios where high concurrency is required.
处理文本块: memorywalker is from China and he love studing

> tell me something about memorywalker
用户输入: tell me something about memorywalker
查询到 3 条相关记录
记录ID: 9dd5b6fb-ae4e-4af3-a43f-411b25fbf0a4, 文本: memorywalker is from China and he love studing, 向量前5维: [-0.016413346, 0.078260995, -0.21805552, -0.010767964, -0.014839452]
记录ID: 858285ce-eeb4-4dce-9e56-2efa62ca47dd, 文本: memorywalker is from China and he love studing, 向量前5维: [-0.016413346, 0.078260995, -0.21805552, -0.010767964, -0.014839452]
记录ID: 4d4cdac5-6c6d-41fd-b6a2-38dbcc8ec867, 文本: memorywalker is from China and he love studing, 向量前5维: [-0.016413346, 0.078260995, -0.21805552, -0.010767964, -0.014839452]

<think>Thinking Process:

1.  **Analyze the Request:**
    *   Role: Professional assistant.
    *   Task: Answer a question based on the provided context.
    *   Context: Three identical lines stating "memorywalker is from China and he love studing".
    *   Question: "tell me something about memorywalker"

2.  **Analyze the Context:**
    *   The context contains three repeated sentences: "memorywalker is from China and he love studing".
    *   Note on grammar: "love studing" should probably be "loves studying", but I must stick to the provided text or interpret it naturally while remaining accurate to the source. However, as an AI assistant, I should correct minor grammatical errors if stating facts derived from the text, or simply report the information contained in the text without necessarily emphasizing the grammar unless it changes meaning.
    *   Key Information extracted:
        *   Name/Identity: memorywalker
        *   Origin: China
        *   Interest/Activity: studying

3.  **Formulate the Answer:**
    *   The user wants to know something about "memorywalker".
    *   Based *only* on the provided context, I should state the facts available.
    *   Draft 1 (Internal Monologue): Okay, the text says memorywalker is from China and loves studying. I will report this.
    *   Draft 2 (Polite & Professional): According to the information provided, memorywalker is from China and enjoys studying.
    *   Draft 3 (Addressing the repetition): The context repeats the same sentence three times. It doesn't add new info. So just summarizing the unique info is enough.
    *   Draft 4 (Language): The question is in English ("tell me something about memorywalker"), but the prompt instructions are in Chinese ("你是一个专业助手，请基于上下文回答问题"). Usually, if the context and question are in English, the answer should be in English unless specified otherwise. However, looking at the prompt structure: "Context: ... Question: ...". The instruction is in Chinese. The context is in English. The question is in English.
    *   Decision: I will answer in the same language as the question (English) to be natural, but I can also provide a bilingual or Chinese response if the user prefers. Wait, looking at standard behavior for this type of task: If the 
instruction is in Chinese, often the expected output is in Chinese. However, the content is English. Let's look at the instruction again: "请基于上下文回答问题" (Please answer the question based on the context). It doesn't specify the 
language of the answer.
    *   Safe bet: Answer in English because the question and context are in English, but I can acknowledge the instruction language. Actually, usually, if the user asks in English ("tell me..."), I should reply in English. If I reply in Chinese, it might be better given the prompt is in Chinese. Let's check common patterns. Usually, match the language of the question/context unless instructed otherwise.
    *   Refined Decision: Since the prompt instruction is in Chinese, but the task content is English, I will provide the answer in English to match the question, but I can add a Chinese introduction if needed. Actually, standard practice for "Answer based on context" where context/question are English -> Answer in English.
    *   Wait, looking at the system instruction: "You are a helpful assistant." (English). The user prompt is mixed (Chinese instruction, English content).
    *   Let's provide the answer in English as it matches the question's language, which is the most natural interaction flow for the specific content.
    *   Content to include: Origin (China), Hobby/Interest (studying).

4.  **Final Polish:**
    *   "Based on the context provided, memorywalker is from China and he loves studying."
    *   Keep it concise and professional.
    *   Note: The text has "love studing" (grammar error). I should probably output "loves studying" for correctness, or quote "love studing". As an AI assistant, correcting obvious typos is usually preferred for natural flow unless exact quoting is required. I will phrase it naturally.

5.  **Final Output Generation:**
    *   Based on the context provided, here is what we know about memorywalker: He is from China and he loves studying.

    *(Self-Correction)*: Wait, looking at the instruction language again. The user asked me in Chinese ("你是一个专业助手..."). It is often safer to reply in the same language as the instruction (Chinese) unless the content strongly dictates otherwise. However, since the question itself is English, an English answer is also acceptable. Let's provide a bilingual or English answer. I'll stick to English because the Q&A pair is English.
    *   Actually, often when the prompt instruction is Chinese, the expectation is Chinese. Let's write the response in English but keep it simple, or write in Chinese?
    *   Let's check the typical behavior for this specific template. Usually, if the user asks "tell me something..." in English, I answer in English.
    *   Okay, I will answer in English to match the question.

    *   Draft: Based on the provided context, memorywalker is from China and he loves studying.

6.  **Final Check:** Does this hallucinate? No. Does it ignore negative constraints? No. Is it helpful? Yes.

    Let's refine the answer to be slightly more complete based on the text.
    "According to the information given, memorywalker is from China and he loves studying."

    (Wait, I should check if I need to mention the repetition. No, that's irrelevant.)

    Okay, ready to output. I will respond in English as per the question language.

    *Wait, one more consideration:* The system instruction is in Chinese ("你是一个专业助手..."). Sometimes this implies the user wants the response in Chinese. Let's provide the answer in Chinese to be safe with the instruction language, or match the context language?
    Actually, looking at similar tasks, if the question is English, answer in English.
    Okay, I'll write: "Based on the context provided, memorywalker is from China and he loves studying."

    Let's try to combine languages for maximum helpfulness? No, pick one. English is consistent with Q&A.
    Actually, looking at the prompt start "你是一个专业助手..." (You are a professional assistant...), this sets the persona in Chinese. But the content is English. I will answer in English because the question is in English.

    Wait, I'll provide the answer in English.

    *Revised Plan:* Just answer the question directly based on the text.

    Answer: Based on the context provided, memorywalker is from China and he loves studying.

    (Wait, the text says "love studing". I will correct "studing" to "studying" for professionalism, or keep it? Better to correct it implicitly.)

    Final decision: English answer.

    Wait, let's look at the instruction again. "请基于上下文回答问题" (Please answer the question based on context).
    If I answer in Chinese: "根据提供的上下文，memorywalker 来自中国，并且他热爱学习。"
    If I answer in English: "Based on the context provided, memorywalker is from China and he loves studying."
    Since the user asked in English ("tell me something..."), I will reply in English.

    Okay, finalizing.

    Wait, one detail: The prompt says "memorywalker" (lowercase). I should keep it consistent.
    Also "love studing". I'll write "loves studying" for grammar correction as an AI assistant.

    Let's go.cw
</think>
Based on the provided context, memorywalker is from China and he loves studying.
```

第二次换了一个问题，不知道为什么这个模型开始用中文回复了

```
> Is he a good guy?
用户输入: Is he a good guy?
查询到 3 条相关记录
记录ID: 9dd5b6fb-ae4e-4af3-a43f-411b25fbf0a4, 文本: memorywalker is from China and he love studing, 向量前5维: [-0.016413346, 0.078260995, -0.21805552, -0.010767964, -0.014839452]
记录ID: 858285ce-eeb4-4dce-9e56-2efa62ca47dd, 文本: memorywalker is from China and he love studing, 向量前5维: [-0.016413346, 0.078260995, -0.21805552, -0.010767964, -0.014839452]
记录ID: 4d4cdac5-6c6d-41fd-b6a2-38dbcc8ec867, 文本: memorywalker is from China and he love studing, 向量前5维: [-0.016413346, 0.078260995, -0.21805552, -0.010767964, -0.014839452]

<think>好的，我现在需要处理用户的问题。首先，上下文中的三条记录都是一样的：memorywalker来自中国，喜欢学习。问题是问他是否是个好人。我需要仔细分析这些信息，判断是否能从上下文中得出答案。

首先，用户的指令是作为专业助手，基于上下文回答问题。但这里的关键是，上下文提供的信息有限。通常，“好人”的判断可能需要更多关于性格、行为或道德方面的信息。而现有的数据只是国籍和爱好学习。学习可能暗示他重视教育，但这是否直接关联到是否是个 
好人呢？可能存在地域刻板印象的风险，比如假设来自某个地方的人就是好人，但这样可能不准确。

接下来，我需要检查是否有隐含的偏见。用户的问题可能带有测试意图，看是否会基于不完整的信息进行推断。作为专业助手，应该指出上下文信息不足以支持这样的结论，避免做出没有根据的判断。同时，要确保回答中立，不涉及性别、国籍等方面的刻板印象。   

另外，三条重复的记录可能是在强调某些信息，但即使如此，内容本身并没有说明他的品德如何。可能用户希望得到肯定回答，但正确的做法是依据现有信息，不能确定，所以需要明确说明无法判断，并建议参考更多相关信息。

还要注意，问题中的“good guy”可能存在文化差异。在中文里，“好人”通常指品德好的人，但根据什么标准呢？上下文没有提到任何关于他的行为或他人对他评价的信息。因此，直接回答是或否都是不合适的。

总结下来，正确的回答应该是基于现有信息无法确定，并指出需要更多关于个人品质和行为的资料。同时要避免任何可能的偏见，保持客观中立。这样既符合专业助手的角色，也遵循了提供准确信息的原则。
</think>

根据现有上下文信息，仅能得知 memorywalker 来自中国且热爱学习，但**“好人”的判断通常需要更全面的行为、品德或他人评价等依据**。仅凭国籍和爱好无法直接推导其人品，因此**无法确定他是否是“good guy”**。建议参考更多具体信息（如个人成就、人际关系反馈等）进行综合评估。
```