# UOS Python 静态构建实现计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 创建 GitHub Actions workflow，在 Debian 10 容器中构建完全静态链接的 Python 二进制包和 virtualenv wheel 包。

**Architecture:** 使用 GitHub Actions 的容器运行功能，在 debian:10 容器中从源码编译所有依赖库的静态版本，然后使用 `MODULE_BUILDTYPE=static` 配置 Python 进行静态构建，最后打包为 tar.gz 和 wheel 格式。

**Tech Stack:** GitHub Actions, Docker (debian:10), Python 3.14, 静态链接构建工具链

---

## 文件结构

| 文件 | 作用 |
|------|------|
| `.github/workflows/build-uos.yml` | 主 workflow 文件，定义构建流程 |
| `docs/superpowers/specs/2026-05-13-uos-python-build-design.md` | 设计文档（已创建） |

---

### Task 1: 创建 workflow 文件基础结构

**Files:**
- Create: `.github/workflows/build-uos.yml`

- [ ] **Step 1: 创建 workflow 文件头部和触发配置**

创建 `.github/workflows/build-uos.yml` 文件，包含：

```yaml
name: Build Python for UOS

on:
  push:
    branches:
      - main
  workflow_dispatch:
    inputs:
      tag:
        description: 'Python version tag to build (e.g., v3.14.5)'
        required: false
        type: string
        default: ''

env:
  FORCE_COLOR: 1

jobs:
  build-python:
    name: Build static Python for UOS
    runs-on: ubuntu-latest
    container:
      image: debian:10
    timeout-minutes: 120
```

- [ ] **Step 2: 验证文件创建成功**

Run: `cat .github/workflows/build-uos.yml`
Expected: 显示文件内容，包含正确的 YAML 结构

---

### Task 2: 添加 checkout 和环境准备步骤

**Files:**
- Modify: `.github/workflows/build-uos.yml`

- [ ] **Step 1: 添加 checkout 步骤**

在 `build-python` job 中添加 steps 部分：

```yaml
    steps:
      - name: Install git for checkout
        run: |
          apt-get update
          apt-get install -y git ca-certificates
          git config --global --add safe.directory "$GITHUB_WORKSPACE"

      - name: Checkout Python source
        uses: actions/checkout@v4
        with:
          ref: ${{ inputs.tag || github.ref }}
          fetch-depth: 0
```

- [ ] **Step 2: 添加构建工具安装步骤**

```yaml
      - name: Install build tools
        run: |
          apt-get install -y \
            build-essential \
            gcc \
            g++ \
            make \
            cmake \
            autoconf \
            automake \
            libtool \
            pkg-config \
            wget \
            curl \
            unzip \
            python3 \
            python3-pip \
            patchelf
```

- [ ] **Step 3: 验证 YAML 语法**

Run: `python3 -c "import yaml; yaml.safe_load(open('.github/workflows/build-uos.yml'))"`
Expected: 无错误输出，YAML 解析成功

---

### Task 3: 添加静态库编译步骤

**Files:**
- Modify: `.github/workflows/build-uos.yml`

- [ ] **Step 1: 创建静态库编译目录**

```yaml
      - name: Create static library build directory
        run: |
          mkdir -p /usr/local/static-libs
          echo "STATIC_LIB_DIR=/usr/local/static-libs" >> $GITHUB_ENV
```

- [ ] **Step 2: 添加 zlib 静态编译**

```yaml
      - name: Build zlib static
        run: |
          cd /tmp
          wget -q https://zlib.net/zlib-1.2.13.tar.gz
          tar xzf zlib-1.2.13.tar.gz
          cd zlib-1.2.13
          ./configure --prefix=/usr/local/static-libs/zlib --static
          make -j$(nproc)
          make install
```

- [ ] **Step 3: 添加 bzip2 静态编译**

