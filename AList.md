# AList

> 存档时间：2026-08-23
> 主题：AList / OpenList，多存储文件聚合、WebDAV、NAS、STRM、302、Emby/Jellyfin 与自动化生态

## 项目介绍

### AList 主项目

- **项目名称**：AlistGo/alist
- **链接**：https://github.com/AlistGo/alist
- **简介**：一个支持多存储的文件列表/WebDAV 程序，后端使用 Go/Gin，前端使用 SolidJS。
- **Star**：50,079（本次检索时）
- **创建时间**：2020-12-23
- **最近更新时间**：2026-08-22
- **许可证**：AGPL-3.0
- **主要技术栈**：Go、Gin、SolidJS；Docker 为主要部署方式之一。
- **核心能力**：
  - 将不同网盘、对象存储、文件协议统一抽象为一层 Storage Driver。
  - 支持本地存储、阿里云盘、OneDrive/SharePoint、Google Drive、百度网盘、夸克、115、123 云盘、迅雷、S3、Azure Blob、FTP/SFTP、SMB、WebDAV、Mega、PikPak、Dropbox 等大量后端。
  - 统一提供 Web 文件管理、WebDAV、预览、上传、删除、重命名、移动、复制、跨存储复制、文件直链、打包下载、离线下载、多线程下载等能力。
  - 支持 PDF、Markdown、代码、文本、图片、视频、音频、字幕及 Office 文档预览。
  - 支持 Docker 部署、Cloudflare Workers 代理、API 文档与 WebDAV 接入。
- **架构价值**：更像一个“统一 Storage Adapter / Gateway”，不是单纯网盘列表。上层可以通过统一 API/WebDAV 访问不同存储后端。
- **典型架构**：`不同 Storage Driver -> AList 统一文件层 -> REST API / WebDAV / 文件直链 -> 播放器、NAS、同步工具、业务系统`。
- **302/流量能力**：部分存储场景可获取真实下载地址后返回重定向，使大文件流量不必全部经过 AList 服务器；是否能直接 302 取决于具体 Storage Driver 与配置。
- **适合二开的场景**：多云文件中心、企业文件聚合、NAS 网盘统一入口、RAG/AI 知识库数据源接入层、媒体库 Storage Gateway、跨存储同步与文件自动化。
- **关联性**：核心主题项目。
- **可验证实现信号**：有完整源码、Storage Drivers、REST API、WebDAV、Docker 部署、官方文档和真实生态项目。

### 值得重点复用的 AList 架构思路

1. **统一 Storage Adapter**：将各平台差异封装在 Driver 中，上层只面对统一文件接口。
2. **WebDAV Gateway**：把不支持 WebDAV 的网盘统一转换为标准协议，交给 Infuse、Kodi、RaiDrive、系统文件管理器、同步软件等消费。
3. **API 自动化入口**：第三方系统可调用 AList 管理/文件 API，实现自动创建存储、同步、归档和上层业务集成。
4. **STRM / 302 媒体链路**：通过 STRM 只保存媒体路径或 URL，媒体服务器管理元数据，真实视频由网盘/CDN 提供，降低本地硬盘和服务器上行带宽需求。
5. **AI/RAG 文件数据源层**：可将 AList Storage Drivers 作为 Google Drive、OneDrive、S3、NAS、FTP、WebDAV 等统一接入层，再接文件扫描、解析、Chunk、Embedding 和 Vector DB。

---

## 全网落地文章与官方实践资料

### 1. 官方：Docker 部署 AList

- **链接**：https://alist-repo.github.io/docs/zh/guide/install/docker.html
- **来源**：AList 官方文档
- **时间**：持续更新
- **摘要**：官方 Docker 安装与运维文档。
- **实现/部署信息**：Docker run、Docker Compose、数据目录持久化、容器升级、管理员密码初始化等。
- **落地场景**：VPS、NAS、Linux 主机、Docker 宿主机部署 AList。
- **热度**：官方资料，无统一互动指标。
- **主题关联性**：高。
- **可验证信号**：官方部署步骤与命令，可直接复现。

### 2. 官方：WebDAV 配置

- **链接**：https://alistgo.com/guide/webdav.html
- **来源**：AList 官方文档
- **时间**：持续更新
- **摘要**：说明 AList 暴露 WebDAV 后的访问路径、权限和客户端用法。
- **实现/架构信息**：默认 WebDAV 路径为 `/dav/`；新版写操作需要对应 WebDAV 管理权限以及 Rename/Delete/Copy 等基础权限。
- **落地场景**：Infuse、Kodi、PotPlayer、RaiDrive、NAS 文件管理器、系统网络磁盘、同步工具。
- **热度**：官方资料。
- **主题关联性**：高。
- **可验证信号**：标准协议入口与官方配置说明。

### 3. 官方：Nginx / Apache / Caddy 反向代理

- **链接**：https://alistgo.com/guide/install/reverse-proxy.html
- **来源**：AList 官方文档
- **时间**：持续更新
- **摘要**：AList 对公网暴露、HTTPS 和反向代理的官方配置参考。
- **实现/架构信息**：反代时需要正确处理 Host、Range、If-Range 等 Header；大文件流媒体场景需要避免代理层不必要缓存，防止把远程大文件先落到本机。
- **落地场景**：公网 AList、域名 HTTPS、影音远程播放、WebDAV 远程访问。
- **热度**：官方资料。
- **主题关联性**：高。
- **可验证信号**：官方反代配置。

### 4. 官方：STRM 驱动

- **链接**：https://alistgo.com/zh/guide/drivers/strm.html
- **来源**：AList 官方文档
- **时间**：2026-05 前后上线/更新
- **摘要**：AList 3.58+ 提供 STRM 相关虚拟驱动能力。
- **实现/架构信息**：可从挂载路径生成/暴露 STRM，用小型文本文件指向真实媒体资源，减少传统全量文件挂载和媒体扫描压力。
- **落地场景**：Emby、Jellyfin、Kodi、NAS 影视中心。
- **热度**：官方资料。
- **主题关联性**：高。
- **可验证信号**：AList 官方版本能力，可直接配置复现。

### 5. Alist + 网盘，不用 NAS 也能打造影音库

- **链接**：https://blog.qust.me/infuse-alist
- **来源/作者**：qust.me / 酱紫表相关实践
- **时间**：2023-04
- **摘要**：用 AList 把网盘暴露给播放器，在没有传统 NAS 的情况下搭建个人影音库。
- **实现/部署信息**：涉及路由器、Android/Termux、绿联、群晖等部署形态；通过 WebDAV 接 Infuse、Kodi、Nova Player 等客户端。
- **落地场景**：Apple TV / iPhone / iPad / Mac 或电视端网盘影音库。
- **热度**：对应 Bilibili 视频超过 31 万播放，见 Bilibili 分组。
- **主题关联性**：高。
- **可验证信号**：有完整文字教程 + 视频演示 + 客户端配置。

### 6. 个人 Obsidian 同步和分享方案：AList + rclone + PicHoro

- **链接**：https://www.horosama.com/archives/256
- **来源/作者**：萌萌哒赫萝
- **时间**：2022-12 / 后续有知乎版本更新
- **摘要**：将 AList 用作个人文件/笔记同步基础设施，而不只是影音工具。
- **实现/架构信息**：AList WebDAV -> rclone -> 本地挂载/同步 -> Obsidian；同时使用对象存储作为图片资源层。
- **落地场景**：跨设备 Obsidian 笔记、文件同步和远程访问。
- **热度**：无统一公开互动指标。
- **主题关联性**：高。
- **可验证信号**：有安装命令、Nginx 配置、rclone 接入路径。

