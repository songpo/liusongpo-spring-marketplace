---
name: spring-ai-cn
description: Spring AI 集成实践，包括 LLM 对接、RAG、向量数据库、Prompt 工程和 AI 应用开发
---

# Spring AI 集成实践指南

当你需要在应用中集成 AI 能力时使用此技能，包括对接各种 LLM、实现 RAG、使用向量数据库等。

## 何时使用

- 集成 OpenAI、Azure OpenAI、Claude 等 LLM
- 实现智能对话机器人
- 实现 RAG（检索增强生成）
- 使用向量数据库进行语义搜索
- 实现文档问答系统
- 实现 AI 辅助的内容生成
- Prompt 工程和优化

## Maven 依赖

```xml
<dependencies>
    <!-- Spring AI Core -->
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-core</artifactId>
        <version>1.0.0-M1</version>
    </dependency>
    
    <!-- OpenAI -->
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
        <version>1.0.0-M1</version>
    </dependency>
    
    <!-- Azure OpenAI（可选） -->
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-azure-openai-spring-boot-starter</artifactId>
        <version>1.0.0-M1</version>
    </dependency>
    
    <!-- Ollama（本地模型，可选） -->
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-ollama-spring-boot-starter</artifactId>
        <version>1.0.0-M1</version>
    </dependency>
    
    <!-- 向量数据库 - Chroma -->
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-chroma-store-spring-boot-starter</artifactId>
        <version>1.0.0-M1</version>
    </dependency>
    
    <!-- 向量数据库 - Pinecone（可选） -->
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-pinecone-store-spring-boot-starter</artifactId>
        <version>1.0.0-M1</version>
    </dependency>
    
    <!-- PDF 文档处理 -->
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-pdf-document-reader</artifactId>
        <version>1.0.0-M1</version>
    </dependency>
</dependencies>

<repositories>
    <repository>
        <id>spring-milestones</id>
        <name>Spring Milestones</name>
        <url>https://repo.spring.io/milestone</url>
    </repository>
</repositories>
```

## 基础配置

### 1. OpenAI 配置

```yaml
# application.yml
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}
      chat:
        options:
          model: gpt-4
          temperature: 0.7
          max-tokens: 2000
      embedding:
        options:
          model: text-embedding-ada-002
```

### 2. Azure OpenAI 配置

```yaml
spring:
  ai:
    azure:
      openai:
        api-key: ${AZURE_OPENAI_API_KEY}
        endpoint: ${AZURE_OPENAI_ENDPOINT}
        chat:
          options:
            deployment-name: gpt-4
            temperature: 0.7
```

### 3. Ollama 配置（本地模型）

```yaml
spring:
  ai:
    ollama:
      base-url: http://localhost:11434
      chat:
        options:
          model: llama2
          temperature: 0.7
```

## 基本对话实现

### 1. 简单对话

```java
package com.example.ai;

import org.springframework.ai.chat.ChatClient;
import org.springframework.ai.chat.ChatResponse;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.stereotype.Service;

@Service
public class ChatService {
    
    private final ChatClient chatClient;
    
    public ChatService(ChatClient chatClient) {
        this.chatClient = chatClient;
    }
    
    public String chat(String message) {
        ChatResponse response = chatClient.call(new Prompt(message));
        return response.getResult().getOutput().getContent();
    }
    
    public String chatWithOptions(String message) {
        ChatResponse response = chatClient.call(
            new Prompt(
                message,
                OpenAiChatOptions.builder()
                    .withModel("gpt-4")
                    .withTemperature(0.7)
                    .withMaxTokens(1000)
                    .build()
            )
        );
        return response.getResult().getOutput().getContent();
    }
}
```

### 2. 流式对话

```java
package com.example.ai;

import org.springframework.ai.chat.ChatClient;
import org.springframework.ai.chat.StreamingChatClient;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.stereotype.Service;
import reactor.core.publisher.Flux;

@Service
public class StreamingChatService {
    
    private final StreamingChatClient streamingChatClient;
    
    public StreamingChatService(StreamingChatClient streamingChatClient) {
        this.streamingChatClient = streamingChatClient;
    }
    
    public Flux<String> streamChat(String message) {
        return streamingChatClient.stream(new Prompt(message))
            .map(response -> response.getResult().getOutput().getContent());
    }
}
```

