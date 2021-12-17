# Maintainer:  Philip Müller <philm[at]manjaro[dog]org>
# Contributor: Helmut Stult

# Arch credits:
# Maintainer: Jonathon Fernyhough <jonathon+m2x+dev>
# Contributor: Alonso Rodriguez <alonsorodi20 (at) gmail (dot) com>
# Contributor: Sven-Hendrik Haase <svenstaro@gmail.com>
# Contributor: Thomas Baechler <thomas@archlinux.org>
# Contributor: James Rayner <iphitus@gmail.com>

pkgbase=nvidia-390xx-utils
pkgname=('nvidia-390xx-utils' 'opencl-nvidia-390xx' 'nvidia-390xx-dkms' 'mhwd-nvidia-390xx')
pkgver=390.147
pkgrel=1
arch=('x86_64')
url="https://www.nvidia.com/"
license=('custom')
options=('!strip')
_pkg="NVIDIA-Linux-x86_64-${pkgver}-no-compat32"
source=("https://us.download.nvidia.com/XFree86/Linux-x86_64/${pkgver}/${_pkg}.run"
        'mhwd-nvidia' 'nvidia-drm-outputclass.conf' 'nvidia-390xx-utils.sysusers'
        '30-nvidia-ignoreabi.conf')
sha256sums=('3fc4b5a7c64326cea79156fc31e8160a89621219df09a4cd268844c3e318accc'
            '11176f1c070bbdbfaa01a3743ec065fe71ff867b9f72f1dce0de0339b5873bb5'
            '089d6dc247c9091b320c418b0d91ae6adda65e170934d178cdd4e9bd0785b182'
            'd8d1caa5d72c71c6430c2a0d9ce1a674787e9272ccce28b9d5898ca24e60a167'
            'b03237d533535df35129925b5a348a4b28fd25c51b5e4cf8908df3b14758eba4')

create_links() {
    # create soname links
    find "$pkgdir" -type f -name '*.so*' ! -path '*xorg/*' -print0 | while read -d $'\0' _lib; do
        _soname=$(dirname "${_lib}")/$(readelf -d "${_lib}" | grep -Po 'SONAME.*: \[\K[^]]*' || true)
        _base=$(echo ${_soname} | sed -r 's/(.*)\.so.*/\1.so/')
        [[ -e "${_soname}" ]] || ln -s $(basename "${_lib}") "${_soname}"
        [[ -e "${_base}" ]] || ln -s $(basename "${_soname}") "${_base}"
    done
}

prepare() {
    sh "${_pkg}.run" --extract-only

    cd "${_pkg}"
    bsdtar -xf nvidia-persistenced-init.tar.bz2

    sed -i 's/__NV_VK_ICD__/libGLX_nvidia.so.0/' nvidia_icd.json.template

    cd kernel
    sed -i "s/__VERSION_STRING/${pkgver}/" dkms.conf
    sed -i 's/__JOBS/`nproc`/' dkms.conf
    sed -i 's/__DKMS_MODULES//' dkms.conf
    sed -i '$iBUILT_MODULE_NAME[0]="nvidia"\
DEST_MODULE_LOCATION[0]="/kernel/drivers/video"\
BUILT_MODULE_NAME[1]="nvidia-uvm"\
DEST_MODULE_LOCATION[1]="/kernel/drivers/video"\
BUILT_MODULE_NAME[2]="nvidia-modeset"\
DEST_MODULE_LOCATION[2]="/kernel/drivers/video"\
BUILT_MODULE_NAME[3]="nvidia-drm"\
DEST_MODULE_LOCATION[3]="/kernel/drivers/video"' dkms.conf

    # Gift for linux-rt guys
    sed -i 's/NV_EXCLUDE_BUILD_MODULES/IGNORE_PREEMPT_RT_PRESENCE=1 NV_EXCLUDE_BUILD_MODULES/' dkms.conf
}