### 7. 实用 Docker 推荐：搭建 AList + WebDAV

- **链接**：https://www.lincol29.cn/usealist
- **来源**：个人技术博客
- **时间**：2024-08
- **摘要**：Docker Compose 部署 AList 并通过 WebDAV 挂载网盘。
- **实现/部署信息**：Compose、容器端口、持久化目录、多网盘配置、WebDAV 客户端接入。
- **落地场景**：个人服务器/NAS 文件聚合。
- **热度**：无统一公开指标。
- **主题关联性**：高。
- **可验证信号**：有 Compose 和可执行配置。

### 8. docker-compose 安装 AList

- **链接**：https://blog.nas.org.cn/34618402/4.html
- **来源**：NAS 技术博客
- **时间**：2023-08
- **摘要**：以 Docker Compose 方式在 NAS/服务器上运行 AList。
- **实现/部署信息**：完整 Compose、管理员密码、数据目录与升级方式。
- **落地场景**：NAS、Docker 主机。
- **热度**：无统一指标。
- **主题关联性**：高。
- **可验证信号**：部署命令可直接复现。

### 9. 绿联 NAS：AList 网盘挂载保姆级教程

- **链接**：https://post.smzdm.com/p/a3mnnwpk/
- **来源**：什么值得买社区
- **时间**：2026
- **摘要**：在绿联 UGOS/NAS Docker 环境中部署 AList，并将云盘挂入 NAS 使用。
- **实现/部署信息**：Docker Compose、数据目录、5244 服务、WebDAV 与 NAS 文件管理/影音库联动。
- **落地场景**：绿联 NAS 多网盘统一入口。
- **热度**：社区文章，具体互动未在本次结果中结构化返回。
- **主题关联性**：高。
- **可验证信号**：设备级实际部署与配置步骤。

### 10. Emby + AList + STRM + 302 实战

- **链接**：https://www.5yyx.com/?p=3998
- **来源**：个人技术站
- **时间**：2025
- **摘要**：通过 STRM 和 302 将云盘媒体接入 Emby，同时减少服务器中转带宽。
- **实现/架构信息**：AutoFilm 生成 STRM、MediaWarp/类似中间层做路径改写，AList 提供网盘访问和直链能力。
- **落地场景**：云端影视库、Emby/Jellyfin、远程播放。
- **热度**：无统一指标。
- **主题关联性**：高。
- **可验证信号**：包含具体组件组合、部署和媒体路径链路。

### 11. 1Panel 可视化部署 AList

- **链接**：https://1panel.cn/docs/v2/user_manual/appstore/alist/
- **来源**：1Panel 官方文档
- **时间**：持续更新
- **摘要**：通过 1Panel 应用商店可视化部署 AList。
- **实现/部署信息**：应用安装、端口、管理员初始化、容器管理、反向代理组合。
- **落地场景**：不想手写 Compose 的 VPS/NAS 用户。
- **热度**：官方资料。
- **主题关联性**：高。
- **可验证信号**：官方应用部署流程。

### 12. 小雅 AList + NAS/WebDAV 部署案例

- **链接**：https://www.sohu.com/a/815358917_122014422
- **来源**：搜狐公开文章
- **时间**：2024
- **摘要**：小雅 AList 生态在 NAS 中的部署与媒体资源组织。
- **实现/部署信息**：xiaoya + AList 多容器、NAS 数据目录、端口、WebDAV/媒体联动。
- **落地场景**：家庭媒体中心。
- **热度**：无结构化互动指标。
- **主题关联性**：中高。
- **可验证信号**：真实多容器部署案例。

---

## GitHub 项目

### 1. AlistGo/alist

- **链接**：https://github.com/AlistGo/alist
- **简介**：AList 主项目，多存储文件列表与 WebDAV Server。
- **主要技术栈**：Go/Gin + SolidJS。
- **落地能力**：Storage Driver、多云盘统一管理、REST API、WebDAV、预览、文件操作、Docker。
- **Star**：50,079
- **最近更新时间**：2026-08-22
- **实践场景**：统一文件层、NAS、WebDAV、影音库、RAG 数据源入口。
- **关联性**：核心。
- **验证信号**：完整源码、官方文档、部署方案、生产生态。

### 2. OpenListTeam/OpenList

- **链接**：https://github.com/OpenListTeam/OpenList
- **简介**：AList 社区分支，当前大量新生态项目同时兼容 OpenList/AList。
- **主要技术栈**：AList 衍生技术栈；本次未单独核实全部语言占比。
- **落地能力**：AList 兼容文件聚合与 WebDAV，成为 2025–2026 年很多衍生项目的新兼容目标。
- **Star**：24,091
- **最近更新时间**：2026-08-14
- **实践场景**：AList 社区替代分支、NAS、媒体库、WebDAV。
- **关联性**：高。
- **验证信号**：公开源码、活跃更新、生态兼容。

### 3. xiaoyaDev/xiaoya-alist

- **链接**：https://github.com/xiaoyaDev/xiaoya-alist
- **简介**：小雅 AList 相关周边和完整媒体生态。
- **主要技术栈**：多容器/Docker 生态；具体语言未在本次结果中完整核实。
- **落地能力**：Emby/Jellyfin 全家桶、元数据、Auto_Symlink、TVBox、Proxy、STRM 等组合安装。
- **Star**：8,419
- **最近更新时间**：2026-08-22
- **实践场景**：家庭云端影视库、大规模网盘媒体资源。
- **关联性**：高。
- **验证信号**：源码 + 部署脚本/文档 + 大规模实际使用。

### 4. power721/alist-tvbox

- **链接**：https://github.com/power721/alist-tvbox
- **简介**：AList proxy server for TVBox，支持播放列表和搜索。
- **主要技术栈**：本次未完整核实。
- **落地能力**：把 AList 作为 Storage Backend，再向 TVBox/播放器提供上层媒体聚合 API；支持搜索、播放、账户、订阅等 API。
- **Star**：3,077
- **最近更新时间**：2026-08-22
- **实践场景**：电视端统一播放、AList 媒体聚合 API。
- **关联性**：高。
- **验证信号**：公开源码、实际 API 与部署文档。

### 5. krau/SaveAny-Bot

- **链接**：https://github.com/krau/SaveAny-Bot
- **简介**：将 Telegram 文件保存到 AList、Disk、WebDAV、S3、rclone 等后端。
- **主要技术栈**：本次未核实。
- **落地能力**：Telegram -> AList/WebDAV/S3 自动归档，支持受限内容和大文件处理。
- **Star**：2,480
- **最近更新时间**：2026-08-22
- **实践场景**：聊天文件自动归档、内容流转中心。
- **关联性**：中高。
- **验证信号**：公开源码 + 多存储后端适配。

### 6. dr34m-cn/taosync

- **链接**：https://github.com/dr34m-cn/taosync
- **简介**：适用于 OpenList v3+/AList 的自动化同步工具。
- **主要技术栈**：本次未核实。
- **落地能力**：在不同存储/目录间自动同步，适合作为 AList 上层数据搬运层。
- **Star**：1,526
- **最近更新时间**：2026-07-28
- **实践场景**：多云同步、NAS 文件归档、备份。
- **关联性**：高。
- **验证信号**：公开源码、直接声明兼容 AList/OpenList。

### 7. traceless/alist-encrypt