### 3. REST API

```java
package com.example.controller;

import org.springframework.http.MediaType;
import org.springframework.web.bind.annotation.*;
import reactor.core.publisher.Flux;

@RestController
@RequestMapping("/api/chat")
public class ChatController {
    
    private final ChatService chatService;
    private final StreamingChatService streamingChatService;
    
    public ChatController(
        ChatService chatService,
        StreamingChatService streamingChatService
    ) {
        this.chatService = chatService;
        this.streamingChatService = streamingChatService;
    }
    
    @PostMapping
    public ChatResponse chat(@RequestBody ChatRequest request) {
        String response = chatService.chat(request.message());
        return new ChatResponse(response);
    }
    
    @PostMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<String> streamChat(@RequestBody ChatRequest request) {
        return streamingChatService.streamChat(request.message());
    }
}

record ChatRequest(String message) {}
record ChatResponse(String response) {}
```

## Prompt 工程

### 1. Prompt Template

```java
package com.example.ai;

import org.springframework.ai.chat.ChatClient;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.ai.chat.prompt.PromptTemplate;
import org.springframework.stereotype.Service;

import java.util.Map;

@Service
public class PromptService {
    
    private final ChatClient chatClient;
    
    public PromptService(ChatClient chatClient) {
        this.chatClient = chatClient;
    }
    
    public String generateProductDescription(String productName, String features) {
        String template = """
            请为以下产品生成一段吸引人的描述：
            
            产品名称：{productName}
            主要特点：{features}
            
            要求：
            1. 突出产品优势
            2. 语言生动有趣
            3. 长度控制在 100 字以内
            """;
        
        PromptTemplate promptTemplate = new PromptTemplate(template);
        Prompt prompt = promptTemplate.create(Map.of(
            "productName", productName,
            "features", features
        ));
        
        return chatClient.call(prompt)
            .getResult()
            .getOutput()
            .getContent();
    }
    
    public String translateText(String text, String targetLanguage) {
        String template = """
            请将以下文本翻译成{targetLanguage}：
            
            {text}
            
            要求：
            1. 保持原文的语气和风格
            2. 确保翻译准确流畅
            3. 只返回翻译结果，不要添加其他说明
            """;
        
        PromptTemplate promptTemplate = new PromptTemplate(template);
        Prompt prompt = promptTemplate.create(Map.of(
            "text", text,
            "targetLanguage", targetLanguage
        ));
        
        return chatClient.call(prompt)
            .getResult()
            .getOutput()
            .getContent();
    }
}
```

### 2. System Message

```java
package com.example.ai;

import org.springframework.ai.chat.ChatClient;
import org.springframework.ai.chat.messages.Message;
import org.springframework.ai.chat.messages.SystemMessage;
import org.springframework.ai.chat.messages.UserMessage;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.stereotype.Service;

import java.util.List;

@Service
public class RoleBasedChatService {
    
    private final ChatClient chatClient;
    
    public RoleBasedChatService(ChatClient chatClient) {
        this.chatClient = chatClient;
    }
    
    public String chatAsExpert(String domain, String question) {
        SystemMessage systemMessage = new SystemMessage(
            "你是一位" + domain + "领域的专家。" +
            "请用专业、准确的语言回答问题，" +
            "并在必要时提供具体的例子和建议。"
        );
        
        UserMessage userMessage = new UserMessage(question);
        
        Prompt prompt = new Prompt(List.of(systemMessage, userMessage));
        
        return chatClient.call(prompt)
            .getResult()
            .getOutput()
            .getContent();
    }
    
    public String chatWithContext(List<String> conversationHistory, String newMessage) {
        SystemMessage systemMessage = new SystemMessage(
            "你是一个友好的助手，会记住对话历史并提供连贯的回答。"
        );
        
        List<Message> messages = new java.util.ArrayList<>();
        messages.add(systemMessage);
        
        // 添加历史对话
        for (int i = 0; i < conversationHistory.size(); i++) {
            if (i % 2 == 0) {
                messages.add(new UserMessage(conversationHistory.get(i)));
            } else {
                messages.add(new AssistantMessage(conversationHistory.get(i)));
            }
        }
        
        // 添加新消息
        messages.add(new UserMessage(newMessage));
        
        Prompt prompt = new Prompt(messages);
        
        return chatClient.call(prompt)
            .getResult()
            .getOutput()
            .getContent();
    }
}
```

