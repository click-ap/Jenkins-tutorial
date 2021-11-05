# Windows 10

## keybase authentication

### 確認有 ssh

    安裝 git for window時就會安裝.

### ssh-keygen 

產生公私鑰，會在個入資料夾內 (eg. c:\Users\John\.ssh\id_rsa) 建立 id_rsa 及 id_rsa.pub
* id_rsa 私鑰
* id_rsa.pub 公鑰

在 linux 使用 ssh -i ~/.ssh/id_rsa 
在 Windows 用 PuTTY 帶 key :

1. puttygen.exe 參考: https://moodletw.wordpress.com/2021/11/03/ssh-key-%e8%bd%89%e5%ad%98%e7%a7%81%e9%91%b0%e7%b5%a6-putty-%e4%bd%bf%e7%94%a8/

![PuTTy.](https://cd9b9d71-a-5741c8b4-s-sites.googlegroups.com/a/click-ap.com/linux/ssh/ssh-keygen/puttygen%E8%BD%89%E5%AD%98%E7%A7%81%E9%91%B0.jpg?attachauth=ANoY7crqImzmWEIzWaGMC1Fm2aPDwP9ZEQfrJFmJgZu6psUH50gHBGvkldgSMzmTIkGn7I-kLqLIFdS48YZ5J9vZaKBQLHAuU-CQ0nEshtf2Jv97iFgHe9KUyouLaUfKM4ibvcsrg9oF8uOPo1c7C16DrTV4yeuFF8-4BUc8jAUwdeQh_QCEI8eGTIIwIE4AuSZP-ETOWXcJS8ftuC7oi_YgRwtee_CxJSlL_ETM6n6zMnzBtN4SQ7l0MwR1sFVR_tvrjsbgDxZvTBitqgroFIT9aQp4VfbqDQ%3D%3D&attredirects=0)


2. 另外一種

https://moodletw.wordpress.com/2021/11/03/%e5%8f%a6%e5%a4%96%e4%b8%80%e7%a8%ae-ppk-%e7%9a%84%e6%87%89%e7%94%a8/

![在Windows工作列的圖示.](https://moodletw.files.wordpress.com/2021/11/image.png)


## WinSCP

WinSCP是 Windows 系統與 Linux 系統交換檔案的好工具,它的名稱由來是因為 Linux內都是用scp作為跨主機交換的命令, 因為它是Windows版本, 所以稱為WinSCP.下載網址如下:

    http://winscp.net/eng/download.php