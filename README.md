# OpenList + FileBrowser Quantum + Meilisearch

本项目使用 OpenList 负责文件列表和存储管理，使用 FileBrowser Quantum 负责 /local 文件的登录、上传、下载和流式 ZIP 打包，使用 Meilisearch 为 OpenList 提供搜索服务。

脚本文件 main.txt 只接管 OpenList 中映射到 /local 和 /nas 的下载、打包下载及上传入口，其他路径继续使用 OpenList 原生逻辑。

## 当前部署结构

| 服务 | 容器端口 | 宿主机端口 | 用途 |
| --- | ---: | ---: | --- |
| OpenList | 5244 | 2000 | 文件列表、认证入口、管理后台 |
| FileBrowser Quantum | 80 | 2100 | 文件上传、单项下载和流式 ZIP 下载 |
| Meilisearch | 7700 | 2700 | OpenList 搜索服务 |

当前三个服务位于同一个 docker-compose.yml 中，FileBrowser 不需要单独的 Compose 文件。

持久化目录对应关系：

| 宿主机目录 | 容器目录 | 用途 |
| --- | --- | --- |
| ./openlist-data | /opt/openlist/data | OpenList 配置和数据库 |
| ./filebrowser-data | /home/filebrowser/data | FileBrowser 配置、数据库和缓存 |
| ./meilisearch/data | /meili_data | Meilisearch 索引数据 |
| ./meilisearch/dumps | /meili_dumps | Meilisearch dumps |
| ./meilisearch/snapshots | /meili_snapshots | Meilisearch 快照 |

## Compose 与脚本的对应关系

### 宿主机文件目录

当前 Compose 使用同一个宿主机目录：

~~~yaml
openlist:
  volumes:
    - "/home/mickey:/local"

filebrowser:
  volumes:
    - "/home/mickey:/local:rw"
~~~

这两个挂载必须指向同一个宿主机目录，否则 OpenList 显示的文件和 FileBrowser 实际下载的文件不一致。

脚本不填写宿主机绝对路径，而是根据 OpenList 页面路径转换 FileBrowser 源内路径：

| OpenList 页面路径 | FileBrowser 源内路径 | 宿主机实际目录 |
| --- | --- | --- |
| /local | / | /home/mickey |
| /nas | /nas | /home/mickey/nas |

脚本对应配置：

~~~js
pathMappings: [
    {
        openlistPrefix: "/local",
        filebrowserPrefix: "/"
    },
    {
        openlistPrefix: "/nas",
        filebrowserPrefix: "/nas"
    }
]
~~~

filebrowserPrefix 是 FileBrowser 源内部的路径，不是宿主机路径，也不是 Docker 挂载路径。

### FileBrowser 源名称

当前 filebrowser-data/config.yaml 定义了：

~~~yaml
sources:
  - path: "/local"
    name: "local"
~~~

脚本必须使用相同的源名称：

~~~js
filebrowserSource: "local"
~~~

如果把 FileBrowser 源名称改成 files，必须同步改为 filebrowserSource: "files"，否则脚本生成的下载和上传 URL 会指向不存在的源。

### FileBrowser 配置和数据库

Compose 中的环境变量与宿主机文件对应如下：

~~~yaml
environment:
  FILEBROWSER_CONFIG: /home/filebrowser/data/config.yaml
  FILEBROWSER_DATABASE: /home/filebrowser/data/database.db
volumes:
  - ./filebrowser-data:/home/filebrowser/data
~~~

因此宿主机上的 filebrowser-data/config.yaml 会被容器读取为配置文件，数据库则保存在 filebrowser-data/database.db。修改 FileBrowser 源名称、认证方式或上传参数时，应修改这个配置文件；这些设置不需要写入 main.txt。

### FileBrowser 端口

当前 Compose 将宿主机 2100 映射到容器 80：

~~~yaml
ports:
  - "2100:80"
~~~

脚本对应：

~~~js
filebrowserPort: "2100"
~~~

这个端口只用于没有命中域名映射时的回退地址。例如从 http://10.2.1.2:2000 访问 OpenList 时，脚本会回退到 http://10.2.1.2:2100。

如果把 Compose 改成：

~~~yaml
ports:
  - "2300:80"
~~~

则脚本要同步改为：

~~~js
filebrowserPort: "2300"
~~~

容器内部端口仍为 80 时，脚本只需要修改宿主机端口；如果同时修改了容器内部端口，还要同步修改 Compose 的右侧端口和 FileBrowser server.port。

### Caddy 域名和非标准端口

通过域名访问 OpenList 时，脚本优先查找 filebrowserBaseByOpenListOrigin：

~~~js
filebrowserBaseByOpenListOrigin: {
    "https://openlist.example.com":
        "https://download.example.com",
    "https://openlist.example.com:8443":
        "https://download.example.com:9443"
}
~~~

左侧必须与浏览器地址栏中的 OpenList Origin 完全一致，包括协议、域名和非标准端口。右侧必须是浏览器可以访问的 FileBrowser 地址。

例如 Caddy 配置为：

~~~caddyfile
https://openlist.example.com:8443 {
    reverse_proxy 127.0.0.1:2000
}

https://download.example.com:9443 {
    reverse_proxy 127.0.0.1:2100
}
~~~

