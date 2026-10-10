include $(TOPDIR)/rules.mk

PKG_NAME:=honk
PKG_VERSION:=0.0.1.beta.84
PKG_RELEASE:=1

PKG_SOURCE:=honk-core-v$(PKG_VERSION)-aarch64-unknown-linux-musl-stock.tar.gz
PKG_SOURCE_URL:=https://github.com/daeuniverse/honk/releases/download/v$(PKG_VERSION)
PKG_HASH:=6dd9d31748ca56a7f69a1ea8d1b9b65c7a0768033b760089147563a2b800703d

PKG_LICENSE:=GPL-3.0-only
PKG_LICENSE_FILES:=LICENSE
PKG_MAINTAINER:=ymxs

# OpenWrt unpacks without stripping components, so point PKG_BUILD_DIR at the
# tarball's top-level directory honk-core-<ver>-<target>/ directly.
PKG_BUILD_DIR:=$(BUILD_DIR)/honk-core-v$(PKG_VERSION)-aarch64-unknown-linux-musl-stock

include $(INCLUDE_DIR)/package.mk

define Package/honk
  SECTION:=net
  CATEGORY:=Network
  TITLE:=honk eBPF transparent proxy
  URL:=https://github.com/daeuniverse/honk
  DEPENDS:=@TARGET_airoha +kmod-sched-core +kmod-sched-bpf +kmod-veth +kmod-nft-core +kmod-nft-queue
endef

define Package/honk/description
  honk is an experimental eBPF-based transparent proxy inspired by dae and
  sing-box, with a Clash-compatible REST/WebSocket API. This package vendors
  the upstream prebuilt aarch64 musl binary with the eBPF object embedded;
  nothing is compiled locally.

  Default config is direct-only (changes no traffic path). Edit
  /etc/honk/config.dae to add nodes/groups. Clash API listens on
  127.0.0.1:9091 to avoid colliding with OpenClash's 9090.
endef

# Prebuilt static musl binary: download + verify + install, nothing to compile.
define Build/Compile
  true
endef

define Package/honk/install
  $(INSTALL_DIR) $(1)/usr/bin
  $(INSTALL_BIN) $(PKG_BUILD_DIR)/honk-core $(1)/usr/bin/honk-core
  $(INSTALL_DIR) $(1)/etc/honk
  $(INSTALL_CONF) ./files/etc/honk/config.dae $(1)/etc/honk/config.dae
  $(INSTALL_DIR) $(1)/etc/init.d
  $(INSTALL_BIN) ./files/etc/init.d/honk $(1)/etc/init.d/honk
  $(INSTALL_DIR) $(1)/etc/uci-defaults
  $(INSTALL_BIN) ./files/etc/uci-defaults/99-honk $(1)/etc/uci-defaults/99-honk
endef

define Package/honk/conffiles
/etc/honk/config.dae
endef

$(eval $(call BuildPackage,honk))
