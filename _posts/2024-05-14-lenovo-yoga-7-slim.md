---
layout: post
title: Lenovo Yoga 7 Slim and Linux compatibility
date: 2024-05-14 01:04 +0200
---
I have recently became a happy owner of Lenovo Yoga 7 Slim gen 8 laptop. 

Normally I'd go with Thinkpad (probably x1 version) and things would "just work" on Linux. But this time Thinkpad-land didn't make me particulary happy with whatever it has to offer and so Lenovo Yoga got bought instead. It comes with modern hardware and more risk that things would not work on Linux.

And I was right, some things didn't work. There were mainly two issues:

1. Laptop refused to go to suspend and immediately wake up.
2. Only 2 out of 6 speakers worked

So, let's roll up our sleeves and fix that. While fixes for that are not available through official sources - patches are flying around on the internet. 

I should provide a disclamer, that I'm using Manjaro, but fixes seem universal for most Linux'es. Using 6.9 kernel, but I verified these fixes on 6.7 version too.

# Subwoofers should be blasting!
1. Checkout this repository [https://github.com/darinpp/yoga-slim-7](https://github.com/darinpp/yoga-slim-7)
2. Check that /etc/rc.local exists, if not then add it:
    ```
        echo "#!/bin/sh" > /etc/rc.local
        chmod +x /etc/rc.local
    ```
3. Now then `/etc/rc.local` is in place, add these couple of lines:
    ```
        cat <<EOF >> /etc/rc.local
        # Disable runtime suspend/resume for tas 2781 as it is not working
        echo on > /sys/bus/i2c/drivers/tas2781-hda/i2c-TIAS2781:00/power/control
        EOF
    ```
4.Copy `TIAS2781RCA4.bin` and `TAS2XXX38BB.bin` files from repo into `/lib/firmware`.
    ```
        cp yoga-slim-7/lib/firmware/TIAS2781RCA4.bin /lib/firmware/
        cp yoga-slim-7/lib/firmware/TAS2XXX38BB.bin /lib/firmware/
    ```
5. Now logout and login and you'll have all speakers working.

I didn't had to do any fixes for alsa mixer, somehow it already adjusted volume for all devices. 

# Suspend should actually suspend.

1. Download [slim7-ssdt](https://gitlab.freedesktop.org/drm/amd/uploads/9fe228c7aa403b78c61fb1e29b3b35e3/slim7-ssdt)
2. Copy it into EFI folder.
    ```
       sudo cp slim7-ssdt /boot/
    ```
4. Run `sudo nano /etc/default/grub`.
    - Find the line starting with `GRUB_CMDLINE_LINUX_DEFAULT` and add the parameter `rtc_cmos.use_acpi_alarm=1` at the end, separated by a space.
    - Add this line `GRUB_EARLY_INITRD_LINUX_CUSTOM="slim7-ssdt"`
    - Save the file (Ctrl+X, Y, Enter).
    - Run `sudo update-grub` to apply the changes.

It might be a good idea, to double check that "secure boot" is not enabled in BIOS. But other then that, suspend should work from now on.

Enjoy. It's a great laptop, trully a beast!