```yaml
      - name: Build bzip2 static
        run: |
          cd /tmp
          wget -q https://sourceware.org/pub/bzip2/bzip2-1.0.8.tar.gz
          tar xzf bzip2-1.0.8.tar.gz
          cd bzip2-1.0.8
          make -j$(nproc) CFLAGS="-fPIC"
          install -d /usr/local/static-libs/bzip2/{lib,include}
          install libbz2.a /usr/local/static-libs/bzip2/lib/
          install bzlib.h /usr/local/static-libs/bzip2/include/
```

- [ ] **Step 4: 添加 xz (lzma) 静态编译**

```yaml
      - name: Build xz (lzma) static
        run: |
          cd /tmp
          wget -q https://tukaani.org/xz/xz-5.4.1.tar.gz
          tar xzf xz-5.4.1.tar.gz
          cd xz-5.4.1
          ./configure --prefix=/usr/local/static-libs/xz \
            --disable-shared --enable-static --with-pic
          make -j$(nproc)
          make install
```

- [ ] **Step 5: 添加 OpenSSL 静态编译**

```yaml
      - name: Build OpenSSL static
        run: |
          cd /tmp
          wget -q https://www.openssl.org/source/openssl-1.1.1w.tar.gz
          tar xzf openssl-1.1.1w.tar.gz
          cd openssl-1.1.1w
          ./Configure no-shared no-dso no-engine \
            linux-x86_64 \
            -fPIC \
            --prefix=/usr/local/static-libs/openssl \
            --openssldir=/usr/local/static-libs/openssl
          make -j$(nproc)
          make install_sw
```

- [ ] **Step 6: 验证静态库编译**

Run: `ls -la /usr/local/static-libs/*/lib/*.a`
Expected: 显示所有静态库文件

---

### Task 4: 添加更多静态库编译步骤

**Files:**
- Modify: `.github/workflows/build-uos.yml`

- [ ] **Step 1: 添加 libffi 静态编译**

```yaml
      - name: Build libffi static
        run: |
          cd /tmp
          wget -q https://github.com/libffi/libffi/releases/download/v3.4.4/libffi-3.4.4.tar.gz
          tar xzf libffi-3.4.4.tar.gz
          cd libffi-3.4.4
          ./configure --prefix=/usr/local/static-libs/libffi \
            --disable-shared --enable-static --with-pic
          make -j$(nproc)
          make install
```

- [ ] **Step 2: 添加 ncurses 静态编译**

```yaml
      - name: Build ncurses static
        run: |
          cd /tmp
          wget -q https://ftp.gnu.org/pub/gnu/ncurses/ncurses-6.4.tar.gz
          tar xzf ncurses-6.4.tar.gz
          cd ncurses-6.4
          ./configure --prefix=/usr/local/static-libs/ncurses \
            --without-shared --without-cxx-shared \
            --with-pic --enable-widec
          make -j$(nproc)
          make install
```

- [ ] **Step 3: 添加 readline 静态编译**

```yaml
      - name: Build readline static
        run: |
          cd /tmp
          wget -q https://ftp.gnu.org/pub/gnu/readline/readline-8.2.tar.gz
          tar xzf readline-8.2.tar.gz
          cd readline-8.2
          ./configure --prefix=/usr/local/static-libs/readline \
            --disable-shared --enable-static --with-pic \
            --with-curses=/usr/local/static-libs/ncurses
          make -j$(nproc) SHLIB_LIBS="-L/usr/local/static-libs/ncurses/lib -lncursesw"
          make install
```

- [ ] **Step 4: 添加 sqlite3 静态编译**

```yaml
      - name: Build sqlite3 static
        run: |
          cd /tmp
          wget -q https://www.sqlite.org/2023/sqlite-autoconf-3410000.tar.gz
          tar xzf sqlite-autoconf-3410000.tar.gz
          cd sqlite-autoconf-3410000
          ./configure --prefix=/usr/local/static-libs/sqlite3 \
            --disable-shared --enable-static
          make -j$(nproc)
          make install
```

