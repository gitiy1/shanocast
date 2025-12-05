# Maintainer: Your Name <your@email.com>
pkgname=shanocast-git
pkgver=r1.efe62cc
pkgrel=1
pkgdesc="A Chromecast receiver implementation based on Openscreen"
arch=('x86_64')
url="https://github.com/rgerganov/shanocast"
license=('BSD')
depends=('ffmpeg' 'sdl2' 'glibc')
makedepends=('git' 'gn' 'ninja' 'python' 'clang')
# 使用 raw 链接直接下载补丁
source=('git+https://chromium.googlesource.com/openscreen.git'
        'https://raw.githubusercontent.com/rgerganov/shanocast/master/shanocast.patch')
sha256sums=('SKIP'
            'SKIP')

prepare() {
  cd "$srcdir/openscreen"

  # === 关键修复 1: 确保工作区纯净 ===
  git reset --hard 934f2462ad01c407a596641dbc611df49e2017b4
  git clean -fdx
  
  # 切换到指定 commit 并更新子模块
  git checkout 934f2462ad01c407a596641dbc611df49e2017b4
  git submodule update --init --recursive

  # === 关键修复 2: 改用 patch 命令 ===
  echo "正在应用 shanocast.patch..."
  patch -p1 -i "$srcdir/shanocast.patch"

  # === 修复 GCC 13/14 和 Arch 环境编译问题 ===

  # 修复 static_credentials.cc 中的 FileUniquePtr 定义
  sed -i '127s/.*/using FileUniquePtr = std::unique_ptr<FILE, int(*)(FILE*)>;/' cast/receiver/channel/static_credentials.cc
  
  # 修复 socket 代码中的未初始化变量
  sed -i '226s/int domain;/int domain = 0;/' platform/impl/stream_socket_posix.cc
  sed -i '106s/int domain;/int domain = 0;/' platform/impl/udp_socket_posix.cc

  # 生成构建文件
  # 移除了无效的 use_custom_libcxx=false 参数
  gn gen out/Default --args="is_debug=false have_ffmpeg=true have_libsdl2=true cast_allow_developer_certificate=true is_clang=false"

  # === 暴力修复 Ninja 构建标志 (针对 Arch 新版编译器) ===
  
  # 1. 移除所有 -Werror (把警告当错误)
  find out/Default -name "*.ninja" -exec sed -i 's/-Werror/-Wno-error/g' {} +
  
  # 2. 移除 GCC 不支持的 Clang 参数
  find out/Default -name "*.ninja" -exec sed -i 's/-Wno-unknown-warning-option//g' {} +

  # 3. 注入编译器标志 (精准区分 C 和 C++)
  
  # [通用标志] C 和 C++ 都需要
  # -include stdint.h: 修复 C++ uint8_t 缺失
  # -D_GNU_SOURCE: 修复 boringssl 隐式声明
  # -Wno-format-security: 通用警告屏蔽
  sed -i 's/\${cflags}/& -include stdint.h -D_GNU_SOURCE -Wno-format-security/g' out/Default/toolchain.ninja
  sed -i 's/\$cflags/& -include stdint.h -D_GNU_SOURCE -Wno-format-security/g' out/Default/toolchain.ninja

  # [C 专用标志] 仅注入到 rule cc 块中，消除 cc1plus 的 "valid for C but not for C++" 警告
  # -Wno-old-style-definition: 修复 zlib 旧式函数定义警告
  # -Wno-deprecated-non-prototype: 修复旧式原型警告
  sed -i '/^rule cc$/,/^$/ s/\${cflags}/& -Wno-old-style-definition -Wno-deprecated-non-prototype/' out/Default/toolchain.ninja
  sed -i '/^rule cc$/,/^$/ s/\$cflags/& -Wno-old-style-definition -Wno-deprecated-non-prototype/' out/Default/toolchain.ninja

  # 修复 FFmpeg 头文件包含路径
  sed -i 's/-I..\/..\"/-I..\/..\"/g' out/Default/toolchain.ninja
  sed -i 's/-I..\/..\"/\"-I..\/..\" \"-I\/usr\/include\/ffmpeg\"/' out/Default/toolchain.ninja
}

build() {
  cd "$srcdir/openscreen"
  ninja -C out/Default cast_receiver
}

package() {
  cd "$srcdir/openscreen"
  install -Dm755 out/Default/cast_receiver "$pkgdir/usr/bin/shanocast"
  echo "安装完成！可以使用 'shanocast <interface>' 运行。"
}
