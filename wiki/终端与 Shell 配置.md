# 终端与 Shell 配置

本文介绍 **Kitty + Zsh + Starship** 的安装、配置与验证。Kitty 负责终端显示，Zsh 提供命令交互，Starship 显示目录、Git 状态等提示符信息。基础术语见[部分概念](部分概念.md)

## 部署顺序

1. 安装 Kitty、Zsh、Starship。
2. 安装并检查 Nerd Font。
3. 创建 Kitty、 Zsh和其插件/Oh My Zsh、 Starship 配置。
4. 在图形桌面打开 Kitty，验证提示符与交互功能。
5. 使用 niri 时，配置打开终端的快捷键。

账户登录 Shell 可以保留 Bash，Kitty 通过自身配置启动 Zsh，无需执行 `chsh`。

## 配置文件

| 配置        | 默认位置                                                     |
| --------- | -------------------------------------------------------- |
| Alacritty | `~/.config/alacritty/alacritty.toml`                     |
| Kitty     | `~/.config/kitty/kitty.conf`                             |
| Ghostty   | `~/.config/ghostty/config.ghostty`（1.2.3 起）；旧版为 `config` |
| Bash      | `~/.bashrc`                                              |
| Zsh       | `~/.zshrc`；设置了 `ZDOTDIR` 时使用其目录                          |
| Fish      | `~/.config/fish/config.fish`                             |
| Starship  | `~/.config/starship.toml`；可用 `STARSHIP_CONFIG` 指定        |

本文使用默认用户目录：配置放在 `~/.config`，字体和补全资源放在 `~/.local/share`，无需额外设置路径变量。完整示例存放在仓库的 `config/` 目录，后文复制命令均在下载或克隆后的**仓库根目录**执行。

## 安装软件

### 安装终端、Shell 与辅助工具

```bash
sudo dnf install kitty zsh git curl unzip fontconfig vim-enhanced less
```

安装后检查：

```bash
kitty --version
zsh --version
command -v zsh
```

### 安装 Starship