- [ ] **Step 5: 添加 mpdecimal 静态编译**

```yaml
      - name: Build mpdecimal static
        run: |
          cd /tmp
          wget -q https://www.bytereef.org/software/mpdecimal/releases/mpdecimal-2.5.1.tar.gz
          tar xzf mpdecimal-2.5.1.tar.gz
          cd mpdecimal-2.5.1
          ./configure --prefix=/usr/local/static-libs/mpdecimal \
            --disable-shared --enable-static
          make -j$(nproc)
          make install
```

- [ ] **Step 6: 验证所有静态库**

Run: `find /usr/local/static-libs -name "*.a" | wc -l`
Expected: 显示静态库数量（应大于 10）

---

### Task 5: 配置和编译 Python

**Files:**
- Modify: `.github/workflows/build-uos.yml`

- [ ] **Step 1: 设置编译环境变量**

```yaml
      - name: Setup Python build environment
        run: |
          export MODULE_BUILDTYPE=static
          export PY_UNSUPPORTED_OPENSSL_BUILD=static
          echo "MODULE_BUILDTYPE=static" >> $GITHUB_ENV
          echo "PY_UNSUPPORTED_OPENSSL_BUILD=static" >> $GITHUB_ENV
          
          echo "STATIC_CFLAGS=-I/usr/local/static-libs/zlib/include \
            -I/usr/local/static-libs/bzip2/include \
            -I/usr/local/static-libs/xz/include \
            -I/usr/local/static-libs/openssl/include \
            -I/usr/local/static-libs/libffi/include \
            -I/usr/local/static-libs/ncurses/include \
            -I/usr/local/static-libs/readline/include \
            -I/usr/local/static-libs/sqlite3/include \
            -I/usr/local/static-libs/mpdecimal/include" >> $GITHUB_ENV
          
          echo "STATIC_LDFLAGS=-L/usr/local/static-libs/zlib/lib \
            -L/usr/local/static-libs/bzip2/lib \
            -L/usr/local/static-libs/xz/lib \
            -L/usr/local/static-libs/openssl/lib \
            -L/usr/local/static-libs/libffi/lib \
            -L/usr/local/static-libs/ncurses/lib \
            -L/usr/local/static-libs/readline/lib \
            -L/usr/local/static-libs/sqlite3/lib \
            -L/usr/local/static-libs/mpdecimal/lib" >> $GITHUB_ENV
```

- [ ] **Step 2: 配置 Python**

```yaml
      - name: Configure Python for static build
        run: |
          ./configure \
            --prefix=/opt/python-uos \
            --disable-shared \
            --with-static-libpython \
            --enable-optimizations \
            --with-lto \
            --with-openssl=/usr/local/static-libs/openssl \
            CFLAGS="${STATIC_CFLAGS}" \
            LDFLAGS="${STATIC_LDFLAGS} -static" \
            MODULE_BUILDTYPE=static \
            PY_UNSUPPORTED_OPENSSL_BUILD=static
```

- [ ] **Step 3: 编译 Python**

```yaml
      - name: Build Python
        run: |
          make -j$(nproc)
```

- [ ] **Step 4: 安装 Python**

```yaml
      - name: Install Python
        run: |
          make install
```

- [ ] **Step 5: 验证 Python 安装**

Run: `/opt/python-uos/bin/python3 --version`
Expected: 显示 Python 版本号

---

### Task 6: 验证静态链接

**Files:**
- Modify: `.github/workflows/build-uos.yml`

- [ ] **Step 1: 添加依赖检查步骤**

