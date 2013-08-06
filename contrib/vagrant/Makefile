#
# Copyright © 2011-2017 Bitcoin Core Developers.
#
# Permission is hereby granted, free of charge, to any person obtaining a copy
# of this software and associated documentation files (the "Software"), to deal
# in the Software without restriction, including without limitation the rights
# to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
# copies of the Software, and to permit persons to whom the Software is
# furnished to do so, subject to the following conditions:
#
# The above copyright notice and this permission notice shall be included in
# all copies or substantial portions of the Software.
#
# THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
# IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
# FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
# AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
# LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
# OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN
# THE SOFTWARE.
#

MAKEFILE := $(word $(words $(MAKEFILE_LIST)),$(MAKEFILE_LIST))

VERSION  := 0.14

SHELL    := $(shell bash --help >/dev/null 2>&1 && echo ba)sh

GREP     := $(shell egrep --help >/dev/null 2>&1 && echo e)grep
PATCH    := $(shell gpatch --help >/dev/null 2>&1 && echo g)patch
SED      := $(shell gsed --help >/dev/null 2>&1 && echo g)sed
WGET     := wget --no-check-certificate \
                 --user-agent=$(shell wget --version | \
                 $(SED) -n 's,GNU \(Wget\) \([0-9.]*\).*,\1/\2,p')

REQUIREMENTS := $(SHELL) bzip2 cut diff $(GREP) gzip $(MAKE) openssl $(PATCH) \
                ruby $(SED) sort tar vagrant wget xargs xz

BUILD_MANIFEST = cd '$(shell dirname '$(MAKEFILE)')'/../.. && \
	find $(1) -print0 | \
	xargs -0 -n1 openssl dgst -sha256 2>/dev/null | \
	openssl dgst -sha256 | \
	$(SED) -n 's,^.*\([0-9a-f]\{64\}\),\1,p'

VALIDATE_MANIFEST = \
	if [ -f "$(1)" ] && echo "$(2)" | diff "$(1)" - 2>&1 >/dev/null; then \
	    rm -f "$(1)"; \
	fi

# ===----------------------------------------------------------------------===

PKGS            :=
CACHE_DIR       := cache
HOST_CACHE_DIR  := $(CACHE_DIR)
GUEST_CACHE_DIR := /vagrant/$(CACHE_DIR)

SOURCEFORGE_MIRROR := downloads.sourceforge.net

PKG             := MacOSX10.11.sdk.tar.gz
$(PKG)_CHECKSUM := d4c877cd1bdb85369d85039c1940a1317c1c6f08a9873f1b9f25e67d6f7da6a9
$(PKG)_URL      :=
PKGS            += $(PKG)

PKG             := osslsigncode-1.7.1.tar.gz
$(PKG)_CHECKSUM := f9a8cdb38b9c309326764ebc937cba1523a3a751a7ab05df3ecc99d18ae466c9
$(PKG)_URL      := https://$(SOURCEFORGE_MIRROR)/project/osslsigncode/osslsigncode/$(PKG)
PKGS            += $(PKG)

# ===----------------------------------------------------------------------===

VAGRANT_UP = \
	vagrant up

VAGRANT_HALT = \
	vagrant ssh -c "nohup sh -c 'sleep 1 && sudo shutdown -h now' >/dev/null"; \
	until vagrant status | grep poweroff; do \
	    sleep 1; \
	done

VAGRANT_RELOAD = \
	$(VAGRANT_HALT); \
	$(VAGRANT_UP)

VAGRANT_DESTROY = \
	vagrant destroy --force

# ===----------------------------------------------------------------------===

.PHONY: all
all:

.PHONY: mostlyclean
mostlyclean:

.PHONY: clean
clean: mostlyclean
	-rm -rf output
	-rm -f .stamp-*
	-$(call VAGRANT_DESTROY)

.PHONY: distclean
distclean: clean
	-rm -rf cache
	-rm -rf .vagrant

.PHONY: maintainer-clean
maintainer-clean: distclean
	@echo 'This command is intended for maintainers to use; it'
	@echo 'deletes files that may need special tools to rebuild.'

.PHONY: dist
dist: all

# ===----------------------------------------------------------------------===