- **链接**：https://github.com/traceless/alist-encrypt
- **简介**：AList 服务代理，为 WebDAV 提供透明加解密。
- **主要技术栈**：代理服务；本次未完整核实语言。
- **落地能力**：WebDAV 下自动加解密，支持加密视频在线播放、加密图片查看，客户端操作透明。
- **Star**：1,504
- **最近更新时间**：2026-07-30
- **实践场景**：不信任云端存储、个人隐私文件、加密媒体库。
- **关联性**：高。
- **验证信号**：源码 + DockerCompose 社区部署文章。

### 8. LeoHaoVIP/AListLiteAndroid

- **链接**：https://github.com/LeoHaoVIP/AListLiteAndroid
- **简介**：面向 Android 的 AList/OpenList 服务端封装，Android 5.0+ 即装即用。
- **主要技术栈**：Android；本次未核实具体语言。
- **落地能力**：无需独立 NAS/VPS，在旧手机或安卓设备运行 AList 服务。
- **Star**：1,310
- **最近更新时间**：2026-07-24
- **实践场景**：旧手机变文件服务器/网盘网关。
- **关联性**：高。
- **验证信号**：公开源码、Android 实际服务端封装。

### 9. AkimioJR/AutoFilm

- **链接**：https://github.com/AkimioJR/AutoFilm
- **简介**：为 Emby/Jellyfin 生成 STRM、追番和媒体库海报的工具。
- **主要技术栈**：本次未完整核实。
- **落地能力**：AList/网盘 -> STRM -> Emby/Jellyfin，减少网盘全量刮削和本地空间占用。
- **Star**：1,031
- **最近更新时间**：2026-08-02
- **实践场景**：自动化媒体库、STRM 文件生成。
- **关联性**：高。
- **验证信号**：公开源码 + 多篇社区部署教程引用。

### 10. AlistGo/docs

- **链接**：https://github.com/AlistGo/docs
- **简介**：AList v3 官方文档仓库。
- **主要技术栈**：文档工程。
- **落地能力**：Storage Drivers、API、WebDAV、部署、反代等官方复现信息。
- **Star**：764
- **最近更新时间**：2026-08-21
- **实践场景**：开发/部署参考。
- **关联性**：高。
- **验证信号**：官方文档源码。

### 11. AkimioJR/MediaWarp

- **链接**：https://github.com/AkimioJR/MediaWarp
- **简介**：Emby/Jellyfin/飞牛影视中间件，面向 STRM 播放链路优化。
- **主要技术栈**：本次未完整核实。
- **落地能力**：播放请求路径改写、代理、客户端控制、脚本嵌入，常与 AutoFilm/AList 组合。
- **Star**：752
- **最近更新时间**：2026-07-15
- **实践场景**：Emby/Jellyfin -> AList/OpenList -> 网盘 302/直链。
- **关联性**：高。
- **验证信号**：源码 + Bilibili/文章实战。

### 12. AmbitiousJun/go-emby2openlist

- **链接**：https://github.com/AmbitiousJun/go-emby2openlist
- **简介**：Go 编写的 Emby + OpenList(AList) 网盘直链反向代理服务。
- **主要技术栈**：Go、Docker Compose。
- **落地能力**：Emby 播放请求映射至 AList/OpenList 网盘直链；适配阿里云盘转码播放，并支持本地目录树生成。
- **Star**：659
- **最近更新时间**：2026-08-15
- **实践场景**：Emby 云盘媒体库、302 播放。
- **关联性**：高。
- **验证信号**：源码、Go 实现、Docker Compose 一键部署。

### 13. AlistGo/alist-web

- **链接**：https://github.com/AlistGo/alist-web
- **简介**：AList V3 前端源码。
- **主要技术栈**：SolidJS。
- **落地能力**：可用于 AList 前端二开、UI/交互定制。
- **Star**：481
- **最近更新时间**：2026-08-05
- **实践场景**：自定义企业文件门户、品牌化前端。
- **关联性**：高。
- **验证信号**：官方前端源码。

### 14. sbwml/luci-app-openlist2

- **链接**：https://github.com/sbwml/luci-app-openlist2
- **简介**：OpenWrt LuCI 对 OpenList/AList 的支持包。
- **主要技术栈**：OpenWrt/LuCI。
- **落地能力**：路由器图形化安装和管理 AList/OpenList。
- **Star**：322
- **最近更新时间**：2026-08-07
- **实践场景**：软路由做常驻 AList 网关。
- **关联性**：高。
- **验证信号**：源码 + 大量路由器部署视频。

### 15. AlliotTech/openalist

- **链接**：https://github.com/AlliotTech/openalist
- **简介**：基于 AList v3.45 的社区分支，强调安全、可定制和用户体验。
- **主要技术栈**：AList 衍生。
- **落地能力**：可作为 AList 替代/二开基线。
- **Star**：245
- **最近更新时间**：2026-07-22
- **实践场景**：社区维护的文件聚合服务。
- **关联性**：中高。
- **验证信号**：公开源码。

### 16. jkoor/Alist-Media-Rename

- **链接**：https://github.com/jkoor/Alist-Media-Rename
- **简介**：获取 TMDB 电影/剧集信息并对 AList 媒体文件进行刮削/重命名。
- **主要技术栈**：本次未核实。
- **落地能力**：把 AList 文件 API/媒体目录接到 TMDB 元数据处理流程。
- **Star**：125
- **最近更新时间**：2026-07-22
- **实践场景**：自动整理网盘影视文件。
- **关联性**：高。
- **验证信号**：源码 + 真实媒体自动化功能。

### 17. BenjiThatFoxGuy-Labs/bclone

- **链接**：https://github.com/BenjiThatFoxGuy-Labs/bclone
- **简介**：rclone 分支，增加 AList、iCloud Photos、Terabox 等后端支持。
- **主要技术栈**：rclone 衍生；具体语言本次未核实。
- **落地能力**：直接把 AList 当作 rclone Storage Backend。
- **Star**：71
- **最近更新时间**：2026-07-29
- **实践场景**：命令行同步、挂载、备份。
- **关联性**：高。
- **验证信号**：源码 + 明确 AList backend。

### 18. adogecheems/alist-beautification

- **链接**：https://github.com/adogecheems/alist-beautification
- **简介**：AList/OpenList 动态美化方案。
- **主要技术栈**：前端注入/主题类方案。
- **落地能力**：无需重做完整前端即可定制 AList 展示层。
- **Star**：62
- **最近更新时间**：2026-08-21
- **实践场景**：公开文件站、企业品牌门户。
- **关联性**：中。
- **验证信号**：源码 + 动态 UI 定制。

### 19. power721/atv-player

- **链接**：https://github.com/power721/atv-player
- **简介**：alist-tvbox 桌面播放器。
- **主要技术栈**：本次未核实。
- **落地能力**：消费 alist-tvbox/AList 的媒体资源。
- **Star**：37
- **最近更新时间**：2026-08-22
- **实践场景**：桌面网盘播放器。
- **关联性**：中高。
- **验证信号**：公开播放器项目。

### 20. chenbin3625/OpenSync

- **链接**：https://github.com/chenbin3625/OpenSync
- **简介**：飞牛 NAS / Docker 下的 AList/OpenList 自动同步工具。
- **主要技术栈**：本次未核实。
- **落地能力**：自动同步 AList/OpenList 中的目录或资源。
- **Star**：32
- **最近更新时间**：2026-08-20
- **实践场景**：fnOS、Docker、NAS 自动归档。
- **关联性**：高。
- **验证信号**：源码 + 明确 AList/OpenList 同步功能。

### 21. Liki4/qnap-openlist-webdav

