# Android Solutions Gradle 构建故障排除

本文档记录了在 `mediapipe/examples/android/solutions` 目录下使用 Gradle 构建项目时遇到的问题、解决步骤以及最终的成功构建流程。

## 1. 初始构建尝试与 Gradle 版本冲突

**操作：**
尝试使用现有的 `gradlew` 脚本进行构建：
```bash
./gradlew assembleDebug
```

**问题 1：**
Gradle 版本不兼容。项目配置使用了较旧的 Gradle 插件，但 `gradle-wrapper.properties` 指向了 Gradle 8.14.3，导致构建失败。

**错误日志：**
```
Task failed with an exception.
...
Cannot use @TaskAction annotation on method IncrementalTask.taskAction$gradle_core() ...
```
这是典型的 Gradle 版本与 Android Gradle Plugin (AGP) 版本不匹配导致的错误。

**尝试修复 (失败)：**
尝试将 Gradle Wrapper 降级到 6.7.1 以匹配旧版 AGP (4.2.0)。
虽然解决了版本不匹配，但在 Java 17 环境下运行旧版 Gradle 导致了严重的类加载错误 (`Could not create service of type ScriptPluginFactory`)，因为旧版 Gradle 不支持 Java 17。

## 2. 升级 Gradle 与 Android Gradle Plugin (AGP)

**决策：**
为了适配当前的开发环境 (Java 17) 和构建工具，决定全面升级项目的构建配置。

**操作步骤：**

1.  **恢复 Gradle Wrapper**: 将 `gradle-wrapper.properties` 恢复为使用 **Gradle 8.14.3**。
    ```properties
    distributionUrl=https\://services.gradle.org/distributions/gradle-8.14.3-bin.zip
    ```

2.  **升级 AGP**: 修改根目录 [`build.gradle`](file:///Volumes/exssd/mediapipe/mediapipe/examples/android/solutions/build.gradle)，将 Android Gradle Plugin 从 `4.2.0` 升级到 **`8.2.0`**。
    ```groovy
    dependencies {
        classpath "com.android.tools.build:gradle:8.2.0"
        // ...
    }
    ```

3.  **升级 Compile SDK**: 修改所有子模块 (`facemesh`, `facedetection`, `hands`) 的 `build.gradle`，将 `compileSdkVersion` 从 `30` 升级到 **`34`**，以满足 AGP 8.x 的要求。

## 3. 适配 AGP 8.0+ 的 Namespace 变更

**问题 2：**
再次构建时报错，提示 `AndroidManifest.xml` 中的 `package` 属性不再被支持用于设置命名空间。

**错误日志：**
```
Execution failed for task ':facedetection:processDebugMainManifest'.
> Incorrect package="com.google.mediapipe.examples.facedetection" found in source AndroidManifest.xml
  Setting the namespace via the package attribute in the source AndroidManifest.xml is no longer supported.
```

**解决方案：**
将包名配置从 `AndroidManifest.xml` 迁移到 `build.gradle` 的 `namespace` 属性中。

1.  **修改 build.gradle**:
    在每个模块的 `android` 块中添加 `namespace`：
    *   `facemesh/build.gradle`: `namespace 'com.google.mediapipe.examples.facemesh'`
    *   `facedetection/build.gradle`: `namespace 'com.google.mediapipe.examples.facedetection'`
    *   `hands/build.gradle`: `namespace 'com.google.mediapipe.examples.hands'`

2.  **修改 AndroidManifest.xml**:
    移除 `<manifest>` 标签中的 `package` 属性。
    ```xml
    <manifest xmlns:android="http://schemas.android.com/apk/res/android">
        <!-- package 属性已移除 -->
        ...
    </manifest>
    ```

## 4. 最终构建成功

**操作：**
执行清理并构建命令：
```bash
./gradlew clean && ./gradlew assembleDebug
```

**结果：**
构建成功 (`BUILD SUCCESSFUL`)。

**产物位置：**
*   **Face Mesh**: `facemesh/build/outputs/apk/debug/facemesh-debug.apk`
*   **Face Detection**: `facedetection/build/outputs/apk/debug/facedetection-debug.apk`
*   **Hands**: `hands/build/outputs/apk/debug/hands-debug.apk`

## 5. 总结

为了在现代环境 (Java 17, Gradle 8+) 中编译该项目，必须进行以下现代化改造：
1.  **Gradle**: 8.14.3
2.  **AGP**: 8.2.0
3.  **Compile SDK**: 34
4.  **Namespace**: 迁移至 `build.gradle` 配置