## RAG（检索增强生成）实现

### 1. 文档加载和分割

```java
package com.example.ai.rag;

import org.springframework.ai.document.Document;
import org.springframework.ai.reader.ExtractedTextFormatter;
import org.springframework.ai.reader.pdf.PagePdfDocumentReader;
import org.springframework.ai.reader.pdf.config.PdfDocumentReaderConfig;
import org.springframework.ai.transformer.splitter.TokenTextSplitter;
import org.springframework.core.io.Resource;
import org.springframework.stereotype.Service;

import java.util.List;

@Service
public class DocumentLoaderService {
    
    public List<Document> loadAndSplitPdf(Resource pdfResource) {
        // 配置 PDF 读取器
        PdfDocumentReaderConfig config = PdfDocumentReaderConfig.builder()
            .withPageExtractedTextFormatter(
                ExtractedTextFormatter.builder()
                    .withNumberOfBottomTextLinesToDelete(3)
                    .withNumberOfTopPagesToSkipBeforeDelete(1)
                    .build()
            )
            .withPagesPerDocument(1)
            .build();
        
        // 读取 PDF
        PagePdfDocumentReader pdfReader = new PagePdfDocumentReader(
            pdfResource,
            config
        );
        List<Document> documents = pdfReader.get();
        
        // 分割文档
        TokenTextSplitter splitter = new TokenTextSplitter();
        return splitter.apply(documents);
    }
    
    public List<Document> loadTextDocuments(List<String> texts) {
        List<Document> documents = texts.stream()
            .map(text -> new Document(text))
            .toList();
        
        TokenTextSplitter splitter = new TokenTextSplitter(
            500,  // chunk size
            100   // overlap
        );
        
        return splitter.apply(documents);
    }
}
```

### 2. 向量存储

```java
package com.example.ai.rag;

import org.springframework.ai.document.Document;
import org.springframework.ai.embedding.EmbeddingClient;
import org.springframework.ai.vectorstore.SearchRequest;
import org.springframework.ai.vectorstore.VectorStore;
import org.springframework.stereotype.Service;

import java.util.List;

@Service
public class VectorStoreService {
    
    private final VectorStore vectorStore;
    private final EmbeddingClient embeddingClient;
    
    public VectorStoreService(
        VectorStore vectorStore,
        EmbeddingClient embeddingClient
    ) {
        this.vectorStore = vectorStore;
        this.embeddingClient = embeddingClient;
    }
    
    public void addDocuments(List<Document> documents) {
        vectorStore.add(documents);
    }
    
    public List<Document> searchSimilarDocuments(String query, int topK) {
        return vectorStore.similaritySearch(
            SearchRequest.query(query)
                .withTopK(topK)
        );
    }
    
    public List<Document> searchWithThreshold(String query, double threshold) {
        return vectorStore.similaritySearch(
            SearchRequest.query(query)
                .withTopK(5)
                .withSimilarityThreshold(threshold)
        );
    }
}
```

### 3. RAG 问答服务

