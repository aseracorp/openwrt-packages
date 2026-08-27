#
# Copyright (C) 2026 aseracorp
#
# This is free software, licensed under the GNU General Public License v2.
#
# mergerfs — FUSE based multi-device union mount filesystem driver.
#
# Packaged from the official prebuilt STATIC Linux binaries published at
# https://github.com/trapexit/mergerfs/releases (mergerfs-<ver>-static-linux_*).
# These are static/static-pie ELF executables with no shared-library runtime
# deps (only libc for the optional LD_PRELOAD helper), so they run on musl
# OpenWrt targets; this package merely re-stages them into an .apk.
#
# The CI workflow downloads the archive matching the build architecture into
# package/mergerfs/prebuilt/ before compiling; building from a plain checkout
# without running that step requires the archive to be present there.

include $(TOPDIR)/rules.mk

PKG_NAME:=mergerfs
PKG_VERSION:=2.42.0
PKG_RELEASE:=1

PKG_BUILD_DIR:=$(BUILD_DIR)/$(PKG_NAME)-$(PKG_VERSION)

PKG_MAINTAINER:=aseracorp <noreply@aseracorp.dev>
PKG_LICENSE:=GPL-3.0-or-later
PKG_LICENSE_FILES:=LICENSE

include $(INCLUDE_DIR)/package.mk

MFS_SRC:=$(TOPDIR)/package/mergerfs/prebuilt

define Build/Prepare
	$(call Build/Prepare/Default)
	mkdir -p $(PKG_BUILD_DIR)
	@if [ ! -f "$(MFS_SRC)/usr/local/bin/mergerfs" ]; then \
		echo "ERROR: $(MFS_SRC)/usr/local/bin/mergerfs not found."; \
		echo "Populate package/mergerfs/prebuilt (see build.yml) or use the CI workflow."; \
		exit 1; \
	fi
	cp -a $(MFS_SRC)/. $(PKG_BUILD_DIR)/
endef

define Build/Compile
	@true
endef

define Package/mergerfs
  SECTION:=utils
  CATEGORY:=Utilities
  SUBMENU:=Filesystem
  TITLE:=FUSE filesystem to provide a dynamic union of the underlying directories
  URL:=https://github.com/trapexit/mergerfs
  # Static binaries; the FUSE kernel module (kmod-fuse) is required to mount.
  # preload.so is intentionally not shipped: OpenWrt's shared-lib scanner
  # cannot auto-satisfy its libc.musl dependency inside an .apk, and mergerfs
  # works without it for standard usage.
  DEPENDS:=+libc
endef

define Package/mergerfs/description
	mergerfs is a FUSE based union filesystem. It takes any number of
	source paths and presents them as a single merged path, with policy
	based file placement, caching and error handling.
endef

define Package/mergerfs/install
	$(INSTALL_DIR) $(1)/usr/sbin $(1)/usr/bin $(1)/usr/lib/mergerfs
	$(INSTALL_BIN) $(PKG_BUILD_DIR)/usr/local/bin/mergerfs $(1)/usr/sbin/mergerfs
	$(INSTALL_BIN) $(PKG_BUILD_DIR)/usr/local/bin/mergerfs-fusermount $(1)/usr/bin/mergerfs-fusermount
	$(INSTALL_BIN) $(PKG_BUILD_DIR)/sbin/mount.mergerfs $(1)/usr/sbin/mount.mergerfs
	$(INSTALL_DIR) $(1)/usr/share/man/man1
	$(INSTALL_DATA) $(PKG_BUILD_DIR)/usr/local/share/man/man1/mergerfs.1 $(1)/usr/share/man/man1/
endef

$(eval $(call BuildPackage,mergerfs))