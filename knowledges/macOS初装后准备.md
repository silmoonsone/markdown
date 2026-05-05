# macOS 准备

## 1. 向 `.zprofile` 添加配置

```shell
alias cls="clear && printf '\e[3J'"
export PATH="$PATH:/Users/USERNAME/.dotnet/tools"
```

## 2. 安装 Homebrew

```shell
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

## 3. 安装 Git LFS

```shell
brew install git-lfs
git lfs install
```

## 4. 其他常用安装

```shell
brew update
brew install git
brew install wget
```