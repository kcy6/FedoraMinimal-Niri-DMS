
# iso最小化系统安装

# 基础网络连接

# 代理

# 软路由
# 安装启动ssh服务
```bash
sudo dnf install -y openssh-server
sudo systemctl enable --now sshd
```

# 与桌面相关的应用启动流程
以我当前情况为例,主机为Fedora+dms greeter+niri+dms

当前的图形登录由 **greetd + DMS Greeter + niri + PAM** 配合完成

## 系统核心组件启动
```
固件 → 引导程序 → Linux 内核 → systemd（PID 1）
```

| 系统组件             | 为桌面提供什么        |
| ---------------- | -------------- |
| `systemd-udevd`  | 发现和管理设备        |
| `systemd-logind` | 管理登录会话、座席和设备访问 |
| `系统 D-Bus`       | 让程序与系统服务通信     |
| `NetworkManager` | 网络连接           |
| `bluetoothd`     | 蓝牙             |
| `greetd`         | 登录管理           |
这些服务大多并行启动，并不是严格一条线依次执行。语言方面，系统配置是：
```ini
# /etc/locale.conf
LANG="en_US.UTF-8"
```

此外系统启动通常由服务依赖或 socket 激活机制触发，**不是 PID 1 按固定顺序写死地逐个调用这些程序**。systemd 根据 unit 配置判断什么时候需要启动它们。

另外，你机器上还有一套**用户 D-Bus**，由用户级 systemd 管理：

```
系统 systemd（PID 1）
└─ systemd --user（用户 y，PID 1407）
   └─ dbus-broker-launch（PID 1536）
      └─ dbus-broker（PID 1537）
```

两套总线用途不同：

- **系统 D-Bus**：供应用与 NetworkManager、蓝牙、logind 等系统服务通信。
- **用户 D-Bus**：供当前用户的桌面应用、通知服务、portal 等通信。

“系统 D-Bus”指的是系统范围的通信总线，**不表示总线进程必须以 root 身份运行**；你这台机器上它使用专门的 `dbus` 用户。
## 登录页面启动
greetd 启动一个专门用于登录的小型图形会话
本机 `/etc/greetd/config.toml` 配置的命令是：
```toml
[terminal]
vt = 1

[default_session]
command = "/usr/bin/dms-greeter --command niri --cache-dir /var/cache/dms-greeter -C /etc/greetd/niri/config.kdl"
user = "greeter"
```
这表示：

- 登录界面运行在 **第 1 个虚拟终端**。
- greetd 以专用账户 **`greeter`** 启动 `dms-greeter`。
- 登录界面的合成器使用 **niri**，读取 `/etc/greetd/niri/config.kdl`。
- 该配置包含 `DMS_RUN_GREETER="1"`，用于登录界面模式。
运行身份是 `greeter`。

这里各自的职责是：

- **greetd**：管理登录过程，通过 PAM 完成身份认证、建立用户会话。
- **dms-greeter**：提供你看到的图形登录界面。
- **登录界面用的 niri**：给登录界面提供 Wayland 显示环境。
**登录时的 niri 和登录后的 niri，是不同用途、不同用户身份的实例。** 前者使用 `/etc/greetd/niri/` 中的配置，后者使用你的用户配置。
因此，你用户目录里的 `LANGUAGE` 设置，也不能直接解释登录界面的语言——登录界面属于 `greeter` 用户的环境。
输入密码验证成功后进入启动流程(通过一个脚本调动)
启动流程:
```
开机
  ↓
系统 systemd（PID 1）启动
  ↓
启动 greetd
  ↓
显示登录界面
  ↓
你输入账号密码，PAM 验证通过
  ↓
PAM 建立用户会话
  ↓
pam_systemd 与 systemd-logind 协作注册会话
  ↓
系统 systemd 启动 user@1000.service
  ↓
你的 systemd --user 启动
  ↓
读取用户环境配置，包括 environment.d 中的 LANGUAGE
  ↓
greetd 启动你的会话入口 niri-session
  ↓
niri-session 请求 systemd --user 启动 niri.service
  ↓
进入桌面
```


`niri-session` 是 niri 提供的“桌面会话启动脚本”(本质为一个脚本)。安装登录界面后，它仍然承担原来的工作，只是从你手动执行，变成登录管理器替你执行。
```
TTY 登录
   ↓
你的 shell
   ↓ 手动输入 niri-session
niri-session
   ↓ 请求 systemd 用户服务启动
niri --session
   ↓
进入桌面
```
因此流程变成：

```
greetd + 图形登录界面
   ↓ 完成认证，选择 Niri 会话
niri-session
   ↓ 请求 systemd 用户服务启动
niri --session
   ↓
进入桌面
```

**登录界面解决“谁登录、启动哪个会话”；`niri-session` 解决“怎样把 niri 会话正确启动并收尾”。**

| 阶段               | `niri-session` 的工作                              |                         |
| ---------------- | ----------------------------------------------- | ----------------------- |
| 准备环境             | 通过登录 shell 执行会话启动流程，使登录 shell 配置有机会生效           |                         |
| 防止重复             | 检查 `niri.service` 是否已经运行                        |                         |
| 同步环境             | 将登录环境导入 systemd 用户服务管理器及 D-Bus 激活环境             |                         |
| 启动桌面             | 执行 `systemctl --user --wait start niri.service` |                         |
| 等待退出             | 保持等待，直到 niri 会话结束                               |                         |
| 清理会话             | 启动 `niri-shutdown.target`，清除显示连接等会话环境变量         |                         |
| 名称               | 是什么                                             | 职责                      |
| `niri-session`   | 启动脚本                                            | 准备、启动、等待、清理整个会话         |
| `niri.service`   | systemd 服务定义                                    | 告诉 systemd 如何管理 niri 进程 |
| `niri --session` | 真正运行的合成器进程                                      | 提供窗口、显示、输入管理，并启用会话集成    |


用户服务管理器的环境生成器读取：
```ini
# ~/.config/environment.d/locale.conf
LANGUAGE=zh_CN:en_US
```
于是，准备传给用户服务的环境包含：
```
LANG=en_US.UTF-8
LANGUAGE=zh_CN:en_US
```