- **链接**：https://github.com/Liki4/qnap-openlist-webdav
- **简介**：QNAP/威联通环境下的 OpenList/AList WebDAV 方案。
- **主要技术栈**：本次未核实。
- **落地能力**：在 QNAP 上暴露/管理 AList/OpenList WebDAV。
- **Star**：20
- **最近更新时间**：2026-07-23
- **实践场景**：威联通 NAS。
- **关联性**：高。
- **验证信号**：设备级落地源码。

### 22. nu1lkali/AlistClientN

- **链接**：https://github.com/nu1lkali/AlistClientN
- **简介**：基于 AList API 的 Flutter 移动客户端，支持 Android。
- **主要技术栈**：Flutter。
- **落地能力**：展示如何直接消费 AList REST API 构建独立客户端。
- **Star**：17
- **最近更新时间**：2026-07-23
- **实践场景**：AList 移动端二开。
- **关联性**：高。
- **验证信号**：源码 + AList API 实际调用。

### 23. alist-org/web-dist

- **链接**：https://github.com/alist-org/web-dist
- **简介**：AList Web 构建产物。
- **主要技术栈**：前端构建产物。
- **落地能力**：部署/替换 Web UI。
- **Star**：17
- **最近更新时间**：2026-08-05
- **实践场景**：AList Web 部署。
- **关联性**：中。
- **验证信号**：官方构建资产。

### 24. delayboy/Magic-AList-For-BaiduDisk

- **链接**：https://github.com/delayboy/Magic-AList-For-BaiduDisk
- **简介**：自制百度网盘同步工具，魔改 AList 并适配 Windows 窗口 UI。
- **主要技术栈**：本次未核实。
- **落地能力**：把 AList 从 Web 服务改造成面向百度网盘的桌面同步工具。
- **Star**：14
- **最近更新时间**：2026-07-21
- **实践场景**：Windows 桌面百度网盘同步。
- **关联性**：中高。
- **验证信号**：AList 魔改源码。

### 25. Alien-Et/Alist-Magisk

- **链接**：https://github.com/Alien-Et/Alist-Magisk
- **简介**：通过 Magisk 将 AList 文件服务集成进 Android，支持 ARM/ARM64。
- **主要技术栈**：Android/Magisk。
- **落地能力**：系统级常驻 AList Server。
- **Star**：14
- **最近更新时间**：2026-08-05
- **实践场景**：安卓盒子、旧手机常驻网盘网关。
- **关联性**：中高。
- **验证信号**：公开模块源码。

### 26. power721/PowerList

- **链接**：https://github.com/power721/PowerList
- **简介**：AList 社区分支。
- **主要技术栈**：AList 衍生。
- **落地能力**：AList 兼容服务端。
- **Star**：10
- **最近更新时间**：2026-08-23
- **实践场景**：替代分支/定制。
- **关联性**：中。
- **验证信号**：公开源码。

### 27. outlook84/openlist-tvbox-gateway

- **链接**：https://github.com/outlook84/openlist-tvbox-gateway
- **简介**：OpenList/AList/WebDAV gateway for TVBox。
- **主要技术栈**：本次未核实。
- **落地能力**：把 AList/WebDAV 资源转换为 TVBox 可消费的网关接口。
- **Star**：9
- **最近更新时间**：2026-07-01
- **实践场景**：电视媒体聚合。
- **关联性**：高。
- **验证信号**：源码 + 明确兼容 AList/WebDAV。

### 28. wab201/alist-tvbox-plugins

- **链接**：https://github.com/wab201/alist-tvbox-plugins
- **简介**：AList-TVBox 插件仓库，带可导入索引、源码和逐插件部署文档。
- **主要技术栈**：插件生态。
- **落地能力**：扩展 AList-TVBox 的数据源与播放能力。
- **Star**：9
- **最近更新时间**：2026-08-22
- **实践场景**：电视端插件化媒体聚合。
- **关联性**：中高。
- **验证信号**：源码 + 部署文档。

### 29. Zhu-junwei/magisk-alist

- **链接**：https://github.com/Zhu-junwei/magisk-alist
- **简介**：Magisk module to run AList on Android devices。
- **主要技术栈**：Android/Magisk。
- **落地能力**：安卓设备部署 AList。
- **Star**：5
- **最近更新时间**：2026-08-06
- **实践场景**：旧手机/电视盒子。
- **关联性**：中。
- **验证信号**：公开模块。

### 30. AlliotTech/openalist-web

- **链接**：https://github.com/AlliotTech/openalist-web
- **简介**：OpenAList/AList V3 衍生前端。
- **主要技术栈**：前端；上游 AList Web 为 SolidJS。
- **落地能力**：自定义文件管理 Web UI。
- **Star**：4
- **最近更新时间**：2026-07-22
- **实践场景**：前端二开。
- **关联性**：中。
- **验证信号**：公开源码。

### 31. jinlin-teck/openlist-strm

- **链接**：https://github.com/jinlin-teck/openlist-strm
- **简介**：从 OpenList/AList 目录生成 `.strm` 文件的轻量服务，带 WebUI。
- **主要技术栈**：Go + Gin/GORM/SQLite + Vue（本次检索得到的项目说明）。
- **落地能力**：无需 FUSE 本地挂载，直接遍历 AList/OpenList 目录生成 STRM；支持手动运行与文件变化监控。
- **Star**：4
- **最近更新时间**：2026-08-01
- **实践场景**：Emby/Jellyfin/Kodi 云盘媒体库。
- **关联性**：高。
- **验证信号**：源码 + WebUI + 增量监控 + 部署能力。

### 32. diannaSIN/plex2alist-

- **链接**：https://github.com/diannaSIN/plex2alist-
- **简介**：Plex 与 AList 联动的魔改版本。
- **主要技术栈**：本次未核实。
- **落地能力**：Plex -> AList 路径/播放链路改造。
- **Star**：3
- **最近更新时间**：2026-08-01
- **实践场景**：Plex 云盘媒体播放实验。
- **关联性**：中。
- **验证信号**：公开魔改源码。

### 33. hrd201/xstrm-suite

- **链接**：https://github.com/hrd201/xstrm-suite
- **简介**：STRM 文件管理系统，面向 Emby + AList 集成。
- **主要技术栈**：本次未完整核实。
- **落地能力**：Admin UI、增量同步、Docker/Nginx 部署、代理友好媒体工作流。
- **Star**：2
- **最近更新时间**：2026-08-16
- **实践场景**：AList STRM 自动化与媒体库维护。
- **关联性**：高。
- **验证信号**：源码 + Docker/Nginx + 增量同步。

### 34. deerwan/songloft-plugin-openlist

- **链接**：https://github.com/deerwan/songloft-plugin-openlist
- **简介**：Songloft 音源插件，通过 OpenList/AList REST API 接入外部网盘音乐库。
- **主要技术栈**：插件；本次未核实具体语言。
- **落地能力**：真实演示“业务应用直接消费 AList REST API”。
- **Star**：2
- **最近更新时间**：2026-08-08
- **实践场景**：网盘音乐库、AList API 二开。
- **关联性**：高。
- **验证信号**：源码 + 明确 REST API 接入。

### 35. fallenwood/DanmakuHub

- **链接**：https://github.com/fallenwood/DanmakuHub
- **简介**：modified AList 的弹幕 Provider。
- **主要技术栈**：本次未核实。
- **落地能力**：在 AList 衍生媒体前端中增加弹幕数据能力。
- **Star**：2
- **最近更新时间**：2026-08-17
- **实践场景**：视频播放二开。
- **关联性**：中。
- **验证信号**：公开源码。

