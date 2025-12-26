# Android FaceMesh 编译与运行故障排除

本文档记录了在 macOS 上编译和运行 FaceMesh Android 项目过程中遇到的问题及解决方案。

## 1. 环境配置 (SDK/NDK)

**问题：**
Bazel 无法定位 Android SDK 和 NDK。

**交互与错误日志：**
运行构建命令：
`bazel build -c opt --config=android_arm64 //mediapipe/examples/android/solutions/facemesh/src/main:facemesh`

系统返回错误：
```
ERROR: .../BUILD:19:15: While resolving toolchains for target //mediapipe/examples/android/solutions/facemesh/src/main:facemesh (cf63efd): No matching toolchains found for types @@bazel_tools//tools/android:sdk_toolchain_type.
To debug, rerun with --toolchain_resolution_debug='@@bazel_tools//tools/android:sdk_toolchain_type'
If platforms or toolchains are a new concept for you, we'd encourage reading https://bazel.build/concepts/platforms-intro.
ERROR: Analysis of target '//mediapipe/examples/android/solutions/facemesh/src/main:facemesh' failed; build aborted
```

**原因分析：**
Bazel 需要在 `WORKSPACE` 文件中明确指定 Android SDK 和 NDK 的路径才能找到对应的构建工具链。

**解决方案：**
更新项目根目录下的 `WORKSPACE` 文件，指向你本地的 Android SDK 和 NDK 路径。

```python
# WORKSPACE
android_sdk_repository(
    name = "androidsdk",
    path = "/Users/your_user/Library/Android/sdk", # 请根据实际情况调整路径
)

android_ndk_repository(
    name = "androidndk",
    path = "/Users/your_user/Library/Android/sdk/ndk/<version>", # 请根据实际情况调整版本
    api_level = 21,
)

bind(
    name = "android/crosstool",
    actual = "@androidndk//:toolchain",
)
```

## 2. zlib 编译错误 (macOS)

**问题：**
在 macOS 上编译时，`zlib` 因 `fdopen` 宏重定义而编译失败。

**错误日志：**
```
external/zlib/zutil.h:147:33: note: expanded from macro 'fdopen'
#        define fdopen(fd,mode) NULL /* No fdopen() */
...
error: expected identifier or '('
```

**解决方案：**
修改 `third_party/zlib.BUILD` 文件，在编译器选项中添加 `-Dfdopen=fdopen`。

```python
# third_party/zlib.BUILD
copts = select({
    "//conditions:default": [
        "-Wno-dangling-else",
        "-Wno-format",
        "-Wno-implicit-function-declaration",
        "-Wno-incompatible-pointer-types",
        "-Wno-incompatible-pointer-types-discards-qualifiers",
        "-Wno-parentheses",
        "-DIOAPI_NO_64",
        "-Dfdopen=fdopen",  # 添加这一行
    ],
}),
```

## 3. 运行时崩溃：缺少 libc++_shared.so

**问题：**
启动"MediaPipe FaceMesh"应用后，点击"START CAMERA"时立即崩溃，报错 `UnsatisfiedLinkError`。

**错误日志：**
```
java.lang.UnsatisfiedLinkError: dlopen failed: library "libc++_shared.so" not found: needed by ... libopencv_java4.so
```

**解决方案：**
OpenCV 库依赖 `libc++_shared.so`，但该库未被打包进 APK。

> **补充说明：OpenCV 的下载与配置**
> OpenCV 是在运行 `bazel build` 命令时，由 Bazel 根据项目根目录 `WORKSPACE` 文件中的定义自动下载和配置的。
> 
> 1.  **定义**：`WORKSPACE` 文件中使用 `http_archive` 规则定义了名为 `android_opencv` 的外部仓库，指向 OpenCV Android SDK 的下载链接（例如 v4.12.0 zip 包）。
> 2.  **配置**：Bazel 下载并解压后，使用 `third_party/opencv_android.BUILD` 文件来描述其构建规则。
> 3.  **引用**：`third_party/BUILD` 文件中的 `opencv` 目标会根据构建平台（如 `android_arm64`）选择对应的 OpenCV 库。
>
> 这里的崩溃问题是因为 OpenCV 的预编译库依赖于 `libc++_shared.so`，而 Bazel 默认没有将其包含在 APK 中。

1.  **复制库文件：**
    从你的 NDK 目录（例如 `sysroot/usr/lib/<arch>/libc++_shared.so`）复制 `libc++_shared.so` 到 `mediapipe/third_party/` 目录。

2.  **更新 `third_party/BUILD`：**
    将 `libc++_shared` 的定义从 `cc_binary` 改为 `cc_import`。

    ```python
    # third_party/BUILD
    cc_import(
        name = "libc++_shared",
        shared_library = "libc++_shared.so",
    )

    cc_library(
        name = "libc++_shared_lib",
        deps = [":libc++_shared"],
        alwayslink = 1,
    )
    ```

3.  **添加应用依赖：**
    在 Android 二进制目标的 `deps` 中添加 `//third_party:libc++_shared_lib`。

    ```python
    # mediapipe/examples/android/solutions/facemesh/src/main/BUILD
    android_binary(
        name = "facemesh",
        deps = [
            # ... 其他依赖 ...
            "//third_party:libc++_shared_lib",
        ],
    )
    ```
