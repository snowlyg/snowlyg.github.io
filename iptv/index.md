# IPTV电信电视频道列表的抓取教程


<!--more-->
### 前景提要

```text
  IPTV 频道的抓包教程其实有很多，但是在看过这个多的教程之后还是踩了一个很无语的坑。真实诠释了：听过了这么多道理，还是过不好这一生。
  因为引用的教程都有很详细的配图，我的教程就不配图了。偷下懒，见谅一下。
```

### 相关教程

- https://www.cnblogs.com/leokale-zz/p/13272694.html
- https://www.right.com.cn/forum/thread-308436-1-1.html
- https://www.right.com.cn/forum/thread-4389926-1-2.html
- https://mozz.ie/posts/extracting-iptv-live-streams
- https://wp.gxnas.com/1189.html
- https://guihet.com/iptvstudy-diyzbsq.html

### 刷机相关工具

- 红米`AC2100` 路由器（刷OPENWRT系统），可以在 `PC` 上通过 `SSH` 工具连接路由器。
- 电信I`PTV`盒子+遥控器（没有可以使用有红外功能的手机替代）
- 显示器一台连接盒子播放电视
- `pc` 一台 （`win11` 系统）
- 电信`IPTV`专线一条

### 抓包流程

1. 专线连接盒子进入设置界面，获取`IPTV`的账号和密码。这一步很简单，对我来说却挺难得，因为我没有遥控器。因为电视盒子是公司的，上一位同事离职交接的时候没有交接遥控器。然后我的手机有没有红外，最后是借用其他同事的手机安装了万能遥控才进入设置界面。
2. 拿到账号和密码以后，我们切换盒子的联网模式到`DHCP`自动获取模式。这一步非常重要，我就是这一步没有设置导致后面一直在走弯路。因为如果是在盒子内拨号的话，那么不管你是使用路由器，还是使用抓包神器，甚至是交换机的端口镜像功能都是没有办法抓到频道的。
3. 登录路由器后台，设置`lan1`（接入`IPTV`专线）,`lan2`（用网线接入到`iptv`盒子）桥接(名称为 `iptv`，后面会用到)，`lan3` 连接 `pc` (用于`SSH`登录)；并在次桥接网卡上`PPPOE`拨号`iptv`(使用前面获取到的账号密码)。打开盒子查看电视频道，如果现在能够正常观看电视，那么你离抓取到电视频道信息只差最后一步。
4. 在 `PC` 上通过 `SSH` 登录路由器，在路由器上安装 tcpdump 工具，其实就是 `wireshark` 一样的工具，只是是命令行版本。
5. 先关闭电视盒子，然后执行 `tcpdump -i iptv -w /tmp/iptv.cap` ,然后打开电视盒子，选择直播列表随意播放一个频道。发现命令行界面出现数据之后，`ctrl+c` 取消 `tcpdump` 命令。
6. 现在 `iptv.cap` 文件到 `PC` 上,使用 `wireshark` 打开，并导出 `HTTP` 对象。找到 `getchannellistHWCTC.jsp` 选择将它保存成文件。
7. 打开 `getchannellistHWCTC.jsp` 文件，文件头部添加代码如下（并注释 `goServicesEntry` 方法）：

```js
  <script>
  const goServicesEntry = function () { };
  var Authentication = {};
  var m3u8Channels = "";
  Authentication['CTCSetConfig'] = function () {
    const regex = /,?(.+?)="(.*?)"/gm;
    if (arguments[0] == 'Channel') {
      var info = {};
      while ((m = regex.exec(arguments[1])) !== null) {
        if (m.index === regex.lastIndex) {
          regex.lastIndex++;
        }
        if (m[1] == "ChannelName") {
          m3u8Channels += "#EXTINF:-1 tvg-name=\"" + m[2] + "\"," + m[2] + "\n"
        }
        if (m[1] == "ChannelURL") {
          const channelURL = m[2].split("|");
          if (channelURL.length == 2) {
            m3u8Channels += channelURL[1] + "\n"
          }
        }

      }
    }
  }
  setTimeout(function () {
    var a = window.document.createElement('a');
    a.href = window.URL.createObjectURL(new Blob([m3u8Channels], { type: 'application/x-mpegURL' }));
    a.download = 'IPTV.m3u8';
    document.body.appendChild(a);
    a.click();
    document.body.removeChild(a);
  }, 1000);
</script>
```

8. 修改 `getchannellistHWCTC.jsp` 文件名为 `getchannellistHWCTC.html` 并在浏览器打开，就可以下载得到后缀 `.m3u8` 的源文件了。

### 路由刷机教程需要根据需求自行搜索，一下附上红米AC2100的刷机教程

- https://supes.top/%E7%BA%A2%E7%B1%B3-%E5%B0%8F%E7%B1%B3-ac2100-%E5%88%B7breed%E5%92%8Copenwrt%E6%95%99%E7%A8%8B/
- https://www.right.com.cn/forum/thread-4023907-1-1.html
- https://www.right.com.cn/forum/forum.php?mod=viewthread&tid=8234820&extra=page%3D1%26filter%3Dtypeid%26typeid%3D43&page=1