## 进入桌面后
```
systemd --user
   ├─ niri
   └─ dms
       └─ qs
```
DMS 和 niri 是协作关系，DMS 由 systemd 启动；它再把自己的界面交给 niri 显示。niri 本身不提供完整桌面环境，需要以下配套组件共同工作。

| 本机组件                              | 主要职责                  | 本机启动方式或关系                            |
| --------------------------------- | --------------------- | ------------------------------------ |
| DMS / `qs`                        | 桌面面板、通知、控制界面等         | `dms.service`                        |
| `fcitx5`                          | 输入法                   | niri 配置的 `spawn-at-startup "fcitx5"` |
| `PipeWire`                        | 音视频流处理                | 用户服务及 socket 激活机制                    |
| `WirePlumber`                     | 音频设备、路由和策略管理          | 与 PipeWire 服务关联                      |
| `pipewire-pulse`                  | 兼容使用 PulseAudio 接口的应用 | 用户服务及 socket                         |
| `xdg-desktop-portal` 及后端          | 文件选择、屏幕共享等桌面接口        | 用户服务，支持按需激活                          |
| `dbus-broker`                     | 用户会话内的消息通信            | 用户 D-Bus 服务                          |
| `gnome-keyring`                   | 保存密码和秘密数据             | 登录集成或服务激活                            |
| GVfs                              | 桌面文件访问、卷和网络位置等支持      | 用户服务和按需启动                            |
| `xwayland-satellite` → `Xwayland` | 让 X11 应用接入 niri 桌面    | 已确认是 niri 下的进程链                      |
这些组件不一定全在同一时刻启动。有些随桌面启动，有些在应用第一次请求功能时才启动。

```
固件 → 引导 → Linux 内核 → 系统 systemd
                              │
                              ├─ 系统服务：网络、蓝牙、logind、系统 D-Bus
                              │
                              ├─ greetd
                              │    ├─ 登录界面：dms-greeter + 登录用 niri
                              │    └─ 登录成功 → niri-session
                              │                        │ 请求启动
                              └─ systemd --user ←──────┘
                                   │
                  environment.d ──→│ LANG / LANGUAGE
                                   │
                                   ├─ niri：显示、输入、窗口管理
                                   │    ├─ fcitx5
                                   │    ├─ Xwayland 支持
                                   │    └─ 快捷键启动终端、启动器
                                   │
                                   ├─ DMS → qs：桌面界面
                                   ├─ PipeWire / WirePlumber：音频
                                   ├─ portals：桌面接口
                                   └─ D-Bus、密钥环、GVfs 等
```

 - `systemd PID 1` 管理整机服务如网络、登录管理器等
 - `systemd --user` 管理用户服务如niri、DMS、音频等
 - `greetd` 登录管理器后台：启动登录界面、协调认证、启动用户会话
 - `DMS Greeter` 看到的登录界面，负责与用户交互





# 中文环境
希望配置的理想环境:桌面和应用显示为中文,整个系统底层为英文

**程序如何选择翻译**:对使用 GNU gettext 的程序，选择消息翻译时通常按以下优先级(LANG最底层,向左逐层覆盖)：
```
LANGUAGE → LC_ALL → LC_MESSAGES → LANG
```
- LANG为所有 locale 类别提供默认值
- LC_MESSAGES单独指定程序消息使用的 locale
- LC_ALL统一覆盖所有 locale 类别
- LANGUA GEGNU gettext 等支持它的翻译机制使用的语言偏好列表



`locale`:当前终端进程的语言环境。
`localectl`:系统默认语言，可能被用户或桌面设置覆盖


## niri
```bash
sudo dnf install -y glibc-langpack-en glibc-langpack-zh langpacks-zh_CN

mkdir /home/y/.config/environment.d
vim /home/y/.config/environment.d/locale.conf
#添加LANGUAGE
LANGUAGE=zh_CN:en_US
```
实现思路:
```
开机
  ↓
系统 systemd（PID 1）启动
  ↓
启动 greetd
  ↓
显示登录界面
  ↓
你输入账号密码，PAM 验证通过
  ↓
PAM 建立用户会话
  ↓
pam_systemd 与 systemd-logind 协作注册会话
  ↓
系统 systemd 启动 user@1000.service
  ↓
你的 systemd --user 启动
  ↓
读取用户环境配置，包括 environment.d 中的 LANGUAGE
  ↓
greetd 启动你的会话入口 niri-session
  ↓
niri-session 请求 systemd --user 启动 niri.service
  ↓
进入桌面
  ↓
niri,dms均继承到LANGUAGE=zh_CN:en_US,后续多为通过其启动的应用同样继承
```

## KDE plasma
```bash
sudo dnf install glibc-langpack-en glibc-langpack-zh langpacks-zh_CN
locale -a | grep -E 'en_US|zh_CN'
localectl status #保留系统英文默认值,如果不是LANG=en_US.UTF-8,执行以下设置
sudo localectl set-locale LANG=en_US.UTF-8

#在 Plasma 中分别设置语言和格式
vim ~/.config/plasma-localerc
#修改LANGUAGE值
[Translations]
LANGUAGE=zh_CN:en_US
```
plasma通过会话启动程序管理,在桌面启动早期准备语言环境,这种管理方式后续所有和plasma桌面相关的都可以很好得继承环境
会话启动程序，就是你登录后，负责把整个桌面组织起来的“开场负责人”。可以理解为：从你登录桌面，到你注销桌面，这一段使用过程，以及为你运行的一组程序。

