+++
title = 'Laravel 專案打包成 Docker image 上傳至 ECR'
date = 2026-04-11T23:48:13+08:00
draft = false
description = '簡單實作 API Gateway，依照 config.yaml 將 Request 反向代理給其他服務，並有 Rate limit、JWT 驗證、Log、metrics 等功能'
toc = true
tags = ['Laravel', 'Docker']
categories = ['Laravel', 'Docker']
+++

# 前言
本來是公司 server 準備改用 Docker 佈版，但因專案時程問題而延宕，最後決定在此記下部分研究成果\
如果 PHP 版本為 8.2 以上，可以考慮使用 [FrankenPHP](https://frankenphp.dev/docs/docker/)。本篇因為公司專案版本較舊，使用 PHP 7.3、Laravel 8 示例

# 打包為 Docker image
Docker base 的選擇，因為 alpine 缺乏一些網路套件如 net-tools、iproute4，並且有 musl libc 的問題（跟 C 有關），最重要的是公司 server OS 使用 Debian。最後選擇了 bullseye

Dockerfile
```
FROM php:7.3-fpm-bullseye

ARG DEBIAN_FRONTEND=noninteractive

# ---------- system packages ----------
RUN apt-get update && apt-get install -y --no-install-recommends \
    nginx \
    supervisor \
    git \
    curl \
    unzip \
    zip \
    libpng-dev \
    libjpeg-dev \
    libfreetype6-dev \
    libzip-dev \
    libonig-dev \
    libxml2-dev \
    vim \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

# ---------- php extensions ----------
RUN docker-php-ext-configure gd --with-freetype-dir=/usr/include/ --with-jpeg-dir=/usr/include/ \
  && docker-php-ext-install -j$(nproc) \
    pdo_pgsql \
    mbstring \
    tokenizer \
    xml \
    gd \
    zip \
    opcache

# ---------- composer ----------
COPY --from=composer:2 /usr/bin/composer /usr/bin/composer

# ---------- config ----------
COPY docker/nginx.conf /etc/nginx/sites-available/default
COPY docker/supervisord.conf /etc/supervisor/conf.d/supervisord.conf

# ---------- app ----------
WORKDIR /var/www/html
COPY . .

RUN composer install \
    --no-dev \
    --optimize-autoloader \
    --no-interaction || true

# ---------- permissions ----------
RUN mkdir -p storage bootstrap/cache \
  && chown -R www-data:www-data storage bootstrap/cache \
  && chmod -R 775 storage bootstrap/cache

EXPOSE 80

CMD ["/usr/bin/supervisord", "-n"]
```
.dockerignore
```
vendor
node_modules
.git
.env
storage
tests
```
docker/nginx.conf
```
server {
    listen 80;
    server_name _;

    root /var/www/html/public;
    index index.php index.html;

    access_log /var/log/nginx/access.log;
    error_log  /var/log/nginx/error.log;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        include fastcgi_params;
        fastcgi_pass 127.0.0.1:9000;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
    }

    location ~ /\. {
        deny all;
    }
}
```
docker/supervisord.conf
```
[supervisord]
nodaemon=true

[program:php-fpm]
command=docker-php-entrypoint php-fpm
priority=10
autostart=true
autorestart=true
stdout_logfile=/dev/stdout
stderr_logfile=/dev/stderr

[program:nginx]
command=nginx -g "daemon off;"
priority=20
autostart=true
autorestart=true
stdout_logfile=/dev/stdout
stderr_logfile=/dev/stderr
```
最後可以在本地試試看
```
docker build -t elizabethhuang/laravel8 .
docker run --name laravel8 -p 8080:80 elizabethhuang/laravel8
```

# 利用 GitHub action 包 image 並上傳至 ECR
考慮上版時的步驟，會有很多 branch 合併、封版測試，完成後確認版號。因此這個包 image 的操作可以限制在有指定版號才動作\
首先，在 ECR 建立 repo。這裡有個小坑，ap-east-2（臺北）不支援 CodePipeline，如果想用 CodePipeline 作為 CI tool 的話，需要注意一下 region\
在 GitHub action secret 新增 AWS_DEPLOY_ROLE 和 AWS_DEFAULT_REGION，這些 AWS OIDC 要用。然後 variable 新增 ECR_REPOSITORY，也就是 ECR repo 名稱

.github/workflows/build.yml
```yml
name: Build Docker image

on:
  workflow_dispatch:

permissions:
  id-token: write
  contents: read

jobs:
  run_job_with_aws:
    runs-on: ubuntu-24.04-arm
    # Check whether the ref contains tag.
    if: startsWith(github.ref, 'refs/tags/v')
    steps:
      - name: Set version tag env
        run: echo "RELEASE_VERSION=${GITHUB_REF#refs/*/}" >> $GITHUB_ENV

      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@main
        with:
          role-to-assume: ${{ secrets.AWS_DEPLOY_ROLE }}
          aws-region: ${{ secrets.AWS_DEFAULT_REGION }}

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build and push Docker image
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          ECR_REPOSITORY: ${{vars.ECR_REPOSITORY }}
          IMAGE_TAG: ${{ env.RELEASE_VERSION }}
        run: |
          docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
```
另外，恭喜現在 Github action public repo 也能使用 arm runner 了！AWS 的 arm 機器比較便宜，但聽前輩說 AWS 相同 arm 不同 instance type 服務效率可能產生巨大差距，因此真的要服務轉移還是實測才準，在自己帳戶玩玩還是可以用 arm 省點摳摳

# Next step...
image 上傳至 ECR 後，接下來就能使用 ECS 部署到 EC2 上了\
我使用 Terraform 啟動整個資源，但目前工作重心轉移到功能開發上，研究進度中斷，若有空閑時間再完善整個 Terraform + GitHub action 部署

