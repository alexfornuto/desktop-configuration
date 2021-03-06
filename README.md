# Desktop Configuration

* Current distribution: **Fedora 27**
* Current hardware: **ThinkPad T580**

## Data to Back Up
* `~/.gnupg/`
* `~/.password-store/`
* `~/.gitconfig`

## Machine Setup
1. Initialize a thumb drive using the [Fedora Media Writer](https://fedoraproject.org/wiki/How_to_create_and_use_Live_USB#Quickstart:_Using_Fedora_Media_Writer).
2. Boot to the USB drive.
3. Reclaim disk space. Disk encryption is good; I use [Opal](https://en.wikipedia.org/wiki/Opal_Storage_Specification) from my ThinkPad BIOS setup, but you can use [LUKS](https://en.wikipedia.org/wiki/Linux_Unified_Key_Setup). I prefer Opal because GNOME Software Center updates require two reboots, and Opal can persist across reboots.
4. Set up a single admin user (no password set for `root`).
5. Reboot into the newly installed Fedora.
6. If installed from a Live ISO, update Fedora using the GNOME Software Center. (A direct `dnf upgrade` can, rarely, cause issues.)
7. Configure the GNOME desktop:

       gsettings set org.gnome.desktop.input-sources xkb-options "['caps:ctrl_modifier']"  # Use Caps Lock as Ctrl
       gsettings set org.gnome.mutter dynamic-workspaces false  # Use static workspaces.
       gsettings set org.gnome.desktop.wm.preferences num-workspaces 1  # Disable all workspace functionality.
       gsettings set org.gnome.desktop.interface clock-show-seconds true
       gsettings set org.gnome.desktop.interface clock-show-date true
       gsettings set org.gnome.desktop.peripherals.mouse natural-scroll true  # Use Apple-style natural scrolling.
       gsettings set org.gnome.shell enabled-extensions "[]"  # Disable the Fedora desktop logo.
       gsettings set org.gnome.desktop.interface clock-show-date true # Show date in the status bar.

       # Terminal:
       gsettings set "org.gnome.Terminal.Legacy.Keybindings:/org/gnome/terminal/legacy/keybindings/" reset-and-clear 'F12'  # Set F12 to Reset and Clear
       TPROFILE=$(gsettings get org.gnome.Terminal.ProfilesList default | tr --delete "'")
       gsettings set "org.gnome.Terminal.Legacy.Profile:/org/gnome/terminal/legacy/profiles:/:$TPROFILE/" scrollback-unlimited true  # Enable unlimited scrollback.

8. Install Google Chrome:

       sudo dnf install https://dl.google.com/linux/direct/google-chrome-stable_current_x86_64.rpm

9. Add [RPM Fusion](https://rpmfusion.org/) repositories:

       sudo dnf install https://download1.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm https://download1.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-$(rpm -E %fedora).noarch.rpm

10. Install other packages:

       sudo dnf install gimp htop inkscape iotop mariadb meld nano php-cli powertop quassel-client tor unbound wireshark-gnome transmission gnome-system-log fatsort nmap-frontend pass ghex composer gnome-builder libvirt-daemon-config-network chrome-gnome-shell

11. Set the editor (in this case, nano, but anything will work):

       mkdir -p ~/.config/environment.d/
       echo "EDITOR=nano" > ~/.config/environment.d/50-editor.conf

12. Add the [Caffeine GNOME Shell extension](https://extensions.gnome.org/extension/517/caffeine/).

13. Configure git:

       git config --global user.name "David Strauss"
       git config --global user.email name@example.com
       git config --global color.ui auto

## Smart Cards

### Tested Hardware

* Reader and card: [YubiKey Neo](https://www.yubico.com/products/yubikey-hardware/yubikey-neo/) and Neo-N
* Reader and card: [YubiKey 4](https://www.yubico.com/products/yubikey-hardware/yubikey4/) and 4 Nano
* Card only: [Fidesmo Dual Interface](http://shop.fidesmo.com/product/fidesmo-card-dual-interface)
* Reader only: [JK-A0100 Series Smartcard Keyboard](http://cherryamericas.com/product/jk-a0100eu-smartcard-keyboard/): Use `enable-pinpad-varlen` in `.gnupg/gpg-agent.conf` for secure PIN entry. The specific tested model was JK-A0100EU-2.
* Reader only: Identiv SCM SPR 532: Should work with secure PIN entry out of the box
* Reader only: Lenovo ThinkPad T560 built-in: No secure PIN entry available
* [LWN Article: A comparison of cryptographic keycards](https://lwn.net/Articles/736231/)

### Machine Setup

1. Disable the GNOME Keyring SSH agent by overriding the desktop file:

       mkdir -p ~/.config/autostart/

       cat <<EOT >> ~/.config/autostart/gnome-keyring-ssh.desktop
       [Desktop Entry]
       Type=Application
       Name=SSH Key Agent
       Exec=/usr/bin/true
       Hidden=true
       EOT

2. Redirect sessions to use the GPG agent for SSH:

       mkdir -p ~/.config/environment.d/

       cat <<EOT >> ~/.config/environment.d/50-ssh-agent.conf
       SSH_AGENT_PID=
       SSH_AUTH_SOCK=${XDG_RUNTIME_DIR}/gnupg/S.gpg-agent.ssh
       EOT

3. Add a user service and corresponding sockets for the GPG agent:

       mkdir -p ~/.config/systemd/user/

       cat <<EOT >> ~/.config/systemd/user/gpg-agent.service
       [Service]
       ExecStart=/usr/bin/gpg-agent --supervised --enable-ssh-support
       ExecReload=/usr/bin/gpgconf --reload gpg-agent
       EOT

       cat <<EOT >> ~/.config/systemd/user/gpg-agent.socket
       [Socket]
       ListenStream=%t/gnupg/S.gpg-agent
       FileDescriptorName=std
       SocketMode=0600
       DirectoryMode=0700

       [Install]
       WantedBy=sockets.target
       EOT

       cat <<EOT >> ~/.config/systemd/user/gpg-agent-ssh.socket
       [Socket]
       ListenStream=%t/gnupg/S.gpg-agent.ssh
       FileDescriptorName=ssh
       Service=gpg-agent.service
       SocketMode=0600
       DirectoryMode=0700

       [Install]
       WantedBy=sockets.target
       EOT

       systemctl --user enable --now gpg-agent.socket gpg-agent-ssh.socket

### Using an Existing Smart Card

1. Complete machine setup (above).
2. Import any existing smart card keys (that were set up according to the directions below):

       gpg2 --card-edit
       > fetch
       > quit
       gpg2 --card-status

3. Import any other keys:

       gpg2 --keyserver keys.gnupg.net --recv-key $KEYID
       gpg2 --keyserver pool.sks-keyservers.net --recv-key $KEYID  # Another database to try.
       gpg2 --keyserver pgp.mit.edu --recv-key $KEYID  # Another database to try.

4. Add trust to any necessary keys:

       gpg2 --edit-key $KEYID
       gpg> trust
       Your decision? 5
       gpg> quit

### Setting Up a New Smart Card

1. Complete machine setup (above).
2. Install the personalization tool:

       sudo dnf install ykpers

3. If the smart card is a YubiKey Neo or YubiKey 4, set the card's mode:

        ykpersonalize -m86

4. Configure the card, generate a key pair, and upload the key:

       gpg2 --change-pin  # Change both the PIN (default is 123456)
                          # and the Admin PIN (default is 12345678).
                          # I use pwgen for the admin PIN.
       gpg2 --card-edit
       gpg/card> admin
       gpg/card> generate # No off-card key backup.
                          # Use a key size of 3072 for everything on YubiKey 4. Default is fine for NEO.
                          # No expiration.
                          # Back up the .rev file.
       gpg/card> quit  # GPG will then print out data, including the key fingerprint
                       # as a long, alphanumeric string.
       gpg2 --keyserver hkps://keys.gnupg.net --send-keys $FINGERPRINT
       gpg2 --keyserver hkp://pool.sks-keyservers.net --send-keys $FINGERPRINT
       gpg2 --keyserver hkp://pgp.mit.edu --send-keys $FINGERPRINT

    **Note:** Revocation certificates are backed up to `~/.gnupg/openpgp-revocs.d`

5. Display the public key in OpenSSH format:

       ssh-add -L

6. Optionally, export the "secret key," which will only be a stub (not the actual key, which is not obtainable):

       gpg2 --export-secret-key --armor $FINGERPRINT > $FINGERPRINT.asc  # Back this up, too.

7. After this is finished, the card should work. You should also have `$FINGERPRINT.asc` and `$FINGERPRINT.rev` backed up. Google Drive and Dropbox are fine for this backup; these files cannot be used to impersonate you.

### Testing and Troubleshooting the Setup

When there's an issue, we can narrow the problem down to an individual component or connection.

* Test that the GPG agent is running and accessible (after an attempt at use):

       systemctl --user status gpg-agent.service
       ls -l $SSH_AUTH_SOCK

       # Optionally, stop it (which will cause reinitialization on use):
       systemctl --user stop gpg-agent.service

* Test that the standard PIN counter hasn't been exhausted. It's the first number returned here:

       gpg2 --card-status|grep "PIN retry counter"

  * If the standard PIN counter has been exhausted, it's possible to unblock (using `gpg2 --card-edit` with `passwd`) as long as the third number (the mangement/admin PIN retry counter) wasn't also zero.

  * If even the management/admin PIN is exhausted, then the entire GPG module needs to be reset: [YubiKey instructions](https://www.yubico.com/support/knowledge-base/categories/articles/reset-applet-yubikey/)

* Test the GPG-to-smart card connection and key trust. The following should prompt for the regular PIN and succeed:

       FINGERPRINT=`gpg2 --card-status | grep "General key info" | grep -o "/[[:alnum:]]* " | grep -o "[[:alnum:]]*"`
       echo "test" | gpg2 --sign --armor --local-user $FINGERPRINT

* Test the OpenSSH client's connection to the GPG agent. The following should output the SSH public key:

       ssh-add -L

### One-Time or Test Usage of the Agent

1. Open a terminal.
2. If you're restarted your computer since using the agent, start it:

       gpg-agent --daemon --enable-ssh-support

3. In any shell where you want to use it, point OpenSSH to the GPG agent:

       export SSH_AUTH_SOCK=${XDG_RUNTIME_DIR}/gnupg/S.gpg-agent.ssh  # Or use the line shown in the output of starting the GPG agent

### Revoking a Key

1. If the key isn't imported locally, follow the "Using an Existing Smart Card" steps first (but skipping the "trust" step).
2. If you don't have the revocation certificate (`.rev`) backed up but have the private key:

       gpg2 --gen-revoke --output=$FINGERPRINT.rev $FINGERPRINT

3. Import the revocation:

       gpg2 --import $FINGERPRINT.rev  # May need to remove colon before the five dashes from file.

4. Publish the revocation:

       gpg2 --keyserver hkps://keys.gnupg.net --send-keys $FINGERPRINT
       gpg2 --keyserver hkps://hkps.pool.sks-keyservers.net --send-keys $FINGERPRINT
       gpg2 --keyserver hkp://pgp.mit.edu --send-keys $FINGERPRINT

## Go Development

1. Install packages:

       sudo dnf install golang

1. Add Go to the path:

       mkdir -p ~/.config/environment.d/
       cat <<EOT >> ~/.config/environment.d/50-golang.conf
       GOPATH=\$HOME/go
       PATH=\$PATH:\$GOROOT/bin:\$GOPATH/bin
       EOT

## PHP Development

1. Install packages:

       sudo dnf install -y nginx mariadb mariadb-server php php-fpm php-mysqlnd php-dbg php-cli php-bcmath php-phpass php-mbstring php-opcache php-gd php-pecl-apcu php-pecl-xdebug

2. Install, start, and configure the database:

       sudo systemctl start mariadb
       mysql_secure_installation

3. Create a directory for web projects (and enable web server access to directories of that type):

       chmod 711 ~
       mkdir ~/public_html
       sudo setsebool -P httpd_enable_homedirs 1    # Enables use of ~/public_html by nginx and PHP-FPM.
       sudo setsebool -P httpd_execmem 1            # Enables PHP's regex compilation.
       sudo setsebool -P httpd_builtin_scripting 1  # Hope to fix: https://bugzilla.redhat.com/show_bug.cgi?id=1510717
       sudo setsebool -P httpd_unified 1            # Same here.
       sudo setsebool -P httpd_enable_cgi 1         # Same here.

4. Add support for `~/public_html` to nginx using `/etc/nginx/default.d/userdir.conf`:

    ```conf
    location @drupal {
        error_log /var/log/nginx/userdir.log notice;
        #rewrite_log on;
        rewrite ^/~([^/]+)/([^/]+)(.*)\?(.*)$ /~$1/$2/index.php?q=$3&$4;
        rewrite ^/~([^/]+)/([^/]+)(/.*?)$ /~$1/$2/index.php?q=$3;
    }

    location ^~ /~ {
        error_log /var/log/nginx/userdir.log notice;

        location ~ ^/~(?<username>[^/]+)(?<path>/.+\.php)$ {
            root /home/$username/public_html;
            try_files $path =404;
            fastcgi_index  index.php;
            include        fastcgi_params;
            fastcgi_param  SCRIPT_FILENAME  $document_root$path;
            fastcgi_param  SCRIPT_NAME      /~$username$path;
            fastcgi_pass   php-fpm;
        }

        location ~ ^/~(?<username>[^/]+)(?<path>/.+?)?$ {
            root /home/$username/public_html;
            try_files $path @drupal;
        }
    }
    ```

5. Configure some PHP-related options:

       echo "apc.rfc1867=1" | sudo tee -a /etc/php.d/40-apcu.ini  # Upload progress tracking.

6. Start services for development (each time they're needed):

       sudo systemctl start mariadb php-fpm nginx

7. Use `~/public_html/$PROJECT/` as the web root, accessible via `http://localhost/~$USER/$PROJECT/`.

8. If new files with the wrong context get added, fix the selinux context:

       restorecon -R ~/public_html

## Dwarf Fortress
1. Download the latest archive.
2. Install necessary packages:

       sudo dnf install -y SDL.i686 SDL_image.i686 SDL_ttf.i686 mesa-libGLU.i686 gtk2.i686 zlib.i686 openal-soft.i686 xterm python qt qt-x11 bzip2 xorg-x11-fonts-Type1

3. Launch with `startlnp`
4. Use `xterm -e` as the custom terminal command configuration.

## OpenRA

1. Download [the binary package for the current release](https://software.opensuse.org/download.html?project=games:openra&package=openra).

2. Install the RPM using Software Center or DNF (in order to install dependencies).

## BIOS Updates

First, acquire the update. For ThinkPads, use [Lenovo's My Products tool](https://account.lenovo.com/us/en/myproducts), click on the product model (after adding yours), then Top Downloads > View All, and finally the "Bootable CD" ISO.

### USB Thumb Drive Method

1. Install the conversion utility:

       sudo dnf install geteltorito

2. Write it to a USB drive:

       geteltorito -o update.img downloaded.iso

3. Open the `update.img` in the Disks utility and restore it to the USB drive.
4. Boot from that drive.

### GRUB2 EFI Chainloader Method

1. **First Time:** Setup:

       sudo dnf install syslinux p7zip p7zip-plugins

       ESPUUID=`sudo grub2-probe --target=fs_uuid /boot/efi/`
       cat >> 40_custom <<EOF
       menuentry "Firmware Update" {
           insmod fat
           insmod part_gpt
           insmod chain
           insmod search_fs_uuid
           search --fs-uuid $ESPUUID
           chainloader /LenovoEFI/Boot/BootX64.efi
       }
       EOF

       sudo mv 40_custom /etc/grub.d/40_custom
       sudo grub2-mkconfig -o /boot/efi/EFI/fedora/grub.cfg

2. **Every Time:** Extract the update to appropriate places in the EFI partition:

       geteltorito -o eltorito.img downloaded.iso
       7z x -oeltorito/ eltorito.img
       sudo rsync --recursive --delete --verbose eltorito/Flash/ /boot/efi/Flash/
       sudo rsync --recursive --delete --verbose eltorito/EFI/ /boot/efi/LenovoEFI/

3. **Every Time:** Reboot and select "Firmware Update."

## Crash Reporting

* Reconfigure using `system-config-abrt`

## Other Resources

* [U2F Local Login](http://blog.liw.fi/posts/u2f-pam/)