package_opencl-nvidia-390xx() {
    pkgdesc="OpenCL implemention for NVIDIA"
    depends=('zlib')
    optdepends=('opencl-headers: headers necessary for OpenCL development')
    provides=("opencl-nvidia=$pkgver" 'opencl-driver')
    conflicts=('opencl-nvidia')

    cd "${_pkg}"

    # OpenCL
    install -D -m644 nvidia.icd "${pkgdir}/etc/OpenCL/vendors/nvidia.icd"
    install -D -m755 "libnvidia-compiler.so.${pkgver}" "${pkgdir}/usr/lib/libnvidia-compiler.so.${pkgver}"
    install -D -m755 "libnvidia-opencl.so.${pkgver}" "${pkgdir}/usr/lib/libnvidia-opencl.so.${pkgver}"

    create_links

    mkdir -p "${pkgdir}/usr/share/licenses"
    ln -s nvidia-utils "${pkgdir}/usr/share/licenses/opencl-nvidia"
}

package_nvidia-390xx-dkms() {
    pkgdesc="NVIDIA drivers - module sources"
    depends=('dkms' "nvidia-390xx-utils=$pkgver" 'libglvnd')
    provides=('NVIDIA-MODULE')

    cd ${_pkg}

    install -dm 755 "${pkgdir}"/usr/src
    cp -dr --no-preserve='ownership' kernel "${pkgdir}/usr/src/nvidia-${pkgver}"

    echo "blacklist nouveau" |
        install -Dm644 /dev/stdin "${pkgdir}/usr/lib/modprobe.d/${pkgname}.conf"

    install -Dt "${pkgdir}/usr/share/licenses/${pkgname}" -m644 "${srcdir}/${_pkg}/LICENSE"
}

