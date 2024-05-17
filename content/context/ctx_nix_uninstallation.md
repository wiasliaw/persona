---
title: "[context] nix uninstallation"
tags:
  - nix
---

## uninstallation for macOS

### 1. 清除掉相關的變數

- 在安裝過程中，會在 `/etc/zshrc`, `/etc/bashrc` 和 `/etc/bash.bashrc` 寫入下方的 shell，刪除即可

```bash
# Nix
if [ -e '/nix/var/nix/profiles/default/etc/profile.d/nix-daemon.sh' ]; then
  . '/nix/var/nix/profiles/default/etc/profile.d/nix-daemon.sh'
fi
# End Nix
```

- Nix 的安裝腳本在安裝之前，會將之前的設定備份起來，如果安裝 nix 之後沒有做什麼修改，可以直接將 `backup-before-nix` 後綴改名回去
  - `/etc/zshrc` -> `/etc/zshrc.backup-before-nix`
  - `/etc/bashrc` -> `/etc/bashrc.backup-before-nix`
  - `/etc/bash.bashrc` -> `/etc/bash.bashrc.backup-before-nix`

### 2. 關閉且移除 nix daemon

```bash
sudo launchctl unload /Library/LaunchDaemons/org.nixos.nix-daemon.plist
sudo rm /Library/LaunchDaemons/org.nixos.nix-daemon.plist
sudo launchctl unload /Library/LaunchDaemons/org.nixos.darwin-store.plist
sudo rm /Library/LaunchDaemons/org.nixos.darwin-store.plist
```

### 3. 移除 `nixbld` group 和 `_nixbuildN` users

- 移除 `nixbld` group

```bash
sudo dscl . -delete /Groups/nixbld
```

- 移除 `_nixbuildN` users。在安裝過程中，nix 建立了足足 32 個 user (`nixbuild1` to `_nixbuild32`)，每個都要移除

```bash
for u in $(sudo dscl . -list /Users | grep _nixbld); do sudo dscl . -delete /Users/$u; done
```

### 4. `sudo vifs`

```bash
sudo vifs
```

檔案打開如下，整行刪除即可

```txt
UUID=XXXXXXXXXXXX /nix apfs rw,noauto,nobrowse,suid,owners
```

### 5. `/etc/synthetic.conf`

```bash
sudo vim /etc/synthetic.conf
```

刪除有關 nix 的行，如果這是唯一一行，那可以將整個檔案刪掉

### 6. 刪除有關 nix 的檔案

```bash
sudo rm -rf /etc/nix /var/root/.nix-profile /var/root/.nix-defexpr /var/root/.nix-channels ~/.nix-profile ~/.nix-defexpr ~/.nix-channels
```

### 7. remove Nix Store volume

```bash
sudo diskutil apfs deleteVolume /nix
```
