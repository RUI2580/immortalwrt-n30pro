include $(TOPDIR)/rules.mk
PKG_NAME:=ttlfix
PKG_VERSION:=1.0.0
PKG_RELEASE:=1
PKG_MAINTAINER:=RUI
PKG_LICENSE:=MIT
include $(INCLUDE_DIR)/package.mk

define Package/ttlfix
  SECTION:=utils
  CATEGORY:=Utilities
  TITLE:=Fix TTL on USB tethering interface
  DEPENDS:=+firewall4 +nftables
endef

define Package/ttlfix/description
  Uses nftables to set ip ttl 64 and ip6 hoplimit 64 on usb0/wwan0.
  Android/iPhone USB tethering drops connections if TTL differs from 64.
endef

define Build/Compile
endef

define Package/ttlfix/install
	$(INSTALL_DIR) $(1)/etc/init.d
	$(INSTALL_BIN) ./files/ttlfix $(1)/etc/init.d/ttlfix
endef

$(eval $(call BuildPackage,ttlfix))
