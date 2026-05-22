---
name: spring-batch-cn
description: Spring Batch 批处理实践，包括 Job、Step、ItemReader/Writer/Processor、分块处理和任务调度
---

# Spring Batch 批处理实践指南

当你需要在 Spring Boot 应用中实现批处理任务时使用此技能，包括大数据量处理、定时任务、ETL 流程和数据迁移。

## 何时使用

- 大批量数据处理
- 定时批处理任务
- ETL（提取、转换、加载）流程
- 数据迁移和同步
- 报表生成
- 文件导入导出
- 数据清洗和转换

## Maven 依赖

```xml
<dependencies>
    <!-- Spring Batch -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-batch</artifactId>
    </dependency>
    
    <!-- 数据库支持（Job Repository） -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-jdbc</artifactId>
    </dependency>
    
    <!-- H2（开发测试） -->
    <dependency>
        <groupId>com.h2database</groupId>
        <artifactId>h2</artifactId>
        <scope>runtime</scope>
    </dependency>
    
    <!-- PostgreSQL（生产环境） -->
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
        <scope>runtime</scope>
    </dependency>
</dependencies>
```

## 1. 基础配置

### 1.1 启用 Batch

```java
package com.example.config;

import org.springframework.batch.core.configuration.annotation.EnableBatchProcessing;
import org.springframework.context.annotation.Configuration;

@Configuration
@EnableBatchProcessing
public class BatchConfig {
}
```

### 1.2 数据库配置

```yaml
# application.yml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/batchdb
    username: postgres
    password: ${DB_PASSWORD}
  batch:
    jdbc:
      initialize-schema: always  # 自动创建 Batch 表
    job:
      enabled: false  # 禁止启动时自动执行 Job
```

## 2. 简单 Job 示例

### 2.1 Tasklet 方式

```java
package com.example.job;

import org.springframework.batch.core.Job;
import org.springframework.batch.core.Step;
import org.springframework.batch.core.job.builder.JobBuilder;
import org.springframework.batch.core.repository.JobRepository;
import org.springframework.batch.core.step.builder.StepBuilder;
import org.springframework.batch.core.step.tasklet.Tasklet;
import org.springframework.batch.repeat.RepeatStatus;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.transaction.PlatformTransactionManager;

@Configuration
public class SimpleJobConfig {
    
    @Bean
    public Job simpleJob(JobRepository jobRepository, Step simpleStep) {
        return new JobBuilder("simpleJob", jobRepository)
            .start(simpleStep)
            .build();
    }
    
    @Bean
    public Step simpleStep(
        JobRepository jobRepository,
        PlatformTransactionManager transactionManager
    ) {
        return new StepBuilder("simpleStep", jobRepository)
            .tasklet(simpleTasklet(), transactionManager)
            .build();
    }
    
    @Bean
    public Tasklet simpleTasklet() {
        return (contribution, chunkContext) -> {
            System.out.println("执行简单任务");
            // 执行业务逻辑
            return RepeatStatus.FINISHED;
        };
    }
}
```

## 3. Chunk 处理（分块处理）

### 3.1 基本 Chunk 配置

```java
package com.example.job;

import com.example.model.User;
import org.springframework.batch.core.Job;
import org.springframework.batch.core.Step;
import org.springframework.batch.core.job.builder.JobBuilder;
import org.springframework.batch.core.repository.JobRepository;
import org.springframework.batch.core.step.builder.StepBuilder;
import org.springframework.batch.item.ItemProcessor;
import org.springframework.batch.item.ItemReader;
import org.springframework.batch.item.ItemWriter;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.transaction.PlatformTransactionManager;

@Configuration
public class ChunkJobConfig {
    
    @Bean
    public Job chunkJob(JobRepository jobRepository, Step chunkStep) {
        return new JobBuilder("chunkJob", jobRepository)
            .start(chunkStep)
            .build();
    }
    
    @Bean
    public Step chunkStep(
        JobRepository jobRepository,
        PlatformTransactionManager transactionManager,
        ItemReader<User> reader,
        ItemProcessor<User, User> processor,
        ItemWriter<User> writer
    ) {
        return new StepBuilder("chunkStep", jobRepository)
            .<User, User>chunk(100, transactionManager)  // 每 100 条提交一次
            .reader(reader)
            .processor(processor)
            .writer(writer)
            .build();
    }
}
```

### 3.2 ItemReader 实现

