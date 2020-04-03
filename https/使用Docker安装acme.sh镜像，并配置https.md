### 使用Docker安装acme.sh镜像，并配置https

***

![https://raw.githubusercontent.com/Emmeet/zhanji/master/image/acme.png]()

> **acme.sh** 实现了 `acme` 协议, 可以从 letsencrypt 生成免费的证书.

以上是官方的一个介绍，说明acme.sh遵从什么规则来生成证书，附上acme.sh的中文官方说明：

[中文文档]([https://github.com/acmesh-official/acme.sh/wiki/%E8%AF%B4%E6%98%8E](https://github.com/acmesh-official/acme.sh/wiki/说明))

按照说明文档来基本可以完成配置，但是我想以docker的方式来配置，于是想把自己实践的一些步骤记录下来，下面开始；

***

> docker将默认使用docker-compose进行

1. docker-compose关于acme.sh镜像的配置

   ```dockerfile
   version: '3'
   services:
     acme:
       image: neilpang/acme.sh:latest
       restart: always
       container_name: acme
       volumes:
               - "/usr/local/nginx_volume/cert:/shared" 
                   
   ```

   由于证书主要配合nginx进行使用，所以在nginx的volume下建立了映射文件夹，/shared 并不是acme生成证书存放的位置，具体生成的证书在哪一会详细说明；

   

2. 执行`docker-compose up -d`启动镜像，并执行`docker exec -it acme /bin/sh`命令进入容器内部，接下来生成证书的操作将在这里执行；

   

3. 进入容器后，acme主要有http和dns两种方式生成证书，这里主要介绍dns方式，执行命令：

   `acme.sh  --issue  --dns -d <yuodomain.com>`

   最新版本的acme执行这条命令后可能会出现如下提示：

   ```shell
   It seems that you are using dns manual mode. Read this link first: https://github.com/acmesh-official/acme.sh/wiki/dns-manual-mode
   ```

   根据提示地址查看文档可知，我们是以手动dns的方式生成，需要在后面加上一串命令，生成的命令变为：

   ```shell
   acme.sh  --issue  --dns -d <yuodomain.com> --yes-I-know-dns-manual-mode-enough-go-ahead-please
   ```

   用来告诉程序，我知道这是手动方式生成，我同意，命令执行成功后，会出现提示：

   ```shell
   Create account key ok.
   Registering account
   Registered
   ACCOUNT_THUMBPRINT='LizlNVt9jp8v2JqA3Z2lnCj_USfv1E-qJRD-e-xL2yQ'
   Creating domain key
   The domain key is here: /acme.sh/<yuodomain.com>/<yuodomain.com>.key
   Multi domain='DNS:<yuodomain.com>'
   Getting domain auth token for each domain
   Getting webroot for domain='<yuodomain.com>'
   Add the following TXT record:
   Domain: '_acme-challenge.<yuodomain.com>'
   TXT value: 'DlT1n6m-LyANn4kSjfI5HuecHShR1rhx-YHgeAEyIRs'
   Please be aware that you prepend _acme-challenge. before your domain
   so the resulting subdomain will be: _acme-challenge.lovedd1314.cn
   Please add the TXT records to the domains, and re-run with --renew.
   Please add '--debug' or '--log' to check more details.
   See: https://github.com/acmesh-official/acme.sh/wiki/How-to-debug-acme.sh
   
   ```

   这时候我们需要到域名控制台，添加一条解析记录，记录类型为TXT，主机记录和值已经给出，另外生成证书存放位置为上方成功提示：`The domain key is here：`后面的文件夹

   

4. 添加完解析记录后继续执行命令：

   ```shell
   acme.sh --renew --dns -d <yuodomain.com> --yes-I-know-dns-manual-mode-enough-go-ahead-please
   ```

   出现成功的提示`Cert success.`，则整个生成过程便以完成，可以将证书映射出去进行使用，需要注意得是Nginx 的配置 `ssl_certificate`  和 `ssl_trusted_certificate` 使用 `fullchain.cer` ，而非 `<domain>.cer` ，否则 [SSL Labs](https://www.ssllabs.com/ssltest/) 的测试会报 `Chain issues Incomplete` 错误

   

5. 最后附上我的Nginx配置

   ```nginx
   server {
   	listen 80;
   	server_name <yuodomain.com>;
   	return 301 https://$http_host$request_uri;
   }
   server {
       listen       443 ssl;
       server_name  <yuodomain.com>;
   
       ssl_certificate  	cert/fullchain.cer;
       ssl_certificate_key cert/<yuodomain.com>.key;
       ssl_session_timeout 5m;
       ssl_protocols TLSV1 TLSv1.1 TLSv1.2;
       ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;
       ssl_prefer_server_ciphers on;
   
       #charset koi8-r;
       #access_log  /var/log/nginx/host.access.log  main;
   
       location / {
           root   /usr/share/nginx/html;
           index  index.html index.htm;
       }
   
       #error_page  404              /404.html;
   
       # redirect server error pages to the static page /50x.html
       #
       error_page   500 502 503 504  /50x.html;
       location = /50x.html {
           root   /usr/share/nginx/html;
       }
   
       # proxy the PHP scripts to Apache listening on 127.0.0.1:80
       #
       #location ~ \.php$ {
       #    proxy_pass   http://127.0.0.1;
       #}
   
       # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
       #
       #location ~ \.php$ {
       #    root           html;
       #    fastcgi_pass   127.0.0.1:9000;
       #    fastcgi_index  index.php;
       #    fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;
       #    include        fastcgi_params;
       #}
   
       # deny access to .htaccess files, if Apache's document root
       # concurs with nginx's one
       #
       #location ~ /\.ht {
       #    deny  all;
       #}
   }
   
   ```

   



