FROM buildpack-deps:jessie

RUN set -ex; \
	if ! command -v gpg > /dev/null; then \
		apt-get update; \
		apt-get install -y --no-install-recommends \
			gnupg2 \
			dirmngr \
		; \
		rm -rf /var/lib/apt/lists/*; \
	fi

# https://gcc.gnu.org/mirrors.html
ENV GPG_KEYS \
# 1024D/745C015A 1999-11-09 Gerald Pfeifer <gerald@pfeifer.com>
	B215C1633BCA0477615F1B35A5B3A004745C015A \
# 1024D/B75C61B8 2003-04-10 Mark Mitchell <mark@codesourcery.com>
	B3C42148A44E6983B3E4CC0793FA9B1AB75C61B8 \
# 1024D/902C9419 2004-12-06 Gabriel Dos Reis <gdr@acm.org>
	90AA470469D3965A87A5DCB494D03953902C9419 \
# 1024D/F71EDF1C 2000-02-13 Joseph Samuel Myers <jsm@polyomino.org.uk>
	80F98B2E0DAB6C8281BDF541A7C8C3B2F71EDF1C \
# 2048R/FC26A641 2005-09-13 Richard Guenther <richard.guenther@gmail.com>
	7F74F97C103468EE5D750B583AB00996FC26A641 \
# 1024D/C3C45C06 2004-04-21 Jakub Jelinek <jakub@redhat.com>
	33C235A34C46AA3FFB293709A328C3A2C3C45C06
RUN set -ex; \
	for key in $GPG_KEYS; do \
		gpg --keyserver ha.pool.sks-keyservers.net --recv-keys "$key"; \
	done

# Last Modified: 2016-06-03
ENV GCC_VERSION 5.4.0
# Docker EOL: 2017-06-03

RUN set -ex; \
	\
	buildDeps=' \
		dpkg-dev \
		flex \
	'; \
	apt-get update; \
	apt-get install -y --no-install-recommends $buildDeps; \
	rm -r /var/lib/apt/lists/*; \
	\
	curl -fSL "http://ftpmirror.gnu.org/gcc/gcc-$GCC_VERSION/gcc-$GCC_VERSION.tar.bz2" -o gcc.tar.bz2; \
	curl -fSL "http://ftpmirror.gnu.org/gcc/gcc-$GCC_VERSION/gcc-$GCC_VERSION.tar.bz2.sig" -o gcc.tar.bz2.sig; \
	gpg --batch --verify gcc.tar.bz2.sig gcc.tar.bz2; \
	mkdir -p /usr/src/gcc; \
	tar -xf gcc.tar.bz2 -C /usr/src/gcc --strip-components=1; \
	rm gcc.tar.bz2*; \
	\
	cd /usr/src/gcc; \
	\
# "download_prerequisites" pulls down a bunch of tarballs and extracts them,
# but then leaves the tarballs themselves lying around
	./contrib/download_prerequisites; \
	{ rm *.tar.* || true; }; \
	\
# explicitly update autoconf config.guess and config.sub so they support more arches/libcs
	for f in config.guess config.sub; do \
		wget -O "$f" "https://git.savannah.gnu.org/cgit/config.git/plain/$f?id=7d3d27baf8107b630586c962c057e22149653deb"; \
# find any more (shallow) copies of the file we grabbed and update them too
		find -mindepth 2 -name "$f" -exec cp -v "$f" '{}' ';'; \
	done; \
	\
	dir="$(mktemp -d)"; \
	cd "$dir"; \
	\
	extraConfigureArgs=''; \
	dpkgArch="$(dpkg --print-architecture)"; \
	case "$dpkgArch" in \
# with-arch: https://anonscm.debian.org/viewvc/gcccvs/branches/sid/gcc-6/debian/rules2?revision=9450&view=markup#l491
# with-float: https://anonscm.debian.org/viewvc/gcccvs/branches/sid/gcc-6/debian/rules.defs?revision=9487&view=markup#l416
# with-mode: https://anonscm.debian.org/viewvc/gcccvs/branches/sid/gcc-6/debian/rules.defs?revision=9487&view=markup#l376
		armel) \
			extraConfigureArgs="$extraConfigureArgs --with-arch=armv4t --with-float=soft" \
# TODO get enough builders to test and verify arm32v5 O:)
			;; \
		armhf) \
			extraConfigureArgs="$extraConfigureArgs --with-arch=armv7-a --with-float=hard --with-fpu=vfpv3-d16 --with-mode=thumb" \
# TODO make[2]: *** No rule to make target '/usr/src/gcc/gcc/sync-builtinsndef', needed by 'tree-ssa-loop-prefetch.o'.  Stop.
# (which the builds takes _forever_ to get to)
			;; \
		\
# with-arch-32: https://anonscm.debian.org/viewvc/gcccvs/branches/sid/gcc-6/debian/rules2?revision=9450&view=markup#l590
		i386) \
			osVersionID="$(set -e; . /etc/os-release; echo "$VERSION_ID")"; \
			case "$osVersionID" in \
				8) extraConfigureArgs="$extraConfigureArgs --with-arch-32=i586" ;; \
				*) extraConfigureArgs="$extraConfigureArgs --with-arch-32=i686" ;; \
			esac; \
# TODO for some reason, libgo + i386 fails on https://github.com/gcc-mirror/gcc/blob/gcc-7_1_0-release/libgo/runtime/proc.c#L154
# "error unknown case for SETCONTEXT_CLOBBERS_TLS"
			;; \
	esac; \
	\
	gnuArch="$(dpkg-architecture --query DEB_BUILD_GNU_TYPE)"; \
	/usr/src/gcc/configure \
		--build="$gnuArch" \
		--disable-multilib \
		--enable-languages=c,c++,fortran,go \
		$extraConfigureArgs \
	; \
	make -j "$(nproc)"; \
	make install-strip; \
	\
	cd ..; \
	\
	rm -rf "$dir"; \
	\
	apt-get purge -y --auto-remove $buildDeps

# gcc installs .so files in /usr/local/lib64...
RUN set -ex; \
	echo '/usr/local/lib64' > /etc/ld.so.conf.d/local-lib64.conf; \
	ldconfig -v

# ensure that alternatives are pointing to the new compiler and that old one is no longer used
RUN set -ex; \
	dpkg-divert --divert /usr/bin/gcc.orig --rename /usr/bin/gcc; \
	dpkg-divert --divert /usr/bin/g++.orig --rename /usr/bin/g++; \
	dpkg-divert --divert /usr/bin/gfortran.orig --rename /usr/bin/gfortran; \
	update-alternatives --install /usr/bin/cc cc /usr/local/bin/gcc 999
