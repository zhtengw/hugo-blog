---
title: "Gentoo配置home-assistant服务"
date: 2019-06-16T20:16:37+08:00
draft: false
toc: true
images:
tags: 
  - gentoo
  - linux
  - server
  - 智能家居
  - NAS
---

Home Assistant 服务配置文件：
`/etc/homeassistant/configuration.yaml`
### AsusWRT 设备列表
确保安装了`dev-python/aioasuswrt`，再将以下内容加入配置文件：
```ini
asuswrt:
  host: 192.168.2.1
  username: admin
  ssh_key: /ssh_aten_key # 对ssh私钥文件的路径有要求，目前发现不能位于 /etc 和 /root
  protocol: ssh
  port: 22
```
### HomeKit 
```shell
# echo "net-dns/avahi mdnsresponder-compat" >> /etc/portage/package.use/server
# emerge -1av net-dns/avahi
```
将以下内容加入配置文件：
```ini
homekit:
```
重启homeassistant服务后会在管理网页的通知栏出现HomeKit配对码，打开iOS的**家庭**配对即可。
如要重新配对，则删除`/etc/homeassistant/.homekit.state`，再重启服务，即会重新显示配对码的通知。

### 小米扫地机器人
确保安装了`dev-python/python-miio`，然后将以下内容加入配置文件：
```ini
vacuum:
  - platform: xiaomi_miio
    host: 192.168.2.181
    token: my-token-code
```
HA目前只能通过token来绑定小米扫地机器人，而获得这个token还得费点功夫[^1]。

### HomeKit 控制小米扫地机器人
控制小米扫地机的HA组件不兼容HA的HomeKit组件[^2]，但可以采用曲线救国的方式实现。HomeKit是支持fan组件的，这里可以将扫地机重映射为一个智能风扇[^3]，风扇档位对应扫地机的清扫强度，便能通过HomeKit简单控制扫地机的开关、调档。将以下内容加入配置文件即可：
```ini
fan:
  - platform: template
    fans:
      xiaomi_fan:
        friendly_name: "Xiaomi Robo"
        value_template: "{%if states('vacuum.xiaomi_vacuum_cleaner') == 'cleaning' %}on{%elif states('vacuum.xiaomi_vacuum_cleaner') == 'paused' %}on{%else %}off{% endif %}"
        speed_template: "{{ state_attr('vacuum.xiaomi_vacuum_cleaner', 'fan_speed') }}"
        turn_on:
          service: vacuum.start
          entity_id: vacuum.xiaomi_vacuum_cleaner
        turn_off:
          service: vacuum.return_to_base
          entity_id: vacuum.xiaomi_vacuum_cleaner
        set_speed:
          service: vacuum.set_fan_speed
          data_template:
            fan_speed: "{{ speed }}"
            entity_id: vacuum.xiaomi_vacuum_cleaner
        speeds:
          - 'off'
          - 'Quiet'
          - 'Balanced'
          - 'Turbo'
          - 'Max'
```

[^1]: [Retrieving the Access Token](https://www.home-assistsant.io/components/vacuum.xiaomi_miio/#retrieving-the-access-token)
[^2]: [Supported Components for HomeKit on HA](https://www.home-assistant.io/components/homekit/#supported-components)
[^3]: [Xiaomi Roborock vacuum as fan in HomeKit](https://community.home-assistant.io/t/xiaomi-roborock-vacuum-as-fan-in-homekit/109236/5)