---

## 知乎

### 1. 网盘挂载工具 AList

- **链接**：https://zhuanlan.zhihu.com/p/2061983327335327200
- **作者**：赫点茶
- **时间**：2026-07-19
- **摘要**：AList 网盘挂载与阿里云盘批量下载实践。
- **实现信息**：Docker 部署、5244 服务、获取管理员密码、挂载阿里云盘、WebDAV；进一步使用 Python 批量下载网盘内容。
- **热度**：知乎搜索结果未返回点赞/收藏等结构化数值。
- **关联性**：高。
- **验证信号**：有 Docker 命令、实际网盘配置、Python 批处理步骤。

### 2. 一键聚合18种网盘！Docker部署AList全攻略

- **链接**：https://zhuanlan.zhihu.com/p/1897422673929289759
- **作者**：二冰
- **时间**：2025-04-24
- **摘要**：通过 Dockge/Docker Compose 把多个网盘和 NAS 本地目录聚合到 AList。
- **实现信息**：`docker-compose.yml`、`5244:5244`、`/opt/alist/data`、本地目录挂载、管理员密码获取。
- **热度**：无结构化互动数值。
- **关联性**：高。
- **验证信号**：Compose 配置和实战步骤。

### 3. Docker 容器基于 WebDAV 通过 AList 挂载百度/阿里云盘

- **链接**：https://zhuanlan.zhihu.com/p/606183479
- **作者**：刘悦的技术博客
- **时间**：2023-02-14
- **摘要**：AList 把公共网盘转换为 WebDAV，本地播放器直接消费远程视频。
- **实现信息**：Docker run 命令、WebDAV 映射、Python3.10 相关接入；强调本地播放器负责解码、服务端主要传输数据。
- **热度**：无结构化互动数值。
- **关联性**：高。
- **验证信号**：完整 Docker 命令和播放链路说明。

### 4. AList 实现网盘 All in One

- **链接**：https://zhuanlan.zhihu.com/p/662035279
- **作者**：亦安酒家的客人
- **时间**：2023-10-21
- **摘要**：多网盘统一管理、WebDAV 与 RaiDrive 本地挂载。
- **实现信息**：Docker run、Docker Compose、多 Storage 驱动、WebDAV、RaiDrive。
- **热度**：无结构化互动数值。
- **关联性**：高。
- **验证信号**：包含部署和客户端挂载配置。

### 5. 绿联私有云 AList Docker 教程：多网盘 + Aria2

- **链接**：https://zhuanlan.zhihu.com/p/683469229
- **作者**：刘琦
- **时间**：2024-02-27
- **摘要**：在绿联 NAS 上部署 AList，挂载阿里云盘、百度网盘等，并在网盘间复制文件。
- **实现信息**：Token/Refresh Token、存储配置、WebDAV 接绿联文件管理、读写缓存设置、多网盘复制。
- **热度**：无结构化互动数值。
- **关联性**：高。
- **验证信号**：真实 NAS 界面操作和多盘互传案例。

### 6. 极空间 Docker：把 AList 挂载的网盘添加到本地

- **链接**：https://zhuanlan.zhihu.com/p/623124915
- **作者**：旧时光
- **时间**：2024-11-01
- **摘要**：通过 WebDAV 将 AList 网盘映射为极空间外部设备。
- **实现信息**：AList 后台先挂载 Storage，再使用 NAS IP + 5244 + WebDAV 方式接入极空间。
- **热度**：无结构化互动数值。
- **关联性**：高。
- **验证信号**：极空间实际设备流程。

### 7. 夸克 + AList + CloudDrive2 + AutoSymlink + Emby 自建影视库

- **链接**：https://zhuanlan.zhihu.com/p/21273422473
- **作者**：0x505951
- **时间**：2025-02-04
- **摘要**：完整的云盘媒体自动化 Docker 栈。
- **实现信息**：AList/Aria2、CloudDrive2、AutoSymlink、Emby 多容器 Compose；FUSE、共享挂载、STRM/符号链接媒体目录。
- **热度**：无结构化互动数值。
- **关联性**：高。
- **验证信号**：给出完整 docker-compose 片段和 volume/port 配置。

### 8. NAS + 网盘融合：Docker 部署 AList、CloudDrive2

- **链接**：https://zhuanlan.zhihu.com/p/715374147
- **作者**：我是阿皮啊
- **时间**：2024-10-23
- **摘要**：把 AList 的网盘挂到绿联 NAS，再接 NAS 影视中心，并通过 DDNS/反代远程访问。
- **实现信息**：AList 添加阿里云盘、`/dav` WebDAV、NAS 网络文件夹、AList WebDAV 策略、Lucky DDNS/反代。
- **热度**：无结构化互动数值。
- **关联性**：高。
- **验证信号**：设备级真实部署和播放测试。

### 9. DIY 家庭小主机 AIO：内容流转中心和家庭影音媒体中心

- **链接**：https://zhuanlan.zhihu.com/p/12971800945
- **作者**：hatchling8502
- **时间**：2024-12-23
- **摘要**：将 AList 作为家庭内容流转中心中的统一网盘层。
- **实现信息**：群晖/虚拟化环境、Docker Compose、AList WebDAV、下载与媒体服务协同。
- **热度**：无结构化互动数值。
- **关联性**：高。
- **验证信号**：完整家庭实验室架构上下文。

### 10. 极空间通过 AList 挂载阿里云盘（详细版）

- **链接**：https://zhuanlan.zhihu.com/p/623048582
- **作者**：旧时光
- **时间**：2024-10-30
- **摘要**：极空间 NAS 小白向 Docker 安装 AList 并挂阿里云盘。
- **实现信息**：镜像安装、容器配置、日志取初始密码、桌面快捷方式、挂载网盘。
- **热度**：无结构化互动数值。
- **关联性**：高。
- **验证信号**：实际 NAS 操作步骤。

### 11. 给没有经验的朋友：Emby 115 STRM 302 稳定方法

- **链接**：https://zhuanlan.zhihu.com/p/788070229
- **作者**：bookmoth
- **时间**：2024-10-12
- **摘要**：通过 AList、Nginx、Emby、STRM 构建 115 网盘直链播放。
- **实现信息**：STRM 路径检查、Emby 媒体库、AList 外网访问、Nginx JS 配置、`embyHost`、`embyMountPath`、`embyApiKey`、`alistAddr`、`alistToken` 等具体参数。
- **热度**：无结构化互动数值。
- **关联性**：高。
- **验证信号**：有明确参数、检查点和路径映射算法。

### 12. STRM 生成、Emby 302 和网盘媒体工具项目攻略

- **链接**：https://zhuanlan.zhihu.com/p/1943793195491234280
- **作者**：网络阁
- **时间**：2025-08-26
- **摘要**：汇总脚本遍历、AutoFilm/AutoSymlink/xStrm、CMS/OneStrm、MoviePilot 等 STRM 生成方式，并说明 AList-Nginx-redir 302 链路。
- **实现信息**：STRM URL、本地路径模式、AList 302、独立 redir 容器、MediaLinker/EmbyExternalUrl 路径改写。
- **热度**：无结构化互动数值。
- **关联性**：高。
- **验证信号**：多种真实工具和架构对比。

### 13. 自建媒体服务器，实现国内外影视远程直链

