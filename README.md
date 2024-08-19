# Distribution Template

This repository serves as a template for new atomic, immutable Arch Linux-based distributions using CommonArch. Included (in this README) is a guide to build your own such distribution using this repository.

CommonArch is a framework and family of Arch Linux-based distributions that run off of OCI container images. Put simply, it enables you to build a Linux distribution from a Dockerfile/Containerfile.

## Requirements

This guide makes the assumption that you already run an existing CommonArch distribution, Arch Linux, or Ubuntu 24.04.

If you're on Windows, Ubuntu or Arch Linux within WSL2 should be sufficient. Follow the guide [here](https://canonical-ubuntu-wsl.readthedocs-hosted.com/en/latest/guides/install-ubuntu-wsl2/) to install Ubuntu in WSL2 on Windows.

You must also have Git installed, either on your host system if you're running Linux, or within your WSL installation on Windows.

## Instructions

1. Begin by using this template to create a new repository on GitHub, and proceed by cloning it locally. Alternatively, you could clone this template repository locally and push it to GitHub as a new repository.

2. In the cloned repository, open `.github/workflows/build.yml`. Here, you can define the desktops you would like to build container images for, which you would later build ISOs from. You can either to choose to build for a subset of the officially supported desktop environments (all of which are included by default; you can remove some of them by editing the list of desktops in the job matrix), or implement support for other desktop environments. However, this guide will cover only the former; adding support for other desktop environments is left as an exercise for the reader.

    ```yaml
    matrix:
        variant: [general, nvidia]
        desktop: [nogui, xfce, mate, budgie, gnome, plasma]
        include:
            - variant: general
            suffix: ''
            - variant: nvidia
            suffix: '-nvidia'
    ```

3. Next, open `Containerfile`. This file is identical to the Dockerfile format in all aspects; you can find the Dockerfile reference [here](https://docs.docker.com/reference/dockerfile). This is where you can make any changes to the container image, for instance by adding `RUN` statements to execute commands within the container image, examples of which are included in the file.

4. If you would like to include any files in the image (for example, for customization), you can do so by copying them to `overlays/common`, which will be merged with the rootfs of the image. As an example, you could include wallpapers by creating the directory `overlays/common/usr/share/backgrounds/` and copying the desired wallpapers over to the newly-created folder.

5. For files specific to each desktop environment you build for (specified in `.github/workflows/build.yml`), copy them to `overlays/<name-of-desktop>`; taking the example of backgrounds again, you could include wallpapers exclusive to the KDE Plasma variant by creating the directory `overlays/plasma/usr/share/backgrounds` and copying the wallpapers there.

6. To install AUR packages, in `Containerfile`, you would want to create a new user (AUR packages cannot be built as root), build and install an AUR helper like `yay` or `paru` as the new user, install any AUR packages you would like to using the helper (for AUR dependency resolution), and finally delete the user. This is demonstrated in the example below, which you could copy to your `Containerfile` (replace `package1 package2 package3` with the list of AUR packages you would like to install):

```docker
RUN useradd -m -s /bin/bash aur && \
    echo "aur ALL=(ALL) NOPASSWD: ALL" > /etc/sudoers.d/aur && \
    mkdir -p /tmp_aur_build && chown -R aur /tmp_aur_build && \
    install-packages-build git base-devel; \
    runuser -u aur -- env -C /tmp_aur_build git clone 'https://aur.archlinux.org/paru-bin.git' && \
    runuser -u aur -- env -C /tmp_aur_build/paru-bin makepkg -si --noconfirm && \
    rm -rf /tmp_aur_build && \
    runuser -u aur -- paru -S --noconfirm package1 package2 package3; \
    userdel -rf aur; rm -rf /home/aur /etc/sudoers.d/aur
```

7. Push your changes to GitHub, and wait for the GitHub Actions workflow run to complete. By default, the image name will be in the format `docker://ghcr.io/<YOUR_USERNAME>/<NAME_OF_REPO>-<NAME_OF_DESKTOP>`, or `docker://ghcr.io/<YOUR_USERNAME>/<NAME_OF_REPO>-nvidia-<NAME_OF_DESKTOP>` for NVIDIA images (images with the proprietary NVIDIA drivers included). For instance, if your GitHub username was `ArchUser` and the name of your GitHub repository `exampledistro`, the name of the GNOME image would be `docker://ghcr.io/ArchUser/exampledistro-gnome`, and for NVIDIA users `docker://ghcr.io/ArchUser/exampledistro-nvidia-gnome`.

8. In the meantime, if you're on Arch Linux (excluding CommonArch; you do not need to install any packages and can skip to the next step if you're already on CommonArch), run the following command:

    ```sh
    sudo pacman -Sy skopeo umoci xorriso grub dosfstools squashfs-tools
    ```

    And if you're on Ubuntu 24.04 (regardless of whether it's in WSL or on real hardware), run:

    ```sh
    sudo apt update; sudo apt install skopeo umoci xorriso grub2 grub-pc grub-efi-amd64 dosfstools systemd-container squashfs-tools
    ```

9. Once the workflow run has completed, clone the repository at https://github.com/CommonArch/iso-builder. Within the cloned repository, run the following command:

    ```sh
    sudo ./iso-builder.sh <NAME_OF_CONTAINER_IMAGE>
    ```

    for each container image you would like to build ISOs from, where `<NAME_OF_CONTAINER_IMAGE>` is the name of the container image (from step 7).

    For example, to build ISOs for each of the example images in step 7, you would run:
    ```sh
    sudo ./iso-builder.sh docker://ghcr.io/ArchUser/exampledistro-gnome
    sudo ./iso-builder.sh docker://ghcr.io/ArchUser/exampledistro-nvidia-gnome
    ```

10. Voila! You should now have installable live ISOs for each of those container images in the current directory. If you're on Windows and completed this guide in WSL2, you can copy the built ISOs from the Linux filesystem in File Explorer. You can test the built ISO on real hardware or in a virtual machine (KVM, Hyper-V, VirtualBox, or VMWare Workstation Player), and it should boot on both legacy BIOS systems and on UEFI systems.