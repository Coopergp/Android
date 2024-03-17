---
Gradle构建速度优化
---

1. #### 使用最新版本的 Gradle 插件
2. #### 开启 Gradle 守护进程：守护进程可以在后台保持一个长时间运行的进程，以便在多次构建中重用已加载的类和资源
   ```
   // gradle.properties
   org.gradle.daemon=true
   ```
3. #### 使用并行构建
   ```
   // gradle.properties
   org.gradle.parallel=true
   ```
4. #### 调整内存分配：适当调整 Gradle 构建过程的内存分配
   ```
   // gradle.properties
   org.gradle.jvmargs=-Xmx4096m
   -XX:MaxPermSize=2048m
   ```
6. #### 启用增量编译：跳过不需要重新编译的部分
    ```
    android{
      ···
      buildFeatures{
      ···
      dataBinding true
      ···
      }
    }
    ```
7. #### 使用缓存
    ```
    // gradle.properties
    org.gradle.caching=true
    ```
8. #### 跳过不需要执行的Task
   命令行中使用 -x 跳过不需要的 task，例如：
   ```
   // --console=verbose 可以输出所有执行的 task
   ./gradlew build --console=verbose -x lint -x test
   ```
9. #### 合理配置ProGuard配置文件

