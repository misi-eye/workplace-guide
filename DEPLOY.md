# 职场图鉴 · 生产部署说明

构建版本见 `index.html` 里的 `<meta name="app-version">`，Service Worker 缓存名同为该版本号。

## 一、发布包内容

```
workplace-guide-dist/
├── index.html          入口（含 OG/Twitter/canonical/安全区 meta）
├── styles.css          样式（已压缩）
├── app.js 等 13 个模块  业务逻辑（已压缩，ES Module）
├── loader.js           第三方库按需加载器
├── chart.umd.min.js    Chart.js（结果页雷达图，按需加载）
├── html2canvas.min.js  分享图渲染（点"生成分享图"时才加载）
├── manifest.json       PWA 清单
├── sw.js               Service Worker（缓存名已注入版本号）
├── icon.svg / icon-192.png / icon-512.png
├── og-image.png        1200×630 社交分享预览图
└── robots.txt
```

纯静态、零后端、零外部请求（Chart.js 与 html2canvas 都是本地文件）。

## 二、上传到服务器

把目录内的**所有文件**（不含本 DEPLOY.md）上传到站点根目录，例如：

```bash
scp -r workplace-guide-dist/* user@server:/var/www/workplace/
```

要求：
1. **必须 HTTPS** —— Service Worker 只在 https（或 localhost）下注册成功。
2. 站点根目录直接放这些文件（不要多套一层子目录），因为 `sw.js` 的作用域是它所在目录。
3. 服务器**不需要任何 rewrite 规则**：本应用没有前端路由路径，所有页面切换都靠 query 参数（`?type=` / `?scene=` / `?r=`），静态托管即可。

## 三、Nginx 参考配置

```nginx
server {
    listen 443 ssl http2;
    server_name workplace.misi-changeye.com.cn;

    ssl_certificate     /path/to/fullchain.pem;
    ssl_certificate_key /path/to/privkey.pem;

    root /var/www/workplace;
    index index.html;

    # 1) HTML 不缓存：保证发版后用户立刻拿到新版（SW 也是网络优先）
    location = /index.html {
        add_header Cache-Control "no-cache, must-revalidate";
    }
    location = / {
        add_header Cache-Control "no-cache, must-revalidate";
    }

    # 2) Service Worker 不缓存，否则旧 SW 会一直活下去
    location = /sw.js {
        add_header Cache-Control "no-cache, must-revalidate";
    }

    # 3) 静态资源短缓存（文件名不含 hash，不宜过长）
    location ~* \.(js|css|png|svg|json)$ {
        add_header Cache-Control "public, max-age=86400";
    }

    # 4) gzip
    gzip on;
    gzip_types text/html text/css application/javascript application/json image/svg+xml;
    gzip_min_length 1024;
}

server {
    listen 80;
    server_name workplace.misi-changeye.com.cn;
    return 301 https://$host$request_uri;
}
```

## 四、发版流程

```bash
cd workplace-guide
python3 build.py          # 生成 ../workplace-guide-dist/ 和 .zip
```

构建脚本会自动：
- 压缩 JS/CSS
- 把构建号写入 `index.html`（`app-version`）和 `sw.js`（缓存名 `workplace-guide-<构建号>`）

**每次发版都必须重新跑一次 build**——缓存名带构建号，用户端会自动丢弃旧缓存、加载新版。

## 五、上线自检清单

| # | 检查项 | 方法 |
|---|--------|------|
| 1 | HTTPS 可访问、HTTP 自动跳 HTTPS | 浏览器打开 `http://域名` 应跳转 |
| 2 | 首页正常，顶栏不显示"重新测评" | 打开首页 |
| 3 | 版本号正确 | 查看源码里的 `app-version` |
| 4 | 完整答题 → 结果页有雷达图 | 走完 40 题 |
| 5 | 分享链接可打开、显示落差句 | 复制 `?r=` 链接，无痕窗口打开 |
| 6 | 场景页直达 | 打开 `?scene=meeting_attack` |
| 7 | 类型页直达 | 打开 `?type=citou` |
| 8 | 离线可用 | 首次访问后关网刷新，应仍能打开 |
| 9 | iOS 加到主屏显示"职场图鉴" | Safari → 分享 → 添加到主屏幕 |
| 10 | 微信/微博分享卡片显示 og-image | 发到文件传输助手看预览图 |

## 六、已知边界

- `?r=` 分享链接的**动态 OG 卡片**对微信/微博静态爬虫无效（纯前端无解，需要短链服务或 SSR）。分享到这些平台时显示的是统一的 `og-image.png`。
- 24 个场景 id 见 `scenes.js`，例如 `meeting_attack`（他在会上怼你）、`credit_steal`（他在抢你的功）。
- 16 个类型 id 见 `types.js`，例如 `shuaguo`（甩锅侠）、`citou`（刺头型）。