```java
package com.example.reader;

import com.example.model.User;
import org.springframework.batch.item.database.JdbcCursorItemReader;
import org.springframework.batch.item.database.builder.JdbcCursorItemReaderBuilder;
import org.springframework.batch.item.file.FlatFileItemReader;
import org.springframework.batch.item.file.builder.FlatFileItemReaderBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.io.ClassPathResource;
import org.springframework.jdbc.core.BeanPropertyRowMapper;

import javax.sql.DataSource;

@Configuration
public class ReaderConfig {
    
    // 从数据库读取
    @Bean
    public JdbcCursorItemReader<User> databaseReader(DataSource dataSource) {
        return new JdbcCursorItemReaderBuilder<User>()
            .name("databaseReader")
            .dataSource(dataSource)
            .sql("SELECT id, name, email, status FROM users WHERE status = 'ACTIVE'")
            .rowMapper(new BeanPropertyRowMapper<>(User.class))
            .build();
    }
    
    // 从 CSV 文件读取
    @Bean
    public FlatFileItemReader<User> csvReader() {
        return new FlatFileItemReaderBuilder<User>()
            .name("csvReader")
            .resource(new ClassPathResource("users.csv"))
            .delimited()
            .names("id", "name", "email", "status")
            .targetType(User.class)
            .linesToSkip(1)  // 跳过标题行
            .build();
    }
}
```

### 3.3 ItemProcessor 实现

```java
package com.example.processor;

import com.example.model.User;
import org.springframework.batch.item.ItemProcessor;
import org.springframework.stereotype.Component;

@Component
public class UserProcessor implements ItemProcessor<User, User> {
    
    @Override
    public User process(User user) throws Exception {
        // 数据转换
        user.setName(user.getName().toUpperCase());
        
        // 数据验证
        if (user.getEmail() == null || !user.getEmail().contains("@")) {
            return null;  // 返回 null 表示跳过该记录
        }
        
        // 数据增强
        user.setProcessedAt(java.time.LocalDateTime.now());
        
        return user;
    }
}
```

### 3.4 ItemWriter 实现

```java
package com.example.writer;

import com.example.model.User;
import org.springframework.batch.item.database.JdbcBatchItemWriter;
import org.springframework.batch.item.database.builder.JdbcBatchItemWriterBuilder;
import org.springframework.batch.item.file.FlatFileItemWriter;
import org.springframework.batch.item.file.builder.FlatFileItemWriterBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.io.FileSystemResource;

import javax.sql.DataSource;

@Configuration
public class WriterConfig {
    
    // 写入数据库
    @Bean
    public JdbcBatchItemWriter<User> databaseWriter(DataSource dataSource) {
        return new JdbcBatchItemWriterBuilder<User>()
            .dataSource(dataSource)
            .sql("""
                INSERT INTO processed_users (id, name, email, status, processed_at)
                VALUES (:id, :name, :email, :status, :processedAt)
                ON CONFLICT (id) DO UPDATE SET
                    name = EXCLUDED.name,
                    email = EXCLUDED.email,
                    status = EXCLUDED.status,
                    processed_at = EXCLUDED.processed_at
                """)
            .beanMapped()
            .build();
    }
    
    // 写入 CSV 文件
    @Bean
    public FlatFileItemWriter<User> csvWriter() {
        return new FlatFileItemWriterBuilder<User>()
            .name("csvWriter")
            .resource(new FileSystemResource("output/processed-users.csv"))
            .delimited()
            .names("id", "name", "email", "status", "processedAt")
            .headerCallback(writer -> writer.write("ID,Name,Email,Status,ProcessedAt"))
            .build();
    }
}
```

## 4. 多步骤 Job

```java
package com.example.job;

import org.springframework.batch.core.Job;
import org.springframework.batch.core.Step;
import org.springframework.batch.core.job.builder.JobBuilder;
import org.springframework.batch.core.repository.JobRepository;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class MultiStepJobConfig {
    
    @Bean
    public Job multiStepJob(
        JobRepository jobRepository,
        Step extractStep,
        Step transformStep,
        Step loadStep
    ) {
        return new JobBuilder("multiStepJob", jobRepository)
            .start(extractStep)
            .next(transformStep)
            .next(loadStep)
            .build();
    }
}
```

## 5. 条件流程

```java
package com.example.job;

import org.springframework.batch.core.Job;
import org.springframework.batch.core.Step;
import org.springframework.batch.core.job.builder.JobBuilder;
import org.springframework.batch.core.repository.JobRepository;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class ConditionalJobConfig {
    
    @Bean
    public Job conditionalJob(
        JobRepository jobRepository,
        Step validateStep,
        Step processStep,
        Step errorStep
    ) {
        return new JobBuilder("conditionalJob", jobRepository)
            .start(validateStep)
            .on("COMPLETED").to(processStep)
            .from(validateStep).on("FAILED").to(errorStep)
            .end()
            .build();
    }
}
```

## 6. Job 参数

