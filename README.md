# 这是什么
chnroute的介绍见 https://code.google.com/p/chnroutes/
本项目chnroute-automation-mac的目的是自动化chnroute，只需要把它添加一次，以后无论网络怎么变化，chnroute都会自动的被适配到新的网络环境中去。

本脚本 Mac Only

# 如何使用
```shell
git clone https://github.com/cattyhouse/chnroute-automation-mac
cd chnroute-automation-mac
./init ＃ 执行init程序，生成所需的命令和脚本文件
mkdir $HOME/bin
cp route-add route-auto route-change route-delete pre_gw $HOME/bin/
cp org.chnroute.automation.plist $HOME/Library/LaunchAgents/
launchctl load -w $HOME/Library/LaunchAgents/org.chnroute.automation.plist
```
搞定，以后就可以撒手不管了，只要电脑网关变化（与上一次对比），chnroute的条目自动适配到新的网关上，网关如果不变，则不会执行任何程序。
# 注意事项
Mac上添加删除路由表是需要root权限的，为了让plist正常运行，需要设置 sudoers, 方法如下：
* 运行 `whoami` 得到你的账户名，比如justin
* 运行 `sudo visudo`，添加一行 `justin  ALL=(ALL) NOPASSWD: ALL`

# 如何删除chnroute
首先你直接用本程序生成的 route-delete 删除是无效的，因为这个操作会被launchd监控到并执行route-auto，而route-auto发现chnroute被删了，就会运行route-add给添加回来，正确的删除方法是：
* 先 `launchctl unload -w $HOME/Library/LaunchAgents/org.chnroute.automation.plist`
* 再执行 sudo $HOME/bin/route-delete