- **链接**：https://zhuanlan.zhihu.com/p/690727400
- **作者**：Thomas
- **时间**：2024-04-04
- **摘要**：从小雅/Emby 实践延伸到自建 AList + Nginx 的路径改写和直链播放。
- **实现信息**：读取 Emby 媒体路径、映射到 AList 网盘路径，通过 Nginx JS/反向代理返回真实资源位置。
- **热度**：无结构化互动数值。
- **关联性**：高。
- **验证信号**：真实 NAS/Emby 反向代理问题和解决路径。

### 14. AList 生成 STRM 接入 NAS 影视中心

- **链接**：https://zhuanlan.zhihu.com/p/31697895664
- **作者**：Stark-C
- **时间**：2025-03-21
- **摘要**：从 AList 中的媒体 URL 生成 STRM，接入绿联 NAS/Emby/Jellyfin 等媒体库。
- **实现信息**：STRM 是只包含 URL/路径的小型文本文件；媒体服务器读取 STRM 后直接访问网盘资源，避免媒体本体下载到 NAS。
- **热度**：无结构化互动数值。
- **关联性**：高。
- **验证信号**：真实 NAS STRM 播放案例。

### 15. 1Panel AList 反向代理出现 Welcome to OpenResty 的排查

- **链接**：https://zhuanlan.zhihu.com/p/32214103073
- **作者**：鸾觞酌醴
- **时间**：2025-03-23
- **摘要**：AList 已用 Docker 安装，再通过 1Panel 建反代站点时的 Nginx/OpenResty 配置错误案例。
- **实现信息**：说明 `location` 应匹配 URI 路径，`/` 与 `server_name` 的职责不同；给出 1Panel 反代问题定位。
- **热度**：无结构化互动数值。
- **关联性**：高。
- **验证信号**：真实故障、根因和修复。

### 16. 家用 NAS 通过 AList 反向代理给 Halo 做存储策略

- **链接**：https://zhuanlan.zhihu.com/p/10500786974
- **作者**：momo
- **时间**：2024-12-09
- **摘要**：把 AList 从“网盘工具”提升为 Halo 博客的外部 Storage Gateway。
- **实现信息**：1Panel -> AList 反向代理 -> HTTPS；给出 Nginx `proxy_pass http://127.0.0.1:5244`、Host/X-Real-IP/X-Forwarded-For/Upgrade 等 Header 配置。
- **热度**：无结构化互动数值。
- **关联性**：高。
- **验证信号**：真实业务系统（Halo）使用 AList 做存储策略。

### 17. AList Docker Compose 部署私人网盘

- **链接**：https://zhuanlan.zhihu.com/p/1898125603472913381
- **作者**：小明同学的笔记
- **时间**：2025-04-22
- **摘要**：Docker Compose 部署 AList 并挂载百度网盘。
- **实现信息**：`5244:5244`、`./data:/opt/alist/data`、PUID/PGID/UMASK、`docker compose up -d`、`docker logs -f alist`、刷新令牌配置。
- **热度**：无结构化互动数值。
- **关联性**：高。
- **验证信号**：可直接复现的 Compose 和配置步骤。

### 18. 服务器搭建 AList + Nginx

- **链接**：https://zhuanlan.zhihu.com/p/1932438376675512899
- **作者**：James
- **时间**：2025-07-26
- **摘要**：VPS 上 Docker 运行 AList，再用 Nginx 绑定域名。
- **实现信息**：创建数据目录、`docker pull`、`docker run`、管理员命令、Nginx `server`/`proxy_pass` 配置。
- **热度**：无结构化互动数值。
- **关联性**：高。
- **验证信号**：完整部署命令和 Nginx 配置。

### 19. 零成本搭建 Zotero WebDAV 同步服务器

- **链接**：https://zhuanlan.zhihu.com/p/2061809870257629077
- **作者**：周树入
- **时间**：2026-07-18
- **摘要**：把 AList 当 Zotero WebDAV 后端，解决 iOS 端只接受受信任 HTTPS 的问题。
- **实现信息**：Azure Ubuntu -> Docker -> AList -> Nginx TLS -> DuckDNS -> Let’s Encrypt -> Zotero；包含证书续签思路。
- **热度**：无结构化互动数值。
- **关联性**：非常高，属于非影音的标准文件协议落地案例。
- **验证信号**：完整端到端架构和客户端约束处理。

### 20. Linux 安装 AList、反向代理教程

- **链接**：https://zhuanlan.zhihu.com/p/650291395
- **作者**：mei232
- **时间**：2023-12-23
- **摘要**：Docker、一键脚本、1Panel 等方式安装 AList，并配置反代。
- **实现信息**：`docker run`、管理员密码命令、官方安装/更新脚本、面板部署思路。
- **热度**：无结构化互动数值。
- **关联性**：高。
- **验证信号**：实际命令与更新方法。

### 21. DockerCompose 快速部署 AList + AList-Encrypt

- **链接**：https://zhuanlan.zhihu.com/p/625032983
- **作者**：太阳是颗黄矮星
- **时间**：2023-04-27
- **摘要**：AList 与 alist-encrypt 双容器透明加密部署。
- **实现信息**：两个 Compose、端口 `5244/5344`、容器数据卷、`host.docker.internal`/`host-gateway` 让 encrypt 容器访问 AList。
- **热度**：无结构化互动数值。
- **关联性**：高。
- **验证信号**：具体 Docker 网络和代理配置。

### 22. 个人 Obsidian 同步和分享方案：AList + rclone + PicHoro

- **链接**：https://zhuanlan.zhihu.com/p/590686830
- **作者**：萌萌哒赫萝
- **时间**：2023-10-23
- **摘要**：AList + rclone 作为跨端笔记/文件层。
- **实现信息**：AList 一键安装、Nginx Range/If-Range 反代配置、站点 URL、rclone 接 WebDAV、Cloudflare R2/PicList 图片存储。
- **热度**：无结构化互动数值。
- **关联性**：非常高，证明 AList 可作为通用文件基础设施。
- **验证信号**：完整命令与 Nginx 配置。

---

## 掘金

> 本批掘金结果来自站点字面搜索补充；Tavily 专用通路在后续补搜时受当前会话权限限制，未新增可归档结果。

### 1. 《阿里云盘 & AList & Kodi》家庭影院搭建指南

- **链接**：https://aicoding.juejin.cn/post/7294841029681168399
- **作者/来源**：掘金
- **时间**：2023-10
- **摘要**：阿里云盘 + AList + Kodi 的家庭影院方案。
- **实现信息**：云服务器部署、AList Storage、Kodi 客户端访问与播放。
- **热度**：本次检索约 3,050 阅读。
- **关联性**：高。
- **验证信号**：具体 AList + Kodi 落地链路。

### 2. 搭建个人影库 TV + Web + Android + iOS

- **链接**：https://juejin.cn/post/7307467804005236763
- **作者/来源**：掘金
- **时间**：2023-12
- **摘要**：Ubuntu + Docker + AList + CloudDrive2 + Jellyfin 多端影音库。
- **实现信息**：服务器部署、多容器媒体服务、网盘接入、多终端消费。
- **热度**：本次检索约 932 阅读。
- **关联性**：高。
- **验证信号**：完整技术组合和部署场景。

### 3. 自动获取 OneDrive 配置并添加 AList 存储

- **链接**：https://juejin.cn/post/7514428702488608787
- **作者/来源**：掘金
- **时间**：2025-06
- **摘要**：用 Python 自动完成 OneDrive OAuth，并调用 AList API 创建 Storage。
- **实现信息**：Azure OAuth -> refresh token -> AList `/api/auth/login` -> `/api/admin/storage/create`；提供完整 Python 源码。
- **热度**：本次检索约 163 阅读。
- **关联性**：非常高，直接体现 AList 作为可编程 Storage Gateway。
- **验证信号**：完整源码、真实 AList Admin API。