.PHONY: check-requirements
define CHECK_REQUIREMENT
	@if ! $(1) --help &>/dev/null; then \
	    if ! which $(1) &>/dev/null; then \
	        echo; \
	        echo 'Missing requirement: $(1)'; \
	        echo; \
	        echo 'Please have a look at "README" to ensure'; \
	        echo 'that your system meets all requirements.'; \
	        echo; \
	        exit 1; \
	    fi; \
	fi

endef
define CHECK_REQUIREMENT_VERSION
	@if ! $(1) --version | head -1 | grep ' \($(2)\)$$' >/dev/null; then \
	    echo; \
	    echo 'Wrong version of requirement: $(1)'; \
	    echo; \
	    echo 'Please have a look at "README" to ensure'; \
	    echo 'that your system meets all requirements.'; \
	    echo; \
	    exit 1; \
	fi

endef
check-requirements: .stamp-check-requirements.h
.stamp-check-requirements.h: $(MAKEFILE)
	@echo '[check requirements]'
	$(foreach REQUIREMENT,$(REQUIREMENTS),$(call CHECK_REQUIREMENT,$(REQUIREMENT)))
	$(call CHECK_REQUIREMENT_VERSION,vagrant,1\.[0-9]\.[0-9])
	@touch "$@"

# ===----------------------------------------------------------------------===

.PHONY: download
download: $(addprefix download-,$(PKGS)) \
          .stamp-download-depends.h

PKG_CHECKSUM = \
	openssl dgst -sha256 '$(HOST_CACHE_DIR)/$(1)' 2>/dev/null | $(SED) -n 's,^.*\([0-9a-f]\{64\}\)$$,\1,p'

CHECK_PKG_ARCHIVE = \
	[ x'$($(1)_CHECKSUM)' == x"`$$(call PKG_CHECKSUM,$(1))`" ]

DOWNLOAD_PKG_ARCHIVE = \
	mkdir -p '$(HOST_CACHE_DIR)' && \
	$(if $($(1)_URL_2), \
	    ( $(WGET) -T 30 -t 3 -O- '$($(1)_URL)' || $(WGET) -O- '$($(1)_URL_2)' ), \
	    $(WGET) -O- '$($(1)_URL)') \
	$(if $($(1)_FIX_GZIP), \
	    | gzip -d | gzip -9n, \
	    ) \
	> '$(HOST_CACHE_DIR)/$(1)'

define PKG_RULE
.PHONY: download-$(1)
download-$(1): $(HOST_CACHE_DIR)/$(1)
$(HOST_CACHE_DIR)/$(1): .stamp-check-requirements.h
	@echo -n "Checking cached package $(1)... "; \
	if ! $(call CHECK_PKG_ARCHIVE,$(1)); then \
	        echo "failed"; \
	     $(call DOWNLOAD_PKG_ARCHIVE,$(1)); \
	     $(call CHECK_PKG_ARCHIVE,$(1)) \
	        || { echo 'Wrong checksum!'; exit 1; }; \
	else \
	        echo "ok"; \
	fi

endef
$(foreach PKG,$(PKGS),$(eval $(call PKG_RULE,$(PKG))))

# ===----------------------------------------------------------------------===

HOST_VM_FILES    := contrib/vagrant/Makefile \
                    contrib/vagrant/Vagrantfile \
                    contrib/vagrant/patch/host-vm-etc-sysctl.conf-100-enable_ipv4_forward.patch
HOST_VM_FILES    := $(sort $(HOST_VM_FILES))
HOST_VM_MANIFEST := $(shell $(call BUILD_MANIFEST,$(HOST_VM_FILES)))

_ := $(shell $(call \
	VALIDATE_MANIFEST, \
	.stamp-host-vm.manifest, \
	$(HOST_VM_MANIFEST) \
))

