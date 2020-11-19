## Step 2: Create the Software Components with PetaLinux

A Vitis platform requires software components. Xilinx provides common software images for quick evaluation. Here since we'd like to demonstrate more software environment customization, we'll use the PetaLinux tools to create the Linux image and sysroot with XRT support, together with some more advanced tweaks. Among all the customizations, the XRT installation and ZOCL device tree setup are mandatory. Other customizations are optional. The customization purposes are explained. Please feel free to pick your desired customization.

Yocto or third-party Linux development tools can also be used as long as they produce the same Linux output products as PetaLinux.

### PetaLinux Project Settings

0. Download and install [petalinux](https://www.xilinx.com/support/download/index.html/content/xilinx/en/downloadNav/embedded-design-tools.html(

1. Setup PetaLinux environment: `source <petaLinux_tool_install_dir>/settings.sh`

2. Create a PetaLinux project named ***zedboard_plnx*** and configure the hw with the XSA file we created before:

   ```bash
   petalinux-create --type project --template zynq --name zedboard_plnx
   cd zedboard_plnx
   petalinux-config --get-hw-description=</absolute path/to/zedboard_platform/>
   ```

   After this step, your directory hierarchy looks like this.

   ```
   - zedboard_platform # Vivado Project Directory
   - zedboard_plnx     # PetaLinux Project Directory
   ```

3. A petalinux-config menu is launched, select ***DTG Settings->MACHINE_NAME***, modify it to ```zedboard```. Select ***OK -> Exit*** to go back to the root menu.

4. Select ***Image Packaging Configuration***, select ***Root File System Type*** as ***EXT4***, and append `ext4` to ***Root File System Formats***. Go back to menu root.

5. Change ***DTG settings -> Kernel Bootargs -> generate boot args automatically*** to NO and update ***User Set Kernel Bootargs*** to `earlycon console=ttyPS0,115200 clk_ignore_unused root=/dev/mmcblk0p2 rw rootwait cma=512M`. 

6. Select ***Exit -> Exit -> Exit -> Yes*** to save the configuration.
cat 

### Customize Root File System, Kernel, Device Tree and U-boot

1. Add user packages by appending the CONFIG_x lines below to the ***<your_petalinux_project_dir>/project-spec/meta-user/conf/user-rootfsconfig*** file.

    ```
   CONFIG_packagegroup-petalinux-xrt
   CONFIG_xrt-dev
   CONFIG_dnf
   CONFIG_e2fsprogs-resize2fs
   CONFIG_parted
   CONFIG_packagegroup-petalinux-self-hosted
   CONFIG_cmake
   CONFIG_xrt-dev
   CONFIG_opencl-clhpp-dev
   CONFIG_opencl-headers-dev
   CONFIG_packagegroup-petalinux-opencv
   CONFIG_packagegroup-petalinux-opencv-dev
   CONFIG_glog
   CONFIG_json-c
   CONFIG_protobuf
   CONFIG_python3-pip
   CONFIG_apt
   CONFIG_dpkg
   CONFIG_json-c-dev
   CONFIG_protobuf-dev
    ```

2. Run ```petalinux-config -c rootfs``` and select ***user packages***, select name of rootfs all the libraries listed above.

3. *Enable OpenSSH and disable dropbear*
   *Dropbear is the default SSH tool in Vitis Base Embedded Platform. If OpenSSH is used to replace Dropbear, the system could achieve 4x times faster data transmission speed (tested on 1Gbps Ethernet environment). Since Vitis-AI applications may use remote display feature to show machine learning results, using OpenSSH can improve the display experience.*

   a) Still in the RootFS configuration window, go to root directory by select ***Exit*** once.</br>
   b) Go to ***Image Features***.</br>
   c) Disable ***ssh-server-dropbear*** and enable ***ssh-server-openssh*** and click Exit.</br>
   ![ssh_settings.png](./images/ssh_settings.png)

    d) Go to ***Filesystem Packages-> misc->packagegroup-core-ssh-dropbear*** and disable ***packagegroup-core-ssh-dropbear***. Go to ***Filesystem Packages*** level by Exit twice.

    e) Go to ***console  -> network -> openssh*** and enable ***openssh***, ***openssh-sftp-server***, ***openssh-sshd***, ***openssh-scp***. Go to root level by Exit four times.

