include $(TOPDIR)/rules.mk

PKG_NAME:=usb-vbus
PKG_VERSION:=1.0.0
PKG_RELEASE:=1
PKG_MAINTAINER:=RUI
PKG_LICENSE:=MIT

include $(INCLUDE_DIR)/package.mk

define Package/usb-vbus
  SECTION:=utils
  CATEGORY:=Utilities
  TITLE:=USB VBUS GPIO control for Netcore N30 Pro
  DEPENDS:=+kmod-gpio-button-hotplug
endef

define Package/usb-vbus/description
  Drives GPIO 535 (MT7981 pin 23 + SoC base 512) high to supply 5V to USB-A port.
  Required for USB Tethering to work since the N30 Pro USB-A has no VBUS from boot.
endef

define Build/Compile
endef

define Package/usb-vbus/install
	$(INSTALL_DIR) $(1)/etc/init.d
	$(INSTALL_BIN) ./files/usb-vbus $(1)/etc/init.d/usb-vbus
endef

$(eval $(call BuildPackage,usb-vbus))