```java
package com.example.ai.rag;

import org.springframework.ai.chat.ChatClient;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.ai.chat.prompt.PromptTemplate;
import org.springframework.ai.document.Document;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

@Service
public class RagService {
    
    private final ChatClient chatClient;
    private final VectorStoreService vectorStoreService;
    
    public RagService(
        ChatClient chatClient,
        VectorStoreService vectorStoreService
    ) {
        this.chatClient = chatClient;
        this.vectorStoreService = vectorStoreService;
    }
    
    public String answerQuestion(String question) {
        // 1. 检索相关文档
        List<Document> relevantDocs = vectorStoreService
            .searchSimilarDocuments(question, 3);
        
        // 2. 构建上下文
        String context = relevantDocs.stream()
            .map(Document::getContent)
            .collect(Collectors.joining("\n\n"));
        
        // 3. 构建 Prompt
        String template = """
            请基于以下上下文回答问题。如果上下文中没有相关信息，请明确说明。
            
            上下文：
            {context}
            
            问题：{question}
            
            回答：
            """;
        
        PromptTemplate promptTemplate = new PromptTemplate(template);
        Prompt prompt = promptTemplate.create(Map.of(
            "context", context,
            "question", question
        ));
        
        // 4. 调用 LLM
        return chatClient.call(prompt)
            .getResult()
            .getOutput()
            .getContent();
    }
    
    public RagResponse answerWithSources(String question) {
        // 检索相关文档
        List<Document> relevantDocs = vectorStoreService
            .searchSimilarDocuments(question, 3);
        
        String context = relevantDocs.stream()
            .map(Document::getContent)
            .collect(Collectors.joining("\n\n"));
        
        String template = """
            请基于以下上下文回答问题。
            
            上下文：
            {context}
            
            问题：{question}
            """;
        
        PromptTemplate promptTemplate = new PromptTemplate(template);
        Prompt prompt = promptTemplate.create(Map.of(
            "context", context,
            "question", question
        ));
        
        String answer = chatClient.call(prompt)
            .getResult()
            .getOutput()
            .getContent();
        
        // 返回答案和来源
        List<String> sources = relevantDocs.stream()
            .map(doc -> doc.getMetadata().get("source"))
            .map(Object::toString)
            .distinct()
            .toList();
        
        return new RagResponse(answer, sources);
    }
}

record RagResponse(String answer, List<String> sources) {}
```

### 4. 文档问答 API

```java
package com.example.controller;

import com.example.ai.rag.DocumentLoaderService;
import com.example.ai.rag.RagService;
import com.example.ai.rag.VectorStoreService;
import org.springframework.ai.document.Document;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.multipart.MultipartFile;

import java.io.IOException;
import java.util.List;

@RestController
@RequestMapping("/api/rag")
public class RagController {
    
    private final DocumentLoaderService documentLoader;
    private final VectorStoreService vectorStoreService;
    private final RagService ragService;
    
    public RagController(
        DocumentLoaderService documentLoader,
        VectorStoreService vectorStoreService,
        RagService ragService
    ) {
        this.documentLoader = documentLoader;
        this.vectorStoreService = vectorStoreService;
        this.ragService = ragService;
    }
    
    @PostMapping("/upload")
    public ResponseEntity<String> uploadDocument(@RequestParam("file") MultipartFile file) 
        throws IOException {
        
        Resource resource = file.getResource();
        List<Document> documents = documentLoader.loadAndSplitPdf(resource);
        vectorStoreService.addDocuments(documents);
        
        return ResponseEntity.ok("文档已上传并索引，共 " + documents.size() + " 个片段");
    }
    
    @PostMapping("/question")
    public ResponseEntity<RagResponse> askQuestion(@RequestBody QuestionRequest request) {
        RagResponse response = ragService.answerWithSources(request.question());
        return ResponseEntity.ok(response);
    }
}

record QuestionRequest(String question) {}
```

## 函数调用（Function Calling）

### 1. 定义函数

```java
package com.example.ai.function;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Description;

import java.util.function.Function;

@Configuration
public class FunctionConfig {
    
    @Bean
    @Description("获取指定城市的当前天气")
    public Function<WeatherRequest, WeatherResponse> getCurrentWeather() {
        return request -> {
            // 实际应用中应该调用真实的天气 API
            return new WeatherResponse(
                request.city(),
                "晴天",
                25.0,
                "°C"
            );
        };
    }
    
    @Bean
    @Description("搜索产品信息")
    public Function<ProductSearchRequest, ProductSearchResponse> searchProducts() {
        return request -> {
            // 实际应用中应该查询数据库
            List<Product> products = productRepository.search(request.keyword());
            return new ProductSearchResponse(products);
        };
    }
}

record WeatherRequest(String city) {}
record WeatherResponse(String city, String condition, double temperature, String unit) {}
record ProductSearchRequest(String keyword) {}
record ProductSearchResponse(List<Product> products) {}
```

### 2. 使用函数调用