4. Enable Package Management

    a) In rootfs config go to ***Image Features*** and enable ***package-management*** and ***debug_tweaks*** option </br>
    b) Set ***package-feed-uris*** to `http://petalinux.xilinx.com/sswreleases/rel-v2020/feeds/zc702-zynq7/`. </br>
    c) Click OK, Exit twice and select Yes to save the changes.

5. *Disable CPU IDLE in kernel config.*

   *CPU IDLE would cause CPU IDLE when JTAG is connected. So it is recommended to disable the selection during project development phase. It can be enabled for production to save power.*</br>
   
   a) Type ```petalinux-config -c kernel```.</br>
   b) Ensure the following items are ***TURNED OFF*** by entering 'n' in the [ ] menu selection: </br>

   - ***CPU Power Mangement > CPU Idle > CPU idle PM support***
   - ***CPU Power Management > CPU Frequency scaling > CPU Frequency scaling***
   </br>
   c) Exit and Save.

6. Update the Device tree.

   Replace the contents of the ***project-spec/meta-user/recipes-bsp/device-tree/files/system-user.dtsi*** file.
   ```
   /include/ "system-conf.dtsi"
   / {
	chosen {
		bootargs = "earlycon console=ttyPS0,115200 clk_ignore_unused root=/dev/mmcblk0p2 rw rootwait cma=512M";
	};
    };


   &amba {
	zyxclmm_drm {
		compatible = "xlnx,zocl";
		status = "okay";
		interrupt-parent = <&axi_intc_0>;
		interrupts = <0  4>, <1  4>, <2  4>, <3  4>,
			     <4  4>, <5  4>, <6  4>, <7  4>,
			     <8  4>, <9  4>, <10 4>, <11 4>,
			     <12 4>, <13 4>, <14 4>, <15 4>,
			     <16 4>, <17 4>, <18 4>, <19 4>,
			     <20 4>, <21 4>, <22 4>, <23 4>,
			     <24 4>, <25 4>, <26 4>, <27 4>,
			     <28 4>, <29 4>, <30 4>, <31 4>;
	};
    };

    &axi_intc_0 {
      xlnx,kind-of-intr = <0x0>;
      xlnx,num-intr-inputs = <0x20>;
      interrupt-parent = <&intc>;
      interrupts = <0 29 4>;
   };
   ```

### Build Image and Prepare for Platform Packaging

1. We would store all the necessary files for Vitis platform creation flow. Here we name it ```zedboard_pkg ```. Then we create a pfm folder inside. 

   ```
   mkdir zedboard_pkg
   cd zedboard_pkg
   mkdir pfm
   ```

   After this step, your directory hierarchy looks like this.

   ```
   - zedboard_platform # Vivado Project Directory
   - zedboard_plnx     # PetaLinux Project Directory
   - zedboard_pkg      # Platform Packaging Directory
     - pfm                  # Platform Packaging Sources
   ```

2. From any directory within the PetaLinux project, build the PetaLinux project.

   ```
   petalinux-build
   ```

3. Create a sysroot self-installer for the target Linux system

   ```
   petalinux-build --sdk
   ```

4. Install sysroot: type ```./images/linux/sdk.sh``` to install PetaLinux SDK, use the `-d` option to provide a full pathname to the output directory ***zedboard_pkg/pfm*** (This is an example ) and confirm.

   - Note: The environment variable ***LD_LIBRARY_PATH*** must not be set when running this command

We would install Vitis AI library and DNNDK into this rootfs during test phase.

***Note: Now HW platform and SW platform are all generated. Next we would [package the Vitis Platform](step3.md).***

<p align="center"><sup>Copyright&copy; 2020 Xilinx</sup></p>