脚本就必须配置：

~~~js
filebrowserBaseByOpenListOrigin: {
    "https://openlist.example.com:8443":
        "https://download.example.com:9443"
}
~~~

如果使用同一个域名的路径反代，也要把完整的 FileBrowser 基础地址写入右侧，例如 https://example.com/filebrowser，并确保 Caddy 将该路径转发到 FileBrowser。

没有匹配的 Origin 时，脚本会使用 当前协议://当前主机:filebrowserPort。因此 HTTPS 页面不能回退到 HTTP 的 2100 端口，否则浏览器可能阻止混合内容请求。

### OpenList 虚拟路径

如果只修改宿主机目录，例如：

~~~yaml
- "/srv/files:/local"
~~~

只要 OpenList 和 FileBrowser 都改成挂载同一个 /srv/files，脚本的 pathMappings 不需要改变。

如果把 OpenList 页面中的虚拟路径 /local 改成 /share，则必须同步修改：

~~~js
pathMappings: [
    {
        openlistPrefix: "/share",
        filebrowserPrefix: "/"
    }
]
~~~

如果把 /nas 改成其他页面路径，也必须修改对应的 openlistPrefix。脚本只会接管列在 pathMappings 中的路径，其他路径保持 OpenList 原生行为。

当前脚本使用一个 FileBrowser 源 local。如果创建多个 FileBrowser 源，脚本目前不能仅靠 pathMappings 为不同路径选择不同源；需要扩展脚本的 URL 构造逻辑。

## 脚本用户配置区

打开 main.txt，只修改文件最前面的 USER_CONFIG。主要配置如下：

~~~js
const USER_CONFIG = {
    pathMappings: [...],
    filebrowserPort: "2100",
    filebrowserSource: "local",
    filebrowserBaseByOpenListOrigin: {...},
    downloadWindowTarget: "_blank",
    uploadWindowTarget: "_blank",
    noticeVisibleMs: 1800,
    noticeFadeMs: 300,
    dragNoticeCooldownMs: 1800,
    messages: {...},
    debug: true
};
~~~

一般只需要修改以下四项：

1. Compose 宿主机端口变化时，修改 filebrowserPort。
2. FileBrowser sources[].name 变化时，修改 filebrowserSource。
3. OpenList 虚拟目录变化时，修改 pathMappings。
4. 使用 Caddy 或非标准端口时，修改 filebrowserBaseByOpenListOrigin。

提示文字、提示时长和是否保留 Console 日志也可以在该区域修改。

## 下载与上传行为

- /local、/nas 中的单个文件和文件夹由 FileBrowser 处理。
- 多选文件和文件夹通过 FileBrowser 下载 API 生成只包含选中项目的 ZIP。
- 点击 OpenList 上传会在新标签页打开对应的 FileBrowser 目录。
- 在受管路径拖拽文件时，OpenList 原生上传框保持显示，但脚本会阻止文件交给 OpenList 上传，并打开 FileBrowser 页面。
- 不在 pathMappings 中的路径继续使用 OpenList 原生下载和上传。

FileBrowser 下载 URL 的核心形式为：

~~~text
/api/resources/download?source=local&file=...&algo=zip
~~~

上传页面的核心形式为：

~~~text
/files/local/目标路径
~~~

## 启动和检查

在 docker-compose.yml 所在目录执行：

~~~bash
docker compose config
docker compose up -d
docker compose ps
~~~

访问地址：

~~~text
OpenList:      http://服务器地址:2000
FileBrowser:   http://服务器地址:2100
Meilisearch:   http://服务器地址:2700
~~~

查看日志：

~~~bash
docker compose logs -f openlist
docker compose logs -f filebrowser
docker compose logs -f meilisearch
~~~

只重启单个服务：

~~~bash
docker compose up -d filebrowser
docker compose up -d openlist
docker compose up -d meilisearch
~~~

## 浏览器调试命令

脚本加载后，在 OpenList Chrome 控制台执行：

~~~js
OpenListFileBrowserDebug.state()
OpenListFileBrowserDebug.state().version
OpenListFileBrowserDebug.scan()
await OpenListFileBrowserDebug.probe()
OpenListFileBrowserDebug.url("/local/temp")
OpenListFileBrowserDebug.url("/nas/电影")
OpenListFileBrowserDebug.uploadUrl("/local/temp")
~~~

更新脚本后使用 Ctrl+F5 强制刷新，避免浏览器继续使用旧脚本。

## 安全和持久化注意事项

- FileBrowser 的配置、数据库和缓存保存在 ./filebrowser-data。
- FileBrowser 当前启用了密码认证，匿名用户不能上传或下载。
- adminUsername 和 adminPassword 只适合首次初始化或内网测试，不要把真实密码提交到公开仓库。
- Meilisearch 的 MEILI_MASTER_KEY 不应留空；生产环境应使用随机密钥，并让 OpenList 使用相同密钥或单独的最小权限 API Key。
- 不建议直接将宿主机 2000、2100、2700 端口暴露到公网，公网访问应通过 Caddy HTTPS 和访问控制。
- ./openlist-data、./filebrowser-data、./meilisearch、.env 和数据库文件应加入 .gitignore，只提交不含凭据的示例配置。

## License

本项目采用 MIT License。
