---
title: debug libuv
category: C/C++
keywords: debug,libuv,C/C++
description: 学习 libuv 或者其它 C/C++相关的技术，感觉又回到了刚开始学习编程的时候，很多的不懂和挑战，不再像用 JavaScript 那样随心所欲，但是越是对底层的学习，越是能了解计算机原理，职业寿命才能变得更长。出于兴趣也好，出于无奈也好，总之新的学习让一切又变得有意思起来。
tags:
  - Node.js
  - libuv
date: 2020-09-28 18:31:53
---

libuv 在 v1.36.0 之后移除了 `gyp_uv.py` ([commit](https://github.com/libuv/libuv/commit/53f3c687fc288708721a5a3d9563febda1b9d2c1))，没办法通过它去创建一个 libuv.a 静态链接库（[v1.35 文档](https://github.com/libuv/libuv/tree/v1.35.0)有详细的介绍），现在我们需要通过`cmake`去创建。

### 构建静态链接库

```shelll
// 下载 libuv
git clone https://github.com/libuv/libuv
cd libuv
mkdir -p build
// DCMAKE_BUILD_TYPE 将其设置为 Debug 模式，不然断点没办法进入 libuv 源码中
 (cd build && cmake -DCMAKE_BUILD_TYPE=Debug ..)
cmake --build build
```

之后 build 目录 如下，libuv_a.a 就是我们需要的 静态链接库。
![build dir](/images/debug-libuv/build.jpg)

### 新建 hello world 项目

通过 CLion 创建一个`helloworld`项目，在 CMakeLists.txt 里添加 libuv 相关的信息，将 libuv 的头文件和源码添加进来，最后把项目和 linuv 链接在一起

```CMakeLists.txt
cmake_minimum_required(VERSION 3.17)
project(helloworld)
add_executable(helloworld main.cpp)

# 前面 clone libuv 绝对路径
set(LIBUVDIR /your/libuv/path)
# 将源码导入
include_directories(${LIBUVDIR}/src)
include_directories(${LIBUVDIR}/include)

add_library(libuv STATIC IMPORTED)
set_target_properties(libuv
        PROPERTIES IMPORTED_LOCATION
        ${LIBUVDIR}/build/libuv_a.a)

# 链接起来
target_link_libraries(helloworld libuv)
```

创建 main.cpp 文件

```cpp
#include <stdio.h>
#include <uv.h>

static void cb(uv_write_t *req, int status) {
  printf("Hello from Callback.\n");
}

int main() {
  uv_tty_t tty;
  uv_write_t req;
  uv_tty_init(uv_default_loop(), &tty, 1, 0);
  char str[] = "Hello UV!\n";
  int len = strlen(str);
  uv_buf_t bufs[] = {uv_buf_init(str, len)};
  uv_write(&req, (uv_stream_t *) &tty, bufs, 1, cb);
  uv_run(uv_default_loop(), UV_RUN_DEFAULT);
  return 0;
}
```

之后就能愉快的打断点调试 libuv 了
![debug libuv](/images/debug-libuv/debug.jpg)

学习 libuv 或者其它 C/C++相关的技术，感觉又回到了刚开始学习编程的时候，很多的不懂和挑战，不再像用 JavaScript 那样随心所欲，但是越是对底层的学习，越是能了解计算机原理，职业寿命才能变得更长。出于兴趣也好，出于无奈也好，总之新的学习让一切又变得有意思起来。
