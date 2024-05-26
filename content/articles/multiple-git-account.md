---
title: 如何操作多個 git account
tags:
  - git
  - published
---

最近因為工作緣故，在工作上需要使用新的 GitHub 去提交程式碼，進而需要在本機上設定個人和公司的 GitHub 設定

## ssh 設定

額外辦理了一個工作用的帳號，自然需要額外產生一個新的 ssh key

### ssh key generation

- 以下以 ed25519 演算法產生 ssh key，同時設定 passphrase

```bash
ssh-keygen -t ed25519 -C "example@example.com" -f ~/.ssh/<work_key>
```

### ssh-agent (macOS only, optional)

- 使用 ssh-agent 保存 passphrase

```bash
eval "$(ssh-agent -s)" && ssh-add -K ~/.ssh/<work_key>
```

## git 設定

要同時使用多個不同的 git account 需要配置不同的 `.gitconfig`

### folder structure

個人 folder 架構設定如下，建立一個 folder 專門放工作用的 repo，在裡面建立一個 .gitconfig 去覆寫更上層的 git config：

```tree
$HOME
├── .gitconfig (global-level)
└── GitHub
    ├── work
    |   └── .gitconfig (profile-like-level)
    └── personal
```

### global-level config

在 global-level 的 `.gitconfig`，做以下的設定，如果建立 git directory 和 includeIf 相符的話，git 會採用指定的設定檔：

```txt
# ~/.gitconfig
[includeIf "gitdir:~/GitHub/work/"]
  path = ~/GitHub/work/.gitconfig
```

### profile-like-level config

profile-like-level 的 `.gitconfig` 設定如下：

```txt
# ~/GitHub/work/.gitconfig
[user]
  name = work_user
  email = work_email

[github]
  user = "work name"

[core]
  sshCommand = "ssh -i ~/.ssh/<work_ssh_key>"
```

### ssh config

最後調整關於工作帳號的 ssh config

```txt
# ~/.ssh/config
Host work
  AddKeysToAgent yes
  HostName github.com
  User git
  IdentityFile ~/.ssh/work_ed25519
  IdentitiesOnly yes
```

## final

設定完畢後，在 `~/GitHub/work/` folder 下的 git 操作都會以工作帳號的設定去執行

## reference

- https://blog.gitguardian.com/8-easy-steps-to-set-up-multiple-git-accounts/
- https://notes.wadeism.net/post/2022-09-12-git-push-with-specify-ssh-key
