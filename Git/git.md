# 配置操作

## 全局配置

配置用户的名字和邮箱

```bash
git config --global user.name '名字'
git config --global user.email '邮箱'
```

## 当前仓库配置

```bash
git config --local user.name '名字'
git config --local user.email '邮箱'
```

## 查看global配置

```bash
git config --global --list
```

## 查看当前仓库配置

```bash
git config --local --list
```

## 删除全局配置

```bash
git config --unset --global 要删除的配置项
```

## 删除当前仓库配置

```bash
git config --unset --local 要删除的配置项
```

# 本地操作

## 查看变更情况

```bash
git status
```

## 将变更添加到暂存区

```bash
git add .
```

## 将仓库的所有变更添加到暂存区

```bash
git add -A
```

## 将指定文件添加到暂存区

```bash
git add 文件1 文件2...
```

## 比较工作区和暂存区的差异

```bash
git diff
git diff 文件1 # 某文件的差异
git diff --cached 文件1 # 某文件暂存区和head的差异

```

