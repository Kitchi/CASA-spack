##
# CASA6 Modular Makefile — Linux / spack variant
#
# Based on the upstream master Makefile. Changes from upstream:
#   - DUseCcache=1 replaced with proper cmake launcher flags throughout
#   - mold linker wired in via cmake linker flags and LDFLAGS for python builds
#   - Hardcoded /usr/lib64/openmpi/bin/ removed; spack env provides openmpi in PATH
#
# Usage:
#   Copy this Makefile into a clean build directory with the spack env active,
#   then: make firstcasa

# Path to patches directory — must be set by user
PATCHDIR    ?=
ifneq ($(PATCHDIR),)
PATCHDIR    := $(realpath $(PATCHDIR))
endif

CASA_BRANCH         = master
CASA_REPO           = https://open-bitbucket.nrao.edu:/scm/casa/casa6.git
CASACORE_DATA_REPO  = ftp://ftp.astron.nl/outgoing/Measures/WSRT_Measures.ztar

CASA_BUILD_TYPE     = RelWithDebInfo
CASACORE_BUILD_TYPE = RelWithDebInfo

LIBSAKURA_VERSION   = 5.2.1
CASASHELL_BRANCH    = master

# Number of cores used for compilation (default: all available in the machine)
NCORES              = $(shell getconf _NPROCESSORS_ONLN)

# Spack view prefix (for library/include paths)
SPACK_VIEW          = $(shell echo $$SPACK_ENV/.spack-env/view)

#oooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooo
#--------------------------------------------------------------------------------------------------------
#
# Package-level dir structure
ROOT        = $(shell pwd)

SRCDIR      = $(ROOT)/src
CASASRC     = $(SRCDIR)/casa6
CASAINSTALL = $(ROOT)/install
CASATESTDIR = $(ROOT)/test
CASAVENVDIR = $(ROOT)/venv
CASABUILD   = $(ROOT)/build


#--------------------------------------------------------------------------------------------------------
#--------------------------------------------------------------------------------------------------------

check-patchdir:
	@if [ -z "$(PATCHDIR)" ]; then \
		echo ""; \
		echo "ERROR: PATCHDIR is not set."; \
		echo ""; \
		echo "  Set it to the spack_env/patches directory, e.g.:"; \
		echo "  make PATCHDIR=/path/to/CASA-spack/linux-aarch64/patches ..."; \
		echo ""; \
		exit 1; \
	fi
	@if [ ! -d "$(PATCHDIR)" ]; then \
		echo ""; \
		echo "ERROR: PATCHDIR=$(PATCHDIR) does not exist."; \
		echo ""; \
		exit 1; \
	fi

firstcasa: check-patchdir init casa-clone libsakura casacore casacpp venv-build casatools casatasks casashell
	@echo  ========================================
	@echo  CASA has been built successfully.
	@echo You can run it with:
	@echo $$ . $(CASAVENVDIR)/bin/activate
	@echo $$ python
	@echo \>\>\> import casatasks

casa: check-patchdir libsakura casacore casacpp venv-build casatools casatasks casashell

clean:
	rm -rf $(SRCDIR) $(CASASRC) $(CASABUILD) $(CASAINSTALL) $(CASATESTDIR) $(CASAVENVDIR)

init:
	mkdir -p $(SRCDIR) $(CASASRC) $(CASABUILD) $(CASAINSTALL) $(CASATESTDIR) $(CASAVENVDIR)

casa-clone: init
	git -C $(SRCDIR) clone -b $(CASA_BRANCH) --recursive $(CASA_REPO)

libsakura:
	curl -L https://github.com/tnakazato/sakura/archive/refs/tags/libsakura-$(LIBSAKURA_VERSION).tar.gz | gunzip | tar -xvf - -C $(SRCDIR)
	cd $(SRCDIR)/sakura-libsakura-$(LIBSAKURA_VERSION) && patch -p1 < $(PATCHDIR)/libsakura-aarch64.patch

	mkdir -p $(CASABUILD)/libsakura
	cmake  \
		-DCMAKE_INSTALL_PREFIX=$(CASAINSTALL) \
		-DCMAKE_BUILD_TYPE=$(CASA_BUILD_TYPE) \
		-DCMAKE_C_COMPILER_LAUNCHER=ccache \
		-DCMAKE_CXX_COMPILER_LAUNCHER=ccache \
		-DBUILD_DOC:BOOL=OFF \
		-DPYTHON_BINDING:BOOL=OFF \
		-DSIMD_ARCH=GENERIC \
		-DENABLE_TEST:BOOL=OFF \
		-DCMAKE_CXX_STANDARD=14 \
		-DCMAKE_POLICY_VERSION_MINIMUM=3.5 \
		$(SRCDIR)/sakura-libsakura-$(LIBSAKURA_VERSION)/libsakura/ \
		-B $(CASABUILD)/libsakura

	$(MAKE) -C $(CASABUILD)/libsakura  install -j $(NCORES)


casacore: casacore-build casacore-configure

