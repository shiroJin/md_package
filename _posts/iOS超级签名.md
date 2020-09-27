---
title: iOS ipa超级签名
---

​	由于对外分外的adhoc包只能安装在provisioning profile中已添加的设备。因此当有新设备添加时，需要单独更新provisioning profile，并对包进行一次重签名。针对新设备包更新的问题，对超级签名的方案进行了调研。

#### 方案概述

​	超级签名主要思路是让需要安装包的设备自行触发设备添加和重签名的操作，下图为超级签名的签名流程。<!--more-->

![整体架构](https://raw.githubusercontent.com/shiroJin/image-storage/master/super_sign.jpg)

1. 设备安装描述文件后，会向服务器发送设备的UDID。
2. 服务器收到UDID后，将UDID注册到开发者账号下，添加到描述文件中。
3. 再生成签名用的provisioning profile，给IPA签名。
4. 然后iPA传Server，使用itms-services方式让用户下载。

#### 技术细节

* 使用配置文件获取UDID

苹果公司允许开发者通过IOS设备和Web服务器之间的某个操作，来获得IOS设备的UDID。这里的一个概述：

1. 在你的Web服务器上创建一个.mobileconfig的XML格式的描述文件；
2. 用户在所有操作之前必须通过某个点击操作完成.mobileconfig描述文件的安装；
3. 在.mobileconfig描述文件中配置好，以及服务器接收数据的URL地址；
4. 当用户设备安装描述文件后，设备会回调你设置的URL，如果你的URL返回302跳转的话，Safari浏览器会跳转到你所给的地址；

配置描述文件模板如下（iOS 10之后，接收请求的服务器必须要支持https，否则无法正常请求）：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
    <dict>
        <key>PayloadContent</key>
        <dict>
            <key>URL</key>
            <string>http://xxxxx</string> <!--接收数据的接口地址-->
            <key>DeviceAttributes</key>
            <array>
                <string>UDID</string>
            </array>
        </dict>
        <key>PayloadOrganization</key>
        <string>dev.xxx.org</string>  <!--组织名称-->
        <key>PayloadDisplayName</key>
        <string>查询设备UDID</string>  <!--安装时显示的标题-->
        <key>PayloadVersion</key>
        <integer>1</integer>
        <key>PayloadUUID</key>
        <string>3C4DC7D2-E475-3375-489C-0BB8D737A653</string>  <!--自己随机填写的唯一字符串，http://www.guidgen.com/ 可以生成-->
        <key>PayloadIdentifier</key>
        <string>dev.xxxx.profile-service</string>
        <key>PayloadDescription</key>
        <string>本文件仅用来获取设备ID</string>   <!--描述-->
        <key>PayloadType</key>
        <string>Profile Service</string>
    </dict>
</plist>
```

* 自动化操作开发者账号

​	fastlane团队的Spaceship公开了Apple Developer Center的API，而且执行速度比解析开发者Web页面快两个数量级，从而在非常短的时间内搞定`Provisioning Profile`。因为spaceship是ruby脚本，所以在更新provisioning profile和签名的操作上采用ruby语言写的脚本。

更新provisioning file流程如下：

1. 登录developer网站（因为双重验证的原因，第一次登录的时候需要额外输入一次验证码。验证有效期为一个月。一个月后需要重新验证一下）。
2. 在账号下添加新设备的UDID
3. 根据team name和provisioning file name找到对应的provisioning profile，并添加新的设备
4. 下载更新后provisioning profile

* 签名

​	fastlane的签名脚本涵盖了绝大部分的签名场景，在这里使用fastlane的sigh进行签名。注意sigh的api比较简单，没有针对插件进行签名。对于包含插件的ipa文件需要自行调用sigh的resign.sh脚本的命令进行签名。

---

#### 更新及签名脚本：

```ruby
require 'spaceship'
require 'sigh'
require 'Open3'
require 'pathname'

def resign(ipa, identity, provisioning_options, output)
  resign_path = find_resign_path
  args = [
    resign_path,
    ipa,
    identity
  ]

  provisioning_options.each { |bundle_id, provisioning_profile|
    args << "-p #{bundle_id}=#{provisioning_profile}"
  }

  output_path = output || ipa
  args << output_path

  command = args.join(' ')
  puts "#{command}"
  Open3.popen3(command) { |i, o, e, t|
    o.each { |line| print line }
    e.each { |line| print line }
  }
end

def find_resign_path
  root = Pathname.new(File.dirname(__FILE__)).realpath
  File.join(root, 'assets', 'resign.sh')
end

HOME = ENV["HOME"]
TEAM_ID = ""
TEAM_NAME = ""
PROFILE_NAME = ""
DEVICE_UDID = ""
DEVICE_NAME = ""
IDENTITY = ""
BUNDLE_ID = ""
IPA_RESOURCE_PATH = "path/to/ipa"
IPA_OUTPUT_PATH = "path/to/output/ipa"

Spaceship::Portal.login("account", "password")
Spaceship::Portal.select_team(team_id: TEAM_ID)

device = Spaceship::Portal.device.find_by_udid(DEVICE_UDID, include_disabled: true)
if not device
  puts 'create new device'
  device = Spaceship::Portal.device.create!(name: DEVICE_NAME, udid: DEVICE_UDID)
elsif device.disabled?
  puts 'enable device'
  device.enable!
end
puts device.name + device.udid

profile = Spaceship::Portal.provisioning_profile.ad_hoc.all.find{ |item| item.name == PROFILE_NAME }
existing = profile.devices.find { |item| item.udid == device.udid }
if not existing
  puts "add device"
  profile.devices << device
  profile = profile.update!
end
provisioning_profile = "#{HOME}/Library/MobileDevice/Provisioning Profiles/#{profile.uuid}.mobileprovision"
if profile.valid? and !File.exist?(provisioning_profile)
  File.write(provisioning_profile, profile.download)
else
  exit 1
end

## resign ipa
provisioning_options = {
    BUNDLE_ID => provisioning_profile
}
resign(IPA_RESOURCE_PATH, IDENTITY, provisioning_options, IPA_OUTPUT_PATH)
```

#### 总结

​	fastlane的自动化工具非常的给力，最难攻克的点fastlane都一一有相应的工具脚本。超级签名的方案更倾向于企业包的分发场景使用，对于APP Store包，apple的testflight已经非常完善了，也提供了公测的功能。