### 4. 1Panel 应用推荐：AList

- **链接**：https://juejin.cn/post/7323099490315042868
- **作者/来源**：掘金
- **时间**：2024-01
- **摘要**：通过 1Panel 部署 AList，并使用多存储/WebDAV 功能。
- **实现信息**：面板应用部署、AList 服务和客户端访问。
- **热度**：本次检索约 395 阅读。
- **关联性**：中高。
- **验证信号**：可复现面板部署。

### 5. N1 盒子部署 OpenList 播放 4K

- **链接**：https://juejin.cn/post/7626697192070823972
- **作者/来源**：掘金
- **时间**：2026
- **摘要**：在低功耗 N1 盒子部署 AList 分支 OpenList 并用于 4K 播放。
- **实现信息**：嵌入式/盒子环境部署、媒体播放。
- **热度**：本次结果未返回稳定阅读数。
- **关联性**：中高。
- **验证信号**：真实低功耗硬件落地。

---

## Bilibili

### 1. infuse + alist + 网盘，不用 NAS 打造影音库教程

- **链接**：https://www.bilibili.com/video/BV1dA411D73y
- **UP 主**：酱紫表
- **时长**：07:51
- **摘要**：通过 AList WebDAV 把网盘接入 Infuse，在 Apple TV/iPhone/iPad/Mac 上构建无需传统 NAS 的影音库。
- **实现信息**：AList -> WebDAV -> Infuse；Infuse 负责海报墙、刮削、播放和客户端解码。
- **热度**：315,692 播放、143 弹幕、6,694 赞、4,792 投币、15,928 收藏、1,769 分享（本次详情接口返回值）。
- **关联性**：非常高。
- **验证信号**：高互动真实教程；视频简介还有文字版教程链接。

### 2. 旧手机变网盘！无需代码，在手机上快速部署 AList

- **链接**：https://www.bilibili.com/video/BV1vq421c7kW
- **UP 主**：在下莫老师
- **摘要**：在 Android 手机上直接运行 AList 服务。
- **实现信息**：手机作为常驻文件/WebDAV Server，避免购买独立 NAS。
- **热度**：217,544 播放。
- **关联性**：高。
- **验证信号**：真实移动设备部署。

### 3. AList 全平台部署教程

- **链接**：https://www.bilibili.com/video/BV1j7aNeUEDG
- **UP 主**：米续期
- **摘要**：跨平台安装 AList。
- **实现信息**：多平台部署、网盘挂载、服务访问。
- **热度**：189,455 播放。
- **关联性**：高。
- **验证信号**：高播放量部署视频。

### 4. AList 在软路由 OpenWrt 安装，挂载多网盘并做电视影音库

- **链接**：https://www.bilibili.com/video/BV1PW4y1V78f
- **UP 主**：向北的平行世界
- **摘要**：OpenWrt 上安装 AList，挂阿里/Google/百度等网盘，并映射到电脑/电视。
- **实现信息**：软路由常驻服务 + 多网盘 + WebDAV/影音客户端。
- **热度**：90,486 播放。
- **关联性**：高。
- **验证信号**：真实路由器环境。

### 5. Termux：无需 Root，手机安装 AList

- **链接**：https://www.bilibili.com/video/BV1A24y1e72r
- **UP 主**：私は学少何
- **摘要**：Android/Termux 部署 AList。
- **实现信息**：无需 Root 的手机服务端方案。
- **热度**：51,066 播放。
- **关联性**：高。
- **验证信号**：真实设备教程。

### 6. Windows 无额外软件挂载 AList 网盘

- **链接**：https://www.bilibili.com/video/BV1LbtUexE7C
- **UP 主**：CestLaVie66
- **摘要**：利用 Windows 自带能力把 AList/WebDAV 挂载为本地资源。
- **实现信息**：AList -> WebDAV -> Windows 文件访问。
- **热度**：32,410 播放。
- **关联性**：高。
- **验证信号**：客户端侧标准协议落地。

### 7. Docker 搭建 AList 挂载各类网盘

- **链接**：https://www.bilibili.com/video/BV1SM4y1t73e
- **UP 主**：搭建小天地
- **摘要**：Docker 部署 AList，并挂载阿里/百度等网盘。
- **实现信息**：容器部署、多 Storage Driver 配置。
- **热度**：27,693 播放。
- **关联性**：高。
- **验证信号**：真实 Docker 教程。

### 8. 云盘 NAS 互通互传：AList 部署配置

- **链接**：https://www.bilibili.com/video/BV1Su411n7W1
- **UP 主**：koryking
- **摘要**：AList 作为 NAS 和云盘之间的文件交换层。
- **实现信息**：网盘挂载、NAS 文件访问/传输。
- **热度**：27,094 播放。
- **关联性**：高。
- **验证信号**：NAS 实战。

### 9. 飞牛 fnOS：AList 本地硬盘 + 网盘一起 WebDAV 分享

- **链接**：https://www.bilibili.com/video/BV1VQzcYvEX4
- **UP 主**：好用斋
- **摘要**：在飞牛 NAS 把本地盘和 AList 网盘统一成 WebDAV。
- **实现信息**：fnOS + AList + 本地 Storage + WebDAV。
- **热度**：26,312 播放。
- **关联性**：高。
- **验证信号**：真实 NAS 系统。

### 10. 路由器安装 AList：iStoreOS / OpenWrt

- **链接**：https://www.bilibili.com/video/BV1vG411Y7k3
- **UP 主**：好用斋
- **摘要**：将 AList 常驻在软路由上。
- **实现信息**：iStoreOS/OpenWrt 部署、常驻网盘 Gateway。
- **热度**：25,007 播放。
- **关联性**：高。
- **验证信号**：真实软路由部署。

### 11. Emby 302 + STRM + MediaWarp 网盘直连

- **链接**：https://www.bilibili.com/video/BV1Xb4tznE8L
- **UP 主**：好用斋
- **摘要**：Emby 播放 STRM 时通过中间件改写至网盘直链，减少服务器上行带宽。
- **实现信息**：Emby -> STRM -> MediaWarp -> AList/OpenList/网盘 302。
- **热度**：24,410 播放。
- **关联性**：高。
- **验证信号**：完整媒体链路演示。

### 12. 让所有网盘支持原画 302 播放

- **链接**：https://www.bilibili.com/video/BV1N9sMz1Egz
- **UP 主**：麒麟猪是我
- **摘要**：围绕 AList/OpenList 生态做网盘 302 原画播放和外网域名处理。
- **实现信息**：302、STRM、域名/路径重定向。
- **热度**：24,734 播放。
- **关联性**：高。
- **验证信号**：真实播放教程。

### 13. PotPlayer + AList = 私有影音库

- **链接**：https://www.bilibili.com/video/BV11d4y1W7cr
- **UP 主**：马蹄铁692
- **摘要**：桌面播放器直接读取 AList 网盘资源。
- **实现信息**：AList/WebDAV -> PotPlayer。
- **热度**：22,102 播放。
- **关联性**：高。
- **验证信号**：客户端实际播放。

### 14. 飞牛 NAS 挂载 AList 网盘

- **链接**：https://www.bilibili.com/video/BV1YEB7YMEV7
- **UP 主**：科技智趣坊
- **摘要**：飞牛 NAS 与 AList 的双向网盘挂载实践。
- **实现信息**：fnOS/NAS -> AList -> 其他网盘。
- **热度**：21,394 播放。
- **关联性**：高。
- **验证信号**：NAS 实际部署。

