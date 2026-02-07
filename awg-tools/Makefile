include $(TOPDIR)/rules.mk

PKG_NAME:=amneziawg-tools
PKG_VERSION:=1.0.20250903

PKG_SOURCE_URL:=https://github.com/amnezia-vpn/amneziawg-tools.git
PKG_SOURCE_VERSION:=v$(PKG_VERSION)
PKG_SOURCE_PROTO:=git

PKG_LICENSE:=GPL-2.0
PKG_LICENSE_FILES:=COPYING

include $(INCLUDE_DIR)/package.mk

define Package/amneziawg-tools
  SECTION:=net
  CATEGORY:=Network
  SUBMENU:=VPN
  URL:=https://github.com/amnezia-vpn/amneziawg-tools
  TITLE:=AmneziaWG userspace tools (awg)
endef

define Package/amneziawg-tools/description
  AmneziaWG userspace configuration tool (awg), a fork of wg-tools
  with support for AmneziaWG obfuscation parameters (Jc, Jmin, Jmax,
  S1-S4, H1-H4, I1-I5).
endef

MAKE_PATH:=src

MAKE_FLAGS += \
	PREFIX=/usr \
	PLATFORM=linux \
	WITH_BASHCOMPLETION= \
	WITH_WGQUICK= \
	WITH_SYSTEMDUNITS=

define Package/amneziawg-tools/install
	$(INSTALL_DIR) $(1)/usr/bin
	$(INSTALL_BIN) $(PKG_BUILD_DIR)/src/wg $(1)/usr/bin/awg
endef

$(eval $(call BuildPackage,amneziawg-tools))
