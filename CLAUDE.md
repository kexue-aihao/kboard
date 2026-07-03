# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概览

这是 V2Board 面板项目，基于 Laravel 8，主要用于代理协议管理。运行环境来自 `README.md` 和 `.env.example`：PHP 7.3+/8.x、Composer、MySQL、Redis。队列默认使用 Redis，Horizon 用于队列进程管理。

Composer 自动加载两个 PSR-4 命名空间：`App\\` 指向 `app/`，`Library\\` 指向 `library/`。

## 常用命令

```bash
composer install
```

安装 PHP 依赖。`init.sh` 中的完整初始化流程会下载 `composer.phar`、执行安装，并在 PHP 8+ 环境额外安装 `joanhey/adapterman`。

```bash
php artisan v2board:install
```

交互式安装：复制 `.env.example`、生成 `APP_KEY`、写入数据库配置、导入 `database/install.sql`，并创建管理员账号。

```bash
php artisan v2board:update
```

执行项目更新 SQL，刷新配置缓存，并通过 `horizon:terminate` 重启队列工作进程。

```bash
php artisan serve
```

本地开发可用的 Laravel HTTP 服务入口。

```bash
php webman.php start
```

通过 Adapterman/Workerman 启动常驻 HTTP 服务，监听 `127.0.0.1:6600`。该模式需要 PHP 8 环境安装 `joanhey/adapterman`，入口会加载 `start.php` 并复用 Laravel Kernel。

```bash
php artisan horizon
```

启动队列工作进程。`pm2.yaml` 也以该命令作为 PM2 进程脚本，日志写入 `storage/logs/queue/queue.log`。

```bash
php artisan schedule:run
```

手动触发 Laravel 计划任务。计划任务包括每分钟更新流量、检查订单/工单、每日统计/重置日志和流量、定时发送提醒邮件、Horizon 指标快照。

```bash
vendor/bin/phpunit
```

运行全部测试。`phpunit.xml` 定义 Unit 与 Feature 两个测试套件，并在测试启动时缓存 config/event，结束后清理 `bootstrap/cache/*.phpunit.php`。

```bash
vendor/bin/phpunit --testsuite Unit
vendor/bin/phpunit --testsuite Feature
vendor/bin/phpunit tests/Unit/ExampleTest.php
vendor/bin/phpunit --filter ExampleTest
```

运行指定测试套件、测试文件或按名称过滤单个测试。

```bash
php artisan config:clear
php artisan config:cache
php artisan horizon:terminate
```

修改配置或 `.env` 后常用的刷新流程。README 的迁移说明要求将 `CACHE_DRIVER` 配置为 `redis` 后执行这些命令。

## 架构结构

### HTTP 入口和路由

`routes/web.php` 只处理前台主题页面、后台入口页面，以及可选的订阅路径。前台主题由 `App\\Services\\ThemeService` 初始化并渲染 `theme::<theme>.dashboard`；后台入口路径来自 `v2board.secure_path`，否则回退到 `frontend_admin_path` 或 `app.key` 的 CRC32B 哈希。

API 路由不在 `routes/api.php` 中。`App\\Providers\\RouteServiceProvider` 将 API 分为 `/api/v1` 和 `/api/v2`，并动态加载 `app/Http/Routes/V1/*.php` 与 `app/Http/Routes/V2/*.php` 中的路由类。新增接口时优先找到对应分区的 Route 类，再映射到 `app/Http/Controllers/V1|V2/...` 控制器。

`app/Http/Kernel.php` 的 `api` 中间件组强制 JSON 响应并处理语言。常用路由中间件包括 `user`、`admin`、`staff`、`client` 和 `log`，权限边界由这些自定义中间件实现。

### 主要业务分区

`app/Http/Controllers/V1` 按访问主体分区：`Admin`、`User`、`Guest`、`Passport`、`Client`、`Server`、`Staff`。后台管理、用户端、登录注册、客户端订阅、节点后端接口和员工接口分别放在对应目录。

`app/Http/Routes/V1` 与控制器分区对应：`AdminRoute`、`UserRoute`、`GuestRoute`、`PassportRoute`、`ClientRoute`、`ServerRoute`、`StaffRoute`。`app/Http/Routes/V2/ServerRoute.php` 目前提供新版服务端配置和证书固定接口。

`app/Services` 放置跨控制器的业务服务，例如订单、支付、套餐、用户、节点、统计、工单、邮件、Telegram 和主题。涉及核心业务流程时先检查对应 Service，而不是把逻辑直接堆到 Controller。

`app/Models` 是 Eloquent 模型层，覆盖用户、订单、套餐、支付、优惠券、礼品卡、知识库、公告、工单、统计以及多种节点协议模型。节点协议相关模型包括 Vmess、Vless、Trojan、Shadowsocks、Hysteria、Tuic、AnyTLS、V2node 等。

`app/Jobs`、`app/Console/Commands` 和 `app/Console/Kernel.php` 共同承担异步任务与定时任务。Horizon 配置中的队列包括 `order_handle`、`traffic_fetch`、`stat`、`send_email`、`send_email_mass`、`send_telegram`。

### 配置和缓存

项目大量使用 `config('v2board.*')` 读取面板设置；这些配置不在默认 Laravel 配置文件列表里显式出现，很多值来自数据库配置缓存。修改安装、主题、安全路径、订阅路径、支付或队列相关逻辑时，需要注意配置缓存和 `php artisan config:cache` 的影响。

`.env.example` 默认使用 MySQL、Redis cache、Redis queue、Redis session。测试环境在 `phpunit.xml` 中覆盖为 array cache/session、sync queue，并使用独立的 phpunit 缓存文件名。

### 前端和主题

仓库没有 `package.json`，没有可见的 Node 构建脚本。前台页面由 Laravel view 和主题系统渲染，后台入口返回 `admin` 视图。主题配置保存在 `config('theme.<theme>')`，缺失时由 `ThemeService` 初始化。

### 安装与更新 SQL

`php artisan v2board:install` 导入 `database/install.sql`，`php artisan v2board:update` 导入 `database/update.sql`。这两个命令直接执行 SQL 文件中的语句，并会刷新配置缓存；排查安装/升级问题时应同时检查 SQL 文件、数据库连接和配置缓存。
