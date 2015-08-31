# 这是什么
`chnroute`的介绍见[chnroute](https://code.google.com/p/chnroutes/), 本项目`chnroute-automation-mac`的目的是自动化`chnroute`，只需要把它添加一次，以后无论网络怎么变化，`chnroute`会自动的被适配到新的网络环境中去。

**本脚本 Mac OS X Only**

# 原理和逻辑
1. `launchd`有一个功能，就是监控文件夹的变化来执行程序，此项目里面，我让launchd监控了： `/Library/Preferences/SystemConfiguration`,当网络变化的时候，这个文件夹一定会变化，而launchd能在秒级别里面检控到并执行程序
2. launchd监控到网络变化，会执行`route-auto`这个脚本，而此脚本会发现当前网关并根据以下情况调用 `route-add` 或者 `route-change`。`route-add` 和 `route-change`的作用就如字面意思上那么简单，就是添加路由和改变路由。
3. 如果当前系统路由的条目数量小于1000个，那么`chnroute`是肯定没有被添加过的（或者之前被删除过），此时`route-auto`这个脚本会获取当前网关，并开始调用 `route-add` 添加`chnroute`，同时将网关的ip写入`pre_gw`供下次用
4. 如果系统路由条目大于1000，说明已经添加过`chnroute`，这时候会将当前的网关ip与`pre_gw`的对比，如果一样，则什么都不执行，如果不一样，则执行`route-change`改变已有的chnroute路由表到新的网关

# 如何使用

```Shell
git clone https://github.com/cattyhouse/chnroute-automation-mac
cd chnroute-automation-mac
./init ＃ 执行init程序，生成所需的命令和脚本文件
mkdir $HOME/bin
cp route-add route-auto route-change route-delete pre_gw $HOME/bin/
cp org.chnroute.automation.plist $HOME/Library/LaunchAgents/
launchctl load -w $HOME/Library/LaunchAgents/org.chnroute.automation.plist
```
搞定，以后就可以撒手不管了，只要电脑网关变化（与上一次对比），`chnroute`的条目自动适配到新的网关上，网关如果不变，则不会执行任何程序。

# 注意事项
Mac上添加删除路由表是需要root权限的，为了让plist正常运行，需要设置 `sudoers`, 方法如下：

1. 运行 `whoami` 得到你的账户名，比如justin
2. 运行 `sudo visudo`，添加一行 `justin  ALL=(ALL) NOPASSWD: ALL`

# 如何删除`chnroute`
首先你直接用本程序生成的 `route-delete` 删除是无效的，因为这个操作会被launchd监控到并执行`route-auto`，而`route-auto`发现`chnroute`被删了，就会运行`route-add`给添加回来，正确的删除方法是

1. 先 `launchctl unload -w $HOME/Library/LaunchAgents/org.chnroute.automation.plist`
2. 再执行 `sudo $HOME/bin/route-delete`

# 小技巧

1. 查看当前生效的路由表，运行 `netstat -nr -f inet`
2. 查看当前生效的路由表条目的数量，运行 `netstat -nr -f inet | sed '1,4d' | wc -l | xargs`