casacore-configure:
	if [ ! -d $(CASAINSTALL)/data ]; then \
		mkdir -p $(CASAINSTALL)/data ; \
		curl -L $(CASACORE_DATA_REPO) | gunzip | tar -xvf - -C $(CASAINSTALL)/data ; \
	fi

	mkdir -p $(CASABUILD)/casacore
	cd $(CASABUILD)/casacore
	cmake \
		-DCMAKE_INSTALL_PREFIX=$(CASAINSTALL) \
		-DDATA_DIR=$(CASAINSTALL)/data \
		-DCMAKE_BUILD_TYPE=$(CASACORE_BUILD_TYPE) \
		-DCMAKE_BUILD_PREFIX=$(CASABUILD) \
		-DCMAKE_C_COMPILER_LAUNCHER=ccache \
		-DCMAKE_CXX_COMPILER_LAUNCHER=ccache \
		-DUSE_OPENMP=ON \
		-DUSE_THREADS=ON \
		-DBUILD_FFTPACK_DEPRECATED=ON \
		-DBUILD_TESTING=OFF \
		-DBUILD_PYTHON3=OFF \
		-DBUILD_DYSCO=ON \
		-DPORTABLE=ON \
		-DUSE_PCH=OFF \
		$(CASASRC)/casatools/casacore \
		-B $(CASABUILD)/casacore

casacore-build : casacore-configure
	$(MAKE) -C $(CASABUILD)/casacore install -j $(NCORES)


casacpp: libsakura casacore casacpp-build

casacpp-needs-configure: $(CASABUILD)/casacpp/Makefile

casacpp-configure: clean_casacpp_build $(CASABUILD)/casacpp/Makefile

clean-casacpp-build:
	rm -rf $(CASABUILD)/casacpp

$(CASABUILD)/casacpp/Makefile:
	if [ -d $(CASABUILD)/casacpp ]; then rm -rf $(CASABUILD)/casacpp; fi
	mkdir -p $(CASABUILD)/casacpp
	cd $(CASABUILD)/casacpp
	PKG_CONFIG_PATH=$(CASAINSTALL)/lib/pkgconfig \
		cmake \
		-DCMAKE_INSTALL_PREFIX=$(CASAINSTALL) \
		-DCMAKE_BUILD_TYPE=$(CASA_BUILD_TYPE) \
		-DPKG_CONFIG_USE_CMAKE_PREFIX_PATH=$(CASAINSTALL) \
		-DCMAKE_C_COMPILER_LAUNCHER=ccache \
		-DCMAKE_CXX_COMPILER_LAUNCHER=ccache \
		$(CASASRC)/casatools/src/code \
		-B $(CASABUILD)/casacpp


casacpp-build : casacpp-needs-configure
	$(MAKE) -C $(CASABUILD)/casacpp  install -j $(NCORES)

venv-build: $(CASAVENVDIR)/bin/activate

$(CASAVENVDIR)/bin/activate:
	-deactivate # Disable any running virtual environments
	python3 -m venv $(CASAVENVDIR)
	. $(CASAVENVDIR)/bin/activate

casatools: casacpp casatools-wheel

casatools-wheel: venv-build
	if [ -d $(CASABUILD)/casatools ]; then rm -rf $(CASABUILD)/casatools; fi
	mkdir -p $(CASABUILD)/casatools
	if [ -d $(CASAINSTALL)/dist ]; then rm -rf $(CASAINSTALL)/dist; fi
	mkdir -p $(CASAINSTALL)/dist

	# Run the build in a venv
	# Disable any potentially running virtual environments beforehand
	-deactivate ; \
		. $(CASAVENVDIR)/bin/activate ; \
		pip install build ; \
		export CMAKE_BUILD_PARALLEL_LEVEL=$(NCORES) ; \
		cd $(CASABUILD)/casatools; PKG_CONFIG_PATH=$(CASAINSTALL)/lib/pkgconfig CMAKE_PREFIX_PATH=$(CASAINSTALL):$(SPACK_VIEW) LDFLAGS="-L$(CASAINSTALL)/lib -L$(SPACK_VIEW)/lib" python3 -m build -o $(CASAINSTALL)/dist $(CASASRC)/casatools ; \
		pip uninstall -y casatools ; \
		pip install $(CASAINSTALL)/dist/casatools*whl ; \
		pip install casadata ; \
		deactivate

casatasks: casatools casatasks-wheel

casatasks-wheel: venv-build
	if [ -d $(CASASRC)/casatasks/dist ]; then rm -rf $(CASASRC)/casatasks/dist; fi
	mkdir -p $(CASASRC)/casatasks/dist
	if [ -d $(CASASRC)/casatasks/build ]; then rm -rf $(CASASRC)/casatasks/build; fi

	# Run the build in a venv
	# Disable any potentially running virtual environments beforehand
	-deactivate ; \
		. $(CASAVENVDIR)/bin/activate ; \
		pip install --upgrade setuptools ; \
		pip install --upgrade wheel ; \
		mkdir -p $(HOME)/.casa/data ; \
		cd $(CASASRC)/casatasks ; \
		./setup.py bdist_wheel ; \
		pip uninstall -y casatasks ; \
		pip install $(CASASRC)/casatasks/dist/casatasks*whl ; \
		\cp -f $(CASASRC)/casatasks/dist/casatasks*whl $(CASAINSTALL)/dist

casashell: casatasks casashell-wheel

casashell-wheel: venv-build
	if [ -d $(SRCDIR)/casashell ]; then rm -rf $(SRCDIR)/casashell; fi
	git -C $(SRCDIR) clone -b $(CASASHELL_BRANCH) --recursive https://open-bitbucket.nrao.edu/scm/casa/casashell.git

	# Run the build in a venv
	# Disable any potentially running virtual environments beforehand
	-deactivate ; \
		. $(CASAVENVDIR)/bin/activate ; \
		cd $(SRCDIR)/casashell ; \
		./setup.py bdist_wheel ; \
		pip uninstall -y casashell ; \
		pip install $(SRCDIR)/casashell/dist/casashell*whl ; \
		\cp -f $(SRCDIR)/casashell/dist/casashell*whl $(CASAINSTALL)/dist

# end
