include $(TOPDIR)/rules.mk
PKG_NAME:=usbwan-watchdog
PKG_VERSION:=1.0.0
PKG_RELEASE:=1
PKG_MAINTAINER:=RUI
PKG_LICENSE:=MIT
include $(INCLUDE_DIR)/package.mk

define Package/usbwan-watchdog
  SECTION:=utils
  CATEGORY:=Utilities
  TITLE:=Watchdog for USB WAN interface
  DEPENDS:=+curl +kmod-gpio-button-hotplug
endef

define Package/usbwan-watchdog/description
  Pings gateway + 223.5.5.5 every minute via cron; on 3 consecutive failures,
  drops and raises USB VBUS (GPIO535 1->0->1) and ifdown/ifup usbwan.
endef

define Build/Compile
endef

define Package/usbwan-watchdog/install
	$(INSTALL_DIR) $(1)/usr/bin
	$(INSTALL_BIN) ./files/usbwan-watchdog $(1)/usr/bin/usbwan-watchdog
	$(INSTALL_DIR) $(1)/etc/crontabs
	echo "* * * * * root /usr/bin/usbwan-watchdog >> /var/log/usbwan-watchdog.log 2>&1" > $(1)/etc/crontabs/root
endef

$(eval $(call BuildPackage,usbwan-watchdog))