```java
package com.example.job;

import org.springframework.batch.core.Job;
import org.springframework.batch.core.Step;
import org.springframework.batch.core.configuration.annotation.StepScope;
import org.springframework.batch.core.job.builder.JobBuilder;
import org.springframework.batch.core.repository.JobRepository;
import org.springframework.batch.core.step.builder.StepBuilder;
import org.springframework.batch.core.step.tasklet.Tasklet;
import org.springframework.batch.repeat.RepeatStatus;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.transaction.PlatformTransactionManager;

@Configuration
public class ParameterizedJobConfig {
    
    @Bean
    public Job parameterizedJob(JobRepository jobRepository, Step paramStep) {
        return new JobBuilder("parameterizedJob", jobRepository)
            .start(paramStep)
            .build();
    }
    
    @Bean
    public Step paramStep(
        JobRepository jobRepository,
        PlatformTransactionManager transactionManager,
        Tasklet paramTasklet
    ) {
        return new StepBuilder("paramStep", jobRepository)
            .tasklet(paramTasklet, transactionManager)
            .build();
    }
    
    @Bean
    @StepScope
    public Tasklet paramTasklet(
        @Value("#{jobParameters['inputFile']}") String inputFile,
        @Value("#{jobParameters['date']}") String date
    ) {
        return (contribution, chunkContext) -> {
            System.out.println("处理文件: " + inputFile);
            System.out.println("日期: " + date);
            return RepeatStatus.FINISHED;
        };
    }
}
```

## 7. Job 启动

```java
package com.example.launcher;

import org.springframework.batch.core.Job;
import org.springframework.batch.core.JobParameters;
import org.springframework.batch.core.JobParametersBuilder;
import org.springframework.batch.core.launch.JobLauncher;
import org.springframework.stereotype.Service;

@Service
public class JobLauncherService {
    
    private final JobLauncher jobLauncher;
    private final Job chunkJob;
    
    public JobLauncherService(JobLauncher jobLauncher, Job chunkJob) {
        this.jobLauncher = jobLauncher;
        this.chunkJob = chunkJob;
    }
    
    public void runJob(String inputFile) throws Exception {
        JobParameters params = new JobParametersBuilder()
            .addString("inputFile", inputFile)
            .addLong("timestamp", System.currentTimeMillis())
            .toJobParameters();
        
        jobLauncher.run(chunkJob, params);
    }
}
```

## 8. 定时调度

```java
package com.example.scheduler;

import org.springframework.batch.core.Job;
import org.springframework.batch.core.JobParameters;
import org.springframework.batch.core.JobParametersBuilder;
import org.springframework.batch.core.launch.JobLauncher;
import org.springframework.scheduling.annotation.EnableScheduling;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;

@Component
@EnableScheduling
public class BatchScheduler {
    
    private final JobLauncher jobLauncher;
    private final Job dailyJob;
    
    public BatchScheduler(JobLauncher jobLauncher, Job dailyJob) {
        this.jobLauncher = jobLauncher;
        this.dailyJob = dailyJob;
    }
    
    // 每天凌晨 2 点执行
    @Scheduled(cron = "0 0 2 * * ?")
    public void runDailyJob() throws Exception {
        JobParameters params = new JobParametersBuilder()
            .addLong("timestamp", System.currentTimeMillis())
            .toJobParameters();
        
        jobLauncher.run(dailyJob, params);
    }
}
```

## 9. 监听器

```java
package com.example.listener;

import org.springframework.batch.core.JobExecution;
import org.springframework.batch.core.JobExecutionListener;
import org.springframework.stereotype.Component;

@Component
public class JobCompletionListener implements JobExecutionListener {
    
    @Override
    public void beforeJob(JobExecution jobExecution) {
        System.out.println("Job 开始: " + jobExecution.getJobInstance().getJobName());
    }
    
    @Override
    public void afterJob(JobExecution jobExecution) {
        if (jobExecution.getStatus().isUnsuccessful()) {
            System.err.println("Job 失败: " + jobExecution.getAllFailureExceptions());
        } else {
            System.out.println("Job 完成: " + jobExecution.getStatus());
        }
    }
}
```

```java
package com.example.listener;

import com.example.model.User;
import org.springframework.batch.core.ItemReadListener;
import org.springframework.stereotype.Component;

@Component
public class ReadListener implements ItemReadListener<User> {
    
    @Override
    public void beforeRead() {
        // 读取前
    }
    
    @Override
    public void afterRead(User item) {
        System.out.println("读取: " + item);
    }
    
    @Override
    public void onReadError(Exception ex) {
        System.err.println("读取错误: " + ex.getMessage());
    }
}
```

## 10. 错误处理

### 10.1 跳过策略

