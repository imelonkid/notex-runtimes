# NoteX Runtimes

这是 NoteX 的公开运行时构建与发布仓库。主应用仓库可以保持私有；客户端匿名读取根目录的
`catalog.json`，再从 GitHub Release 或已配置的国内镜像下载运行时包。

## 三个包

| id | 内容 | 大小（压缩） |
|---|---|---|
| `python-sci` | python-build-standalone 3.12 加 numpy / pandas / matplotlib / scipy / pillow / jedi | 约 120 MB |
| `java-jdk` | Temurin JDK 17 用 jlink 裁到只含 JShell 与常用标准库模块 | 约 60 MB |
| `js-node` | 官方 Node 20 去掉文档和头文件，保留 npm | 约 25 MB |

包名：`<id>-<version>-<platform>-<arch>.tar.gz`，解压后顶层直接是 `bin/`。
`version` 是上游版本加构建号，比如 `3.12.14-1`；改了依赖包或裁剪方式就把构建号加一。

## 本机构建

```bash
./build-node.sh                       # 最快，先用它验证环境
JDK_HOME=/path/to/jdk ./build-java.sh # 有本机 JDK 就不用下载
PIP_INDEX_URL=https://pypi.tuna.tsinghua.edu.cn/simple ./build-python.sh
```

每个脚本都会做自检：Python 在隔离模式下导入全部包并用 Agg 画一张图，Java 用源码启动器跑一个用到 JShell 和 AWT 的小程序，Node 跑一段 vm。产物在 `dist/`。

国内构建时上游下载可以换镜像：`PBS_MIRROR`（python-build-standalone）、`JDK_URL`（清华的 Adoptium 镜像）、`NODE_MIRROR`（默认已是 npmmirror）。

## 生成清单

```bash
node make-catalog.mjs --dist dist --out dist/catalog.json \
  --base https://<bucket>.<endpoint>/v1 \
  --base https://gitee.com/<owner>/<repo>/releases/download/v1
```

`--base` 的顺序就是应用尝试下载的顺序，国内镜像放前面。只重打了一种架构时加 `--merge 上一份.json`，其它平台的条目会保留。

## 发布

```bash
export OSS_BUCKET=notex-runtimes OSS_ENDPOINT=oss-cn-hangzhou.aliyuncs.com \
       OSS_ACCESS_KEY_ID=… OSS_ACCESS_KEY_SECRET=… \
       GITEE_OWNER=… GITEE_REPO=notex-runtimes GITEE_TOKEN=…
TAG=v1 ./publish.sh
```

### 阿里云 OSS 要配的

- 桶设为公共读，或者前面挂 CDN。
- **CORS 规则**必须加：来源 `*`，方法 `GET, HEAD`，允许头 `*`。应用是从网页里 `fetch` 清单的，没有这条浏览器会拦。包本身走 curl 不受影响。
- 建议用 `v1/` 这样的前缀，将来清单格式升级时换 `v2/`，老版本应用不受影响。

### 码云要配的

- 建一个公开仓库，`catalog.json` 提交到仓库根目录，Release 附件放包。
- 私人令牌需要 `projects` 权限。码云单个附件上限 100 MB（企业版更大），`python-sci` 超过的话只能放 OSS，清单里码云那条地址会 404，应用会自动跳到下一个。
- 码云 Release 附件同名会冲突，`publish.sh` 会先删旧的再传。

### GitHub Actions

仓库使用 `.github/workflows/build-runtimes.yml`。手动触发后，两台 macOS runner 各出一种架构，汇总后生成清单；勾上 `publish` 才会发布 GitHub Release 并把正式清单写回仓库。要同步 OSS 或码云时，在仓库 Secrets 里填写上面的对应变量；不填则只使用 GitHub Release。

## 应用侧怎么接

应用默认从 `https://raw.githubusercontent.com/imelonkid/notex-runtimes/master/catalog.json` 读取清单；工作流正式发布后会自动写回该文件。
用户也可以在 `~/.notex/config.json` 里写 `runtimeCatalog` 换源。

安装后的目录是 `~/.notex/runtimes/<id>/<version>/`，应用把它当普通候选运行时对待，并在启动时隔离本机环境。