package_nvidia-390xx-utils() {
    pkgdesc="NVIDIA drivers utilities"
    depends=('xorg-server' 'libglvnd' 'egl-wayland' 'mhwd' 'jansson' 'gtk3' 'libxv' 'libvdpau' 'libxnvctrl-390xx')
    optdepends=('xorg-server-devel: nvidia-xconfig'
                'opencl-nvidia: OpenCL support')
    provides=('vulkan-driver' 'opengl-driver' "nvidia-utils=$pkgver" 'nvidia-390xx-libgl')
    conflicts=('nvidia-libgl' 'nvidia-utils' 'nvidia-390xx-libgl')
    replaces=('nvidia-libgl')
    install="${pkgname}.install"

    # Exceedingly hacky workaround for Xorg 1.21 until Nvidia updates the driver
    install -D -m644 30-nvidia-ignoreabi.conf "${pkgdir}/etc/X11/xorg.conf.d/30-nvidia-ignoreabi.conf"

    cd "${_pkg}"

    # Check http://us.download.nvidia.com/XFree86/Linux-x86_64/${pkgver}/README/installedcomponents.html
    # for hints on what needs to be installed where.

    # X driver
    install -D -m755 nvidia_drv.so "${pkgdir}/usr/lib/xorg/modules/drivers/nvidia_drv.so"

    # GLX extension module for X
    install -D -m755 "libglx.so.${pkgver}" "${pkgdir}/usr/lib/nvidia/xorg/libglx.so.${pkgver}"
    # Ensure that X finds glx
    ln -s "libglx.so.${pkgver}" "${pkgdir}/usr/lib/nvidia/xorg/libglx.so.1"
    ln -s "libglx.so.${pkgver}" "${pkgdir}/usr/lib/nvidia/xorg/libglx.so"

    # GLVND
    install -D -m755 "libGLX_nvidia.so.${pkgver}" "${pkgdir}/usr/lib/libGLX_nvidia.so.${pkgver}"
    #ln -s "libGLX_nvidia.so.${pkgver}" "${pkgdir}/usr/lib/libGLX_indirect.so.0"
    install -D -m755 "libEGL_nvidia.so.${pkgver}" "${pkgdir}/usr/lib/libEGL_nvidia.so.${pkgver}"

    # EGL and OpenGL ES libraries
    install -D -m755 "libGLESv1_CM_nvidia.so.${pkgver}" "${pkgdir}/usr/lib/libGLESv1_CM_nvidia.so.${pkgver}"
    install -D -m755 "libGLESv2_nvidia.so.${pkgver}" "${pkgdir}/usr/lib/libGLESv2_nvidia.so.${pkgver}"
    install -D -m644 "10_nvidia.json" "${pkgdir}/usr/share/glvnd/egl_vendor.d/10_nvidia.json"

    # OpenGL core library
    install -D -m755 "libnvidia-glcore.so.${pkgver}" "${pkgdir}/usr/lib/libnvidia-glcore.so.${pkgver}"
    install -D -m755 "libnvidia-eglcore.so.${pkgver}" "${pkgdir}/usr/lib/libnvidia-eglcore.so.${pkgver}"
    install -D -m755 "libnvidia-glsi.so.${pkgver}" "${pkgdir}/usr/lib/libnvidia-glsi.so.${pkgver}"

    # misc
    install -D -m755 "libnvidia-ifr.so.${pkgver}" "${pkgdir}/usr/lib/libnvidia-ifr.so.${pkgver}"
    install -D -m755 "libnvidia-fbc.so.${pkgver}" "${pkgdir}/usr/lib/libnvidia-fbc.so.${pkgver}"
    install -D -m755 "libnvidia-encode.so.${pkgver}" "${pkgdir}/usr/lib/libnvidia-encode.so.${pkgver}"
    install -D -m755 "libnvidia-cfg.so.${pkgver}" "${pkgdir}/usr/lib/libnvidia-cfg.so.${pkgver}"
    install -D -m755 "libnvidia-ml.so.${pkgver}" "${pkgdir}/usr/lib/libnvidia-ml.so.${pkgver}"

    # Vulkan ICD
    install -D -m644 "nvidia_icd.json.template" "${pkgdir}/usr/share/vulkan/icd.d/nvidia_icd.json"

    # VDPAU
    install -D -m755 "libvdpau_nvidia.so.${pkgver}" "${pkgdir}/usr/lib/vdpau/libvdpau_nvidia.so.${pkgver}"

    # nvidia-tls library
    install -D -m755 "libnvidia-tls.so.${pkgver}" "${pkgdir}/usr/lib/libnvidia-tls.so.${pkgver}"
    install -D -m755 "tls/libnvidia-tls.so.${pkgver}" "${pkgdir}/usr/lib/tls/libnvidia-tls.so.${pkgver}"

    # CUDA
    install -D -m755 "libcuda.so.${pkgver}" "${pkgdir}/usr/lib/libcuda.so.${pkgver}"
    install -D -m755 "libnvcuvid.so.${pkgver}" "${pkgdir}/usr/lib/libnvcuvid.so.${pkgver}"

    # PTX JIT Compiler (Parallel Thread Execution (PTX) is a pseudo-assembly language for CUDA)
    install -D -m755 "libnvidia-ptxjitcompiler.so.${pkgver}" "${pkgdir}/usr/lib/libnvidia-ptxjitcompiler.so.${pkgver}"

    # Fat (multiarchitecture) binary loader
    install -D -m755 "libnvidia-fatbinaryloader.so.${pkgver}" "${pkgdir}/usr/lib/libnvidia-fatbinaryloader.so.${pkgver}"

    # DEBUG
    install -D -m755 nvidia-debugdump "${pkgdir}/usr/bin/nvidia-debugdump"

    # nvidia-xconfig
    install -D -m755 nvidia-xconfig "${pkgdir}/usr/bin/nvidia-xconfig"
    install -D -m644 nvidia-xconfig.1.gz "${pkgdir}/usr/share/man/man1/nvidia-xconfig.1.gz"

    # nvidia-bug-report
    install -D -m755 nvidia-bug-report.sh "${pkgdir}/usr/bin/nvidia-bug-report.sh"

    # nvidia-smi
    install -D -m755 nvidia-smi "${pkgdir}/usr/bin/nvidia-smi"
    install -D -m644 nvidia-smi.1.gz "${pkgdir}/usr/share/man/man1/nvidia-smi.1.gz"

    # nvidia-cuda-mps
    install -D -m755 nvidia-cuda-mps-server "${pkgdir}/usr/bin/nvidia-cuda-mps-server"
    install -D -m755 nvidia-cuda-mps-control "${pkgdir}/usr/bin/nvidia-cuda-mps-control"
    install -D -m644 nvidia-cuda-mps-control.1.gz "${pkgdir}/usr/share/man/man1/nvidia-cuda-mps-control.1.gz"

    # nvidia-modprobe
    # This should be removed if nvidia fixed their uvm module!
    install -D -m4755 nvidia-modprobe "${pkgdir}/usr/bin/nvidia-modprobe"
    install -D -m644 nvidia-modprobe.1.gz "${pkgdir}/usr/share/man/man1/nvidia-modprobe.1.gz"

    # nvidia-persistenced
    install -D -m755 nvidia-persistenced "${pkgdir}/usr/bin/nvidia-persistenced"
    install -D -m644 nvidia-persistenced.1.gz "${pkgdir}/usr/share/man/man1/nvidia-persistenced.1.gz"
    install -D -m644 nvidia-persistenced-init/systemd/nvidia-persistenced.service.template "${pkgdir}/usr/lib/systemd/system/nvidia-persistenced.service"
    sed -i 's/__USER__/nvidia-persistenced/' "${pkgdir}/usr/lib/systemd/system/nvidia-persistenced.service"

    # application profiles
    install -D -m644 nvidia-application-profiles-${pkgver}-rc "${pkgdir}/usr/share/nvidia/nvidia-application-profiles-${pkgver}-rc"
    install -D -m644 nvidia-application-profiles-${pkgver}-key-documentation "${pkgdir}/usr/share/nvidia/nvidia-application-profiles-${pkgver}-key-documentation"

    install -D -m644 LICENSE "${pkgdir}/usr/share/licenses/nvidia-utils/LICENSE"
    install -D -m644 README.txt "${pkgdir}/usr/share/doc/nvidia/README"
    install -D -m644 NVIDIA_Changelog "${pkgdir}/usr/share/doc/nvidia/NVIDIA_Changelog"
    cp -r html "${pkgdir}/usr/share/doc/nvidia/"
    ln -s nvidia "${pkgdir}/usr/share/doc/nvidia-utils"

    # distro specific files must be installed in /usr/share/X11/xorg.conf.d
    install -D -m644 "${srcdir}/nvidia-drm-outputclass.conf" "${pkgdir}/usr/share/X11/xorg.conf.d/10-nvidia-drm-outputclass.conf"

    install -Dm644 "${srcdir}/nvidia-390xx-utils.sysusers" "${pkgdir}/usr/lib/sysusers.d/$pkgname.conf"

    # nvidia-settings
    install -D -m755 nvidia-settings "${pkgdir}/usr/bin/nvidia-settings"
    install -D -m644 nvidia-settings.1.gz "${pkgdir}/usr/share/man/man1/nvidia-settings.1.gz"
    install -D -m644 nvidia-settings.desktop "${pkgdir}/usr/share/applications/nvidia-settings.desktop"
    install -D -m644 nvidia-settings.png "${pkgdir}/usr/share/pixmaps/nvidia-settings.png"
    install -D -m755 "libnvidia-gtk3.so.$pkgver" "$pkgdir/usr/lib/libnvidia-gtk3.so.$pkgver"
    sed -e 's:__UTILS_PATH__:/usr/bin:' -e 's:__PIXMAP_PATH__:/usr/share/pixmaps:' -i "${pkgdir}/usr/share/applications/nvidia-settings.desktop"

    create_links
}

    package_mhwd-nvidia-390xx() {
    pkgdesc="MHWD module-ids for nvidia $pkgver"
    arch=('any')

    install -d -m755 "${pkgdir}/var/lib/mhwd/ids/pci/"

    # Generate mhwd database
    sh -e ${srcdir}/mhwd-nvidia \
    ${srcdir}/${_pkg}/README.txt \
    ${srcdir}/${_pkg}/kernel/nvidia/nv-kernel.o_binary \
    > ${pkgdir}/var/lib/mhwd/ids/pci/nvidia-390xx.ids
    # add PCIID: 1b82 Nvidia Gforce 1070 Ti
    sed -i 's/1b81 1b84/1b81 1b82 1b84/g' ${pkgdir}/var/lib/mhwd/ids/pci/nvidia-390xx.ids
}
