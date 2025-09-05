# COPR Makefile for automated beanstalkd builds from git master
# Builds from latest git commit like nginx-mod-nchan

# COPR provides outdir, set default for local testing
outdir ?= .

# Default target for COPR
srpm:
	@echo "=== Starting beanstalkd SRPM build from GitHub master ==="
	@echo "Output directory: $(outdir)"
	@echo "Working directory: $$(pwd)"
	@echo "User: $$(whoami)"

	# Install git if not available
	@if ! command -v git >/dev/null 2>&1; then \
		echo "=== Git not found, installing it ==="; \
		if command -v dnf >/dev/null 2>&1; then \
			echo "Using DNF to install git..."; \
			dnf install -y git || (echo "ERROR: Failed to install git with dnf" && exit 1); \
		elif command -v yum >/dev/null 2>&1; then \
			echo "Using YUM to install git..."; \
			yum install -y git || (echo "ERROR: Failed to install git with yum" && exit 1); \
		else \
			echo "ERROR: No package manager found to install git"; \
			exit 1; \
		fi; \
		echo "Git installed successfully"; \
	else \
		echo "=== Git is already available ==="; \
	fi

	# Verify git is now available
	@which git || (echo "ERROR: git still not found after installation attempt" && exit 1)

	# Clone beanstalkd from GitHub
	@echo "=== Cloning beanstalkd from GitHub ==="; \
	rm -rf beanstalkd-master; \
	git clone --depth=50 https://github.com/beanstalkd/beanstalkd.git beanstalkd-master || exit 1; \
	\
	echo "=== Getting beanstalkd commit info ==="; \
	COMMIT=$$(cd beanstalkd-master && git rev-parse HEAD); \
	SHORTCOMMIT=$$(cd beanstalkd-master && git rev-parse --short HEAD); \
	VERSION=$$(cd beanstalkd-master && (git describe --tags --abbrev=0 2>/dev/null | sed 's/^v//') || echo "1.13"); \
	echo "Commit: $$COMMIT"; \
	echo "Short commit: $$SHORTCOMMIT"; \
	echo "Version: $$VERSION"; \
	\
	echo "=== Determining release number ==="; \
	FULL_VERSION="$$VERSION"_"$$SHORTCOMMIT"; \
	FEDORA_VER=$$(rpm -E %fedora 2>/dev/null || echo "42"); \
	echo "Checking COPR for existing builds of version $$FULL_VERSION on Fedora $$FEDORA_VER"; \
	COPR_URL="https://download.copr.fedorainfracloud.org/results/reversejames/beanstalkd/fedora-$$FEDORA_VER-x86_64"; \
	BUILD_DIRS=$$(curl -s "$$COPR_URL/" 2>/dev/null | grep -oE '[0-9]+-beanstalkd' | head -10); \
	EXISTING_RELEASES=""; \
	for dir in $$BUILD_DIRS; do \
		RPMS=$$(curl -s "$$COPR_URL/$$dir/" 2>/dev/null | grep -oE "beanstalkd-$$VERSION-[0-9]+\.git$$SHORTCOMMIT\.fc[0-9]+\.x86_64\.rpm" | head -1); \
		if [ -n "$$RPMS" ]; then \
			REL=$$(echo "$$RPMS" | sed "s/.*-$$VERSION-\([0-9]\+\)\.git$$SHORTCOMMIT\.fc[0-9]\+\.x86_64\.rpm/\1/"); \
			EXISTING_RELEASES="$$EXISTING_RELEASES $$REL"; \
		fi; \
	done; \
	if [ -n "$$EXISTING_RELEASES" ]; then \
		MAX_REL=$$(echo $$EXISTING_RELEASES | tr ' ' '\n' | sort -n | tail -1); \
		RELEASE_NUM=$$(($$MAX_REL + 1)); \
		echo "Found existing release $$MAX_REL, incrementing to $$RELEASE_NUM"; \
	else \
		RELEASE_NUM=1; \
		echo "No existing builds found for $$FULL_VERSION, starting with release 1"; \
	fi; \
	\
	echo "=== Creating beanstalkd tarball ==="; \
	tar czf beanstalkd-$$SHORTCOMMIT.tar.gz -C . beanstalkd-master --transform "s/^beanstalkd-master/beanstalkd-$$COMMIT/" || exit 1; \
	\
	echo "=== Creating systemd service file ==="; \
	( \
	echo '[Unit]'; \
	echo 'Description=Beanstalkd Fast Workqueue Service'; \
	echo 'After=network.target'; \
	echo ''; \
	echo '[Service]'; \
	echo 'User=beanstalkd'; \
	echo 'Type=simple'; \
	echo 'EnvironmentFile=-/etc/sysconfig/beanstalkd'; \
	echo 'ExecStart=/usr/bin/beanstalkd $$ADDR $$PORT $$USER $$MAX_JOB_SIZE $$BINLOG_DIR $$BINLOG_SIZE $$BINLOG_FSYNC_PERIOD'; \
	echo 'Restart=always'; \
	echo ''; \
	echo '[Install]'; \
	echo 'WantedBy=multi-user.target'; \
	) > beanstalkd.service || exit 1; \
	\
	echo "=== Creating sysconfig file ==="; \
	( \
	echo '# System configuration for the beanstalkd daemon'; \
	echo ''; \
	echo '#'; \
	echo '# beanstalkd commandline options'; \
	echo '#'; \
	echo ''; \
	echo 'ADDR=-l 0.0.0.0'; \
	echo 'PORT=-p 11300'; \
	echo 'USER=-u beanstalkd'; \
	echo ''; \
	echo '# Job size is left to the default. Uncomment and set it'; \
	echo '# to have it take affect.'; \
	echo '#MAX_JOB_SIZE=-z 65535'; \
	echo ''; \
	echo '# Using the binlog is off by default.'; \
	echo '#'; \
	echo '# The direcory to house the binlog.'; \
	echo '#BINLOG_DIR=-b /var/lib/beanstalkd/binlog'; \
	echo '#'; \
	echo '# fsync the binlog at most once every N milliseconds.'; \
	echo '# setting this to 0 means '"'"'always fsync'"'"'. If this is unset,'; \
	echo '# and the binlog is used, then no explicit fsync is ever'; \
	echo '# performed.  That is, the -F option is used.'; \
	echo '#BINLOG_FSYNC_PERIOD=-f some-value'; \
	echo '#'; \
	echo '# The size of each binlog file.  This is rounded'; \
	echo '# up to the nearest 512 byte boundary.'; \
	echo '#BINLOG_SIZE=-s 10485760'; \
	) > beanstalkd.sysconfig || exit 1; \
	\
	echo "=== Generating spec file ==="; \
	( \
	echo '%global _hardened_build 1'; \
	echo ''; \
	echo "# Git commit hash for tracking upstream changes"; \
	echo "%global commit $$COMMIT"; \
	echo '%global shortcommit %(c=%{commit}; echo $${c:0:7})'; \
	echo ''; \
	echo '%define beanstalkd_user      beanstalkd'; \
	echo '%define beanstalkd_group     %{beanstalkd_user}'; \
	echo '%define beanstalkd_home      %{_localstatedir}/lib/beanstalkd'; \
	echo '%define beanstalkd_binlogdir %{beanstalkd_home}/binlog'; \
	echo ''; \
	echo 'Name:           beanstalkd'; \
	echo "Version:        $$VERSION"; \
	echo "Release:        $$RELEASE_NUM.git%{shortcommit}%{?dist}"; \
	echo 'Summary:        A simple, fast work-queue service'; \
	echo ''; \
	echo 'License:        MIT'; \
	echo 'URL:            https://beanstalkd.github.io/'; \
	echo '# Using git snapshot from master branch'; \
	echo 'Source0:        beanstalkd-%{shortcommit}.tar.gz'; \
	echo 'Source1:        beanstalkd.service'; \
	echo 'Source2:        beanstalkd.sysconfig'; \
	echo ''; \
	echo 'BuildRequires:  systemd gcc gcc-c++'; \
	echo ''; \
	echo 'Requires(pre):    shadow-utils'; \
	echo 'Requires(post):   systemd'; \
	echo 'Requires(preun):  systemd'; \
	echo 'Requires(postun): systemd'; \
	echo ''; \
	echo '%description'; \
	echo 'beanstalkd is a simple, fast work-queue service. Its interface is generic,'; \
	echo 'but was originally designed for reducing the latency of page views in'; \
	echo 'high-volume web applications by running most time-consuming tasks'; \
	echo 'asynchronously.'; \
	echo ''; \
	echo '%prep'; \
	echo '%autosetup -n beanstalkd-%{commit}'; \
	echo ''; \
	echo '%build'; \
	echo '# Fix signal handler function signature'; \
	echo "sed -i 's/handle_sigterm_pid1()/handle_sigterm_pid1(int sig)/' main.c"; \
	echo '# Remove -Werror from the Makefile to avoid build failures on warnings'; \
	echo "sed -i 's/-Werror//' Makefile"; \
	echo 'make LDFLAGS="%{?__global_ldflags}" CFLAGS="%{optflags} -Wno-stringop-truncation" %{?_smp_mflags}'; \
	echo ''; \
	echo '%check'; \
	echo 'make check'; \
	echo ''; \
	echo '%install'; \
	echo 'make install PREFIX=%{buildroot}%{_prefix}'; \
	echo '%{__install} -p -D -m 0644 %{SOURCE1} %{buildroot}%{_unitdir}/beanstalkd.service'; \
	echo '%{__install} -p -D -m 0644 %{SOURCE2} %{buildroot}%{_sysconfdir}/sysconfig/beanstalkd'; \
	echo '%{__install} -d -m 0755 %{buildroot}%{beanstalkd_home}'; \
	echo '%{__install} -d -m 0755 %{buildroot}%{beanstalkd_binlogdir}'; \
	echo '%{__install} -d -m 00755 %{buildroot}%{_mandir}/man1'; \
	echo '%{__install} -p -m 0644 doc/beanstalkd.1 %{buildroot}%{_mandir}/man1/'; \
	echo ''; \
	echo '%pre'; \
	echo 'getent group %{beanstalkd_group} >/dev/null || groupadd -r %{beanstalkd_group}'; \
	echo 'getent passwd %{beanstalkd_user} >/dev/null || \'; \
	echo '    useradd -r -g %{beanstalkd_user} -d %{beanstalkd_home} -s /sbin/nologin \'; \
	echo '    -c "beanstalkd user" %{beanstalkd_user}'; \
	echo 'exit 0'; \
	echo ''; \
	echo '%post'; \
	echo '# make the binlog dir after installation, this is so SELinux does not complain'; \
	echo '# about the init script creating the binlog directory'; \
	echo '# See RhBug 558310'; \
	echo 'if [ -d %{beanstalkd_home} ]; then'; \
	echo '    %{__install} -d %{beanstalkd_binlogdir} -m 0755 \'; \
	echo '        -o %{beanstalkd_user} -g %{beanstalkd_group} \'; \
	echo '        %{beanstalkd_binlogdir}'; \
	echo 'fi'; \
	echo '%systemd_post beanstalkd.service'; \
	echo ''; \
	echo '%preun'; \
	echo '%systemd_preun beanstalkd.service'; \
	echo ''; \
	echo '%postun'; \
	echo '%systemd_postun_with_restart beanstalkd.service'; \
	echo ''; \
	echo '%files'; \
	echo '%license LICENSE'; \
	echo '%{_unitdir}/beanstalkd.service'; \
	echo '%{_bindir}/beanstalkd'; \
	echo '%{_mandir}/man1/beanstalkd.1*'; \
	echo '%config(noreplace) %{_sysconfdir}/sysconfig/beanstalkd'; \
	echo '%attr(0755,%{beanstalkd_user},%{beanstalkd_group}) %dir %{beanstalkd_home}'; \
	echo '%ghost %attr(0755,%{beanstalkd_user},%{beanstalkd_group}) %dir %{beanstalkd_binlogdir}'; \
	echo ''; \
	echo '%changelog'; \
	echo "* $$(date '+%a %b %d %Y') COPR Builder - $$VERSION-$$RELEASE_NUM.git%{shortcommit}"; \
	echo '- Automated build from GitHub master branch'; \
	echo '- Git commit: %{commit}'; \
	) > beanstalkd.spec || exit 1; \
	echo "Spec file generated"; \
	\
	echo "=== Building SRPM ==="; \
	rpmbuild -bs \
		--define "_sourcedir $$(pwd)" \
		--define "_specdir $$(pwd)" \
		--define "_builddir $$(pwd)" \
		--define "_srcrpmdir $(outdir)" \
		--define "_rpmdir $$(pwd)" \
		beanstalkd.spec || exit 1; \
	\
	echo "=== SRPM build complete ==="; \
	ls -la $(outdir)/*.src.rpm

clean:
	@rm -f *.tar.gz *.spec *.src.rpm *.service *.sysconfig *.rpm
	@rm -rf beanstalkd-master

.PHONY: srpm clean