.PHONY: host-vm
host-vm: .stamp-host-vm.manifest
.stamp-host-vm.manifest: \
	.stamp-check-requirements.h \
	$(foreach FILE,$(HOST_VM_FILES),$(shell echo $(FILE) | $(SED) 's:^contrib/vagrant/::g'))
	
	-$(call VAGRANT_DESTROY)
	
	$(call VAGRANT_UP)
	
	vagrant ssh --command "echo 'grub-common hold' | sudo dpkg --set-selections"
	
	vagrant ssh --command "sudo apt-get update"
	vagrant ssh --command "sudo apt-get upgrade -y"
	$(call VAGRANT_RELOAD)
	
	vagrant ssh --command "sudo apt-get install -y \
	    git lzop make patch ruby xz-utils zip"
	vagrant ssh --command "sudo apt-get install -y \
	    debootstrap lxc python-vm-builder qemu-utils"
	
	vagrant ssh --command "sudo apt-get install -y bridge-utils"
	vagrant ssh --command "sudo $(SHELL) -c 'cd / && \
	    patch -p1 < /vagrant/patch/host-vm-etc-sysctl.conf-100-enable_ipv4_forward.patch'"
	
	vagrant ssh --command "sudo apt-get install -y apt-cacher-ng"
	
	# Need to restart networking at before using the host-vm, but might as well
	# shut down the VM at this point since any future build targets will first
	# bring it back up.
	-$(call VAGRANT_HALT)
	
	echo $(HOST_VM_MANIFEST) > "$@"

# ===----------------------------------------------------------------------===

GITIAN_BUILDER_FILES    := contrib/vagrant/Makefile \
                           contrib/vagrant/patch/gitian-builder-ad3f9cc4.patch
GITIAN_BUILDER_FILES    := $(sort $(GITIAN_BUILDER_FILES))
GITIAN_BUILDER_MANIFEST := $(shell $(call BUILD_MANIFEST,$(GITIAN_BUILDER_FILES)))

_ := $(shell $(call \
	VALIDATE_MANIFEST, \
	.stamp-gitian-builder.manifest, \
	$(GITIAN_BUILDER_MANIFEST) \
))

.PHONY: gitian-builder
gitian-builder: .stamp-gitian-builder.manifest
.stamp-gitian-builder.manifest: \
	.stamp-host-vm.manifest \
	$(foreach FILE,$(GITIAN_BUILDER_FILES),$(shell echo $(FILE) | $(SED) 's:^contrib/vagrant/::g'))
	
	$(call VAGRANT_UP)
	
	vagrant ssh --command "\
	    if [ -d ~/gitian-builder ]; then \
	        sudo rm -rf ~/gitian-builder; \
	    fi"
	vagrant ssh --command "\
	    git clone 'https://github.com/devrandom/gitian-builder' ~/gitian-builder"
	vagrant ssh --command "cd ~/gitian-builder && \
	    git checkout ad3f9cc4c2c8c0899961a366f5b9fbd1483b0ee3"
	vagrant ssh --command "cd ~/gitian-builder && \
	    patch -p1 < /vagrant/patch/gitian-builder-ad3f9cc4.patch"
	vagrant ssh --command "cd ~/gitian-builder && \
	    ln -s '$(GUEST_CACHE_DIR)' inputs"
	vagrant ssh --command "cd ~/gitian-builder && \
	    ln -s '$(GUEST_CACHE_DIR)' cache"
	
ifdef VAGRANT_CYCLE
	-$(call VAGRANT_HALT)
endif
	
	echo $(GITIAN_BUILDER_MANIFEST) > "$@"

# ===----------------------------------------------------------------------===

GITIAN_BASE_VM_FILES    := contrib/vagrant/Makefile
GITIAN_BASE_VM_FILES    := $(sort $(GITIAN_BASE_VM_FILES))
GITIAN_BASE_VM_MANIFEST := $(shell $(call BUILD_MANIFEST,$(GITIAN_BASE_VM_FILES)))

_ := $(shell $(call \
	VALIDATE_MANIFEST, \
	"$(HOST_CACHE_DIR)/gitian-base-vm.manifest", \
	$(GITIAN_BASE_VM_MANIFEST) \
))

_ := $(shell \
	if [ ! -e "$(HOST_CACHE_DIR)/gitian-base-vm.manifest" ]; then \
	    rm -f "$(HOST_CACHE_DIR)/gitian-base-vm.tar.lzo"; \
	fi \
)

_ := $(shell $(call \
	VALIDATE_MANIFEST, \
	.stamp-gitian-base-vm.manifest, \
	$(GITIAN_BASE_VM_MANIFEST) \
))

$(HOST_CACHE_DIR)/gitian-base-vm.manifest: $(HOST_CACHE_DIR)/gitian-base-vm.tar.lzo
	echo $(GITIAN_BASE_VM_MANIFEST) > "$@"
