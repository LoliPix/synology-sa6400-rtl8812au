## 给SA6400群晖添加RTL8812AU无线网卡驱动
1. 去releases页面下载`ko`结尾的驱动文件上传到群晖某个位置(例如:`/volume1/homes/LoliPix/Backup/NAS/8812au.ko`)(Tips:我这里是rr引导的黑群晖,白群晖跟我有些操作不一样,但是驱动是适用的,如果是白群晖可以去问一个ai让它模仿我下面的操作)
2. 在群晖`控制面板--任务计划`,关闭原有的计划`Wireless`,创建新的计划(`新增--触发的任务--用户定义的脚本`),起一个任务名称,事件选择开机,并且选择root执行.接着来到`任务设置`,把下面的代码添加进`用户定义脚本`里面
```bash
#!/bin/sh

KO="/volume1/homes/LoliPix/Backup/NAS/8812au.ko" # 这里改成自己的驱动地址

# 1. 加载 RTL8812AU 驱动
if ! /usr/sbin/lsmod 2>/dev/null | grep -q '^8812au '; then
    /sbin/insmod "$KO"
    sleep 2
fi

# 2. 等待无线接口出现
for i in 1 2 3 4 5 6 7 8 9 10; do
    if /bin/ls /sys/class/net/eth8* >/dev/null 2>&1; then
        break
    fi
    sleep 1
done

# 3. 启动 RR 的无线网络连接脚本
/usr/bin/wireless_supplicant.sh "*" "你的WIFI名称i" "你的WIFI密码"
```
3. 最后在启动脚本之前,前往`控制面板--网络--常规--高级设置--启用多网关`勾选上,接着启用那个计划,最后重启NAS验证一下.
