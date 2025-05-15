---
layout: post
title:  "编译 Nginx for Android aarch64"
date:   2025-5-14 22:30:35 +0800
categories: nginx android aarch64
---

参考：<https://blog.csdn.net/weixin_43793181/article/details/116494853>

本篇文章主要介绍了如何在ubuntu上为Android平台编译Nginx，使用了NDK进行交叉编译。首先需要下载Nginx源码和依赖库的源码，然后配置环境变量并设置NDK路径。接着是编译依赖库，最后编译Nginx。

编译环境：Ubuntu 24.04

编译需要的工具和源代码版本：

1. android NDK： [android-ndk-r23c](https://github.com/android/ndk/wiki/Home/e05d396317f6e1e5e00e3fe7d4f223ad54a354f8)
2. nginx：       1.25.3
3. openssl：     1.1.1w
4. pcre：        8.45
5. zlib：        1.2.13

以下是完整的、**基于 `android-ndk-r23c` 的 Nginx 1.25.3 Android aarch64 构建脚本**。包括：

- 下载并解压所有依赖（含 `android-ndk-r23c`）
- 构建静态 nginx 二进制，可推送到 Android 7.1.2 aarch64 测试

---

## 📁 目录结构建议

```
nginx-android/
├── 00_init_tools.sh
├── 01_download.sh
├── 02_build_deps.sh
├── 03_build_nginx.sh
└── archives/
└── sources/
```

---

## 1. `00_init_tools.sh`安装必要工具

```s
#!/bin/bash

sudo apt update && sudo apt install -y build-essential autoconf libtool

# 更新系统并安装基础工具
sudo apt update
sudo apt install -y \
    build-essential \
    autoconf \
    automake \
    libtool \
    pkg-config \
    cmake \
    git \
    wget \
    curl \
    unzip \
    zlib1g-dev \
    libpcre3-dev \
    libssl-dev  # 如果需 SSL 支持（如 OpenSSL）
```

## 01_download.sh

```s
#!/bin/bash
set -euo pipefail

# 配置区（可自定义版本）
NGINX_VERSION="1.25.3"
OPENSSL_VERSION="1.1.1w"
PCRE_VERSION="8.45"
ZLIB_VERSION="1.2.13"
HTTP_FLV_MODULE_TAG="v1.2.9"

# 创建目录结构
mkdir -p {sources,archives}

# 下载函数
download() {
    local url=$1
    local file=$2
    if [ ! -f "archives/$file" ]; then
        echo "下载: $file"
        wget -qc "$url" -O "archives/$file" || {
            echo "下载失败: $url"
            exit 1
        }
    else
        echo "已存在: $file"
    fi
}

# 下载所有源码
download "https://nginx.org/download/nginx-${NGINX_VERSION}.tar.gz" "nginx-${NGINX_VERSION}.tar.gz"
download "https://www.openssl.org/source/openssl-${OPENSSL_VERSION}.tar.gz" "openssl-${OPENSSL_VERSION}.tar.gz"
download "https://sourceforge.net/projects/pcre/files/pcre/${PCRE_VERSION}/pcre-${PCRE_VERSION}.tar.gz" "pcre-${PCRE_VERSION}.tar.gz"
download "https://zlib.net/zlib-${ZLIB_VERSION}.tar.gz" "zlib-${ZLIB_VERSION}.tar.gz"

# download "https://github.com/winshining/nginx-http-flv-module/archive/refs/tags/v1.2.9.tar.gz" "nginx-http-flv-module-1.2.9.tar.gz"
#克隆模块仓库
if [ ! -d "sources/nginx-http-flv-module" ]; then
    git clone --depth 1 --branch "$HTTP_FLV_MODULE_TAG" \
        https://github.com/winshining/nginx-http-flv-module.git \
        sources/nginx-http-flv-module
fi

# 解压源码
for pkg in nginx openssl pcre zlib; do
    version_var="${pkg^^}_VERSION"
    tar -xzf "archives/${pkg}-${!version_var}.tar.gz" -C sources/
done

echo "所有源码已准备就绪: ./sources/"
```

---

## 02_build_deps.sh

```s
#!/bin/bash
set -euo pipefail

# 配置区
OPENSSL_VERSION="1.1.1w"
PCRE_VERSION="8.45"
ZLIB_VERSION="1.2.13"
LIBXCRYPT_VERSION="4.4.36"

# 环境检查
[ ! -d "sources" ] && { echo "请先运行 download_sources.sh"; exit 1; }

# NDK配置
export ANDROID_NDK="${ANDROID_NDK:-$(pwd)/sources/android-ndk-r23c}"
export API_LEVEL=24
export ARCH="aarch64"
export OUT_DIR="$(pwd)/android-libs"

# 工具链设置
export TOOLCHAIN="$ANDROID_NDK/toolchains/llvm/prebuilt/linux-x86_64"
export HOST="${ARCH}-linux-android"
export CC="${TOOLCHAIN}/bin/${HOST}${API_LEVEL}-clang"
export CXX="${TOOLCHAIN}/bin/${HOST}${API_LEVEL}-clang++"
export AR="${TOOLCHAIN}/bin/llvm-ar"
export RANLIB="${TOOLCHAIN}/bin/llvm-ranlib"
export CFLAGS="--sysroot=${TOOLCHAIN}/sysroot -fPIC"

# 修复PCRE编译
compile_pcre() {
    cd "sources/pcre-${PCRE_VERSION}"

    # 关键修复：禁用man手册和不需要的UTF支持
    ./configure \
        --host="$HOST" \
        --prefix="$OUT_DIR" \
        --disable-shared \
        --enable-static \
        --enable-jit \
        --disable-cpp \
        --disable-pcre16 \
        --disable-pcre32 \
        --enable-unicode-properties \
        --enable-pcregrep-libz=no \
        --enable-pcregrep-libbz2=no \
        --enable-pcretest-libreadline=no \
        --mandir=/tmp  # 重定向man手册到临时目录

    # 直接调用make而不是make install避免man手册问题
    make -j$(nproc) install-binPROGRAMS install-libLTLIBRARIES install-data \
        install-pkgconfigDATA install-includeHEADERS

    cd ../..
}

# OpenSSL编译保持不变
compile_openssl() {
    cd "sources/openssl-${OPENSSL_VERSION}"

    # 完全清除旧配置
    make distclean >/dev/null 2>&1 || true

    # 设置 Android 专用环境
    export ANDROID_NDK_HOME="$ANDROID_NDK"
    export PATH="$TOOLCHAIN/bin:$PATH"

    # 使用完整的 perl Configure 命令
    perl Configure \
        android-arm64 \
        -D__ANDROID_API__=$API_LEVEL \
        --prefix="$OUT_DIR" \
        --openssldir="$OUT_DIR/ssl" \
        no-shared \
        no-weak-ssl-ciphers \
        no-ssl3 \
        no-comp \
        -fstack-protector-strong \
        -fPIC \
        -DANDROID \
        CC="$CC" \
        AR="$AR" \
        RANLIB="$RANLIB" \
        LD="$TOOLCHAIN/bin/ld.lld"

    # 修复 Makefile 中的安装规则
    sed -i 's/|| true//g' Makefile
    sed -i 's/$(INSTALL) -m 644 $$i/$(INSTALL) -m 644 $$i ||:/g' Makefile


    # 编译并安装
    make depend
    make -j$(nproc)

    # 手动安装库文件
    mkdir -p "$OUT_DIR/lib"
    cp libssl.a libcrypto.a "$OUT_DIR/lib/"

    # 安装头文件
    mkdir -p "$OUT_DIR/include/openssl"
    cp include/openssl/*.h "$OUT_DIR/include/openssl/"

    cd ../..
}

compile_zlib(){
    cd "sources/zlib-${ZLIB_VERSION}"

    make clean || true

    # 直接使用 zlib 的 configure
    CHOST=$HOST \
    CC="$CC" \
    AR="$AR" \
    RANLIB="$RANLIB" \
    CFLAGS="$CFLAGS" \
    ./configure --static --prefix="$OUT_DIR"

    make -j$(nproc)
    make install
    cd ../..
}

compile_libxcrypt(){
    cd "sources/libxcrypt-${LIBXCRYPT_VERSION}"

    ./configure -prefix="$OUT_DIR" --host=aarch64-linux-android --disable-shared --enable-static --disable-obsolete-api  CFLAGS="$CFLAGS -Wno-error"

    make -j$(nproc)
    make install

    cd ../..
}

# 执行编译
mkdir -p "$OUT_DIR"
compile_zlib
compile_libxcrypt
compile_pcre
compile_openssl

echo "依赖库编译完成: $OUT_DIR"
```

---

## 调整nginx源代码，解决编译报错

编译nginx时会报错，需要修改如下三个文件：

1. 打开`auto/cc/name`找到`ngx_feature_run=yes`改为`ngx_feature_run=no`。

2. 打开`auto/types/sizeof`找到15行的`ngx_size=`改为`ngx_size=4`，找到43行的

```cts
ngx_size=`$NGX_AUTOTEST`
```

改为：

```c
ngx_size=4 #`$NGX_AUTOTEST`
```

3. 打开`src/os/unix/ngx_files.c`文件，找到`ngx_open_glob`函数和`ngx_close_glob`函数。

找到`ngx_open_glob`函数：

```c
ngx_open_glob(ngx_glob_t *gl)
{
    int  n;

    n = glob((char *) gl->pattern, 0, NULL, &gl->pglob);

    if (n == 0) {
        return NGX_OK;
    }

#ifdef GLOB_NOMATCH

    if (n == GLOB_NOMATCH && gl->test) {
        return NGX_OK;
    }

#endif

    return NGX_ERROR;
}
```

改为：

```c
ngx_open_glob(ngx_glob_t *gl)
{
    return NGX_OK;
}
```

找到`ngx_close_glob`函数：

```c++
ngx_close_glob(ngx_glob_t *gl)
{
    globfree(&gl->pglob);
}
```

改为：

```c++
ngx_close_glob(ngx_glob_t *gl)
{
    //globfree(&gl->pglob);
}
```

---

## 03_build_nginx.sh

```s
#!/bin/bash
set -euo pipefail

# 使用绝对路径定义NDK
export ANDROID_NDK="${ANDROID_NDK:-$(pwd)/sources/android-ndk-r23c}"
export TOOLCHAIN="$ANDROID_NDK/toolchains/llvm/prebuilt/linux-x86_64"

# 设置完整的交叉编译环境
export PATH="$TOOLCHAIN/bin:$PATH"
export CC="$TOOLCHAIN/bin/aarch64-linux-android24-clang"  # 使用完整路径
export CXX="$TOOLCHAIN/bin/aarch64-linux-android24-clang++"
export AR="$TOOLCHAIN/bin/llvm-ar"
export LD="$TOOLCHAIN/bin/ld.lld"
export RANLIB="$TOOLCHAIN/bin/llvm-ranlib"
export STRIP="$TOOLCHAIN/bin/llvm-strip"

# 设置sysroot和编译标志
export SYSROOT="$TOOLCHAIN/sysroot"
export CFLAGS="--sysroot=$SYSROOT -fPIC -DANDROID -I$(pwd)/android-libs/include  -DNGX_HAVE_GCC_ATOMIC=1 -DNGX_HAVE_ATOMIC_OPS=1  -DNGX_HAVE_MAP_ANON=1"
export LDFLAGS="--sysroot=$SYSROOT -static -fuse-ld=lld -L$(pwd)/android-libs/lib -lssl -lcrypto -lz -lpcre -lcrypt"

# 验证编译器
echo "=== 编译器验证 ==="
$CC -v
echo "int main(){return 0;}" > test.c
$CC test.c -o test && file test
rm -f test test.c

# 进入Nginx源码目录
cd "$(pwd)/sources/nginx-1.25.3"

# 清理旧配置
[ -f Makefile ] && make clean
rm -rf objs

# 配置Nginx（关键修改：使用绝对路径）
./configure \
    --crossbuild=Linux::aarch64 \
    --prefix=/usr/local/nginx-android \
    --add-module=../nginx-http-flv-module-1.2.9 \
    --with-cc="$CC" \
    --with-cc-opt="$CFLAGS" \
    --with-ld-opt="$LDFLAGS" \
    --with-http_ssl_module \
    --with-http_mp4_module \
    --without-pcre2  # 明确禁用PCRE2

make -j$(nproc)

echo "编译成功！二进制位置: objs/nginx"
echo "验证架构:"
file objs/nginx
```

---

## ✅ 使用方式

```bash
chmod +x 01_download.sh 02_extract.sh 03_build.sh

./01_download.sh
./02_extract.sh
./03_build.sh
```

---

## 🚀 推送并测试到 Android

```bash
adb push sources/output/sbin/nginx /data/local/tmp/nginx
adb shell
cd /data/local/tmp
chmod +x nginx
./nginx -V
```
