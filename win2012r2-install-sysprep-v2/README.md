#WIN2012R2 WINM IMAGE

    CREATED: 8/19/2015
    EXPIRES: 2/15/2016

    MD5 (win2012r2-install-sysprep-v2.wim)  = bd02edc47ed3f813fd8db713829e0702
    MD5 (Autounattended-Win2012r2-OOBE.xml) = 2c6dca87fd81977d4498382f37e92fce

###PACKER
This WIM is from the eval win2012 iso. It was built using boxcutter. To
rebuild it place the win2012r2 eval iso into ./iso, untar the boxcutter.tgz
and run the build.sh script. This will use packer (0.8.5 at the time)
to launch the iso into VMware. It is configured with cygwin, a vagrant
user, and some windows tweaks. Then it is shutdown and a vagrant .box
is created. This box can be used over and over to create new vm instances.

###WIM
To create a WIM that can be imaged onto a new machine, we have to boot into
WinPE the windows preinstallation environment. You have to install the 
Windows Advanced Deployment Kit, then open a ADK shell and run the following
to create and mount a clean WinPE image. This image is customized with
drivers and scripts, resulting in a new boot.wim that we copy as winpe.wim

    C:\> copype amd64 C:\WinPE_amd64
    C:\> imagex /mountrw C:\WinPE_amd64\media\sources\boot.wim 1 C:\WinPE_amd64\mount
    C:\> copy C:\path\to\imagex.exe C:\WinPE_amd64\mount\Windows\System32
    C:\> drvinst /inf:C:\Drivers\network\ixfoo.inf /inject:C:\WinPE_amd64\mount
    C:\> drvinst /inf:C:\Drivers\storage\isfoo.inf /inject:C:\WinPE_amd64\mount
    C:\> drvinst /inf:C:\Drivers\usb\iufoo.inf /inject:C:\WinPE_amd64\mount
    C:\> echo wpeinit > C:\WinPE_amd64\mount\Windows\System32\startnet.cmd
    C:\> echo wpeutil WaitForNetwork >> C:\WinPE_amd64\mount\Windows\System32\startnet.cmd
    C:\> echo net use Y: \\10.10.10.10\share >> C:\WinPE_amd64\mount\Windows\System32\startnet.cmd
    C:\> imagex /unmount /commit C:\WinPE_amd64\mount
    C:\> copy C:\WinPE_amd64\media\source\boot.wim Y:\wwwroot\winpe\winpe.wim


###PXE
To winpe.wim must be put unto a webserver so we can PXE boot over HTTP into it.
This is the pxelinux config to chain boot to a http server:

    :winpe
    dhcp net0
    set boot-url http://10.10.10.10/winpe/
    chain --replace ${boot-url}boot.ipxe || goto failed

You can find wimboot [here](http://ipxe.org/wimboot),
the other files are from the win2012r2 ISO, and the winpe.wim is the one we
customized. The boot.ipxe file mentioned about should look like this with
the files being relative to it (in other words this is the layout of
http://10.10.10.10/winpe/).

    #!ipxe
    kernel wimboot
    initrd bootmgr.exe              bootmgr.exe
    initrd boot/bcd                 BCD
    initrd boot/fonts/wgl4_boot.ttf wgl4_boot.ttf
    initrd boot/fonts/chs_boot.ttf  chs_boot.ttf
    initrd boot/fonts/cht_boot.ttf  cht_boot.ttf
    initrd boot/fonts/jpn_boot.ttf  jpn_boot.ttf
    initrd boot/fonts/kor_boot.ttf  kor_boot.ttf
    initrd boot/boot.sdi            boot.sdi
    initrd winpe.wim                boot.wim
    boot

###IMAGEX

PXE boot into your winpe.wim and make sure you can reach the network. The startnet.cmd
file placed into the winpe.wim should've been executed and the share should contain
the wim file produced by the packer step. The drive needs to be partitioned, and the
wim file imaged onto it.

    --snip diskpart.txt --
    list disk
    select disk 0
    clean
    create partition primary
    select partition 1
    active
    format quick compress fs=ntfs label=OS
    assign letter=C
    exit
    --snip--

    X:\> diskpart /s diskpart.txt
    X:\> imagex.exe /apply "Y:\wimfiles\win2012r2-install-sysprep-v2.wim" 1 C:\

Next the new image has to be made bootable.

    X:\> cd /d C:\Windows\System32
    C:\Windows\System32> bcdedit.exe
    C:\Windows\System32> bcdboot.exe C:\Windows /s C: /f ALL
    C:\Windows\System32> bcdedit.exe /timeout 5

Reboot and you're done.

    C:\Windows\System32> wpeutil reboot

###Sysprep Autounattend

The win2012r2-install-sysprep-v2.wim image was sysprepped before it was imaged. With
the sysprep the Autounattended-Win2012r2-OOBE.xml file was included to configure the
initial login. This allows for a fully unattended install. This particular xml runs
`puppet agent --waitforcert` the first time.


    Amit Bakshi
    8/19/2015
