# Nginx使用Godaddy SSL证书

## 购买Godaddy SSL证书

- 打开Godaddy官网 http://www.godaddy.com/
- 点击网站导航里的 Hosting & SSL >> SSL Certificates
- 直接点击"Get Started"在出来选择页面选择你的需求购买
- 回到你的控制台点击"SSL CERTIFICATES"就可以看到你刚买的SSL了，点击其后面的"Set Up"激活这个SSL
- 激活成功后，我们在"SSL CERTIFICATES"一栏看到证书已激活，点击后面的"Launch"按钮进入下一步
- 这时需要你输入 CSR ，页面不要关，我们去服务器上生成一个CSR

## 服务器生成 CSR

- root登录服务器确保服务器上安装了openssl，如未安装使用yum命令安装`yum install openssl openssl-devel`
- 命令行生成私钥和CSR文件，期间会要求填入相关信息，如实填写即可

  ```bash
  openssl genrsa -des3 -out domain.com.key 2048
  openssl req -new -key domain.com.key -out domain.com.csr
  ```

- 保存好生成的domain.com.key和domain.com.csr文件，后续证书安装的时候需要使用到domain.com.key私钥文件

## Godaddy验证

- 复制上面生成好的CSR信息粘贴到godaddy的输入框里，点击提交，等待审核，需要验证域名所有权。

## 下载证书

- 审核验证通过后，需要去证书下载页面下载对应服务器类型的证书，Nginx请选择其他
- 下载的证书压缩包包含两个文件，例如 `269d355bc7e546eb.crt`,`gd_bundle-g2-g1.crt`，把证书文件上传到服务器上

## 安装证书

- 把证书文件和olami.ai.key 文件放置同一个文件夹，例如:`/etc/nginx/ssl.domain.com`,确保不会存在同名文件夹

- 进行证书合并 `cat 269d355bc7e546eb.crt gd_bundle-g2-g1.crt > domain.com.crt`

- 在`/etc/nginx/ssl.domain.com`目录生成dhparam2048.pem文件，`cd /etc/nginx/ssl.domain.com && openssl dhparam -out dhparam2048.pem 2048`

- 修改Nginx主机配置

  ```conf

  server{

    listen       101.251.70.129:443 ssl;
    listen       101.251.70.129:80;
    server_name  domain.com;
    server_name  www.domain.com;

    ssl_certificate /etc/nginx/ssl.domain.com/domain.com.crt;
    ssl_certificate_key /etc/nginx/ssl.domain.com/domain.com.key;

    ssl_prefer_server_ciphers on;
    ssl_dhparam /etc/nginx/ssl.domain.com/dhparam2048.pem;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers "EECDH+ECDSA+AESGCM EECDH+aRSA+AESGCM EECDH+ECDSA+SHA384 EECDH+ECDSA+SHA256 EECDH+aRSA+SHA384 EECDH+aRSA+SHA256 EECDH+aRSA+RC4 EECDH EDH+aRSA !aNULL !eNULL !LOW !3DES !MD5 !EXP !PSK !SRP !DSS !RC4";
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;

    ....
  }

  ```

-重启nginx服务，`nginx -s reload`,如果不生效，使用如下命令重启nginx进程

  ```bash
  #Centos 7
  systemctl restart nginx
  #Centos 6
  service nginx restart
  ```