# bsapplist

BIMSee 主程序的更新清单与发布包仓库。

BIMSee 的 `帮助 → 检查更新` 和启动后 24 小时一次的静默检查都读取这里的清单；
插件列表在另一个仓库 `bspluginlist`。

## 目录结构

```
applist_win.json             Windows 主程序更新清单
packages/win64/              发布包留档（BIMSeePro_<版本>.zip，不参与下载）
```

实际下载地址是 Gitee 发行版附件（见下方 `download` 字段说明）。

`applist_linux.json` / `applist_macos.json` 为预留文件名，对应平台有发布包后再添加。

## 清单格式

```json
{
  "schemaVersion": 1,
  "updated": "2026-10-29",
  "latest": {
    "version": "3.1.0",
    "releaseNotes": "本次更新的说明，显示在更新提示里",
    "download": "packages/win64/BIMSeePro_3.1.0.zip",
    "sha256": "<ZIP 的 SHA-256>",
    "size": 16463452
  }
}
```

- `version`：三段式 `X.Y.Z`，必须和发布包里
  `apps/bimsee/BimseeVersion.h` 的 `BIMSEE_PRODUCT_VERSION_STR` 一致；
  主程序按它判断是否有新版本。
- `download`：**Gitee 发行版附件的绝对地址**，例如
  `https://gitee.com/bimsee/bsapplist/releases/download/v3.1.0/BIMSeePro_3.1.0.zip`。
  不能用 `packages/...` 相对路径：Gitee 的 raw 接口对大于 1 MB 的文件要求登录，
  匿名下载会返回 403（`large file require login for access`），而发行版附件可以匿名下载。
  GitHub 与 Gitee 使用**同一份清单**，海外用户也从 Gitee 下载（包只有十几 MB）。
- `sha256` / `size`：下载后校验，不匹配则拒绝安装（`size` 需为整数）。
- `latest` 是必填对象；`schemaVersion`、`updated` 目前仅作标识，主程序不解析。

## 发布新版本

1. 在 BIMSee 源码里改 `apps/bimsee/BimseeVersion.h` 的版本号，然后打包：

   ```bash
   cmake --build build/win-x64 --config Release --target package-bimsee
   # 输出 dist/BIMSeePro_<版本>.zip
   ```

   包里只有主程序 + `ifcloader`；其他插件由 `bspluginlist` 单独安装。

2. 在 Gitee 上建发行版（标签 `v<版本>`，如 `v3.1.0`），把 ZIP 作为**发行版附件**上传。
   附件上传后形如：
   `https://gitee.com/bimsee/bsapplist/releases/download/v3.1.0/BIMSeePro_3.1.0.zip`

   也可以走 API（需要带 `projects` 权限的私人令牌）：

   ```bash
   curl -X POST "https://gitee.com/api/v5/repos/bimsee/bsapplist/releases?access_token=<令牌>" \
        -H "Content-Type: application/json" \
        -d '{"tag_name":"v3.1.0","name":"BIMSeePro 3.1.0","target_commitish":"main"}'
   curl -X POST "https://gitee.com/api/v5/repos/bimsee/bsapplist/releases/<release_id>/attach_files?access_token=<令牌>" \
        -F "file=@BIMSeePro_<版本>.zip"
   ```

3. 取 SHA-256 和字节数，更新 `applist_win.json` 的 `latest`
   （`version` / `releaseNotes` / `download` / `sha256` / `size`），同时改 `updated` 日期：

   ```bash
   sha256sum BIMSeePro_<版本>.zip
   stat -c%s BIMSeePro_<版本>.zip
   ```

   可另存一份到 `packages/win64/` 随仓库留档（**不参与下载**）。

4. 提交并推送到**两个**远程，保持镜像一致（中国大陆用户走 Gitee）：

   ```bash
   git add -A
   git commit -m "发布 BIMSeePro <版本>"
   git push origin main        # github.com/bimsee/bsapplist
   git push gitee main         # gitee.com/bimsee/bsapplist
   ```

   主程序从 `raw.githubusercontent.com/bimsee/bsapplist/main/applist_win.json`
   或 `gitee.com/bimsee/bsapplist/raw/main/applist_win.json` 拉取清单，
   Gitee 的 raw 有缓存，推送后可能需要几分钟才生效。

## 更新时主程序会做什么

确认更新后：下载 ZIP → 校验 `size`/`sha256` → 生成临时 PowerShell updater →
关闭 BIMSee → updater 等待进程退出后解压覆盖安装目录并重启。

updater 会把包内文件覆盖到安装目录（先删旧 `plugins/ifcloader/`）。
由于包本身只含主程序、`Fonts/`、`viewer.dat` 和 `plugins/ifcloader/`，
**`plugins/` 下其他已安装插件和 `.bimsee-state.ini` 不受影响**。
