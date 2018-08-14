# nginx

## 1.安装

```shell
mkdir -p /soft/nginx;
cd /soft/nginx;
wget http://nginx.org/download/nginx-1.14.0.tar.gz;
tar -zxvf /soft/nginx/nginx-1.14.0.tar.gz;
cd /soft/nginx/nginx-1.14.0;
yum install -y pcre pcre-devel;
yum install -y zlib zlib-devel;
yum install -y openssl openssl-devel;
./configure --sbin-path=/usr/local/nginx/nginx --conf-path=/usr/local/nginx/nginx.conf --pid-path=/usr/local/nginx/nginx.pid --with-http_ssl_module;
make;
make install;
/usr/local/nginx/nginx;
```

