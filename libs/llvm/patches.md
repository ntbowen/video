已保存到 Memo ✅

---

````markdown
# LLVM 22.1.3 完整构建修复记录（最终版）

> 环境：zagwrt · x86_64 · musl · feeds/video/libs/llvm · 2026-04-27/28
> SPIRV-LLVM-Translator：21.1.1

---

## 一、修复汇总

| 修改项 | 问题现象 | 修复内容 |
|--------|----------|----------|
| `101-fix-spirv-translator-llvm22-api.patch` | `LLVMToSPIRVDbgTran.cpp` 编译失败 | LLVM 22 API 变更 |
| `102-fix-spirv-reader-lifetime-llvm22.patch` | `SPIRVReader.cpp` Lifetime API 参数变更 | 移除 `Size` 参数 |
| `103-fix-spirv-passplugin-header-llvm22.patch` | `PassPlugin.cpp` 头文件路径变更 | `llvm/Passes/` → `llvm/Plugins/` |
| Makefile `Build/Prepare` 补丁应用 | target build 未应用 patch | 在 `Build/Prepare` 中补加 patch 命令 |
| Makefile libclc `.spv` 安装路径 | `spirv-mesa3d-.spv` 找不到 | `share/clc/` → `lib/clang/*/lib/libclc/` |
| Makefile `prepare_builtins` 移除 | host install 报 `No such file` | LLVM 22 已删除此工具 |

---

## 二、patch 101：SPIRV Translator API 变更

**文件**：`feeds/video/libs/llvm/patches-spirv/101-fix-spirv-translator-llvm22-api.patch`

修复 `LLVMToSPIRVDbgTran.cpp` 中因 LLVM 22 API 变更导致的编译失败。

---

## 三、patch 102：Lifetime API 参数变更

**文件**：`feeds/video/libs/llvm/patches-spirv/102-fix-spirv-reader-lifetime-llvm22.patch`

**错误**：
```
error: no matching function for call to
'llvm::IRBuilder<>::CreateLifetimeStart(llvm::Value*&, llvm::ConstantInt*&)'
candidate expects 1 argument, 2 provided
```

**根因**：LLVM 22 移除了 `CreateLifetimeStart/End` 的 `Size` 参数。

修复 `SPIRVReader.cpp` 中三处 `CreateLifetimeStart(Var, S)` / `CreateLifetimeEnd(Var, S)` → 移除第二个参数。

---

## 四、patch 103：PassPlugin.h 路径变更

**文件**：`feeds/video/libs/llvm/patches-spirv/103-fix-spirv-passplugin-header-llvm22.patch`

```patch
--- a/lib/SPIRV/PassPlugin.cpp
+++ b/lib/SPIRV/PassPlugin.cpp
@@ -49,7 +49,7 @@
 #include "SPIRVWriter.h"
 
 #include "llvm/Passes/PassBuilder.h"
-#include "llvm/Passes/PassPlugin.h"
+#include "llvm/Plugins/PassPlugin.h"
 
 using namespace llvm;
```

**根因**：LLVM 22 将 `PassPlugin.h` 从 `llvm/Passes/` 移到 `llvm/Plugins/`。

**验证**：
```bash
find build_dir/target-x86_64_musl/llvm-mesa/llvm-project-22.1.3.src/llvm/include/ \
    -name "PassPlugin.h"
# 输出：.../llvm/include/llvm/Plugins/PassPlugin.h
```

---

## 五、Makefile：Build/Prepare 补全 patch 应用

**根因**：`Host/Prepare` 解压并应用了 101/102/103 patch，但 `Build/Prepare` **只解压，没有应用 patch**，导致 target build 使用未打 patch 的源码。

**修复后 `Build/Prepare` 块**：

