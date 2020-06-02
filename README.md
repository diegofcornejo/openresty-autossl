# openresty-autossl
OpenResty
- Autossl
-- Local Store nginx.store.conf
-- Redis Store nginx.redis.conf
- Dynamic Proxy
-- MySQL

=======INSTALLATION=======
https://www.storyblok.com/tp/automatic-ssl-multi-tenant


======References==========
https://changelogfy.com/blog/how-i-setup-custom-domain-ssl-with-lets-encrypt-for-changelog-pages/
https://blog.readme.com/auto-generating-ssl-certificates-for-custom-domains-using-lets-encrypt/

Copy the IPv4 Public IP from the EC2 instance dashboard and connect to your instance using your private key.

ssh -i your_key.pem ec2-user@YOUR_EC2_IPInstall OpenResty
We will need to install OpenResty on the remote instance. What is OpenResty®? OpenResty® is a full-fledged web platform that integrates the standard Nginx core, LuaJIT and many carefully written Lua libraries. Lua gives OpenResty/nginx the power to make automatic SSL possible.


sudo yum-config-manager --add-repo https://openresty.org/package/amazon/openresty.repo
sudo yum install openresty
sudo yum install openresty-resty
On newer Amazon machines I got following error installing OpenResty:

https://openresty.org/package/amazon/2/x86_64/repodata/repomd.xml: \[Errno 14\] HTTPS Error 404 - Not Found
To fix that error you need to edit the repo file with sudo vim /etc/yum.repos.d/openresty.repo and exchange the $releasever placeholder of the baseurl to “latest” baseurl=https://openresty.org/package/amazon/latest/$basearch.Install LuaRocks
LuaRocks is the package manager we need to install the lua-resty-auto-ssl package.


wget http://luarocks.org/releases/luarocks-2.0.13.tar.gz
tar -xzvf luarocks-2.0.13.tar.gz
cd luarocks-2.0.13/
./configure --prefix=/usr/local/openresty/luajit \
--with-lua=/usr/local/openresty/luajit/ \
--lua-suffix=jit \
--with-lua-include=/usr/local/openresty/luajit/include/luajit-2.1
make
sudo make installInstall ggc
https://github.com/GUI/lua-resty-auto-ssl requires ggc for the installation process so we install it with yum.

sudo yum install gccSetup a user group
As lua-resty-auto-ssl needs to write to the directory /etc/resty-auto-ssl we’ll add the user group www to our ec2-user.


sudo groupadd www
sudo usermod -a -G www ec2-userInstall lua-resty-auto-ssl
Using the package manger luarocks we install lua-resty-auto-ssl and create the directory where the library will write it’s files to.


sudo /usr/local/openresty/luajit/bin/luarocks install lua-resty-auto-ssl
sudo mkdir /etc/resty-auto-ssl
sudo chown -R root:www /etc/resty-auto-ssl/
sudo chmod -R 775 /etc/resty-auto-sslGenerate a self signed fallback certificate
We will need a self signed fallback certificate as a fallback to be able to start nginx.


sudo openssl req -new -newkey rsa:2048 -days 3650 -nodes -x509 \
-subj '/CN=sni-support-required-for-valid-ssl' \
-keyout /etc/ssl/resty-auto-ssl-fallback.key \
-out /etc/ssl/resty-auto-ssl-fallback.crtEdit nginx.conf
After backing up our original nginx.conf we open vim to insert the required configuration for the server.


sudo mv /usr/local/openresty/nginx/conf/nginx.conf /usr/local/openresty/nginx/conf/nginx.backup.conf
sudo vim /usr/local/openresty/nginx/conf/nginx.conf
Insert following content to the nginx.conf
/usr/local/openresty/nginx/conf/nginx.conf


user ec2-user www;
events {
worker_connections 1024;
}

http {
lua_shared_dict auto_ssl 1m;
lua_shared_dict auto_ssl_settings 64k;
resolver 8.8.8.8 ipv6=off;

init_by_lua_block {
auto_ssl = (require "resty.auto-ssl").new()
auto_ssl:set("allow_domain", function(domain)
return true
end)
auto_ssl:init()
}

init_worker_by_lua_block {
auto_ssl:init_worker()
}

server {
listen 443 ssl;
ssl_certificate_by_lua_block {
auto_ssl:ssl_certificate()
}
ssl_certificate /etc/ssl/resty-auto-ssl-fallback.crt;
ssl_certificate_key /etc/ssl/resty-auto-ssl-fallback.key;
}

server {
listen 80;
location /.well-known/acme-challenge/ {
content_by_lua_block {
auto_ssl:challenge_server()
}
}
}

server {
listen 127.0.0.1:8999;
client_body_buffer_size 128k;
client_max_body_size 128k;

location / {
content_by_lua_block {
auto_ssl:hook_server()
}
}
}
}Start OpenResty
As the final step we start OpenResty as a system service.

sudo service openresty startChange DNS record
To test it out point your domain or subdomain to the IP address of your EC2 instance and open the browser with **https://**subdomain.yourdomain.com.
Debugging
If you get an error or a invalid certificate checkout what’s happening tailing the nginx error.log. I had some directory rights issues and found it out by watching the error.log while reloading the website with https.

tail -F /usr/local/openresty/nginx/logs/error.logLog rotation
To enable log rotation for Resty we also need to add a logrotate configuration like following:

$ sudo vim /etc/logrotate.d/resty
/etc/logrotate.d/resty


/usr/local/openresty/nginx/logs/*.log {
compress
copytruncate
create 0644 root root
delaycompress
missingok
rotate 7
sharedscripts
postrotate
kill -USR1 `cat /usr/local/openresty/nginx/logs/nginx.pid`
endscript
}
Then create the logrotation cronjob

$ sudo crontab -e

0 21 * * * /usr/sbin/logrotate -v /etc/logrotate.d/resty
