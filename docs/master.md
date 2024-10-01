# 主時鐘端
## 需求
- 本文以Pi5實作
- OS RaspberryPi OS Lite
- GPS NEO-M8N + GPS天線 SMA介面有源天線 

# 安裝步驟

## 腳位對接及設定

| 樹梅派腳位       | GPS腳位 |
| ---------------- | ------- |
| GPIO18           | PPS     |
| GPIO14(UART_TXD) | RX      |
| GPIO15(UART_RXD) | TX      |
| GND              | GND     |
| 5V               | VCC     |
---
### 啟用UART0
### 設定 GPIO18 為 PPS 腳位


```
sudo nano /boot/firmware/config.txt
```
在最底下加上
```
dtparam=uart0=on
dtoverlay=pps-gpio,gpiopin=18
```
重啟
```
sudo reboot
```
重啟後測試看看
```
sudo cat /dev/ttyAMA0
```
如果有出現下面之類的訊息代表成功
```
$GNVTG,,T,,M,0.162,N,0.300,K,D*3E

$GNGGA,052224.00,2459.10707,N,12120.49902,E,2,05,3.56,210.4,M,16.7,M,,0000*4D

$GNGSA,A,3,12,05,,,,,,,,,,,5.39,3.56,4.04*15

$GNGSA,A,3,75,71,74,,,,,,,,,,5.39,3.56,4.04*14

$GPGSV,3,1,10,05,19,119,30,10,16,318,,12,21,140,38,13,05,058,23*74

$GPGSV,3,2,10,15,35,050,19,18,57,228,,23,50,338,,24,72,040,*7E

$GPGSV,3,3,10,32,13,272,,50,60,167,38*74

$GLGSV,3,1,09,65,18,309,,71,19,065,35,72,39,013,,73,15,036,*6D

$GLGSV,3,2,09,74,45,073,25,75,38,152,40,86,14,193,,87,38,241,*64

```


- 安裝PPS-tools

```
sudo apt install pps-tools -y
```
安裝後測試
```
sudo ppstest /dev/pps0
```
如果有出現
```
source 0 - assert 1725859406.005707255, sequence: 97 - clear  0.000000000, sequence: 0
source 0 - assert 1725859407.005745853, sequence: 98 - clear  0.000000000, sequence: 0
source 0 - assert 1725859408.005788363, sequence: 99 - clear  0.000000000, sequence: 0
source 0 - assert 1725859409.005826316, sequence: 100 - clear  0.000000000, sequence: 0
```
代表安裝成功

## 安裝 GPSd

```
sudo apt install gpsd gpsd-clients -y
```

安裝後修改 /etc/default/gpsd

```
sudo nano /etc/default/gpsd
```
將對應的欄位修改成
```
DEVICES="/dev/ttyAMA0 /dev/pps0"
GPSD_OPTIONS="-n"
USBAUTO="false"
```
啟動gpsd並加入開機自動啟動
```
sudo systemctl enable gpsd
sudo systemctl start gpsd
```

測試看看
```
cgps
```