```java
package com.example.ai.function;

import org.springframework.ai.chat.ChatClient;
import org.springframework.ai.chat.ChatResponse;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.ai.openai.OpenAiChatOptions;
import org.springframework.stereotype.Service;

import java.util.List;

@Service
public class FunctionCallingService {
    
    private final ChatClient chatClient;
    
    public FunctionCallingService(ChatClient chatClient) {
        this.chatClient = chatClient;
    }
    
    public String chatWithFunctions(String message) {
        ChatResponse response = chatClient.call(
            new Prompt(
                message,
                OpenAiChatOptions.builder()
                    .withFunctions(List.of("getCurrentWeather", "searchProducts"))
                    .build()
            )
        );
        
        return response.getResult().getOutput().getContent();
    }
}
```

## 图像生成

```java
package com.example.ai.image;

import org.springframework.ai.image.ImageClient;
import org.springframework.ai.image.ImagePrompt;
import org.springframework.ai.image.ImageResponse;
import org.springframework.ai.openai.OpenAiImageOptions;
import org.springframework.stereotype.Service;

@Service
public class ImageGenerationService {
    
    private final ImageClient imageClient;
    
    public ImageGenerationService(ImageClient imageClient) {
        this.imageClient = imageClient;
    }
    
    public String generateImage(String prompt) {
        ImageResponse response = imageClient.call(
            new ImagePrompt(
                prompt,
                OpenAiImageOptions.builder()
                    .withModel("dall-e-3")
                    .withQuality("hd")
                    .withN(1)
                    .withHeight(1024)
                    .withWidth(1024)
                    .build()
            )
        );
        
        return response.getResult().getOutput().getUrl();
    }
}
```

## 最佳实践

### 1. 错误处理

```java
package com.example.ai;

import org.springframework.ai.chat.ChatClient;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.retry.annotation.Backoff;
import org.springframework.retry.annotation.Retryable;
import org.springframework.stereotype.Service;

@Service
public class ResilientChatService {
    
    private final ChatClient chatClient;
    
    public ResilientChatService(ChatClient chatClient) {
        this.chatClient = chatClient;
    }
    
    @Retryable(
        maxAttempts = 3,
        backoff = @Backoff(delay = 1000, multiplier = 2)
    )
    public String chatWithRetry(String message) {
        try {
            return chatClient.call(new Prompt(message))
                .getResult()
                .getOutput()
                .getContent();
        } catch (Exception e) {
            throw new ChatServiceException("调用 AI 服务失败", e);
        }
    }
}
```

### 2. 缓存

```java
package com.example.ai;

import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Service;

@Service
public class CachedChatService {
    
    private final ChatService chatService;
    
    public CachedChatService(ChatService chatService) {
        this.chatService = chatService;
    }
    
    @Cacheable(value = "chatResponses", key = "#message")
    public String chat(String message) {
        return chatService.chat(message);
    }
}
```

### 3. 成本控制

```java
package com.example.ai;

import org.springframework.stereotype.Component;

import java.util.concurrent.atomic.AtomicInteger;

@Component
public class TokenUsageTracker {
    
    private final AtomicInteger totalTokens = new AtomicInteger(0);
    
    public void trackUsage(int promptTokens, int completionTokens) {
        int total = promptTokens + completionTokens;
        totalTokens.addAndGet(total);
        
        // 记录到数据库或监控系统
        logUsage(promptTokens, completionTokens, total);
    }
    
    public int getTotalTokens() {
        return totalTokens.get();
    }
    
    private void logUsage(int promptTokens, int completionTokens, int total) {
        // 实现日志记录
    }
}
```

## 配置文件

```yaml
# application.yml
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}
      base-url: https://api.openai.com
      chat:
        options:
          model: gpt-4
          temperature: 0.7
          max-tokens: 2000
      embedding:
        options:
          model: text-embedding-ada-002
    
    vectorstore:
      chroma:
        client:
          host: localhost
          port: 8000
        collection-name: documents
        initialize-schema: true

# 缓存配置
spring:
  cache:
    type: caffeine
    caffeine:
      spec: maximumSize=1000,expireAfterWrite=1h
```

## 相关技能

- `/springboot-project-planner-cn` - 规划 AI 功能需求
- `/springboot-security-cn` - 保护 AI API
- `/springboot-testing-cn` - 测试 AI 功能