$(HOST_CACHE_DIR)/gitian-base-vm.tar.lzo: \
	.stamp-host-vm.manifest \
	.stamp-gitian-builder.manifest \
	$(foreach FILE,$(GITIAN_BASE_VM_FILES),$(shell echo $(FILE) | $(SED) 's:^contrib/vagrant/::g'))
	
	$(call VAGRANT_UP)
	
	vagrant ssh --command "cd ~/gitian-builder && \
	    GITIAN_HOST_IP=10.0.2.15 bin/make-base-vm --lxc --suite trusty --arch amd64"
	vagrant ssh --command "cd ~/gitian-builder && \
	    tar cvfS gitian-base-vm.tar base-*"
	vagrant ssh --command "cd ~/gitian-builder && \
	    lzop --best --delete gitian-base-vm.tar"
	vagrant ssh --command "\
	    mkdir -p '$(GUEST_CACHE_DIR)'"
	vagrant ssh --command "\
	    mv ~/gitian-builder/gitian-base-vm.tar.lzo '$(GUEST_CACHE_DIR)/gitian-base-vm.tar.lzo'"
	
ifdef VAGRANT_CYCLE
	-$(call VAGRANT_HALT)
endif

.stamp-gitian-base-vm.manifest: \
	.stamp-host-vm.manifest \
	.stamp-gitian-builder.manifest \
	$(HOST_CACHE_DIR)/gitian-base-vm.tar.lzo
	
	$(call VAGRANT_UP)
	
	vagrant ssh --command "cd ~/gitian-builder && \
	    rm -f base-*"
	vagrant ssh --command "cd ~/gitian-builder && \
	    tar xvfS '$(GUEST_CACHE_DIR)/gitian-base-vm.tar.lzo'"
	
ifdef VAGRANT_CYCLE
	-$(call VAGRANT_HALT)
endif
	
	echo $(GITIAN_BASE_VM_MANIFEST) > "$@"

.PHONY: gitian-base-vm
gitian-base-vm: \
	$(HOST_CACHE_DIR)/gitian-base-vm.manifest \
	.stamp-gitian-base-vm.manifest

# ===----------------------------------------------------------------------===

ifneq ($(shell git log -1 2>&1 >/dev/null && echo true),true)
$(error Must be called from within a git repository; exiting)
endif

SOURCE_FILES    := $(shell \
	git ls-tree --name-only --full-tree -r `git log -1 --format="%H"` \
	    | $(GREP) -v '^contrib/vagrant/')
SOURCE_FILES    := $(sort $(SOURCE_FILES))
SOURCE_MANIFEST := $(shell $(call BUILD_MANIFEST,$(SOURCE_FILES)))

_ := $(shell $(call \
	VALIDATE_MANIFEST, \
	$(HOST_CACHE_DIR)/src-local.manifest, \
	$(SOURCE_MANIFEST) \
))

_ := $(shell \
	if [ ! -e "$(HOST_CACHE_DIR)/src-local.manifest" ]; then \
	    rm -f "$(HOST_CACHE_DIR)/src-local.tar.xz"; \
	fi \
)

.PHONY: bundle-source
bundle-source: $(HOST_CACHE_DIR)/src-local.manifest
$(HOST_CACHE_DIR)/src-local.manifest: $(HOST_CACHE_DIR)/src-local.tar.xz
	echo $(SOURCE_MANIFEST) >'$@'
$(HOST_CACHE_DIR)/src-local.tar.xz: \
	$(foreach FILE,$(SOURCE_FILES),$(shell echo $(FILE) | $(SED) 's:^:../../:g'))
	
	mkdir -p git-tree
	
	$(SHELL) -c "cd git-tree && git init"
	$(SHELL) -c "cd git-tree && git remote add local ../../.."
	$(SHELL) -c "cd git-tree && git pull --no-tags local HEAD:master"
	
	mkdir -p '$(HOST_CACHE_DIR)'
	rm -f '$(HOST_CACHE_DIR)'/src-local.{tar,tar.xz}
	COPYFILE_DISABLE=1 tar -C git-tree -rf '$(HOST_CACHE_DIR)'/src-local.tar .
	xz --fast '$(HOST_CACHE_DIR)'/src-local.tar
	
	rm -rf git-tree

