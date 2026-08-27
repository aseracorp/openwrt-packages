#
# Copyright (C) 2026 aseracorp
#
# This is free software, licensed under the GNU General Public License v2.
# See LICENSE for more information.
#
# Cosmos Cloud (<https://cosmos-cloud.io/>) packaged from the official
# prebuilt release archives (`cosmos-cloud-<ver>-<arch>.zip`, published on
# aseracorp/Cosmos-Server and aseracorp/Cosmos-Server-legacyArchs).
#
# Because the Cosmos Cloud runtime is shipped as prebuilt statically-linked
# binaries (Go) plus web assets, this package does NOT compile anything: the
# CI workflow downloads the prebuilt archive matching the build architecture,
# extracts it into package/cosmoscloud/prebuilt, and this Makefile merely
# re-packages those files into an OpenWrt .apk.
#

include $(TOPDIR)/rules.mk

PKG_NAME:=cosmoscloud
# apk requires apk-valid version strings (the upstream '-unstable013' suffix
# is invalid), so PKG_VERSION uses the broad release number. The exact
# prebuilt archive (e.g. 0.23.0-unstable013) is selected by the CI workflow's
# COSMOS_VERSION env var, independent of this field.
PKG_VERSION:=0.23.0
PKG_RELEASE:=1

PKG_BUILD_DIR:=$(BUILD_DIR)/$(PKG_NAME)-$(PKG_VERSION)

PKG_MAINTAINER:=aseracorp <noreply@aseracorp.dev>
PKG_LICENSE:=Apache-2.0 WITH Commons-Clause
PKG_LICENSE_FILES:=LICENCE

include $(INCLUDE_DIR)/package.mk

# Directory populated by the CI workflow with the prebuilt Cosmos Cloud
# release (see .github/workflows/build.yml). Absent when built from a
# plain checkout without running the workflow.
COSMOS_SRC:=$(TOPDIR)/package/cosmoscloud/prebuilt

# Pull the prebuilt tree into the package build dir.
define Build/Prepare
	$(call Build/Prepare/Default)
	mkdir -p $(PKG_BUILD_DIR)
	@if [ ! -d "$(COSMOS_SRC)" ]; then \
		echo "ERROR: $(COSMOS_SRC) not found."; \
		echo "Cosmos Cloud is packaged from prebuilt binaries."; \
		echo "Populate package/cosmoscloud/prebuilt (see build.yml) or use the CI workflow."; \
		exit 1; \
	fi
	cp -a $(COSMOS_SRC)/. $(PKG_BUILD_DIR)/
endef

# Prebuilt binaries — nothing to compile.
define Build/Compile
	@true
endef

define Package/cosmoscloud
  SECTION:=net
  CATEGORY:=Network
  TITLE:=Cosmos Cloud — self-hosted home server secure gateway (prebuilt)
  URL:=https://cosmos-cloud.io/
  # Runtime deps = external commands the static Cosmos daemon shells out to
  # (see azukaar/Cosmos-Server source, exec.* calls). Binaries are statically
  # linked, so no libc/soname deps - these are the tools it invokes at runtime:
  #   bash, curl, yq, coreutils-df (GNU df for disk stats; other coreutils are
  #   covered by busybox), util-linux (mount/lsblk/umount), parted,
  #   mdadm, iptables (iptables-save), fuse3-utils (fusermount), zstd.
  # Feature deps: docker/dockerd (managed apps), samba4-server (shares),
  # avahi-nodbus-daemon (mDNS/discovery), findutils-xargs (constellation).
  # snapraid and mergerfs are CONDITIONAL deps: they are pulled in only on
  # architectures where that package is actually built for the arch
  # (CONFIG_PACKAGE_snapraid / CONFIG_PACKAGE_mergerfs), so the Cosmos package
  # does not force an uninstallable dependency where no matching arch package
  # exists. tailscale/restic remain optional extras (not hard deps).
  DEPENDS:=+bash +curl +yq +coreutils-df +util-linux +parted +mdadm +iptables +fuse3-utils +zstd \
	+docker +dockerd +samba4-server +avahi-nodbus-daemon +findutils-xargs \
	$(if $(CONFIG_PACKAGE_snapraid),+snapraid) \
	$(if $(CONFIG_PACKAGE_mergerfs),+mergerfs)
endef

define Package/cosmoscloud/description
	Cosmos Cloud is a self-hosted home server secure gateway and server
	manager: authentication, let's encrypt HTTPS, reverse proxy, anti-DDoS
	and app management. This package ships the official prebuilt binaries
	(Go, statically linked) and web assets for the build architecture.
endef

define Package/cosmoscloud/conffiles
/etc/cosmos/.env
/etc/cosmos/config.txt
endef

define Package/cosmoscloud/install
	$(INSTALL_DIR) $(1)/opt/cosmos
	cp -a $(COSMOS_SRC)/. $(1)/opt/cosmos/
	$(INSTALL_DIR) $(1)/etc/init.d
	$(INSTALL_BIN) ./files/etc/init.d/CosmosCloud $(1)/etc/init.d/CosmosCloud
	$(INSTALL_DIR) $(1)/etc/cosmos
	touch $(1)/etc/cosmos/.env
	touch $(1)/etc/cosmos/config.txt
endef

$(eval $(call BuildPackage,cosmoscloud))