若出現下列畫面代表正常運作中
```
lqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqklqqqqqqqqqqqqqqqqqqSeen 25/Used  8k
x Time:        2024-09-09T05:29:22.000Z (18)xxGNSS   PRN  Elev   Azim   SNR Usex
x Latitude:         24.98501960 N           xxGP  5    5  17.0  122.0  22.0  Y x
x Longitude:       121.34164570 E           xxGP 12   12  23.0  138.0  32.0  Y x
x Alt (HAE, MSL):    235.137,    218.434 m  xxGP 25   25  13.0  173.0  30.0  Y x
x Speed:             0.39 km/h              xxGL  7   71  18.0   69.0  25.0  Y x
x Track (true, var):   338.7,  -4.6     deg xxGL 10   74  44.0   68.0  16.0  Y x
x Climb:             0.00 m/min             xxGL 11   75  42.0  150.0  40.0  Y x
x Status:         3D DGPS FIX (9 secs)      xxQZ  2  194  26.0  142.0  34.0  Y x
x Long Err  (XDOP, EPX):  2.08, +/-  7.8 m  xxQZ  3  195  37.0  173.0  37.0  Y x
x Lat Err   (YDOP, EPY):  1.25, +/-  4.7 m  xxGP 10   10  18.0  319.0   0.0  N x
x Alt Err   (VDOP, EPV):  2.41, +/- 17.8 m  xxGP 13   13   4.0   61.0  23.0  N x
x 2D Err    (HDOP, CEP):  2.43, +/- 16.9 m  xxGP 15   15  33.0   53.0  25.0  N x
x 3D Err    (PDOP, SEP):  3.42, +/- 17.6 m  xxGP 18   18  55.0  223.0   0.0  N x
x Time Err  (TDOP):       2.35              xxGP 23   23  52.0  340.0   0.0  N x
x Geo Err   (GDOP):       4.14              xxGP 24   24  68.0   37.0   0.0  N x
x Speed Err (EPS):       +/-  2.2 km/h      xxGP 32   32  14.0  275.0   0.0  N x
x Track Err (EPD):        n/a               xxSB128   41  39.0  242.0   0.0  N x
x Time offset:            2.008669352 s     xxSB129   42  54.0  141.0   0.0  N x
x Grid Square:            PL04qx06          xxSB137   50  60.0  167.0  37.0  N x
```

## 安裝Chrony

安裝Chrony可以讓伺服器自動校時
```
sudo apt install chrony -y
```
安裝後修改 /etc/chrony/chrony.conf
```
sudo nano /etc/chrony/chrony.conf
```
在文件最下面加上
```
# serve time to the network
allow

# refclock from gpsd
refclock SOCK /run/chrony.ttyAMA0.sock refid GPS precision 1e-1 offset 0.9999 delay 0.2
refclock SOCK /run/chrony.pps0.sock refid PPS precision 1e-7

# sync the real-time clock
rtcsync
makestep 1.0 -1
```
重啟chrony 並重新啟動gpsd 沒有重啟gpsd 等等的GPS 以及 PPS 數值會呈現 0
```
sudo systemctl restart chrony
sudo systemctl restart gpsd
```
確認chrony服務是否正常運作
```
watch chronyc sources -v
```
稍等個幾分鐘後chrony就會自己切成與PPS同步 如果沒有的話就重開機 重開機治百病
```
Every 2.0s: chronyc sources -v                                                                    raspberrypi: Mon Sep  9 14:06:18 2024


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
#x GPS                           0   4   377    12  -1002ms[-1002ms] +/-  200ms
#* PPS                           0   4   377    12  -2252us[-2418us] +/- 8936us
^+ tw.ntp.twds.com.tw            2   6    77    42   -317us[ +803us] +/- 7421us
^+ t1.time.tw1.yahoo.com         2   6    77    42  +1955us[+3076us] +/- 8066us
^- ntp.tipsy.coffee              2   6    77    43   +625us[+1745us] +/-   19ms
^+ t2.time.tw1.yahoo.com         2   6    77    43   +181us[+1301us] +/- 6125us

```
## 安裝PTP Server

```
sudo apt install linuxptp -y
```

編輯/etc/linuxptp/ptp4l.conf
```
sudo nano /etc/linuxptp/ptp4l.conf
```
將對應的欄位修改成
```
priority1               127
masterOnly              1
```
啟動PTP服務
```
systemctl enable ptp4l@eth0
systemctl start ptp4l@eth0
```
建立系統服務
```
cd /lib/systemd/system
sudo cp phc2sys\@.service sys2phc-eth0.service
```
編輯sys2phc-eth0.service
```
sudo nano sys2phc-eth0.service
```
將對應的欄位修改成
```
Requires=ptp4l@eth0.service
After=ptp4l@eth0.service
```
啟動phc2sys (這個步驟好像需要他一直在背景執行 所以我把他丟到背景去了)
```
sudo  nohup phc2sys -w -c eth0 -s CLOCK_REALTIME >/dev/null 2>&1 &
```
啟動服務
```
systemctl daemon-reload
systemctl enable sys2phc-eth0
systemctl start sys2phc-eth0
```