DEPLOY_SOURCE_BUNDLE = \
        vagrant ssh --command "rm -rf ~/'$(1)' || true"; \
        vagrant ssh --command "mkdir -p ~/'$(1)'"; \
        vagrant ssh --command "tar -C ~/'$(1)' -xf '$(GUEST_CACHE_DIR)/src-local.tar.xz'"

# ===----------------------------------------------------------------------===

.PHONY: download-depends
download-depends: .stamp-download-depends.h
.stamp-download-depends.h: \
	.stamp-host-vm.manifest \
	$(HOST_CACHE_DIR)/src-local.manifest
	
	mkdir -p $(HOST_CACHE_DIR)/common
	
	$(call VAGRANT_UP)
	
	$(call DEPLOY_SOURCE_BUNDLE,bitcoin)
	
	vagrant ssh --command "\
	    make -C ~/bitcoin/depends download SOURCES_PATH='$(GUEST_CACHE_DIR)'/common"
	
	vagrant ssh --command "rm -rf ~/bitcoin"
	
ifdef VAGRANT_CYCLE
	-$(call VAGRANT_HALT)
endif
	
	touch "$@"

# ===----------------------------------------------------------------------===

OUTPUT_DIR         := output
HOST_OUTPUT_DIR    := $(OUTPUT_DIR)
GUEST_OUTPUT_DIR   := /vagrant/$(OUTPUT_DIR)
PATCH_DIR          := patch
HOST_PATCH_DIR     := $(PATCH_DIR)
GUEST_PATCH_DIR    := /vagrant/$(PATCH_DIR)

TARGETS            :=

TARGET             := bitcoin-linux
$(TARGET)_NAME     := $(TARGET)-$(VERSION)
$(TARGET)_OUTPUT   := $($(TARGET)_NAME)-gitian.zip
$(TARGET)_SCRIPT   := contrib/gitian-descriptors/gitian-linux.yml
$(TARGET)_PKGS     :=
$(TARGET)_DEPS     :=
$(TARGET)_FILES    := $($(TARGET)_SCRIPT) \
                      contrib/vagrant/cache/src-local.tar.xz
$(TARGET)_FILES    += $(foreach PACKAGE, \
                                $($(TARGET)_PKGS), \
                                contrib/vagrant/cache/$(PACKAGE))
$(TARGET)_FILES    := $(sort $($(TARGET)_FILES))
$(TARGET)_MANIFEST := $(call BUILD_MANIFEST,$($(TARGET)_FILES))
TARGETS            += $(TARGET)

TARGET             := bitcoin-osx
$(TARGET)_NAME     := $(TARGET)-$(VERSION)
$(TARGET)_OUTPUT   := $($(TARGET)_NAME)-gitian.zip
$(TARGET)_SCRIPT   := contrib/gitian-descriptors/gitian-osx.yml
$(TARGET)_PKGS     :=
$(TARGET)_DEPS     :=
$(TARGET)_FILES    := $($(TARGET)_SCRIPT) \
                      contrib/vagrant/cache/src-local.tar.xz
$(TARGET)_FILES    += $(foreach PACKAGE, \
                                $($(TARGET)_PKGS), \
                                contrib/vagrant/cache/$(PACKAGE))
$(TARGET)_FILES    := $(sort $($(TARGET)_FILES))
$(TARGET)_MANIFEST := $(call BUILD_MANIFEST,$($(TARGET)_FILES))
TARGETS            += $(TARGET)

TARGET             := bitcoin-win
$(TARGET)_NAME     := $(TARGET)-$(VERSION)
$(TARGET)_OUTPUT   := $($(TARGET)_NAME)-gitian.zip
$(TARGET)_SCRIPT   := contrib/gitian-descriptors/gitian-win.yml
$(TARGET)_PKGS     := MacOSX10.11.sdk.tar.gz
$(TARGET)_DEPS     :=
$(TARGET)_FILES    := $($(TARGET)_SCRIPT) \
                      contrib/vagrant/cache/src-local.tar.xz
$(TARGET)_FILES    += $(foreach PACKAGE, \
                                $($(TARGET)_PKGS), \
                                contrib/vagrant/cache/$(PACKAGE))
$(TARGET)_FILES    := $(sort $($(TARGET)_FILES))
$(TARGET)_MANIFEST := $(call BUILD_MANIFEST,$($(TARGET)_FILES))
TARGETS            += $(TARGET)

