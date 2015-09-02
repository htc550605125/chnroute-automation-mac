# 这是什么
`chnroute`的介绍见[chnroute](https://code.google.com/p/chnroutes/), 本项目`chnroute-automation-mac`的目的是自动化`chnroute`，只需要把它添加一次，以后无论网络怎么变化，`chnroute`会自动的被适配到新的网络环境中去。

**本脚本 Mac OS X Only （原则上其思路可以用到Linux上，但是需要找到相应的实现方法，如果具备监控网络变化的工具，Linux上也可以实现这个功能）**
# TODO
经某同学提示，chnroute.txt 可以用 [cidrmerge](http://cidrmerge.sourceforge.net) 来合并一下的，可以将原来的5824条路由变成3757条，这样的话加载修改删除的速度就快很多了。有待测试这 3757条是否和chnroute的5824条包含的ip是否一样。

# 解决了什么问题
chnroute有以下问题：

1. 添加chnroute之后，如果电脑重启动，需要重新手动添加
2. 当连接到新的网络的时候，chnroute还是绑在旧的网关上，导致路由表里面的ip都无法连接，需要手动删除再重新添加

这些都是很郁闷的事情，本程序完美解决了这两个问题。

# 原理和逻辑
1. `launchd`有一个功能`WatchPaths`，就是监控文件的变化来执行程序，此项目里面，我让launchd监控了： `/Library/Preferences/SystemConfiguration/com.apple.airport.preferences.plist`,当Wi-Fi网络变化的时候，这个文件一定会变化，而launchd能在秒级别里面检控到并执行程序 (如果你用的不仅仅是Wi-Fi，请查找`/Library/Preferences/SystemConfiguration/`下对应的文件，并增加到plist的`WatchPaths`下面)
2. launchd监控到网络变化，会执行`route-auto`这个脚本，而此脚本会发现当前网关并根据以下情况调用 `route-add` 或者 `route-change`。`route-add` 和 `route-change`的作用就如字面意思上那么简单，就是添加路由和改变路由。
3. 如果当前系统路由的条目数量小于1000个，那么`chnroute`是肯定没有被添加过的（或者之前被删除过），此时`route-auto`这个脚本会获取当前网关，并开始调用 `route-add` 添加`chnroute`，同时将网关的ip写入`pre_gw`供下次用
4. 如果系统路由条目大于1000，说明已经添加过`chnroute`，这时候会将当前的网关ip与`pre_gw`的对比，如果一样，则什么都不执行，如果不一样，则执行`route-change`改变已有的chnroute路由表到新的网关

# 插曲
我发现切换Wi-Fi的时候，只要开始与新的Wi-Fi握手成功，`/Library/Preferences/SystemConfiguration/com.apple.airport.preferences.plist` 这个文件就会变化，但是这个时候还没有获取ip，而launchd就让route-auto执行了，执行的结果就是什么也不做，而等到获取到了ip，那个监控的文件却不再变化，所以route-auto不会再次执行，导致这个脚本失去意义。于是在route-auto的前面加上了如下代码，确保ip获取到了才执行其余下的代码。

```Shell
_date="date +%F-%H:%M:%S"
until [ -n "`netstat -nr -f inet | grep UHLWIi`" ];do
echo "`$_date` Gateway is not ready, keep trying every second"
sleep 1
done
echo "`$_date` Gateway found, continue to work"
```
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

# 如何Debug

开两个终端窗口，一个运行 `tail -f $HOME/chnroute.auto.log` ，另外一个运行 `sudo tail -f /var/log/system.log | grep "route-"`, 然后试着开关网络，或者切换网络到不同的Wi-Fi热点（比如从路由器的Wi-Fi切换到手机热点上），第一个窗口可以看到`chnroute`是被`添加，改变 还是没反应`，第二个窗口可以看到`route-auto`在什么情况下触发（关闭Wi-Fi不会触发，只有Wi-Fi获取了网络，或者是改变了网络，这个才会触发）

# 效果

这期间Wi-Fi从路由器切换到手机的个人热点，再切换回路由器，再切换到个人热点，再切换到路由器。多次切换，`chnroute`都能自动的适配到新的网关上去。（注意我把`route-add` `route-change`的输出重定向到了` > /dev/null 2>&1` 了，所以看不到那5000多条add和change的输出。），从log可以看出来，要完全add 或者 change chnroute，需要大约13s的时间，毕竟有接近6000条路由。

`tail -f $HOME/chnroute.auto.log`

```
2015-08-31-21:34:38 Current Gateway is 192.168.12.1
2015-08-31-21:39:38 Gateway is not ready, keep trying every second
2015-08-31-21:39:40 Gateway is not ready, keep trying every second
2015-08-31-21:39:41 Gateway is not ready, keep trying every second
2015-08-31-21:39:42 Gateway is not ready, keep trying every second
2015-08-31-21:39:43 Gateway is not ready, keep trying every second
2015-08-31-21:39:44 Gateway is not ready, keep trying every second
2015-08-31-21:39:45 Gateway is not ready, keep trying every second
2015-08-31-21:39:47 Gateway is not ready, keep trying every second
2015-08-31-21:39:48 Gateway is not ready, keep trying every second
2015-08-31-21:39:49 Gateway is not ready, keep trying every second
2015-08-31-21:39:50 Gateway is not ready, keep trying every second
2015-08-31-21:39:51 Gateway is not ready, keep trying every second
2015-08-31-21:39:52 Gateway found, continue to work
2015-08-31-21:39:53 Changing chnroute ......
2015-08-31-21:40:06 Changing chnroute done
2015-08-31-21:40:06 Current Gateway is 172.20.10.1
2015-08-31-21:40:43 Gateway is not ready, keep trying every second
2015-08-31-21:40:44 Gateway found, continue to work
2015-08-31-21:40:45 Changing chnroute ......
2015-08-31-21:40:57 Changing chnroute done
2015-08-31-21:40:57 Current Gateway is 192.168.12.1
2015-08-31-21:43:00 Gateway is not ready, keep trying every second
2015-08-31-21:43:02 Gateway is not ready, keep trying every second
2015-08-31-21:43:03 Gateway is not ready, keep trying every second
2015-08-31-21:43:04 Gateway is not ready, keep trying every second
2015-08-31-21:43:05 Gateway found, continue to work
2015-08-31-21:43:05 Changing chnroute ......
2015-08-31-21:43:18 Changing chnroute done
2015-08-31-21:43:18 Current Gateway is 172.20.10.1
2015-08-31-21:44:16 Gateway found, continue to work
2015-08-31-21:44:16 Changing chnroute ......
2015-08-31-21:44:28 Changing chnroute done
2015-08-31-21:44:28 Current Gateway is 192.168.12.1

```


# 如何删除`chnroute`
首先你直接用本程序生成的 `route-delete` 删除是无效的，因为这个操作会被launchd监控到并执行`route-auto`，而`route-auto`发现`chnroute`被删了，就会运行`route-add`给添加回来，正确的删除方法是

1. 先 `launchctl unload -w $HOME/Library/LaunchAgents/org.chnroute.automation.plist`
2. 再执行 `sudo $HOME/bin/route-delete`

# 小技巧

1. 查看当前生效的路由表，运行 `netstat -nr -f inet`
2. 查看当前生效的路由表条目的数量，运行 `netstat -nr -f inet | sed '1,4d' | wc -l | xargs`
