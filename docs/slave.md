# 從時鐘端
## 需求
- 本文以Pi5實作
- OS: RaspberryPi OS Lite
 
# 安裝步驟

## 安裝 chrony

```
sudo apt install chrony -y
```
-----
編輯/etc/chrony/chrony.conf
```
sudo nano /etc/chrony/chrony.conf
```
在最下面加上master的IP address
```
server 192.168.1.226 iburst
```
重啟chrony服務
```
sudo systemctl restart chrony
```
檢查是否同步成功
```
watch chronyc sources -v
```
會發現已經跟master同步時間
```
Every 2.0s: chronyc sources -v                                                                    raspberrypi: Tue Sep 10 13:19:25 2024


  .-- Source mode  '^' = server, '=' = peer, '#' = local clock.
 / .- Source state '*' = current best, '+' = combined, '-' = not combined,
| /             'x' = may be in error, '~' = too variable, '?' = unusable.
||                                                 .- xxxx [ yyyy ] +/- zzzz
||      Reachability register (octal) -.           |  xxxx = adjusted offset,
||      Log2(Polling interval) --.      |          |  yyyy = measured offset,
||                                \     |          |  zzzz = estimated error.
||                                 |    |           \
MS Name/IP address         Stratum Poll Reach LastRx Last sample
===============================================================================
^- 123-204-232-128.adsl.dyn>     2   6    17    16    +30ms[  +30ms] +/-   60ms
^- 2001-b031-5c02-ff00-0000>     2   6    17    16  -2157us[-2164us] +/-   28ms
^- t2.time.tw1.yahoo.com         2   6    17    16   -372us[ -378us] +/- 3689us
^- 114-34-171-136.hinet-ip.>     2   6    17    16  -1003us[-1009us] +/- 4642us
^* 192.168.1.226                 1   6    17    15  +1921ns[-6614ns] +/-  114us
```
-----


server 192.168.1.226
refclock SHM 2 refid PTP precision 1e-7 stratum 1


安裝linuxptp
```
sudo apt install linuxptp -y
```
並把PTP納入參考時時鐘源
需修改chrony.conf
```
sudo nano /etc/chrony/chrony.conf
```
在最下面加上

```
refclock SHM 2 refid PTP precision 1e-7 stratum 1
```
編輯配置文件
```
sudo nano /etc/linuxptp/ptp4l.conf
```
將對應的欄位修改成
```
slaveOnly               1
clock_servo             ntpshm
ntpshm_segment          2
time_stamping           software
summary_interval        10
```
啟動服務
```
sudo systemctl enable ptp4l@eth0
sudo systemctl start ptp4l@eth0
```
讓系統時鐘跟網卡時鐘同步

```
sudo ptp4l -i eth0 -m -H -s
sudo phc2sys -s eth0 -c CLOCK_REALTIME -w
```
重啟chrony服務
```
sudo systemctl restart chrony
```
