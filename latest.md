# My Note : GCP部署
###### tags:`GCP`, `Deployment`, `Nginx`, `Laravel`

# 事前 
1. 在自己電腦上安裝 gcloud sdk
    * [macOS](https://formulae.brew.sh/cask/google-cloud-sdk) : `brew cask install google-cloud-sdk`
    * [Ubuntu](https://cloud.google.com/sdk/docs/quickstart-debian-ubuntu?hl=zh-tw)
    * [Windows](https://cloud.google.com/sdk/docs/quickstart-windows?hl=zh-tw) 
3. `gcloud init` 
4. `gcloud auth login`
5. 查看 project `gcloud config list project`
（`gcloud projects create`==PROJECT_ID==）

# Step 1 建立執行個體
1. `gcloud compute instances create test1023 --zone asia-east1-b --image-project ubuntu-os-cloud --image-family ubuntu-1804-lts` 
    >一定要這兩個參數：`--image-project` ,  `--image-family`
    * 若全都用預設，就只要下指令: `gcloud compute instances create test1023`即可
    * 查看[可用映像檔](https://cloud.google.com/compute/docs/images?hl=zh-tw)
    查詢：`gcloud compute images list`
    * 查看[可用區域](https://cloud.google.com/compute/docs/regions-zones/?hl=zh-tw#available)
    查詢：`gcloud compute zones list`
    * 查詢目前VM個體：`gcloud compute instances list`
    
2. 進入 `gcloud compute ssh "test1023"`
(`gcloud compute ssh --zone "asia-east1-b" "test1023"`)
3. Enter passphrase for key '/Users/sarahcheng/.ssh/google_compute_engine': 輸入SSH 鑰匙密碼
4. 開瀏覽器到 [gcloud console](https://console.cloud.google.com/) 將防火牆打開
![](https://i.imgur.com/qO078qH.png)


:::danger
如果要刪除 `gcloud compute instances delete test1023`
:::

# Step 2 安裝所需套件
* 看作業系統： 指令`lsb_release -a`  及 `cat /etc/*release`
## 2-1 安裝 php

1. 看 php 版本：`php -v`，更新到 7.3
    ```=
     #安裝常用的套件管理工具
    $sudo apt-get install software-properties-common
    
     #增加一個php的ppa
    $sudo add-apt-repository ppa:ondrej/php
    
     #BJ4惹
    $sudo apt-get update
    
     #安裝php
    $sudo apt-get install -y php7.3
    
     #同時安裝 php-mbstring、php-gd、php-xml、zip
    $sudo apt-get install php-mbstring php-gd php-xml zip
    ```
    * 再看一次確認安裝成功 `php -v`

>ppa：Personal Package Archives
開發者可以自行發行軟體並且透過 Ubuntu 幫使用者更新。
白話點就是透過 PPA 在自己的 Ubuntu 上面安裝這些軟體，在未來可以透過 更新管理員 來更新這些軟體的版本。

:::danger
若vm的作業系統是debian，則下以下指令安裝php
```
#安裝apt-transport-https（確保APT可以在HTTPS中執行）、lsb-release（識別系統的Linux發行版的版本）及ca-certificates（CA憑證工具）
sudo apt-get install apt-transport-https lsb-release ca-certificates -y

#加入packages.sury.org的GPG key
sudo wget -O /etc/apt/trusted.gpg.d/php.gpg https://packages.sury.org/php/apt.gpg

#將packages.sury.org寫入至Debian的sources list套件來源清單
echo "deb https://packages.sury.org/php/ $(lsb_release -sc) main" | sudo tee /etc/apt/sources.list.d/php.list

#更新sources list清單
sudo apt-get update

#安裝php7.3-fpm
sudo apt-get install php7.3-fpm -y

#同時安裝 php-mbstring、php-gd、php-xml、zip
sudo apt-get install php7.3-mbstring php7.3-gd php7.3-xml zip

```
參考網址：https://www.kjnotes.com/devtools/82
:::

## 2-2 安裝 composer 
* 安裝指令：`curl -sS https://getcomposer.org/installer | sudo php -- --install-dir=/usr/local/bin --filename=composer`
* 查看是否安裝成功：`composer -V`
## 2-3 [安裝 mysql](https://blog.johnsonlu.org/install-mysql-on-ubuntu/)
1. `sudo apt-get install mysql-server`

    > 若要安裝特定版本`apt-get install mysql-server-5.7` 
    >  若要設密碼：`$mysql_secure_installation`
7. 啟動 mysql-server: 
    `systemctl start mysql`
8. 查看是否成功啟動：
    `systemctl |grep  mysql`
9. 進入 mysql server:
    `mysql -u root`
    更改有權限或要修改的使用者本身已登入 mysql 的密碼
    `mysql> SET PASSWORD FOR '目標使用者'@'主機' = PASSWORD('密碼');`
    > 查看目前使用者及其身份驗證方式：
`SELECT User, Host, plugin FROM mysql.user;`
    
    1. 新增使用者(用root的話可以省略這步)
     `CREATE USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY '密碼';`
    
    2. 給予新建的使用者存取DB的權限：
`GRANT ALL PRIVILEGES ON *.* TO 'root'@'localhost';` 
(＊.＊ 代表所有DB的所有 table)
    
    3. 查看目前 mysql server的使用者：
`select user,plugin,authentication_string from mysql.user;` 
    
    4. 進入mysql，修改root驗證密碼的plugin
`update mysql.user set authentication_string=PASSWORD('密碼'), plugin='mysql_native_password' where user='root'; `
   
   5. 刷新msyql系統權限相關的表
    `flush privileges;`

5. 讓php和mysql 連線
```
sudo apt-get install php7.3-mysql
```

    

## 2-4 安裝 Nginx
1. 查看Port 80 是否被佔用：`netstat -utlnp | grep 80`  
    (參數意義 udp, tcp, listen, numeric, process)
    > numeric 的意思是將名稱數字化(IP)，例：原本的 localhost 會變成 127.0.0.1 
2. Ubuntu 的映像檔已內建 Apache2，預設是開啟聽著 80 port， 查看 apache2 狀態：`systemctl status apache2`
3. 我們要改用 Nginx, 但是因為安裝 Nginx 後它也會自動聽 80 port，所以先把 apache2 停掉：`systemctl stop apache2`
4. 安裝 Nginx：`sudo apt-get install nginx -y`
5. 安裝 Nginx 與 PHP溝通之套件 `sudo apt-get install php7.3-fpm`


## 2-5 安裝 git
1. `sudo apt-get update`
2. `sudo apt-get upgrade`
3. `sudo apt-get install git`

# Step 3 部署 github 專案
## 3-1 專案下載 
* `git clone https://github.com/laravel/laravel.git`
## 3-2 Nginx 設定
1. 修改 Nginx 設定檔中的讀取路徑為我們要部署的專案
`vim /etc/nginx/sites-available/default`   
    * 將專案路徑＋/public 寫在 root 後 
    ```
    #root /var/www/html;
    root /home/boa1417/專案名稱/index.php的所屬資料夾;
    # Add index.php to the list if you are using php
    index index.php;
    ```
    如圖：![](https://i.imgur.com/EajSnwX.png)
    * location 改成
    ```
      location / {
        try_files $uri $uri/ /index.php?$query_string;
      }
    ```
    * 拿掉 php 那段的註解，套件改 7.3 版本，`php7.3-fpm.sock;`
    
    ```
    location ~ \.php$ {
      include snippets/fastcgi=php.conf;
      fastcgi+pass unix:/var/run/php/php7.3-fpm.sock;
    }
    ```
![](https://i.imgur.com/9wEkfqw.png)

    
2. 讓 niginx 重新載入修改完的設定檔：
    `nginx -s reload`
   PS:若出現下列錯誤 [參考連結](https://www.jianshu.com/p/4f8b57632e2b)
   ```
   nginx: [error] open() "/run/nginx.pid" failed (2: No such file or directory)
   // 可輸入下列指令
   $nginx
   // 執行該指令之後，會在 /usr/local/var/run/ 路徑下，創建一個名爲 nginx.pid 的文件 )
   ```
   或重新開啟nginx，讓nginx listen 80 port
   ```
   systemctl start nginx
   ```
4. 進入專案安裝套件和設定環境：
    `cd laravel`
    1. `composer install`
    2. `mv .env.example .env`
    3. 產生 access key：
        `php artisan key:generate`

## 3-3 資料庫 Operation
1. 進入 `mysql -u root -p`
    * `CREATE DATABASE testdb;`
    * `show databases;`
    * `use testdb;`
    * `show tables;` (現在是空的)
2. 另外開一個 ssh 連線過去
    1. `apt-get install php7.3-mysql`
    2. 進到專案目錄下：`cd laravel`
    3. 修改專案的環境設定 .env 檔： `vim .env` 
        ```
        DB_CONNECTION=mysql
        DB_HOST=127.0.0.1
        DB_PORT=3306
        DB_DATABASE=testdb
        DB_USERNAME=sarah
        DB_PASSWORD=00000
        ```
    4. 執行指令建立 tabel: `php artisan migrate` (原本git clone 下來的專案就有寫好的 migration了)
3. 回到 mysql  
    * `show tables;` (現在多了三個 table)
    * 離開 mysql `exit`
## 3-4 更改權限 
在專案目錄下：
* 先到專案路徑下看擁有者 `ls -al` 
原本狀態：
```
total 84
drwxrwxr-x. 3 boa1417 boa1417 4096 Mar 16 15:57 .
drwx------. 4 boa1417 boa1417  169 Mar 16 15:57 ..
drwxrwxr-x. 8 boa1417 boa1417  163 Mar 16 10:39 .git
-rw-rw-r--. 1 boa1417 boa1417   14 Mar 16 10:39 .gitignore
-rw-rw-r--. 1 boa1417 boa1417 1060 Mar 16 10:39 README.md
-rw-rw-r--. 1 boa1417 boa1417  580 Mar 16 10:39 addComment.php
-rw-rw-r--. 1 boa1417 boa1417  555 Mar 16 10:39 addPosts.php
-rw-rw-r--. 1 boa1417 boa1417  646 Mar 16 10:39 addReply.php
-rw-rw-r--. 1 boa1417 boa1417 3069 Mar 16 10:39 allComments.php
-rw-rw-r--. 1 boa1417 boa1417 1126 Mar 16 10:39 allReplies.php
-rw-rw-r--. 1 boa1417 boa1417 2710 Mar 16 10:39 board.php
-rw-rw-r--. 1 boa1417 boa1417 1352 Mar 16 10:39 createTable.php
-rw-rw-r--. 1 boa1417 boa1417   91 Mar 16 10:39 db.php
-rw-rw-r--. 1 boa1417 boa1417  540 Mar 16 10:39 dislike.php
-rw-rw-r--. 1 boa1417 boa1417  107 Mar 16 10:39 env.example.php
-rw-rw-r--. 1 boa1417 boa1417  130 Mar 16 15:57 env.php
-rw-rw-r--. 1 boa1417 boa1417  158 Mar 16 10:39 header.php
-rw-rw-r--. 1 boa1417 boa1417 2594 Mar 16 10:39 index.php
-rw-rw-r--. 1 boa1417 boa1417 2632 Mar 16 10:39 like.php
-rw-rw-r--. 1 boa1417 boa1417  196 Mar 16 10:39 logout.php
-rw-rw-r--. 1 boa1417 boa1417 3168 Mar 16 10:39 register.php
-rw-rw-r--. 1 boa1417 boa1417 1635 Mar 16 10:39 style.css
-rw-rw-r--. 1 boa1417 boa1417 3400 Mar 16 10:39 view.php

```

* 修改擁有群組為 www-data
    `sudo chown -R boa1417.www-data .`
    * "chown" change 擁有者
    * "-R" 往下層所有目錄遞迴 
    * "sarahcheng.www-data" 擁有者.擁有群組
    * "." 代表當前目錄
* `sudo chmod -R 2770 ./storage/`
    * "chmod" change mode
    * "-R" 往下層所有目錄遞迴 
    * "2770" 2代表 SGID [參考](http://www.vixual.net/blog/archives/224)
        * 770分別代表：
            * 擁有者的存取權限
            * 擁有群組的存取權限
            * 其他人的存取權限
    * "." 代表當前目錄

    到專案路徑下看擁有者是否更改成功 `ls -al` 



# P.S.
- GCP instance 的 IP，可在 GCP GUI 複製 `外部 IP`。
- 將舊資料備份 `mysqldump --opt -usarahcheng -p laravel_shop > test.sql`
- 或這樣備份 `mysqldump -u 使用者名稱 -p 要匯出的資料 > 匯出後要叫的檔名.sql`
- 透過 ssh 將檔案傳到 GCE instance(AWS也一樣) `scp test.sql test1023: --zone asia-east1-b`

- 查看目前機器中所有開放的 Port
    ```bash
    $netstat -utlnp
    ```

* 基本上不用安裝[laravel](https://laravel.com/docs/6.x/installation)(不太可能在server上開發)，不過還是提供參考指令： 
    :::success
    1. `apt-get install php7.3-zip`
    2. `composer global require laravel/installer`
    2. `vi ~/.bash_profile`
        * `export PATH="$HOME/.composer/vendor/bin:$PATH"`
    3. `source ~/.bash_profile
    4. `laravel -V`

    :::

- Machine type `f1-micro` 只有 600M 的記憶體，但 Composer [據說](https://www.reddit.com/r/PHP/comments/2j15ff/making_composer_use_less_ram/cl7kk4t/) 會用到 512M 以上，目前聽到的方案：
    1. 直接開足夠規格的機器：n1-standard-1 以上 (3.7G 記憶體)
    2. 在需要 Composer 時才擴增記憶體，但機器需要進關機狀態
    3. 本地建置完，再傳到機器內

- [記錄] 把整個資料夾都修改權限 `sudo chmod -R 2770 .`，解決了 502 錯誤
- [記錄] 後來卡在 404
    - 如果是放在 /var/www/larave/public 可以運作
    - 需要 `sudo chmod 777 -R ./storage/` ([來源](https://laracasts.com/discuss/channels/laravel/permission-denied-for-laravel-storage-logs?reply=483075))，但似乎是有點危險的權限 (?)

## 複製本機資料庫資料到遠端
本地備份資料：
```
mysqldump -uroot -p 資料庫名 > 備份的檔案名稱.sql
```
複製到instance上：
```
gcloud compute scp 備份的檔案名稱.sql instance名:/home/使用者名稱
```

instance上的mysql 建立database後，匯入資料
```
 mysql -uroot -p 資料庫名 < 備份的檔案名稱.sql
```
## 複製遠端資料庫資料到本機
由於對遠端來說本機的22 port是鎖住的，所以不能在遠端的terminal下指令
本機指令 
```
scp 遠端使用者名稱@遠端ip:檔案絕對路徑/檔名 本機目的資料夾路徑 
```

例如：
```
scp ubuntu@13.113.32.70:/home/ubuntu/stock.sql /Users/admin
```

## 加入新 ssh 使用者

1. 在 Instance 加入你的公鑰，(GCP console)。
2. 注意：登入的帳號，就是公鑰最後面顯示的名稱
3. 用 ssh 指令連線。

`$ ssh  name@my_instance_ip`
(預設抓 ~/.ssh/id_rsa )

比如說

`$ ssh louis@35.XX.230.OO`

如果要指定用特定哪一組公私鑰，可加上 `-i`
`$ ssh  -i  specified_private_key  name@my_instance_ip (edited)`

## Apache2 是在裝 php7.3 時的相依套件

> $apt-get install php7.3

> The following additional packages will be installed:
  **apache2** apache2-bin apache2-data apache2-utils libapache2-mod-php7.3 ...


## 踩過的坑：本機無法用ssh或gcloud compute ssh連上instance

試過新增一個instance，image選擇debian和centOS都能使用gcloud連線，唯獨ubuntu連線會出現以下情況：

1. 使用ssh連線：
```
✘ admin@admins-MacBook-Pro$ssh boa1417@35.201.188.202
boa1417@35.201.188.202: Permission denied (publickey).
```

Ans：
原來是ubuntu認不得admin，我的本機這位使用者，連線的公鑰也是錯的，因此連線時有兩個選項：
1. 使用ssh連線，指定要使用哪一組公鑰，像這邊要用.ssh裡的my-gcp這組
(gcloud預設是用~/.ssh/google_compute_engine這組來連線)
```
$ssh -i ~/.ssh/my-gcp boa1417@35.201.188.202
```
然後要輸入passphrase密碼才能順利連線。

2. 使用gcloud連線，指定user
```
$gcloud compute ssh boa1417@instance0205 --zone asia-east1-b
```

```
$sudo vipw
```
![](https://i.imgur.com/IJRyWgC.jpg)


可以看到user名單，若連線時本機的user名稱沒有出現在此list中，就會出現permission denied。

```
admin@35.201.188.202: Permission denied (publickey).
ERROR: (gcloud.compute.ssh) [/usr/bin/ssh] exited with return code [255].
```
## 踩過的坑：瀏覽器一直自動下載我部署上去的source code

php沒生效

可能是因為：
1. php版本打架
Solution：移除其中一個
例如同時存在php7.3和7.4，移除其中一個
```
sudo apt autoremove php7.4*
```
2. 設定檔沒改好
好像在php7.4之後，nginx的設定檔就必須把 
```
include snippets/fastcgi-php.conf;
```
這段uncomment，因為php的相依套件都是寫在snippets裡

## 同台server run兩個專案
1. 建立軟連結
```
sudo vim /etc/nginx/site-enable/建立和site-available的軟連結
```
/etc/nginx/site-available/裡可以有很多個設定檔
所以在/etc/nginx/site-enable/裡建立一個連結對應site-available的一個檔案就好
2. 修改/etc/nginx/site-available/裡面的nginx設定檔
```
##
# You should look at the following URL's in order to grasp a solid understanding
# of Nginx configuration files in order to fully unleash the power of Nginx.
# https://www.nginx.com/resources/wiki/start/
# https://www.nginx.com/resources/wiki/start/topics/tutorials/config_pitfalls/
# https://wiki.debian.org/Nginx/DirectoryStructure
#
# In most cases, administrators will remove this file from sites-enabled/ and
# leave it as reference inside of sites-available where it will continue to be
# updated by the nginx packaging team.
#
# This file will automatically load configuration files provided by other
# applications, such as Drupal or Wordpress. These applications will be made
# available underneath a path with that package name, such as /drupal8.
#
# Please see /usr/share/doc/nginx-doc/examples/ for more detailed examples.
##

# Default server configuration
#
server {

	listen 80 default_server;
	listen [::]:80 default_server;

	# SSL configuration
	#
	# listen 443 ssl default_server;
	# listen [::]:443 ssl default_server;
	#
	# Note: You should disable gzip for SSL traffic.
	# See: https://bugs.debian.org/773332
	#
	# Read up on ssl_ciphers to ensure a secure configuration.
	# See: https://bugs.debian.org/765782
	#
	# Self signed certs generated by the ssl-cert package
	# Don't use them in a production server!
	#
	# include snippets/snakeoil.conf;

	#root /var/www/html;
	# Add index.php to the list if you are using PHP
	root /home/ubuntu/chuen_stock/dist;
	index index.php index.html index.htm index.nginx-debian.html;
	server_name _;

    location /api {
                # First attempt to serve request as file, then
                # as directory, then fall back to displaying a 404.
		index index.php index.html index.htm index.nginx-debian.html;
                try_files $uri $uri/ /index.php?$query_string;
    }
	location ~ \.php$ {
        	root /home/ubuntu/stockAnalyser/public/;
		include snippets/fastcgi-php.conf;
		fastcgi_pass unix:/var/run/php/php-fpm.sock;
	}

	# pass PHP scripts to FastCGI server
	#

	#	# With php-fpm (or other unix sockets):
	#	# With php-cgi (or other tcp sockets):
	#	fastcgi_pass 127.0.0.1:9000;

	# deny access to .htaccess files, if Apache's document root
	# concurs with nginx's one
	#
	#location ~ /\.ht {
	#	deny all;
	#}
}


# Virtual Host configuration for example.com
#
# You can move that to a different file under sites-available/ and symlink that
# to sites-enabled/ to enable it.
#
#server {
#	listen 80;
#	listen [::]:80;
#
#	server_name example.com;
#
#	root /var/www/example.com;
#	index index.html;
#
#	location / {
#		try_files $uri $uri/ =404;
#	}
#}
```
