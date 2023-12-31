# Use an official Python runtime as parent image
# FROM buildpack-deps:22.04
FROM ubuntu:22.04

# Set image locale
ENV LANG en_US.UTF-8
ENV LANGUAGE en_US:en
ENV TZ=America/New_York

# Expose some ports to host by default
EXPOSE 8080 8081 8082 8083 8084 8085

# Define Neovim COC (command completion) extensions to be installed
ARG COC='coc-css coc-cssmodules coc-docker coc-fzf-preview coc-git coc-html coc-htmldjango coc-jedi coc-json coc-lua coc-markdownlint coc-perl coc-powershell coc-pydocstring coc-sh coc-sql coc-swagger coc-tsserver coc-xml coc-yaml'

# We'll use Lazygit: here are configs
ARG LG='lazygit'
ARG LG_GITHUB='https://github.com/jesseduffield/lazygit/releases/download/v0.40.2/lazygit_0.40.2_Linux_x86_64.tar.gz'
ARG LG_ARCHIVE='lazygit.tar.gz'

# We'll use python-3.12.0
ARG PYTHON_VERSION='3.12.0'

# Create directory for projects
RUN mkdir -p /app/workspace

# Set the working directory in the container to workspace
WORKDIR /app
RUN echo > /app/workspace/build.log

# Bash aliases
COPY ./home/ /root/

# Update repos and install software
RUN set -eux; \
	echo "Update repos and install software" >> /app/workspace/build.log; \
	apt-get update 2>&1 | tee /app/workspace/build.log; \
	apt-get -y install \
		bash \
		curl \
		fzf \
		ripgrep \
		tree \
		git \
		xclip \
		python3 \
		python3-pip \
		ninja-build \
		gettext \
		libexpat1-dev \
		libtool \
		libtool-bin \
		autoconf \
		automake \
		cmake \
		sed \
		gcc \
		g++ \
		gdb \
		pkg-config \
		zip \
		unzip \
		2>&1 | tee /app/workspace/build.log

# Install nvm
SHELL ["/bin/bash", "-c"]
ENV NVM_DIR="/root/.nvm"
RUN set -eux; \
	echo "Install nvm" >> /app/workspace/build.log; \
	curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.5/install.sh | bash; \
	[ -s "$NVM_DIR/nvm.sh" ] && sed -i '/NVM_SCRIPT_SOURCE/s/$_/bash/' "$NVM_DIR/nvm.sh"; \
	echo done | tee /app/workspace/build.log