本文采用[官方安装脚本](https://starship.rs/guide/)方式。先下载到临时目录，阅读脚本后再执行

```bash
starship_stage=$(mktemp -d)
curl -fL https://starship.rs/install.sh -o "$starship_stage/install.sh"
less "$starship_stage/install.sh"
```

确认下载成功并检查内容后，在同一个 Bash 会话中执行：

```bash
sh "$starship_stage/install.sh"
starship --version
```


## 字体准备

### 安装 Nerd Font

Nerd Fonts 提供提示符常用的图标字形。以下选择 JetBrainsMono Nerd Font；已有合适字体时可跳过下载，直接检查字体名称。

从 [Nerd Fonts 官方发布页](https://github.com/ryanoasis/nerd-fonts/releases)下载 `JetBrainsMono.zip`，解压到用户字体目录。例如，将压缩包保存为 `$HOME/Downloads/JetBrainsMono.zip` 后执行：

```bash
font_archive="$HOME/Downloads/JetBrainsMono.zip"
font_dir="$HOME/.local/share/fonts/JetBrainsMonoNerd"
mkdir -p "$font_dir"
unzip -n "$font_archive" -d "$font_dir"
fc-cache -f
```

### 检查字体是否可用

```bash
fc-list | grep -i 'JetBrainsMono.*Nerd' | head
fc-match 'JetBrainsMono Nerd Font'
```

预期返回 JetBrainsMono Nerd Font 对应字体。`fc-match` 即使找不到指定字体也可能返回替代字体，需核对输出名称。安装字体后还要在终端配置中选中它；中文字符显示另需具有相应字形的字体。

## 配置 Kitty

创建配置目录并复制示例：

```bash
mkdir -p "$HOME/.config/kitty"
cp -i config/kitty.conf "$HOME/.config/kitty/kitty.conf"
```

示例通过 `shell /usr/bin/zsh` 启动 Zsh，并设置字体、边距与配色，其余选项沿用默认值。

完整配置见 [kitty.conf](../config/kitty.conf)。

字体大小以点计；Kitty 的边距单位与 Alacritty 不同，相同数值不保证在两者中呈现相同宽度。保存后运行 `kitty` 检查启动输出及外观。详细选项见 [Kitty 配置文档](https://sw.kovidgoyal.net/kitty/conf/)。

## 配置 Zsh

本节采用独立安装、直接加载插件的方案。插件的安装与配置均在本节完成，完整加载顺序保存在 [zshrc](../config/zshrc) 中。Oh My Zsh 的替代思路见下方补充，两套初始化方式应择一使用。

### 组件与作用

| 组件 | 作用 | 安装方案 |
| --- | --- | --- |
| Zsh 补全系统 | 按 Tab 补全命令、路径和参数，显示候选菜单 | 随 Zsh 提供，无需另外安装；通过 `compinit` 初始化 |
| zsh-completions | 为更多命令提供补全定义，扩充 Tab 补全范围 | 可选，从上游 Git 仓库下载 |
| zsh-autosuggestions | 输入时显示建议，示例先匹配历史记录，再尝试补全 | 通过 DNF 安装，随后在 `.zshrc` 中加载 |
| zsh-syntax-highlighting | 为命令、选项、字符串与路径着色，便于辨认输入 | 通过 DNF 安装，在配置末尾加载 |

自动建议用于提示可能要输入的内容，Tab 补全用于选择候选项；语法高亮只提供视觉提示，不保证命令正确或执行安全。

### 安装自动建议

```bash
sudo dnf install zsh-autosuggestions
```

安装后，配置模板通过 `source /usr/share/zsh-autosuggestions/zsh-autosuggestions.zsh` 加载插件，并设置建议策略和文字颜色。仅安装软件包不会自动为个人 `.zshrc` 添加这些设置。[上游安装说明](https://github.com/zsh-users/zsh-autosuggestions/blob/master/INSTALL.md)

### 安装语法高亮

```bash
sudo dnf install zsh-syntax-highlighting
```

配置模板先定义高亮颜色，再加载 `/usr/share/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh`。该加载语句放在其他插件和交互设置之后，避免后续修改影响高亮行为。[上游安装说明](https://github.com/zsh-users/zsh-syntax-highlighting/blob/master/INSTALL.md)

### 可选：安装额外补全定义

Zsh 已有基础补全，`zsh-completions` 用于扩展支持的命令。需要时执行：

```bash
mkdir -p "$HOME/.local/share/zsh"
git clone --depth 1 https://github.com/zsh-users/zsh-completions.git \
  "$HOME/.local/share/zsh/zsh-completions"
```

配置模板会在目录存在时，将其中的 `src` 加入 `fpath`，然后执行 `compinit`。跳过该扩展不会禁用 Zsh 自带补全。[上游说明](https://github.com/zsh-users/zsh-completions)

### 加载配置

完成两个 DNF 插件和前文 Starship 的安装后，在仓库根目录执行：

```bash
cp -i config/zshrc "$HOME/.zshrc"
```

完整配置见 [zshrc](../config/zshrc)，包含补全、历史记录、别名、自动建议、Starship 初始化和输入高亮。已有 `.zshrc` 时按功能合并，不重复添加插件加载语句；若不使用某个插件，应同时移除模板中对应的加载与设置。

### 补充：Oh My Zsh 替代方案

Oh My Zsh 提供基础 Zsh 设置、插件加载和提示符主题。后续可扩展为 **Kitty + Zsh + Oh My Zsh 主题及插件** 的独立方案。本节先说明替代关系，完整安装、配置模板和迁移验证步骤待补充；不作为当前主方案的必需步骤。[Oh My Zsh 官方说明](https://github.com/ohmyzsh/ohmyzsh)

| 当前组件 | Oh My Zsh 方案中的处理 |
| --- | --- |
| Starship | 使用 Oh My Zsh 主题负责提示符，并移除 `starship init zsh`；外观和模块能力取决于所选主题 |
| 手动执行 `compinit` | 交由框架初始化，避免重复执行 |
| zsh-autosuggestions | 仍使用该外部插件，可安装到框架的自定义插件目录，再由 `plugins=(...)` 加载 |
| zsh-syntax-highlighting | 仍使用该外部插件，按上游说明配置加载顺序；采用框架加载后移除原来的重复加载语句 |
| zsh-completions | 仍是可选扩展；保留时需在框架初始化前将其 `src` 加入 `fpath` |
| 历史容量、个人配色和快捷键 | 按需迁移，并检查是否被框架默认设置覆盖 |

这里由 **Oh My Zsh 的主题替代 Starship**，由框架接管插件加载；自动建议和语法高亮仍由原插件实现。外部插件需要另行安装，写入 `plugins=(...)` 不会自动下载，也不会自动更新这些外部仓库。

若后续采用自定义插件目录的安装方式，应明确其更新入口，并避免同时加载 DNF 安装和 Git 下载的两份插件。迁移时先备份 `.zshrc`，保留原配置和 Starship 配置文件，验证新方案后再决定是否清理旧组件；恢复时可还原原配置。



## 配置 Starship 外观

要使shell读取到Starship配置需要进行初始化见[starship初始化](https://starship.rs/guide/)

前面的 `.zshrc` 示例已经包含 Starship 初始化，无需重复添加。Starship 默认读取 `~/.config/starship.toml`；没有该文件时使用默认样式。

### 提示符样式与模块

创建配置目录并复制示例：

```bash
mkdir -p "$HOME/.config"
cp -i config/starship.toml "$HOME/.config/starship.toml"
```

示例启用显示当前 Shell 的模块，其余模块沿用默认值。若已有 `[shell]`，修改其字段即可；若自定义顶层 `format`，还需确认它包含 `$shell` 或 `$all`。

完整配置见 [starship.toml](../config/starship.toml)。模块与格式说明见 [Starship 配置文档](https://starship.rs/config/#shell)。

### 使用预设

预设适合作为新配置的起点。已有自定义配置时，先生成候选文件，比较后再合并，不直接覆盖 `starship.toml`：

```bash
starship preset nerd-font-symbols \
  -o "$HOME/.config/starship.nerd-font-symbols.example.toml"
```

该命令会写入指定候选文件，文件已有内容时也应先备份。更多预设见 [Starship 预设列表](https://starship.rs/presets/)。

## 验证与常见问题

完成主要方案后，先检查 Zsh 配置语法：

```bash
zsh -f -n "$HOME/.zshrc"
```

语法检查不执行插件，也不能证明运行时功能完整。在图形会话中运行 `kitty`，打开新窗口后执行：

```zsh
printf 'Zsh 版本：%s\n' "$ZSH_VERSION"
```

预期显示 Zsh 版本。`$SHELL` 可能仍显示账户登录 Shell，不用于判断当前交互 Shell。随后确认：

- 字体和图标可读，字号、光标与背景符合预期。
- Zsh 的 Tab 补全、历史建议和输入着色分别生效。
- Starship 显示提示符，目录变化时内容更新，无初始化错误。

| 现象 | 优先检查 |
| --- | --- |
| 图标为方框 | Nerd Font 是否安装，以及终端实际选用的字体 |
| 新窗口仍是另一个 Shell | 终端显式启动设置、账户默认 Shell 与启动器参数 |
| Starship 未显示 | 命令是否在 PATH 中、当前 Shell 是否加载了初始化片段 |
| 提示符重复或样式被覆盖 | 是否同时加载了多个提示符工具或重复初始化 |
| Zsh 没有额外补全 | `src` 是否加入 `fpath`、是否在 `compinit` 前配置 |
| Zsh 插件找不到 | 安装状态及软件包实际路径 |
| 历史记录没有保存 | `HISTFILE` 路径与权限、历史选项，以及 Shell 是否正常退出 |
| 终端颜色不同于示例 | 是否加载了其他主题、应用是否自行指定颜色 |

## 可选：配置 niri 终端快捷键

在 `~/.config/niri/config.kdl` 的 `binds` 块中加入以下**合并片段**；已有 `Mod+T` 时替换原绑定，避免重复：

```kdl
Mod+T hotkey-overlay-title="Open a Terminal: kitty" { spawn "kitty"; }
```

保存后执行 `niri validate` 检查配置，再按 `Mod+T` 确认能够打开 Kitty。

## 可选终端

以下方案可替代 Kitty，无需与主要方案全部安装。

### Alacritty

以下保留为可选终端方案。未安装时执行：

```bash
sudo dnf install alacritty
```

创建目录并复制外观配置：

```bash
mkdir -p "$HOME/.config/alacritty"
cp -i config/alacritty.toml "$HOME/.config/alacritty/alacritty.toml"
```

示例包含字体、窗口与完整配色，未列出的选项使用默认值。已有配置应合并到对应 TOML 表中，避免重复 `[window]`、`[font]` 等表名。字号、透明度和配色均可调整，选项含义见 [Alacritty 配置文档](https://alacritty.org/config-alacritty.html)。

完整配置见 [alacritty.toml](../config/alacritty.toml)。

保存后在已有终端中运行 `alacritty` 打开新窗口，检查是否有配置错误提示，并检查字体、光标、配色与背景透明度。透明不等于背景模糊，模糊效果还依赖合成器能力和配置。

### Ghostty

**安装：** 截至 2026-10-09，官方安装文档列出的 Fedora 方案包括社区维护的 `scottames/ghostty` COPR。以下命令会启用该第三方仓库并安装软件，需要管理员认证；它不是 Fedora 官方仓库软件，也不是 Ghostty 官方分发的 Linux 二进制包。已有其他来源时保留原安装渠道。[Ghostty 安装说明](https://ghostty.org/docs/install/binary#fedora)

```bash
sudo dnf copr enable scottames/ghostty
sudo dnf install ghostty
ghostty --version
```

**配置文件：** Ghostty 1.2.3 起使用 `config.ghostty`，同时兼容旧文件名 `config`。若两个文件都存在，后加载的旧文件可能覆盖新文件中的值，因此优先编辑实际使用的文件，不重复创建两套配置。[配置格式与查找规则](https://ghostty.org/docs/config)

新配置使用以下路径；旧版本或已有 `config` 的安装应调整文件名。修改已有文件前先保留备份。

```bash
mkdir -p "$HOME/.config/ghostty"
cp -i config/config.ghostty "$HOME/.config/ghostty/config.ghostty"
```

示例设置字体、边距、透明度与基础配色，其余选项沿用默认值。Ghostty 使用自己的 `key = value` 格式；已有文件按字段合并，注释单独成行。

完整配置见 [config.ghostty](../config/config.ghostty)。

字号和边距以点计。光标选项设置默认样式，Shell 集成或运行中的应用仍可能调整光标。具体选项见 [Ghostty 配置参考](https://ghostty.org/docs/config/reference)。

保存后从已有终端运行 `ghostty`，检查配置报错、字体和透明度。调整后使用新窗口确认效果；已有实例可能需要先重新加载配置。

### 为可选终端指定 Zsh

未设置时，终端按自身默认规则选择 Shell。若仅希望某个终端运行 Zsh，先确认 `command -v zsh` 返回实际路径，再选择对应配置。

Alacritty 的**可选合并片段**：

```toml
[terminal.shell]
program = "/usr/bin/zsh"
```

Ghostty 的**可选合并片段**：

```conf
command = /usr/bin/zsh
```

使用 Fish 时换成其实际路径，例如 `/usr/bin/fish`。这只改变该终端启动的程序，不修改账户登录 Shell。

## 可选 Shell 设置

以下内容按需使用；主要方案无需配置 Fish 或更改账户登录 Shell。

### Bash

使用现有 Bash 即可，不需要为了提示符另换 Shell。以下是 `~/.bashrc` 的别名**合并片段**，已存在时修改原定义：

```bash
alias ll='ls -alh'
```

Bash 的补全和历史设置可保留发行版现有配置。如需使用 Starship，在 `~/.bashrc` 的交互配置部分加入：

```bash
eval "$(starship init bash)"
```

### Fish

未安装时执行：

```bash
sudo dnf install fish
mkdir -p "$HOME/.config/fish"
vim "$HOME/.config/fish/config.fish"
```

Fish 自带交互补全、输入建议和语法高亮。以下为 `config.fish` 的**合并片段**，放入已有的交互判断块或新建一处：

```fish
if status is-interactive
    set -g fish_greeting
    alias ll 'ls -alh'
    starship init fish | source
end
```

这里通过启动配置关闭欢迎信息，便于随文件迁移。若原先用 `set -U fish_greeting` 保存了通用变量，仅删除这一行不会撤销旧的通用设置；恢复时需一并检查。交互功能见 [Fish 官方文档](https://fishshell.com/docs/current/interactive.html)。

### 可选：修改账户默认 Shell

只有希望更改账户登录 Shell 时才需要此步骤。先记录当前值，并确认目标 Shell 已安装且列在 `/etc/shells`：

```bash
getent passwd "$(id -un)" | cut -d: -f7
cat /etc/shells
command -v zsh
```

以 Zsh 为例，确认后执行：

```bash
chsh -s /usr/bin/zsh
```

该操作可能要求身份认证。重新登录后再次用 `getent` 检查；已经打开的 Shell 不会随之切换，终端显式配置的启动程序也可能覆盖账户默认选择。`$SHELL` 不能可靠证明当前正在运行哪个交互 Shell。

## 更新与恢复

DNF 安装的工具通过 DNF 更新；Ghostty 的 COPR 安装同样通过该仓库更新。手动安装的 Starship 沿用其安装方式。额外补全仓库单独更新：

```bash
git -C "$HOME/.local/share/zsh/zsh-completions" pull --ff-only
```

仓库有个人修改时先检查差异，保留修改；更新后重新验证补全和 Shell 启动。

恢复按实际改动范围进行：

1. 恢复对应配置备份，或只删除本页新增的字段、别名和插件加载语句；保留其他设置。
2. 停用 Starship 时移除对应 Shell 中的初始化行，保留其他 Shell 配置。
3. 修改过账户默认 Shell 时，用 `chsh -s` 指定此前记录且仍可用的原路径。
4. Fish 若存在旧的空 `fish_greeting` 通用变量，可在 Fish 中用 `set -eU fish_greeting` 移除；同时检查配置文件是否仍设置该变量。
5. 打开新窗口检查。配置错误导致 Shell 无法正常交互时，可从可用终端或 TTY 使用 `bash --norc`、`zsh -f` 或 `fish --no-config` 启动临时环境，再修复文件。

若需卸载通过上述 COPR 安装的 Ghostty，可执行 `sudo dnf remove ghostty`；仅在该仓库也是此次新增且不再使用时，执行 `sudo dnf copr disable scottames/ghostty`。用户配置保留，便于恢复。

撤销外观配置通常无需卸载软件。确需卸载时，DNF 软件使用包管理器处理；手动安装的软件先确认实际二进制及所属来源，字体和克隆仓库只移除自行创建且不再使用的内容。

## 验证范围

本文提供 Fedora 环境的部署步骤与配置示例，尚未完成全新系统上的端到端部署验证。实际使用时应完成前述检查；不同 Fedora 版本的软件包、字体名称和桌面设置可能存在差异。