```makefile
define Build/Prepare
    $(call Build/Prepare/Default)
    $(STAGING_DIR_HOST)/bin/libdeflate-gzip -dc $(DL_DIR)/$(SPIRV_LLVM_TRANSLATOR_FILE) \
        | $(TAR) -C $(PKG_BUILD_DIR)/llvm/projects $(TAR_OPTIONS)
    @echo "Applying SPIRV-Translator LLVM22 API fix (target build)..."
    patch -p1 -d $(PKG_BUILD_DIR)/llvm/projects/SPIRV-LLVM-Translator-$(SPIRV_LLVM_TRANSLATOR_VERSION) \
        < $(CURDIR)/patches-spirv/101-fix-spirv-translator-llvm22-api.patch
    patch -p1 -d $(PKG_BUILD_DIR)/llvm/projects/SPIRV-LLVM-Translator-$(SPIRV_LLVM_TRANSLATOR_VERSION) \
        < $(CURDIR)/patches-spirv/102-fix-spirv-reader-lifetime-llvm22.patch
    patch -p1 -d $(PKG_BUILD_DIR)/llvm/projects/SPIRV-LLVM-Translator-$(SPIRV_LLVM_TRANSLATOR_VERSION) \
        < $(CURDIR)/patches-spirv/103-fix-spirv-passplugin-header-llvm22.patch
endef
```

> ⚠️ patch 路径使用 `$(CURDIR)/patches-spirv/` 而非 `patches/`，避免 OpenWrt 自动扫描。

---

## 六、Makefile：libclc .spv 安装路径修复

**问题**：
```
cp: cannot stat '.../ipkg-install/usr/share/clc/spirv-mesa3d-.spv': No such file or directory
cp: cannot stat '.../ipkg-install/usr/share/clc/spirv64-mesa3d-.spv': No such file or directory
```

**根因**：LLVM 22 将 libclc 的 `.spv` 安装路径从 `usr/share/clc/` 改为 `usr/lib/clang/22/lib/libclc/`。

**Makefile 修改**（`Build/Install` 段，第 272-273 行）：

```makefile
# 旧（LLVM < 22）：
$(CP) $(PKG_INSTALL_DIR)/usr/share/clc/spirv{,64}-mesa3d-.spv \
    $(CMAKE_HOST_INSTALL_PREFIX)/share/clc

# 新（LLVM 22）：
$(CP) $(PKG_INSTALL_DIR)/usr/lib/clang/*/lib/libclc/spirv{,64}-mesa3d-.spv \
    $(CMAKE_HOST_INSTALL_PREFIX)/share/clc
```

**执行**：
```bash
sed -i 's|usr/share/clc/spirv|usr/lib/clang/*/lib/libclc/spirv|g' \
    feeds/video/libs/llvm/Makefile
```

---

## 七、Makefile：移除 prepare_builtins

**问题**：
```
install: cannot stat '.../build/bin/prepare_builtins': No such file or directory
```

**根因**：`prepare_builtins` 在 LLVM 22 中已被移除。

**修复**：
```bash
sed -i 's/ prepare_builtins//' feeds/video/libs/llvm/Makefile
```

---

## 八、构建流程

### 完整重建（从零）
```bash
rm -rf build_dir/hostpkg/llvm-mesa/ build_dir/target-x86_64_musl/llvm-mesa/
make package/feeds/video/llvm/compile V=sc -j$(nproc) 2>&1 | tee /tmp/llvm-build.log
```

### 仅重跑 host Install
```bash
rm -f build_dir/hostpkg/llvm-mesa/llvm-project-22.1.3.src/.built
rm -f staging_dir/hostpkg/stamp/.llvm-project_installed
make package/feeds/video/llvm/host-compile V=sc
```

### 仅重跑 target Install
```bash
rm -f build_dir/target-x86_64_musl/llvm-mesa/llvm-project-22.1.3.src/.built
make package/feeds/video/llvm/compile V=sc
```

---

## 九、验证结果

```
time: package/feeds/video/llvm/host-compile#242.36#92.29#94.61   ✅
time: package/feeds/video/llvm/compile#19184.88#1496.05#812.04   ✅
```

```bash
# host .spv 文件到位
ls staging_dir/host/llvm-mesa-22.1.3.Linux-x86_64/share/clc/
# spirv-mesa3d-.spv  spirv64-mesa3d-.spv  ✅

# target staging stamp
ls staging_dir/target-x86_64_musl/stamp/.llvm_installed  ✅
```

---

## 十、注意事项

1. `Host/Prepare` 和 `Build/Prepare` 是两个独立阶段，patch **必须在两处都应用**
2. `.spv` 复制逻辑在 `Build/Install` 中，但目标是 host staging；host `.built` stamp 失效时需重跑 `host-compile`
3. `prepare_builtins` 从 LLVM 22 起被移除，升级时需同步清理工具列表
4. libclc 路径规律：`usr/lib/clang/<major>/lib/libclc/`，glob `*` 可兼容未来小版本
````
