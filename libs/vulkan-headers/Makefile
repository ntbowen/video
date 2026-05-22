include $(TOPDIR)/rules.mk

PKG_NAME:=vulkan-headers
PKG_VERSION:=1.4.352
PKG_SOURCE:=Vulkan-Headers-$(PKG_VERSION).tar.gz
PKG_SOURCE_URL:=https://codeload.github.com/KhronosGroup/Vulkan-Headers/tar.gz/v$(PKG_VERSION)?
PKG_HASH:=4850909d22a8a9767c27daea2b972e49d7c298560573d5b6221ee50db9bf49db

PKG_MAINTAINER:=Daniel Golle <daniel@makrotopia.org>
PKG_LICENSE:=Apache-2.0 OR MIT
PKG_LICENSE_FILES:=LICENSE.md

PKG_BUILD_DIR:=$(BUILD_DIR)/Vulkan-Headers-$(PKG_VERSION)

CMAKE_BINARY_SUBDIR:=build
CMAKE_INSTALL:=1

include $(INCLUDE_DIR)/package.mk
include $(INCLUDE_DIR)/cmake.mk

define Package/vulkan-headers
  SECTION:=devel
  CATEGORY:=Development
  TITLE:=Vulkan Headers
  URL:=https://github.com/KhronosGroup/Vulkan-Headers
  BUILDONLY:=1
endef

define Package/vulkan-headers/description
Vulkan header files and API registry.
endef

define Package/vulkan-headers/install
true
endef

$(eval $(call BuildPackage,vulkan-headers))