```java
package com.example.config;

import org.springframework.batch.core.Step;
import org.springframework.batch.core.repository.JobRepository;
import org.springframework.batch.core.step.builder.StepBuilder;
import org.springframework.batch.item.ItemProcessor;
import org.springframework.batch.item.ItemReader;
import org.springframework.batch.item.ItemWriter;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.transaction.PlatformTransactionManager;

@Configuration
public class SkipConfig {
    
    @Bean
    public Step skipStep(
        JobRepository jobRepository,
        PlatformTransactionManager transactionManager,
        ItemReader reader,
        ItemProcessor processor,
        ItemWriter writer
    ) {
        return new StepBuilder("skipStep", jobRepository)
            .chunk(100, transactionManager)
            .reader(reader)
            .processor(processor)
            .writer(writer)
            .faultTolerant()
            .skip(Exception.class)
            .skipLimit(10)  // 最多跳过 10 条
            .build();
    }
}
```

### 10.2 重试策略

```java
package com.example.config;

import org.springframework.batch.core.Step;
import org.springframework.batch.core.repository.JobRepository;
import org.springframework.batch.core.step.builder.StepBuilder;
import org.springframework.batch.item.ItemProcessor;
import org.springframework.batch.item.ItemReader;
import org.springframework.batch.item.ItemWriter;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.transaction.PlatformTransactionManager;

@Configuration
public class RetryConfig {
    
    @Bean
    public Step retryStep(
        JobRepository jobRepository,
        PlatformTransactionManager transactionManager,
        ItemReader reader,
        ItemProcessor processor,
        ItemWriter writer
    ) {
        return new StepBuilder("retryStep", jobRepository)
            .chunk(100, transactionManager)
            .reader(reader)
            .processor(processor)
            .writer(writer)
            .faultTolerant()
            .retry(Exception.class)
            .retryLimit(3)  // 最多重试 3 次
            .build();
    }
}
```

## 11. 并行处理

### 11.1 多线程 Step

```java
package com.example.config;

import org.springframework.batch.core.Step;
import org.springframework.batch.core.repository.JobRepository;
import org.springframework.batch.core.step.builder.StepBuilder;
import org.springframework.batch.item.ItemProcessor;
import org.springframework.batch.item.ItemReader;
import org.springframework.batch.item.ItemWriter;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.task.SimpleAsyncTaskExecutor;
import org.springframework.transaction.PlatformTransactionManager;

@Configuration
public class ParallelConfig {
    
    @Bean
    public Step parallelStep(
        JobRepository jobRepository,
        PlatformTransactionManager transactionManager,
        ItemReader reader,
        ItemProcessor processor,
        ItemWriter writer
    ) {
        return new StepBuilder("parallelStep", jobRepository)
            .chunk(100, transactionManager)
            .reader(reader)
            .processor(processor)
            .writer(writer)
            .taskExecutor(new SimpleAsyncTaskExecutor())
            .throttleLimit(10)  // 最多 10 个线程
            .build();
    }
}
```

### 11.2 分区 Step

```java
package com.example.config;

import org.springframework.batch.core.Step;
import org.springframework.batch.core.partition.support.Partitioner;
import org.springframework.batch.core.repository.JobRepository;
import org.springframework.batch.core.step.builder.StepBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.task.SimpleAsyncTaskExecutor;
import org.springframework.transaction.PlatformTransactionManager;

@Configuration
public class PartitionConfig {
    
    @Bean
    public Step partitionStep(
        JobRepository jobRepository,
        PlatformTransactionManager transactionManager,
        Partitioner partitioner,
        Step workerStep
    ) {
        return new StepBuilder("partitionStep", jobRepository)
            .partitioner("workerStep", partitioner)
            .step(workerStep)
            .gridSize(10)  // 10 个分区
            .taskExecutor(new SimpleAsyncTaskExecutor())
            .build();
    }
}
```

## 12. REST API 控制

```java
package com.example.controller;

import org.springframework.batch.core.Job;
import org.springframework.batch.core.JobParameters;
import org.springframework.batch.core.JobParametersBuilder;
import org.springframework.batch.core.launch.JobLauncher;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/batch")
public class BatchController {
    
    private final JobLauncher jobLauncher;
    private final Job chunkJob;
    
    public BatchController(JobLauncher jobLauncher, Job chunkJob) {
        this.jobLauncher = jobLauncher;
        this.chunkJob = chunkJob;
    }
    
    @PostMapping("/start")
    public String startJob(@RequestParam String inputFile) throws Exception {
        JobParameters params = new JobParametersBuilder()
            .addString("inputFile", inputFile)
            .addLong("timestamp", System.currentTimeMillis())
            .toJobParameters();
        
        jobLauncher.run(chunkJob, params);
        
        return "Job started successfully";
    }
}
```

## 配置文件

```yaml
# application.yml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/batchdb
    username: postgres
    password: ${DB_PASSWORD}
  batch:
    jdbc:
      initialize-schema: always
    job:
      enabled: false
  task:
    execution:
      pool:
        core-size: 10
        max-size: 20
```

## 相关技能

- `/springboot-project-planner-cn` - 规划批处理需求
- `/springboot-testing-cn` - 测试批处理任务
- `/springboot-observability-cn` - 监控批处理性能