| 对比       | niri＋`environment.d`    | Plasma＋`plasma-localerc`       |
| -------- | ----------------------- | ------------------------------ |
| 谁读取      | systemd 的用户环境生成器        | Plasma 会话启动程序                  |
| 主要用途     | 设置通用环境变量，语言只是其中一种       | 设置 Plasma 的语言和区域格式             |
| 环境如何传播   | 用户服务管理器 → niri、DMS → 应用 | Plasma 准备环境 → 同步服务启动环境 → 桌面和应用 |
| 是否依赖特定桌面 | 不专属于 niri               | 属于 Plasma 配置体系                 |
| 作用范围     | 获得该环境的用户服务及后代进程         | Plasma 会话及获得其启动环境的程序           |
## GNOME
```bash
mkdir /home/y/.config/environment.d
vim /home/y/.config/environment.d/locale.conf
#添加LANGUAGE
LANGUAGE=zh_CN:en_US

```
和KDE相似
# 中文输入法
## Fcitx5+rime+雾凇输入方案+万象输入模型安装
```bash
#安装输入框架和输入引擎
sudo dnf install git curl fcitx5 fcitx5-rime fcitx5-configtool google-noto-sans-cjk-fonts google-noto-color-emoji-fonts

#让系统重新扫描字体并更新缓存
fc-cache -f

#安装输入方案
git clone https://github.com/rime/plum.git ~/plum
cd ~/plum
env rime_frontend=fcitx5-rime bash rime-install iDvel/rime-ice
env rime_frontend=fcitx5-rime bash rime-install iDvel/rime-ice:others/recipes/grammar:schema=rime_ice

#日常更新雾凇和词库
cd ~/plum
env rime_frontend=fcitx5-rime bash rime-install iDvel/rime-ice
#需要单独更新万象模型时
cd ~/plum
env rime_frontend=fcitx5-rime bash rime-install iDvel/rime-ice:others/recipes/grammar:schema=rime_ice
#更新 Plum 工具本身
git -C ~/plum pull --ff-only

#同时额外添加Fcitx5启动到niri配置内使其自启动
```

| 名称           | 是什么             | 负责什么                                    |
| ------------ | --------------- | --------------------------------------- |
| **Fcitx5**   | 输入法框架           | 连接应用、切换输入法、显示候选框；可以搭载 Rime 等输入法         |
| **Rime／中州韵** | 输入法引擎           | 按方案处理输入编码、查询词典、组织候选，并记录个人用词             |
| **雾凇拼音输入方案** | Rime 的方案和配套配置   | 提供全拼／双拼规则、词库、Lua 扩展等；你选中的是全拼 `rime_ice` |
| **万象语法模型**   | Rime 可使用的语言模型数据 | 提供词语搭配的统计信息，辅助候选排序；不是独立输入法              |
| **Plum／东风破** | 安装和更新输入方案的工具    | 下载方案、词库和模型，应用配置补丁；日常打字不需要运行它            |
## 配置
- 设置默认为雾凇输入方案
```bash
mkdir -p ~/.local/share/fcitx5/rime
vim ~/.local/share/fcitx5/rime/default.custom.yaml #添加如下内容

patch:
  # 这里的 rime_ice_suggestion 为雾凇方案的默认预设
  __include: rime_ice_suggestion:/
```
- 配置F4提供的快捷切换方案
`ls /usr/share/rime-data/*.schema.yaml` 命令可以看到当前所有可用的输入方案，去掉文件名的 `.schema.yaml` 就是 `schema` 名，写进 `default.custom.yaml` 后可以在 F4 菜单里切换不同的输入方案
```
patch:
  schema_list:
    - schema: luna_pinyin_simp
    - schema: rime_ice
    - schema: wanxiang
    - schema: double_pinyin_flypy
    - schema: wubi86
    - schema: bopomofo
```
>`luna_pinyin_simp` 是 RIME 自带的明月拼音；`wanxiang` 是万象拼音；`double_pinyin_flypy` 是小鹤双拼（由 `rime-ice-git` 提供）；`bopomofo` 是注音输入法。

- 添加万象输入法联想模型
```
vim ~/.local/share/fcitx5/rime/rime_ice.custom.yaml

patch:
  "grammar/language": wanxiang-lts-zh-hans
```
- 默认使用英文标点
雾凇拼音默认输出中文标点（，。？！）。如果你日常聊天、写代码更习惯英文标点，可以让它默认就处于英文标点状态，不用每次手动切换：

```shell
vim ~/.local/share/fcitx5/rime/rime_ice.custom.yaml

patch:
  # switches 是方案里的开关列表，@1 即第二项「中英标点」开关
  # reset: 1 表示每次启动都重置到第二个状态，也就是英文标点
  "switches/@1/reset": 1
```

> 如果前面配置语法模型时已经创建过这个文件，把 `"switches/@1/reset": 1` 加进已有的 `patch:` 里即可，不要出现两个 `patch:`。
重启输入法生效。之后想临时切回中文标点，按 `Ctrl+Shift+3` 即可。
- 配置自己的快捷短语
```bash
cp -a ~/.local/share/fcitx5/rime/custom_phrase.txt \
  ~/.local/share/fcitx5/rime/my_custom_phrase.txt

vim /home/y/.local/share/fcitx5/rime/rime_ice.custom.yaml

# 在现有patch下添加
  "custom_phrase/user_dict": my_custom_phrase
```
- 添加特定词库
```bash
cd ~/.local/share/fcitx5/rime

curl -fL --retry 3 \
  https://raw.githubusercontent.com/gaboolic/rime-frost/master/cn_dicts_cell/computer.dict.yaml \
  -o computer.dict.yaml

curl -fL --retry 3 \
  https://raw.githubusercontent.com/scomper/Rime/master/dicts/IT.dict.yaml \
  -o IT.dict.yaml
  
head -n 16 computer.dict.yaml
head -n 16 IT.dict.yaml

cat > ~/.local/share/fcitx5/rime/rime_ice_work.dict.yaml <<'EOF'
# Rime dictionary
# encoding: utf-8
---
name: rime_ice_work
version: "2026-09-24"
sort: by_weight
import_tables:
  - cn_dicts/8105
  - cn_dicts/base
  - cn_dicts/ext
  - cn_dicts/tencent
  - cn_dicts/others
  - rime_ice
  - computer
  - IT
...
EOF

vim ~/.local/share/fcitx5/rime/rime_ice.custom.yaml

#在patch下添加
  "translator/dictionary": rime_ice_work
  "translator/user_dict": rime_ice
  
#更新和维护
#将rime_ice.custom.yaml中的  "translator/dictionary": rime_ice_work改为rime_ice
#要添加或减少词库时主要维护 rime_ice_work.dict.yaml 中的 import_tables 列表
#查看其名字name: electronics写入import_tables如  - electronics
cat /home/y/.local/share/fcitx5/rime/electronics.dict.yaml
```
- 兼容性
```bash
# 创建用户家里的环境变量配置目录，编辑输入法环境变量
mkdir -p ~/.config/environment.d/
vim ~/.config/environment.d/fcitx5.conf

XMODIFIERS=@im=fcitx
QT_IM_MODULES="wayland;fcitx"
QT_IM_MODULE=fcitx
SDL_IM_MODULE=fcitx
```
- 预编辑文本内光标不随字符移动
在 Fcitx5 配置 → 附加组件 → Rime → 配置 中：
取消勾选“将嵌入式预编辑光标固定在预编辑开头”
- 取消候选框每个字的注释
```bash
vim /home/y/.local/share/fcitx5/rime/rime_ice.custom.yaml

patch:
  "translator/spelling_hints": 0
```