### 15. AList 网盘挂载到本地目录 / rclone 教程

- **链接**：https://www.bilibili.com/video/BV17w4m1r73a
- **UP 主**：爱折腾的员工
- **摘要**：使用 rclone/挂载手段把 AList 远程文件映射到本地目录。
- **实现信息**：AList/WebDAV/rclone、本地目录访问。
- **热度**：17,543 播放。
- **关联性**：高。
- **验证信号**：本地挂载实际操作。

### 16. AList 搭 WebDAV 私有网盘 / 低配 NAS

- **链接**：https://www.bilibili.com/video/BV16g4y1d7kM
- **UP 主**：TSOGview
- **摘要**：低性能设备运行 AList 作为 WebDAV Server。
- **实现信息**：低配 NAS + AList + WebDAV。
- **热度**：17,378 播放。
- **关联性**：高。
- **验证信号**：低配设备实测。

### 17. 群晖 Docker 搭 AList 多网盘聚合

- **链接**：https://www.bilibili.com/video/BV1qL411R7uc
- **UP 主**：是喻先森
- **摘要**：群晖 Docker 部署 AList 并整合多个网盘。
- **实现信息**：Synology + Docker + AList。
- **热度**：16,453 播放。
- **关联性**：高。
- **验证信号**：群晖实战。

### 18. 无需 Rclone：AList-Strm 建网盘媒体库

- **链接**：https://www.bilibili.com/video/BV1VzLEzYE7u
- **UP 主**：好用斋
- **摘要**：直接从 AList 生成 STRM，减少本地挂载复杂度。
- **实现信息**：AList -> STRM -> 媒体服务器刮削/播放。
- **热度**：15,413 播放。
- **关联性**：高。
- **验证信号**：真实媒体库生成演示。

### 19. AList 服务器实现网盘资源共享

- **链接**：https://www.bilibili.com/video/BV11p4y177Ay
- **UP 主**：AMGYzhea
- **摘要**：服务器部署 AList 对外提供统一文件访问。
- **实现信息**：VPS/服务器 + AList + 多网盘。
- **热度**：14,939 播放。
- **关联性**：高。
- **验证信号**：服务器实战。

### 20. 用 AList 搭建云盘家庭 Emby 影音库

- **链接**：https://www.bilibili.com/video/BV1khgPeeE3x
- **UP 主**：劈山葫芦娃
- **摘要**：把 AList 云盘资源接入 Emby。
- **实现信息**：AList -> Emby 媒体库/播放链路。
- **热度**：13,400 播放。
- **关联性**：高。
- **验证信号**：真实 Emby 影音库演示。

### 21. 无需反代：AList + alist-strm + Kodi 302

- **链接**：https://www.bilibili.com/video/BV1PjHseREaJ
- **UP 主**：Jioyzen
- **摘要**：只用 AList 和 alist-strm 容器，把网盘媒体接入 Kodi 并实现直链播放。
- **实现信息**：AList -> alist-strm -> Kodi；减少 Emby/Nginx 复杂度。
- **热度**：9,539 播放。
- **关联性**：高。
- **验证信号**：明确最小容器组合。

### 22. OpenList + Emby 302 全流程容器部署

- **链接**：https://www.bilibili.com/video/BV19cM8zkEHx
- **UP 主**：双栈工坊工作室
- **摘要**：AList 分支 OpenList 与 Emby 的 302 播放全流程。
- **实现信息**：容器部署、网盘媒体、路径/直链。
- **热度**：7,701 播放。
- **关联性**：高。
- **验证信号**：全流程容器教程。

### 23. AList 和 WebDAV 基础使用

- **链接**：https://www.bilibili.com/video/BV1nAdKYhEAz
- **UP 主**：凉冠
- **摘要**：基础 WebDAV 客户端连接和 AList 文件访问。
- **实现信息**：AList `/dav` -> WebDAV client。
- **热度**：6,904 播放。
- **关联性**：高。
- **验证信号**：协议实操。

### 24. AList V3 挂载 Google Drive

- **链接**：https://www.bilibili.com/video/BV18v4y1W7vo
- **UP 主**：阿博特-安稳
- **摘要**：AList Storage Driver 接 Google Drive。
- **实现信息**：Google Drive 授权/配置 -> AList。
- **热度**：7,167 播放。
- **关联性**：高。
- **验证信号**：真实 Driver 配置。

### 25. AList 接 kkFileView 在线预览

- **链接**：https://www.bilibili.com/video/BV1RBN9e5ExV
- **UP 主**：Oo天亮讲晚安O
- **摘要**：AList 与 kkFileView 联动，增强 Office/文档在线预览。
- **实现信息**：AList 文件资源 -> kkFileView 预览服务。
- **热度**：2,504 播放。
- **关联性**：高。
- **验证信号**：明确外部服务集成。

### 26. AList 对接对象存储

- **链接**：https://www.bilibili.com/video/BV1bsvQe3EHz
- **UP 主**：吾要won
- **摘要**：AList Storage Driver 接对象存储。
- **实现信息**：对象存储配置、AList 文件访问。
- **热度**：707 播放。
- **关联性**：高。
- **验证信号**：Storage Driver 实操。

---

## 关联组件：Infuse

- **产品**：Infuse（Firecore）
- **角色**：Apple 生态的视频播放器/影音库客户端，不是 AList/Emby 这种服务端。
- **支持平台**：iPhone、iPad、Apple TV、Mac、Vision Pro。
- **与 AList 的关系**：AList 把阿里云盘、115、夸克、OneDrive、Google Drive、S3、NAS 等统一暴露为 WebDAV；Infuse 连接 AList 的 WebDAV 后负责海报墙、元数据、播放、观看进度与终端解码。
- **典型架构**：`网盘 -> AList -> WebDAV -> Infuse -> Apple TV`。
- **价值**：对于主要使用 Apple TV/iOS/macOS 的个人影音库，可以不部署 Emby/Jellyfin，直接用 AList + Infuse；需要多用户、服务端媒体库、复杂 STRM/302 时再引入 Emby/Jellyfin。
- **验证信号**：Bilibili “infuse+alist+网盘，不用 NAS 打造影音库教程”本次查询为 315,692 播放、15,928 收藏。

---

## 当前最值得直接复用的方向

1. **统一多网盘 Storage Adapter**：AList/OpenList，适合作为业务系统的文件接入层。
2. **网盘 -> WebDAV Gateway**：AList，适合系统文件管理器、Infuse、Kodi、RaiDrive、同步软件。
3. **WebDAV 透明加密**：alist-encrypt。
4. **网盘 -> STRM -> Emby/Jellyfin**：AutoFilm、openlist-strm、xstrm-suite。
5. **Emby -> AList/OpenList -> 302/直链**：go-emby2openlist、MediaWarp。
6. **AList -> TV/播放器**：alist-tvbox、openlist-tvbox-gateway、atv-player。
7. **多网盘自动同步**：TaoSync、OpenSync。
8. **Telegram -> AList 自动归档**：SaveAny-Bot。
9. **Android 自带 AList Server**：AListLiteAndroid、Alist-Magisk、magisk-alist。
10. **业务系统调用 AList API 自动建 Storage**：掘金 OneDrive Python 示例，展示 OAuth + AList Admin API 的完整自动化链路。
11. **知识库/RAG 数据源统一入口**：复用 AList Storage Drivers，向上接文件扫描、解析、Chunk、Embedding 和 Vector DB，是本批结果中最值得进一步二开的非影音方向之一。
