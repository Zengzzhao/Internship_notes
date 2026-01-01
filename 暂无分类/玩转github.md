# github pages

GitHub Pages 是 GitHub 提供的「静态网站托管服务」



# github actions

**yml文件配置**

```yaml
# workflow的名称。如果省略该字段，默认为当前workflow的文件名
name: Release
# 指定触发workflow的条件
on:
  # 定时自动触发
  schedule:
    - cron: "0 */2 * * *" # cron表达式，每2小时执行一次
  # 允许任何时候手动触发
  workflow_dispatch:
  
  push: # push事件触发workflow
    branches:
      - main # 只要在main分支上push才出发workflow

# workflow文件的主体是jobs字段，表示要执行的一项或多项任务
jobs:
  release:
    name: Release # name就是job任务说明
    permissions: # 工作流该job对仓库权限
      contents: write
    runs-on: ubuntu-latest # 运行所需要的虚拟机环境
		timeout-minutes: 10 # 最多运行时间
		# steps字段指定每个Job的运行步骤，可以包含一个或多个步骤
    steps:
      - name: Install Dependencies
        run: pnpm install # 在终端执行该shell命令，与uses同时只能有一个
        
      - name: Publish to npm # name就是steps步骤说明
        id: changesets
        uses: changesets/action@v1 # 使用写好的actions(owner/repo@ref),changesets/action@v1表示changesets的 actions,使用v4这个版本
        with: # 给uses的action传递参数
          publish: pnpm changeset publish #  无 changeset的xxx.md 时执行的发布命令
        env: # 给当前step设置环境变量
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }} # 用于创建 PR
          NPM_TOKEN: ${{ secrets.NPM_TOKEN }} # 用于发布到 npm
          
```

## 常用job的step

**常用自己写的run**

```yaml
# 安装依赖
- name: Install Dependencies
	run: pnpm install
# 打包
- name: Build Packages
  run: pnpm run build
# 运行仓库中某个js脚本
- name: Run Js Script
	run: node xxx.js
# 提交修改应用到当前拉取代码的默认分支main
- name: Commit & Push
	run: |
		git config user.name github-actions
		git config user.email github-actions@github.com
		git add -A .
		git commit -m "🤖 Auto Update Daily Quotations"
		git push

```

**常用action**

```yaml
# 将当前仓库代码拉到runner工作目录
- name: Checkout repository
	uses: actions/checkout@v4
# 在runner上安装并切换指定版本的pnpm
- name: Install pnpm
	uses: pnpm/action-setup@v2
  with:
    version: 10
# 在runner上安装并切换指定版本的node
- name: Set up Node.js
	uses: actions/setup-node@v4
  with:
    node-version: 22
    cache: pnpm

```