define TARGET_RULE
.PHONY: $(1)
$(1): $(HOST_OUTPUT_DIR)/$($(1)_OUTPUT)
$(HOST_OUTPUT_DIR)/$($(1)_OUTPUT): \
	.stamp-gitian-base-vm.manifest \
	$(HOST_CACHE_DIR)/src-local.manifest \
	
	mkdir -p $(HOST_OUTPUT_DIR)
	
	$(call VAGRANT_UP)
	
	$(call DEPLOY_SOURCE_BUNDLE,bitcoin)
	
	echo `$($(1)_MANIFEST)` > .$(1).manifest
	
	if [ ! -f $(HOST_CACHE_DIR)/$(1)/`cat .$(1).manifest`/$($(1)_OUTPUT) ]; then \
	    vagrant ssh --command "cd ~/gitian-builder && \
	        USE_LXC=1 LXC_BRIDGE=lxcbr0 LXC_GUEST_IP=10.0.3.5 ./bin/gbuild --package bitcoin=src-local.tar.xz --commit bitcoin=`git log -1 --format="%H"` ../bitcoin/$($(1)_SCRIPT)" || exit 1; \
	    vagrant ssh --command "cd ~/gitian-builder/build/out && \
	        if [ ! -f $($(1)_OUTPUT) ]; then \
	            zip -r $($(1)_OUTPUT) * || exit 1; \
	        fi" || exit 1; \
	    vagrant ssh --command "mkdir -p '$(GUEST_CACHE_DIR)/$(1)/'`cat .$(1).manifest`/" || exit 1; \
	    vagrant ssh --command "cd ~/gitian-builder && \
	        mv build/out/$($(1)_OUTPUT) \
	           '$(GUEST_CACHE_DIR)/$(1)/'`cat .$(1).manifest`/" || exit 1; \
	    vagrant ssh --command "cd ~/gitian-builder && \
	        mv result/$($(1)_NAME)-res.yml \
	           '$(GUEST_CACHE_DIR)/$(1)/'`cat .$(1).manifest`'/$(1)-$($(1)_VERSION)-gitian-res.yml'" || exit 1; \
	    vagrant ssh --command "cd ~/gitian-builder && \
	        mv var/build.log \
	           '$(GUEST_CACHE_DIR)/$(1)/'`cat .$(1).manifest`'/$(1)-$($(1)_VERSION)-gitian-build.log'" || exit 1; \
	    vagrant ssh --command "cd ~/gitian-builder && \
	        mv var/install.log \
	           '$(GUEST_CACHE_DIR)/$(1)/'`cat .$(1).manifest`'/$(1)-$($(1)_VERSION)-gitian-install.log'" || exit 1; \
	fi
	
	cp '$(HOST_CACHE_DIR)/$(1)/'`cat .$(1).manifest`'/$($(1)_OUTPUT)' \
	   '$(HOST_CACHE_DIR)/'
	cp '$(HOST_CACHE_DIR)/$(1)/'`cat .$(1).manifest`'/$($(1)_OUTPUT)' \
	   '$(HOST_OUTPUT_DIR)/'
	cp '$(HOST_CACHE_DIR)/$(1)/'`cat .$(1).manifest`'/$(1)-$($(1)_VERSION)-gitian-res.yml' \
	   '$(HOST_OUTPUT_DIR)/'
	cp '$(HOST_CACHE_DIR)/$(1)/'`cat .$(1).manifest`'/$(1)-$($(1)_VERSION)-gitian-build.log' \
	   '$(HOST_OUTPUT_DIR)/'
	cp '$(HOST_CACHE_DIR)/$(1)/'`cat .$(1).manifest`'/$(1)-$($(1)_VERSION)-gitian-install.log' \
	   '$(HOST_OUTPUT_DIR)/'
	
	rm -f .$(1).manifest
	
	vagrant ssh --command "rm -rf ~/bitcoin"
	
ifdef VAGRANT_CYCLE
	-$(call VAGRANT_HALT)
endif

endef
$(foreach TARGET,$(TARGETS),$(eval $(call TARGET_RULE,$(TARGET))))

.PHONY: all
all: $(foreach TARGET,$(TARGETS),$(shell echo $(TARGET)))

#
# End of File
#
