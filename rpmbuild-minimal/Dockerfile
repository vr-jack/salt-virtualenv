FROM fedora:24

# Install packages
RUN dnf -y install mock rpm-build vim-enhanced

# Tweak vim config
RUN echo "colorscheme elflord" >>/root/.vimrc
RUN echo "noremap <leader>\ :nohlsearch<CR>" >>/root/.vimrc

# Add a non-privileged user and add to mock group so it has build privs
RUN useradd -m builder
RUN gpasswd -a builder mock
RUN echo "cat /etc/motd" >>/home/builder/.bash_profile

# Set up build env
WORKDIR /home/builder
RUN echo "_topdir /home/builder/rpmbuild" >.rpmmacros
RUN mkdir -p rpmbuild/{BUILD,BUILDROOT,SOURCES,SPECS,SRPMS,RPMS}
RUN mkdir bin

# Copy files
COPY mockbuild bin/
COPY motd /etc/motd
COPY SRPMS/* rpmbuild/SRPMS/
COPY RPMS/* rpmbuild/RPMS/

# Fix ownership
RUN chown -R builder:builder /home/builder

# Expand the SRPM so users can inspect it later
RUN su - builder -c 'rpm -ivh /home/builder/rpmbuild/SRPMS/salt-2016.3.4-2.el5_erik.src.rpm'

CMD su - builder