# Install node and npm
RUN set -eux; \
	echo "Install node and npm" >> /app/workspace/build.log; \
	[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"; \
	[ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"; \
	nvm install node 2>&1 | tee /app/workspace/build.log; \
	nvm install-latest-npm 2>&1 | tee /app/workspace/build.log

# Python plugins for Nvim + msgpack-rpc API scripting for Nvim processes
RUN set -eux; \
	echo "Install plugins for Nvim" >> /app/workspace/build.log; \
	pip3 install pynvim 2>&1 | tee /app/workspace/build.log

# NodeJS package manager tie-in with Nvim
RUN set -eux; \
	echo "Install NodeJS package manager tie-in with Nvim" >> /app/workspace/build.log; \
	[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"; \
	npm i -g neovim 2>&1 | tee /app/workspace/build.log

# Neovim
RUN set -eux; \
	echo "Install Nvim" >> /app/workspace/build.log; \
	mkdir -p /root/tmp; \
	cd /root/tmp; \
	git clone https://github.com/neovim/neovim; \
	cd /root/tmp/neovim; \
	git checkout stable; \
	nproc="$( nproc )"; \
	make -j${nproc}; \
	make install; \
	rm -rf /root/tmp

# Neovim spell check dictionaries
RUN mkdir -p /root/.local/share/nvim/site/spell
# COPY ./spell/ /root/.local/share/nvim/site/spell/

# Install vim-plug
RUN set -eux; \
	echo "Install vim-plug" >> /app/workspace/build.log; \
	curl -fLo /root/.local/share/nvim/site/autoload/plug.vim --create-dirs https://raw.githubusercontent.com/junegunn/vim-plug/master/plug.vim

# COC extensions
RUN set -eux; \
	echo "Install COC extensions" >> /app/workspace/build.log; \
	mkdir -p /root/.config/coc/extensions; \
	cd /root/.config/coc/extensions; \
	[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"; \
	npm install $COC --global --only=prod 2>&1 | tee /app/workspace/build.log

# Lazygit
RUN set -eux; \
	echo "Install Lazygit" >> /app/workspace/build.log; \
	mkdir -p /root/tmp; \
	cd /root/tmp; \
	curl -L -o $LG_ARCHIVE $LG_GITHUB; \
	tar xzvf $LG_ARCHIVE; \
	mv $LG /usr/bin/; \
	rm -rf /root/tmp

# Install python
ENV PATH /usr/local/bin:$PATH
ENV LANG C.UTF-8
RUN set -eux; \
	echo "Install python" >> /app/workspace/build.log; \
	apt-get install -y --no-install-recommends libbluetooth-dev tk-dev uuid-dev; \
	rm -rf /var/lib/apt/lists/*; \
	mkdir -p /root/tmp; \
	cd /root/tmp; \
	curl -L -o python.tar.xz "https://www.python.org/ftp/python/${PYTHON_VERSION%%[a-z]*}/Python-$PYTHON_VERSION.tar.xz"; \
	mkdir -p /usr/src/python; \
	tar --extract --directory /usr/src/python --strip-components=1 --file python.tar.xz; \
	rm -f python.tar.xz; \
	cd /usr/src/python; \
	gnuArch="$( dpkg-architecture --query DEB_BUILD_GNU_TYPE )"; \
	./configure \
		--build="$gnuArch" \
		--enable-loadable-sqlite-extensions \
		--enable-optimizations \
		--enable-option-checking=fatal \
		--enable-shared \
		--with-lto \
		--with-system-expat \
		--without-ensurepip \
	; \
	nproc="$( nproc )"; \
	EXTRA_CFLAGS="$( dpkg-buildflags --get CFLAGS )"; \
	LDFLAGS="$( dpkg-buildflags --get LDFLAGS )"; \
	make -j "$nproc" \
		"EXTRA_CFLAGS=${EXTRA_CFLAGS:-}" \
		"LDFLAGS=${LDFLAGS:-}" \
		"PROFILE_TASK=${PROFILE_TASK:-}" \
	; \
# prevent accidental usage of a system installed libpython of the same version
	rm python; \
	make -j "$nproc" \
		"EXTRA_CFLAGS=${EXTRA_CFLAGS:-}" \
		"LDFLAGS=${LDFLAGS:--Wl},-rpath='\$\$ORIGIN/..lib'" \
		"PROFILE_TASK=${PROFILE_TASK:-}" \
		python \
	; \
	make install; \
	\
# enable GDB to load debugging data
	bin="$( readlink -ve /usr/local/bin/python3 )"; \
	dir="$( dirname "$bin")"; \
	mkdir -p "/usr/share/gdb/auto-load/$dir"; \
	cp -vL Tools/gdb/libpython.py "/usr/share/gdb/auto-load/$bin-gdb.py"; \
	cd /; \
	rm -rf /usr/src/python; \
	find /usr/local -depth \
		\( \
			\( -type d -a \( -name test -o -name tests -o -name idle_test \) \) \
			-o \( -type f -a \( -name '*.pyc' -o -name '*.pyo' -o -name 'libpython*.a' \) \) \
		\) -exec rm -rf '{}' + \
	; \
	ldconfig; \
	python3 --version | tee -a /app/workspace/build.log

# set LANG back to our preferred locale after building python
ENV LANG en_US.UTF-8

# make some useful symlinks that are expected to exist
RUN set -eux; \
	echo "make symlinks" >> /app/workspace/build.log; \
	for src in idle3 pydoc3 python3 python3-config; do \
		dst="$( echo "$src" | tr -d 3 )"; \
		[ -s "/usr/local/bin/$src" ]; \
		[ ! -e "/usr/local/bin/$dst" ]; \
		ln -svT "$src" "/usr/local/bin/$dst"; \
	done

ENV PYTHON_PIP_VERSION 23.3.1
RUN set -eux; \
	echo "Install pip" >> /app/workspace/build.log; \
	curl -sSL https://bootstrap.pypa.io/get-pip.py -o get-pip.py; \
	export PYTHONDONTWRITEBYTECODE=1; \
	python get-pip.py  \
		--disable-pip-version-check \
		--no-cache-dir \
		--no-compile \
		"pip==$PYTHON_PIP_VERSION" \
	; \
	rm -f get-pip.py; \
	pip --version | tee /app/workspace/build.log


# Install any needed packages specified in requirements.txt
RUN mkdir -p /root/tmp
COPY ./requirements.txt /root/tmp/
RUN set -eux; \
	echo "Install packages from requirements.txt" >> /app/workspace/build.log; \
	test -f /root/tmp/requirements.txt && pip3 install --trusted-host pypi.python.org -r /root/tmp/requirements.txt 2>&1 | tee /app/workspace/build.log

# add current directory contents into the container
# ADD ./workspace /root/workspace

# CMD ["python3"]

# Best current practices:
# 1.  Keep your Dockerfiles lean and efficient.  Avoid installing unnecessary packages and clean up after yourself to
#	keep the image size down.
#
# 2.  Use .dockerignore files.  These work like .gitignore files.  They prevent unwanted files from being added to your
#	Docker images.
#
# 3.  Build your applications to be environment-agnostic as much as possible.  This means minimizing the number of
#	environment-specific configurations you need.
#
# 4.  Use environment variables for configuration.  This allows you to keep sensitive information out of your
#	Dockerfiles.
#
# 5.  Regularly update your images to get the latest security patches.  You can automate this process with CI/CD
#	pipelines.


