
## 使用 ssh 登人 GitLab

進入 ssh folder, (%userprofile%/.ssh)
建立 config 的文字檔

    Host gitlab.com
    User git
    Hostname gitlab.com
    IdentityFile ~/.ssh/id_rsa
    IdentitiesOnly yes

確定在 .ssh 內有 id_rsa 私鑰.

參考: https://zeckli.github.io/zh/2017/10/01/resolve-gitlab-permission-denied-zh.html