```yaml
      - name: Verify static linking
        run: |
          echo "=== Checking Python binary dependencies ==="
          ldd /opt/python-uos/bin/python3 || true
          
          echo "=== Expected: only glibc dependencies ==="
          ldd /opt/python-uos/bin/python3 2>&1 | grep -E "(libpthread|libdl|libc\.so)" || echo "Found expected glibc deps"
          
          echo "=== Checking for unwanted dynamic libs ==="
          if ldd /opt/python-uos/bin/python3 2>&1 | grep -qE "(libssl|libcrypto|libz|libbz2|liblzma|libffi|libncurses|libreadline|libsqlite3)"; then
            echo "ERROR: Found unwanted dynamic library dependencies"
            ldd /opt/python-uos/bin/python3
            exit 1
          fi
          echo "SUCCESS: No unwanted dynamic library dependencies found"
```

- [ ] **Step 2: 添加模块验证步骤**

```yaml
      - name: Verify Python modules
        run: |
          /opt/python-uos/bin/python3 -c "import ssl; print('SSL:', ssl.OPENSSL_VERSION)"
          /opt/python-uos/bin/python3 -c "import zlib; print('zlib OK')"
          /opt/python-uos/bin/python3 -c "import bz2; print('bz2 OK')"
          /opt/python-uos/bin/python3 -c "import lzma; print('lzma OK')"
          /opt/python-uos/bin/python3 -c "import sqlite3; print('sqlite3:', sqlite3.sqlite_version)"
          /opt/python-uos/bin/python3 -c "import ctypes; print('ctypes OK')"
          /opt/python-uos/bin/python3 -c "import curses; print('curses OK')"
          /opt/python-uos/bin/python3 -c "import readline; print('readline OK')"
          /opt/python-uos/bin/python3 -c "import decimal; print('decimal OK')"
```

- [ ] **Step 3: 验证模块输出**

Run: `/opt/python-uos/bin/python3 -c "import ssl,zlib,bz2,lzma,sqlite3,ctypes,curses,readline,decimal"`
Expected: 无错误，所有模块成功导入

---

### Task 7: 打包和上传 artifacts

**Files:**
- Modify: `.github/workflows/build-uos.yml`

- [ ] **Step 1: 获取 Python 版本**

```yaml
      - name: Get Python version for package naming
        run: |
          PYTHON_VERSION=$(/opt/python-uos/bin/python3 -c "import sys; print(sys.version.split()[0])")
          echo "PYTHON_VERSION=${PYTHON_VERSION}" >> $GITHUB_ENV
```

- [ ] **Step 2: 创建 Python tarball**

```yaml
      - name: Create Python tarball
        run: |
          cd /opt
          tar -czf python-${{ env.PYTHON_VERSION }}-uos-x86_64.tar.gz python-uos
          mv python-${{ env.PYTHON_VERSION }}-uos-x86_64.tar.gz $GITHUB_WORKSPACE/
```

- [ ] **Step 3: 下载 virtualenv wheel**

```yaml
      - name: Download virtualenv wheel
        run: |
          mkdir -p /tmp/wheels
          pip3 download virtualenv -d /tmp/wheels --no-deps
          cp /tmp/wheels/virtualenv-*.whl $GITHUB_WORKSPACE/
```

- [ ] **Step 4: 上传 Python artifact**

```yaml
      - name: Upload Python artifact
        uses: actions/upload-artifact@v4
        with:
          name: python-uos-${{ env.PYTHON_VERSION }}
          path: python-${{ env.PYTHON_VERSION }}-uos-x86_64.tar.gz
          retention-days: 30
```

- [ ] **Step 5: 上传 virtualenv artifact**

```yaml
      - name: Upload virtualenv artifact
        uses: actions/upload-artifact@v4
        with:
          name: virtualenv-wheel
          path: virtualenv-*.whl
          retention-days: 30
```

- [ ] **Step 6: 验证 artifacts 创建**

Run: `ls -la *.tar.gz *.whl`
Expected: 显示 Python tarball 和 virtualenv wheel 文件

---

### Task 8: 推送代码并触发构建

**Files:**
- None（Git 操作）

- [ ] **Step 1: 提交 workflow 文件**