## 美化fcitx5
这里改变的是输入法候选框样式
```bash
mkdir -p ~/.local/share/fcitx5-theme-sources
#catppuccin是一个有统一配色风格的主题家族Catppuccin 给不同软件提供适配版本，这里安装的是Fcitx5主题集
git clone https://github.com/catppuccin/fcitx5.git \
  ~/.local/share/fcitx5-theme-sources/catppuccin
#fcitx5-themes-candlelight,含类似macos的主题样式
git clone git@github.com:thep0y/fcitx5-themes-candlelight.git \
~/.local/share/fcitx5-theme-sources/fcitx5-themes-candlelight

cd ~/.local/share/fcitx5-theme-sources/catppuccin
bash enable-rounded.sh
mkdir -p ~/.local/share/fcitx5/themes
cp -r src/. ~/.local/share/fcitx5/themes/

cd ~/.local/share/fcitx5-theme-sources/fcitx5-themes-candlelight
#将想要使用的主题复制到相应目录
cp -r <theme-name> ~/.local/share/fcitx5/themes/
```
后续更新示例:
```bash
cd ~/.local/share/fcitx5-theme-sources/catppuccin
git pull --ff-only
bash enable-rounded.sh
cp -r src/. ~/.local/share/fcitx5/themes/

cd ~/.local/share/fcitx5-theme-sources/fcitx5-themes-candlelight
git pull
cp -r macOS-dark ~/.local/share/fcitx5/themes/
vim  ~/.local/share/fcitx5/themes/macOS-dark/theme.conf
#修改以下两项为
[InputPanel/Background]
Image=panel-compat.png

[InputPanel/Highlight]
Image=highlight-compat.png
```
重启输入法
```
fcitx5 -r
```
# obsidian
[obsidian官网](https://obsidian.md/)
[obsidian官网文档](https://obsidian.md/help/)

|               | 位置                                             |
| ------------- | ---------------------------------------------- |
| 应用程序          | `~/Applications/Obsidian.AppImage`             |
| 菜单启动器快捷方式     | `~/.local/share/applications/obsidian.desktop` |
| 笔记库（示例，可自行选择） | `~/Documents/Obsidian`/                        |
 
`AppImage`方式安装,本节指令使用bash执行,其他shell可能不适配
```bash
#缺少依赖补上:
sudo dnf install fuse fuse-libs -y

url=$(curl -fsSL https://obsidian.md/download |
  grep -oE 'https://github\.com/obsidianmd/obsidian-releases/releases/download/[^/]+/Obsidian-[0-9.]+\.AppImage' |
  head -n 1)

printf '%s\n' "$url"

mkdir -p ~/Applications
curl -fL --retry 3 "$url" -o ~/Applications/Obsidian.AppImage.download &&
chmod +x ~/Applications/Obsidian.AppImage.download &&
mv -f ~/Applications/Obsidian.AppImage.download ~/Applications/Obsidian.AppImage

#启动
~/Applications/Obsidian.AppImage

```
## 添加到系统的应用菜单/fuzzel程序启动器
```bash
mkdir -p ~/.local/share/applications
cat > ~/.local/share/applications/obsidian.desktop <<EOF
[Desktop Entry]
Type=Application
Name=Obsidian
Comment=Markdown Knowledge base
Exec=sh -c 'exec "$HOME/Applications/Obsidian.AppImage" "$@"' sh %U
Icon=obsidian
Terminal=false
Categories=Office;
EOF
```
> [!info]- 快捷方式字段说明
>
> | 字段 | 作用 |
> | --- | --- |
> | `[Desktop Entry]` | 主配置段的固定名称，告诉读取者下面是应用入口信息 |
> | `Type=Application` | 表示这是可以运行的应用 |
> | `Name=Obsidian` | 菜单中显示的名称 |
> | `Comment=…` | 应用说明，可能用于提示文字或搜索，不是代码注释 |
> | `Exec=…` | 启动命令；这里执行指定的 AppImage |
> | `%U` | 多个 URL 的参数占位符；没有传入 URL 时会移除 |
> | `Icon=…` | 图标的绝对路径，也可以是图标主题中的名称 |
> | `Terminal=false` | 不要求在终端窗口中运行 |
> | `Categories=Office;` | 将应用归入办公类；分类列表使用分号分隔 |

## 解决obsidian图标未显示问题
```bash
mkdir -p ~/.local/share/icons
curl -fL https://obsidian.md/favicon.svg -o ~/.local/share/icons/obsidian.svg

sed -i "s|^Icon=.*|Icon=$HOME/.local/share/icons/obsidian.svg|" ~/.local/share/applications/obsidian.desktop
#这条命令会直接修改 Obsidian 快捷方式文件，把图标设置为你下载的 SVG 图片,将图片下载到~/.local/share/icons/。
#修改前Icon=obsidian，修改后Icon=/home/y/.local/share/icons/obsidian.svg
#fuzzel没有显示图标但dms应用管理器已经显示了图标,同时本机图标路径正确，疑似与该 SVG 的 CSS 样式和 Fuzzel的NanoSVG 渲染兼容性有关。可尝试将 CSS 类名对应的颜色直接写入图形属性。
sed -i 's/class="purple"/fill="#9974F8"/g' ~/.local/share/icons/obsidian.svg
#9974F8取自官网图标,官网以后更改图标结构，命令就不合适了
```

> [!tip]- 补充信息
> Fuzzel 和 DMS 分别读取什么？
> 两者都使用 XDG 标准的应用入口，默认来源相同,就fuzzel,dms菜单这种情况而言读取是相同的,但隐藏规则、排序、搜索和刷新方式可能不同，最终显示的列表不一定完全一样
> `~/.local/share/applications/`当前用户的应用入口,如`obsidian.desktop`
> `/usr/local/share/applications/`本机额外安装的应用入口
> `/usr/share/applications/`系统软件包安装的应用入口`Alacritty.desktop`,`org.mozilla.firefox.desktop`等
> - **Fuzzel**：收集这些目录中的 `.desktop` 文件，生成可搜索的应用列表。[Fuzzel 手册](https://man.archlinux.org/man/fuzzel.1.en)
> - **DMS 菜单**：通过 Quickshell 的 `DesktopEntries` 获取应用列表
> Fuzzel 通常是每次调用时启动，选完应用就退出，不是一直运行的后台服务。修改后及时刷新
> 如果卡死了执行`pkill -x fuzzel`,`fuzzel` 
> DMS 会响应应用列表变化，但图标内容或缓存未必立即刷新；先重新打开菜单，仍未更新可执行 dms restart。注销重登录作为最后的处理方式。

## 快捷键


## 页面美化


## 一键安装配置脚本bash install-obsidian.sh
[[Fedora Minimal+Niri+DMS文档对应脚本#obsidian.sh]]

# 命令的别名
### ll=ls -alh
`bash`
```bash
echo "alias ll='ls -alh'" >> ~/.bashrc
source ~/.bashrc
```
`zsh`
```
echo "alias ll='ls -alh'" >> ~/.zshrc
source ~/.zshrc
```
`fish`
```
alias --save ll='ls -alh'
```
# 部分软甲的安装
## vscode
**1. 导入微软签名密钥：**

```
sudo rpm --import https://packages.microsoft.com/keys/microsoft.asc
```

**2. 添加 VS Code 软件源：**

```
sudo tee /etc/yum.repos.d/vscode.repo > /dev/null <<'EOF'
[code]
name=Visual Studio Code
baseurl=https://packages.microsoft.com/yumrepos/vscode
enabled=1
gpgcheck=1
gpgkey=https://packages.microsoft.com/keys/microsoft.asc
EOF
```

**3. 安装并启动：**

```
sudo dnf install code
code
```
## telegram
```
# 安装，可能弹出管理员认证
flatpak install --system flathub org.telegram.desktop

# 启动
flatpak run org.telegram.desktop
```

安装后查看实际版本：

```
flatpak info --system org.telegram.desktop
```
后续更新：
```
flatpak update --system org.telegram.desktop
```


# 截图相关
## 功能需求
- 区域截图,显示器截图,窗口截图,长截图
- 默认只复制到剪贴板
- 贴图,贴图后编辑,贴图后翻译其内容
- OCR

## 相关工具

 `grim`:真正负责“抓取屏幕像素”
 `slurp`:让你用鼠标“框选区域”
 `Swappy`:编辑标注工具,更轻量，但功能和交互明显更基础。
 `Satty:`编辑标注工具
 `wl-clipboard` 是让命令行程序读写 Wayland 剪贴板的工具包。 它包含两个命令：wl-copy把数据放进剪贴板,wl-paste把剪贴板中的数据取出来
 `cliphist`:剪贴板管理工具
 `mark-shot`:一个功能综合齐全的截图工具,基本可以实现所有截图相关需求


## 截图方式比较

niri+dms默认存在三种截图方式(选择niri原生截图)

|对比项|niri 原生截图|DMS 调用 niri 并编辑|DMS 自带截图命令|
|---|---|---|---|
|典型命令|`niri msg action screenshot`|`dms ipc call niri screenshot`|`dms screenshot`|
|谁负责截图|niri|仍然是 niri|DMS 截图工具，具体捕获方式依环境而定|
|是否自动打开编辑器|否|正常保存截图后打开 Swappy 等编辑器|默认不打开编辑器|
|默认保存行为|由 `screenshot-path` 控制|DMS 显式指定文件路径|默认保存文件并复制到剪贴板|
|`screenshot-path null` 是否管用|管用|不能阻止 DMS 指定路径保存|不控制该命令的输出策略|
|只复制、不保存|设置 `null`，或截图界面按 `Ctrl + C`|按 `Ctrl + C`，但不会自动打开编辑器|使用 `--no-file`|
|适合什么需求|简单截图、只复制|截图后立即标注，接受保留原图|用命令控制截图类型、格式和输出|


 

## 基础截图配置
`如果希望所有 niri 截图默认都不保存`
~/.config/niri/config.kdl搜索screenshot,把配置文件中原有的screenshot-path "..."那一行替换为：screenshot-path null
`修改原快捷键`
```
// 区域截图
Mod+Shift+S { screenshot; }

// 当前显示器截图
Mod+Shift+A { screenshot-screen; }

// 当前窗口截图
Mod+Shift+W { screenshot-window; }
```
`配置打开剪贴板的快捷键。` 
Win + V 已用于切换窗口浮动。可以在 /home/y/.config/niri/config.kdl的 binds {}内加入：
```
Mod+Ctrl+V { spawn "dms" "ipc" "call" "clipboard" "toggle"; }
```

## mark-shot



# Firefox
## 插件安装
划词翻译,篡改猴,Dark Reader,Global Speed,uBlock Origin
 






# 终端美化
## 安装starship
Starship 是一款用 Rust 语言编写的跨平台、跨 Shell 的开源终端提示符（Prompt）定制工具。它可以让你的命令行界面变得非常美观、轻量且高效。
Starship 主题常使用 Nerd Fonts 提供的图标字形（glyph）。为了避免图标显示为方框或乱码，建议在终端模拟器中使用 Nerd Font。
```bash
mkdir -p ~/.local/share/fonts/JetBrainsMonoNerd
curl -fLo /tmp/JetBrainsMono.zip https://github.com/ryanoasis/nerd-fonts/releases/latest/download/JetBrainsMono.zip
unzip /tmp/JetBrainsMono.zip -d ~/.local/share/fonts/JetBrainsMonoNerd
fc-cache -fv

fc-list | grep -i "JetBrainsMono.*Nerd" | head
fc-match "JetBrainsMono Nerd Font"
正常应该看到类似：JetBrainsMonoNerdFont-Regular.ttf: "JetBrainsMono Nerd Font" "Regular"
```

```
curl -sS https://starship.rs/install.sh | sh
starship --version
```
## 配置 Alacritty
```bash
mkdir -p ~/.config/alacritty
vim ~/.config/alacritty/alacritty.toml
```
字体 + 字号 + 边距 + 透明度 + 光标。[主题设计参考网站](https://github.com/alacritty/alacritty-theme)
> [!NOTE]- 参考配置
> ```toml
> # ── 窗口 ──────────────────────────────────
> [window]
> padding = { x = 10, y = 10 }
> dynamic_padding = true
> opacity = 0.95
> 
> # ── 字体 ──────────────────────────────────
> [font]
> size = 11.0
> 
> [font.normal]
> family = "JetBrainsMono Nerd Font"
> style = "Regular"
> 
> [font.bold]
> family = "JetBrainsMono Nerd Font"
> style = "Bold"
> 
> [font.italic]
> family = "JetBrainsMono Nerd Font"
> style = "Italic"
> 
> # ── 光标 ──────────────────────────────────
> [cursor]
> style = { shape = "Beam", blinking = "On" }
> 
> # ── Shell ─────────────────────────────────
> [terminal.shell]
> program = "/usr/bin/fish"
> 
> # ── 基础配色 ──────────────────────────────
> [colors.primary]
> background = "#1a1b26"        # Tokyo Night 深蓝背景
> foreground = "#cdd6f4"        # Mocha 浅灰蓝文字
> dim_foreground = "#7f849c"
> bright_foreground = "#e6e9ff"
> 
> # 淡紫色光标
> [colors.cursor]
> text = "#1a1b26"
> cursor = "#b4befe"
> 
> [colors.vi_mode_cursor]
> text = "#1a1b26"
> cursor = "#a6e3a1"
> 
> # 选中文字：蓝灰底，亮色文字
> [colors.selection]
> text = "#e6e9ff"
> background = "#3b4261"
> 
> # 搜索结果：普通匹配用蓝色，当前匹配用奶油黄
> [colors.search.matches]
> foreground = "#1a1b26"
> background = "#89b4fa"
> 
> [colors.search.focused_match]
> foreground = "#1a1b26"
> background = "#f9e2af"
> 
> # 链接提示标签
> [colors.hints.start]
> foreground = "#1a1b26"
> background = "#f9e2af"
> 
> [colors.hints.end]
> foreground = "#1a1b26"
> background = "#b4befe"
> 
> # ── 标准 ANSI 调色板 ──────────────────────
> [colors.normal]
> black = "#32344a"
> red = "#f38ba8"               # 柔和粉红
> green = "#a6e3a1"             # 浅绿
> yellow = "#f9e2af"            # 奶油黄
> blue = "#7aa2f7"              # Tokyo Night 蓝
> magenta = "#bb9af7"           # Tokyo Night 紫
> cyan = "#94e2d5"              # 薄荷青
> white = "#bac2de"
> 
> # ── 明亮 ANSI 调色板 ──────────────────────
> [colors.bright]
> black = "#737aa2"             # 灰色文字保持可读性
> red = "#ff9eb5"
> green = "#b5edb0"
> yellow = "#ffedc2"
> blue = "#89b4fa"
> magenta = "#c7a9ff"
> cyan = "#a6eee2"
> white = "#e6e9ff"
> 
> # 部分程序使用的扩展颜色
> [[colors.indexed_colors]]
> index = 16
> color = "#fab387"             # 桃橙
> 
> [[colors.indexed_colors]]
> index = 17
> color = "#f5e0dc"             # 浅玫瑰
> ```

## 配置Kitty
```bash
mkdir -p ~/.config/kitty
vim ~/.config/kitty/kitty.conf
```

> [!NOTE]- 参考配置
> ```conf
> # 普通文字使用的字体
> font_family JetBrainsMono Nerd Font
> 
> # 自动选择与上面字体匹配的粗体、斜体、粗斜体
> bold_font auto
> italic_font auto
> bold_italic_font auto
> 
> # 字号：11 pt（点）
> font_size 11.0
> 
> # 文字与窗口边缘之间的留白：四边各 10 pt
> window_padding_width 10
> 
> # 背景不透明度：95%，即轻微透明
> background_opacity 0.95
> 
> # 文本光标形状：竖线 |
> cursor_shape beam
> 
> # 光标闪烁间隔：0.5 秒
> cursor_blink_interval 0.5
> 
> # Kitty 默认启动的 Shell
> shell /usr/bin/zsh
> ```

## 初始化starship
```bash
echo 'eval "$(starship init bash)"' >> ~/.bashrc
echo 'eval "$(starship init zsh)"' >> ~/.zshrc
fish需要手动修改
vim ~/.config/fish/config.fish
starship init fish | source

# 注意这部分指令应该分别在其对应的shell环境下运行
# Bash 中
source ~/.bashrc
# Zsh 中
source ~/.zshrc
# Fish 中
source ~/.config/fish/config.fish

nano ~/.config/starship.toml
```
starship.toml用于配置starship美化项[参考主题](https://starship.rs/presets/)
```zsh
starship preset nerd-font-symbols -o ~/.config/starship.toml
```
显示当前使用的是哪个shell
```toml
[shell]
disabled = false
bash_indicator = "bash"
zsh_indicator = "zsh"
fish_indicator = "fish"
unknown_indicator = "shell"
style = "bold #b4befe"
format = "[$indicator]($style) "
```
# shell美化
## fish
关闭欢迎提示信息:
```bash
set -U fish_greeting
```
## zsh
```bash
#设置为默认shell
chsh -s /usr/bin/zsh
#安装插件来提供语法检查、自动补全、Tab 选择、历史记录,
sudo dnf --repo=fedora --repo=updates install zsh-autosuggestions zsh-syntax-highlighting
mkdir -p ~/.local/share/zsh
git clone --depth 1 https://github.com/zsh-users/zsh-completions.git \
    ~/.local/share/zsh/zsh-completions
    
vim ~/.zshrc

#检查zsh配置并生效
zsh -n ~/.zshrc && exec zsh

#以后更新这套补全规则时执行
git -C ~/.local/share/zsh/zsh-completions pull --ff-only
```

> [!NOTE]- 参考配置
> ```
> ZSH_AUTOSUGGEST_STRATEGY=(history completion)
> 
> # 蓝灰色，与实际输入的文字区分
> ZSH_AUTOSUGGEST_HIGHLIGHT_STYLE='fg=#737aa2'
> 
> source /usr/share/zsh-autosuggestions/zsh-autosuggestions.zsh
> 
> # ── Starship 提示符 ───────────────────────
> eval "$(starship init zsh)"
> 
> # ── 输入高亮配色 ──────────────────────────
> typeset -A ZSH_HIGHLIGHT_STYLES
> 
> # 普通文字
> ZSH_HIGHLIGHT_STYLES[default]='fg=#cdd6f4'
> 
> # 有效命令：薄荷青
> ZSH_HIGHLIGHT_STYLES[command]='fg=#94e2d5'
> ZSH_HIGHLIGHT_STYLES[builtin]='fg=#94e2d5'
> ZSH_HIGHLIGHT_STYLES[function]='fg=#94e2d5'
> ZSH_HIGHLIGHT_STYLES[alias]='fg=#94e2d5'
> ZSH_HIGHLIGHT_STYLES[hashed-command]='fg=#94e2d5'
> 
> # 未识别的命令：柔和粉红
> ZSH_HIGHLIGHT_STYLES[unknown-token]='fg=#f38ba8'
> 
> # 关键字：淡紫
> ZSH_HIGHLIGHT_STYLES[reserved-word]='fg=#bb9af7'
> ZSH_HIGHLIGHT_STYLES[precommand]='fg=#bb9af7'
> 
> # 参数选项：蓝色
> ZSH_HIGHLIGHT_STYLES[single-hyphen-option]='fg=#89b4fa'
> ZSH_HIGHLIGHT_STYLES[double-hyphen-option]='fg=#89b4fa'
> 
> # 字符串：浅绿
> ZSH_HIGHLIGHT_STYLES[single-quoted-argument]='fg=#a6e3a1'
> ZSH_HIGHLIGHT_STYLES[double-quoted-argument]='fg=#a6e3a1'
> 
> # 路径：浅蓝，并带下划线
> ZSH_HIGHLIGHT_STYLES[path]='fg=#89b4fa,underline'
> 
> # 管道、重定向等符号
> ZSH_HIGHLIGHT_STYLES[commandseparator]='fg=#b4befe'
> ZSH_HIGHLIGHT_STYLES[redirection]='fg=#b4befe'
> 
> # 语法高亮最后加载
> source /usr/share/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh
> 
> ```

# 程序启动器
常见启动器:

| 启动器 | 主要特点 |
| --- | --- |
| **Fuzzel** | Wayland 原生，轻量，适合搜索并启动应用 |
| **Rofi** | 模式、主题和脚本扩展丰富；目前主线已支持 Wayland |
| **Wofi** | 基于 GTK 的 Wayland 启动器 |
| **Tofi** | 偏极简的 Wayland 启动器 |
| **Ulauncher / Albert** | 偏向带扩展功能的桌面搜索启动器 |
| **KRunner** | KDE 生态中的运行、搜索和操作入口 |
| **DMS 启动器** | 与 DMS 桌面界面、搜索和插件整合 |

## Fuzzel
面向 Wayland 桌面的轻量应用启动器，可以理解成键盘操作的“开始菜单”
- 输入应用名称，筛选结果，按 Enter 启动。
- 读取应用的 `.desktop` 启动项，显示名称、图标。
- 支持配置字体、颜色、大小等外观。
- 还提供 `--dmenu` 选择器模式，可由脚本传入选项，制作其他菜单；这些扩展用途需要额外脚本配合。(普通模式下，候选列表是应用程序；`--dmenu` 模式下，**候选列表由你提供**，可以是目录、操作名称、表情或其他文本。)
---
 一般规则是读取 \${XDG_DATA_HOME:-\$HOME/.local/share}/applications，以及 $XDG_DATA_DIRS 各目录下的 applications 子目录。
但**不能理解成“一个显示项必定对应一个独立软件”**：
- 有些 `.desktop` 文件设置了隐藏条件，不会显示。
- 同一个软件可以有多个启动项。
- 开启 Fuzzel 的 `show-actions` 后，一个文件里的“新建窗口”等操作也可成为单独的显示项。
- 同一 desktop ID 的用户条目可以覆盖系统条目。[Desktop Entry 规范](https://specifications.freedesktop.org/desktop-entry/latest-single/)
---
### 不同桌面环境下应用的显示项
一个应用显示多个相同功能的选项如`GNOME System Monitor(设置了 OnlyShowIn=KDE;),System Monitor(设置了 NotShowIn=KDE;)`:推荐在\${XDG_CONFIG_HOME:-\$HOME/.config}/fuzzel/fuzzel.ini首个 `[colors]` 之前的全局区域加入：
```bash
filter-desktop=yes #filter-desktop=yes 的作用是：让 Fuzzel 根据当前桌面环境，筛选应该显示的应用入口。
```
### 应用需要fuzzel调用终端时
如btop++,需要Fuzzel 调用终端,在fuzzel.ini添加
```
terminal=alacritty -e
```
### 外观配置：改变fuzzel长什么样
Fuzzel 自己管理字体、配色、尺寸和边框，可以通过配置文件调整。默认用户配置位置是：
```bash
mkdir -p ${XDG_CONFIG_HOME:-$HOME/.config}/fuzzel/
vim ${XDG_CONFIG_HOME:-$HOME/.config}/fuzzel/fuzzel.ini
```

```ini
[colors]
background=002b36cc
text=839496ff
prompt=93a1a1ff
placeholder=586e75ff
input=839496ff
match=cb4b16ff
selection=073642ff
selection-text=93a1a1ff
selection-match=cb4b16ff
counter=586e75ff
border=fdf6e3ff
```


# 文件管理器
| 文件管理器                        | 所属生态／特点               |
| ---------------------------- | --------------------- |
| **Nautilus / Files**         | GNOME，界面简洁，适合日常文件操作   |
| **Dolphin**                  | KDE，功能和可配置项丰富，支持分栏等操作 |
| **Thunar**                   | Xfce，偏轻量、传统的文件管理界面    |
| **Nemo**                     | Cinnamon，传统布局，功能较丰富   |
| **PCManFM / PCManFM-Qt**     | LXDE／LXQt，偏轻量         |
| **Caja**                     | MATE，传统桌面文件管理器        |
| **Yazi / ranger / nnn / lf** | 在终端中使用，主要通过键盘操作       |
## Nautilus
`nautilus-open-any-terminal`:第三方扩展补充功能:右键添加“在此处打开 终端”
```bash
# 启用该项目提供的第三方 COPR 源
sudo dnf copr enable monkeygold/nautilus-open-any-terminal

# 安装扩展及 Python 支持
sudo dnf install nautilus-open-any-terminal nautilus-python

# 将 nautilus-open-any-terminal 扩展使用的终端设为 Alacritty。指定使用 Alacritty；以普通用户执行，不加 sudo
gsettings set com.github.stunkymonkey.nautilus-open-any-terminal terminal alacritty
#恢复扩展的默认值
gsettings reset com.github.stunkymonkey.nautilus-open-any-terminal terminal
#恢复后默认使用哪个终端(通常为gnome-terminal)
gsettings get com.github.stunkymonkey.nautilus-open-any-terminal terminal

#退出正在运行的 Nautilus
nautilus -q
```
`sushi`:选中文件后按空格快速预览，减少打开完整应用的次数
```bash
sudo dnf install sushi -y
```
# 桌面概览背景与桌面保持一致
编辑 cconfig,添加以下两处配置：

1. 在**已有的 `layout { ... }` 内**加入这一行，保留其他设置：

```
background-color "transparent"
```

2. 在文件末尾添加这个**顶层区块**，不要放进 `layout` 内：

```
layer-rule {
    match namespace="^quickshell$"
    place-within-backdrop true
}
```

这样壁纸会作为固定背景，概览中的工作区和窗口在它上方缩小。[niri 官方说明](https://github.com/niri-wm/niri/blob/main/docs/wiki/Overview.md)

```
overview {
    //窗口缩放比例(0 到 0.75)
    zoom 0.5
    //关闭工作区阴影，让背景更干净
    workspace-shadow {
        off
    }
}
```

保存后运行语法检查：

```
niri validate
```

然后按你已配置的 **Mod+O** 查看效果。恢复时删除上述新增内容即可。


# DMS内置命令和接口
```bash
dms ipc call <目标模块> <操作> [操作参数...]
dms ipc call launcher toggle
```

|部分|含义|
|---|---|
|`dms`|执行 DMS 命令行程序|
|`ipc`|使用进程间通信接口，联系运行中的 DMS|
|`call`|调用某个接口|
|`launcher`|目标模块：启动器|
|`toggle`|操作：打开时关闭，关闭时打开|

# niri/config.kdl文件配置
## 自行修改过的快捷键速查
	- Mod+Shift+S    区域截图
	- Mod+Shift+A    当前显示器截图
	- Mod+Shift+W    当前窗口截图
	- Mod+Ctrl+V     打开剪贴板
	- Alt+Q 打开或唤出 Firefox“划词翻译”的独立窗口，并将它设为浮动、切换焦点到该窗口
## 截图相关
[[Fedora Minimal+Niri+DMS#基础截图(niri 自带截图)]]
## 概览相关

[桌面概览背景与桌面保持一致](#桌面概览背景与桌面保持一致)


## 窗口圆角
```
window-rule {
    geometry-corner-radius 12
    clip-to-geometry true
}
```
- `12` 是圆角半径，可以改成 `8`、`16` 等。
- `clip-to-geometry true` 才会实际裁剪窗口圆角，否则只影响边框和焦点框。
- 不写 `match` 就应用到所有窗口。[官方说明](https://niri-wm.github.io/niri/Configuration%3A-Window-Rules.html#geometry-corner-radius)

## 隐藏软件的放大缩小全屏
```
prefer-no-csd
```

## 启动Fcitx5
`spawn-at-startup "fcitx5"`


## niri 的焦点环 `focus-ring`
```
focus-ring {
    width 2
    active-color "#b4befe"
    inactive-color "#3b4261"
}
```

- `width 2`：边框变细。
- `active-color`：活动显示器上焦点窗口的边框颜色。
- `inactive-color`：非活动显示器上焦点环的颜色。
## 鼠标相关
鼠标移到哪个窗口，哪个窗口就自动获得焦点，无需点击在`input` 块里面添加：
```
input {
    focus-follows-mouse
}
```
如果不希望聚焦时带动桌面横向滚动，改用：
```
input {
    focus-follows-mouse max-scroll-amount="0%"
}
```
### 替换锁屏,swaylock
改为使用dms greeter的锁屏(如果使用的是dms项目的登录管理器)
```
Super+Alt+L hotkey-overlay-title="Lock the Screen: dms" { spawn "dms" "ipc" "call" "lock" "lock"; }
```



# 卸载程序

# Flatpak
**1. 查看当前源**

```
flatpak remotes --show-details
```

`flathub` 是源名称，实际使用哪个服务器取决于 URL。

**2. 切换到官方源**

```
sudo flatpak remote-modify --system flathub \
  --url=https://dl.flathub.org/repo
```

刷新应用目录元数据，然后检查：

```bash
flatpak update --system --appstream flathub 
#刷新应用目录
flatpak remotes --system --show-details 
#查看软件源配置
```

预期 `flathub` 的 URL 变为 `https://dl.flathub.org/repo`。刷新元数据不会升级已安装应用。

**3. 如果官方下载慢，恢复中科大源**

```
sudo flatpak remote-modify --system flathub \
  --url=https://mirrors.ustc.edu.cn/flathub
```

中科大提供的是**缓存服务**：官方说明元数据每小时同步，未命中的文件可能重定向到 Flathub，所以仍需要能够连接官方站点。[中科大配置说明](https://mirrors.ustc.edu.cn/help/flathub.html)


# 一些零碎记录
[部分概念](wiki/部分概念.md)
[遇到的一些零碎问题](wiki/遇到的一些零碎问题.md)