```bash
git add .github/workflows/build-uos.yml docs/superpowers/specs/2026-05-13-uos-python-build-design.md
git commit -m "feat: add GitHub Actions workflow for UOS static Python build"
```

- [ ] **Step 2: 推送到远程仓库**

```bash
git push origin main
```

- [ ] **Step 3: 验证 push 成功**

Run: `git log -1 --oneline`
Expected: 显示最新提交记录

---

### Task 9: 监控构建进度

**Files:**
- None（监控操作）

- [ ] **Step 1: 查看 GitHub Actions 运行状态**

访问 GitHub Actions 页面或使用 gh CLI：
```bash
gh run list --limit 5
```

Expected: 显示最新的 workflow run

- [ ] **Step 2: 每 60 秒检查构建状态**

循环检查直到构建完成：
```bash
# 手动监控命令
gh run watch
```

- [ ] **Step 3: 检查构建结果**

构建完成后：
```bash
gh run view --log
```

Expected: 显示构建日志，检查是否有错误

---

### Task 10: 错误排查和修复循环

**Files:**
- Modify: `.github/workflows/build-uos.yml`（如有错误）

- [ ] **Step 1: 分析错误日志**

如果构建失败，查看详细日志：
```bash
gh run view --log-failed
```

- [ ] **Step 2: 定位错误原因**

根据错误信息，确定问题所在：
- 静态库编译失败：检查库版本、编译选项
- Python 配置失败：检查 configure 输出、环境变量
- Python 编译失败：检查编译日志、缺少的依赖
- 模块验证失败：检查模块是否正确链接

- [ ] **Step 3: 修复问题**

根据错误原因修改 workflow 文件：
```yaml
# 示例：修复 OpenSSL 编译问题
- name: Build OpenSSL static
  run: |
    # 调整编译选项
    ./Configure no-shared no-dso -fPIC ...
```

- [ ] **Step 4: 提交并推送修复**

```bash
git add .github/workflows/build-uos.yml
git commit -m "fix: resolve build error - {error description}"
git push origin main
```

- [ ] **Step 5: 等待新构建完成**

重复 Task 9 的监控步骤，直到构建成功。

---

## 自检清单

### 1. Spec Coverage Check

| Spec Requirement | Task Coverage |
|------------------|---------------|
| 创建 workflow 文件 | Task 1, Task 2 |
| 静态编译所有依赖库 | Task 3, Task 4 |
| Python 静态构建 | Task 5 |
| 验证静态链接 | Task 6 |
| 打包 tar.gz 和 wheel | Task 7 |
| 推送代码触发构建 | Task 8 |
| 监控构建进度 | Task 9 |
| 错误排查循环 | Task 10 |
| push 到 main + 可选 tag | Task 1（workflow_dispatch inputs） |

**无遗漏项**

### 2. Placeholder Scan

检查计划中的所有步骤：
- 无 "TBD", "TODO", "implement later"
- 无 "Add appropriate error handling"
- 无 "Write tests for the above"
- 所有代码步骤都有完整的代码块
- 所有命令都有明确的预期输出

**无 placeholder 问题**

### 3. Type Consistency

检查环境变量和路径：
- `STATIC_LIB_DIR` 在 Task 3 定义，后续任务使用一致
- `STATIC_CFLAGS` 和 `STATIC_LDFLAGS` 在 Task 5 定义，在 configure 中使用
- `PYTHON_VERSION` 在 Task 7 定义，在 artifact 上传中使用
- 所有路径 `/usr/local/static-libs/*` 在各任务中保持一致

**类型和命名一致**

---

## 执行选择

Plan complete and saved to `docs/superpowers/plans/2026-05-13-uos-python-build-plan.md`. 

两种执行选项：

**1. Subagent-Driven (recommended)** - 我为每个任务派发新 subagent，任务间进行审查，快速迭代

**2. Inline Execution** - 在当前会话中使用 executing-plans 执行任务，批量执行并设置检查点

选择哪种方式？