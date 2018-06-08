# Setup_WIFIAP
#### 讓樹莓派成為一個無線網路AP分享機

這裡介紹如何將手邊的樹莓派3B設定成一個無線網路AP分享器。讓其他裝置能夠透過數枚怕來上網。
主要原理可以分成三層，第一層，我們必須先將樹梅派連上有線網路，並固定住自己的IP。第二層我們將有線網路的封包轉發給無線網路，
這邊我們必須先設定好無線網路IP以及DHCP伺服器來分配使用者IP。最後一層我們需要外部的HOSTAPD的服務套件，
透過這個套件才能將無線網路卡真正變成一個AP分享器。

其實這個方法網路上也有非常多的文章在講述，不過我自己在嘗試的時候常遇到  

1.hostapd無法啟用無線網卡driver  
2.發出去沒有IP  
3.有IP卻無法連上網路 

因為多數文章細節的版本實在太多了，我想說統整一個比較完整且詳細的版本。

## 1.網路設定

首先，如果你拿到一張樹梅派並且開始透過SSH開始連線了，
為了讓每次連線不用重找位址，你應該也已經會設定固定乙太網路IP，
如果沒有，建議先設定 `eth0`
(個人習慣用nano改文檔，習慣用vim的人請自行代換)

    sudo nano /etc/dhcpcd.conf

請在最下方加入

    interface eth0
    static ip_adress=192.168.137.100 #137改成你自己乙太網域的名稱
    static routers=192.168.137.1
    static domain_name_servers=8.8.8.8
    static domain_search=8.8.4.4

以上儲存後，下次再用ssh連結就不用再找一次位址，固定位址就是`ip_adress`

***
以上寫法我後來發現，如果把乙太網路差到網路孔上，反而會無法上網，可能原因是我們把IP及routers固定了，如果換成其他網域可能轉發會出問題。
所以其實不要去更改反而會比較好。
***

接下來我們要先去設定無線網路`wlan0`的部分

     sudo nano /etc/netwoek/interfaces

找到有關`wlan0`的那行，應該會找這樣

    allow-hotplug wlan0
    iface wlan0 inet manual
    wpa-conf /etc/wpa_..... #他會自己去找設定

把他們全槓掉(或是前頭加上#你懂得)
加上新的設定

    auto wlan0
    iface wlan0 inet static #w同樣是靜態 
     address 192.168.2.1 #2是我自己設的網域，你可以自己選。但請把她記起來，之後要用到。
     netmask 255.255.255.0
     network 192.168.2.0 #這裡的2也是自選，但務必跟address一樣，基本上我後面都是用2代表，就不多述
     broadcast 192.168.2.255
 
完成後存檔。

接著我們要去設定一個分配伺服器DHCP，這樣使用者才能取得IP來上網


## 2. DHCP設定

這部分分成兩種方法，其實個人推薦使用第二種，不過兩者可以交互嘗試。

### 2.1 isc-dhcp-sever
先在樹莓派安裝DHCP伺服器

    sudo apt-get install isc-dhcp-server
 
安裝好後先去編輯設定檔

    sudo nano /etc/dhcp/dhcpd.conf

將以下全槓掉

    #option definitions common to all supported networks...
    #option domain-name "example.org";
    #option domain-name-servers ns1.example.org, ns2.example.org;

並在任一定點，設定此為DHCP官方伺服器

    authoritative;

以及設定要分配的網段，這邊的網段就跟剛剛我們設定的無線網路卡相同

    subnet 192.168.2.0 netmark 255.255.255.0{
      range 192.168.2.10 192.168.2.200;
      option broadcast-address 192.168.2.255;
      option routers 192.168.2.1;
      default-lease-time 600;
      max-lease-time 7200;
      option domain-name "local"
    }

完成後存檔。
接著我們要將此服務的`interface`設定指到`wlan0`
所以先進到`/etc/default/isc-dhcp-server`找到特定行，進行編輯

    #On what interfaces should the DHCP server (dhcpd) serve DHCP requests?
    #Separate multiple interfaces with spaces, e.g. "eth0 eth1".
    INTERFACES="wlan0"

完成後儲存。
現在已經將伺服器設定好了  可以先嘗試啟用，並檢查有沒有錯誤。

    sudo server isc-dhcp-server restart

如果有無法解決的錯誤或者是走到後面還是有問題，可以看看2.2的方法，沒問題的話可繼續到3.轉發設定

### 2.2 DNSMASQ

這個套件本身除了提供DHCP，還有DNS服務，功能與上面類似。
先下載套件

    sudo apt-get install dnsmasq 

一般來說，我們用不太到這個套件全部的功能，而如果你有心也可以研究這一個強大的套件，
不過方便起見，我們直接複製一個範本來做修改。

    sudo mv /etc/dnsmasq.conf /etc/dnsmasq.conf.orig
    sudo nano /etc/dnsmasq.conf

我們將其修改成

    interface=wlan0
    listen-address=192.168.2.1 #這邊是之前的無線網卡IP位指
    resolv-file=/etc/resolv.dnsmasq
    mini-port=49152
    bind-interfaces
    server=8.8.8.8
    cache-size=10000
    domain-needed
    bogus-priv
    dhcp-range=192.168.2.10, 192.168.2.200,255.255.255.0, 12h
    #以下可寫可不寫
    dhcp-option=1,255.255.255.0
    dhcp-option=3,192.168.2.1
    dhcp-option=6,192.168.2.1,8.8.8.8,168.95.192.1
    dhcp-authoritative

完成後存檔。之後複製原本本機的DNS設定、並且修改成`wlan0`的自訂IP

    sudo cp /etc/resolv.conf /etc/resolv.dnsmasq
    sudo sed -i 's/127.0.0.1/192.168.2.1/g' /etc/resolv.dnsmasq

如果深怕沒改到的話，直接進到文檔裡面改也可以，反正裡面也只有一個位址，把它改成你剛剛指定的網卡IP就可以了。

最後重新啟動dnsmasq來測試有沒有錯誤訊息。
通常這邊很容易寫錯字，排除寫錯，如果有些是可不寫的，可以先槓掉在測試看看。

    sudo service dnsmasq restart


## 3.轉發設定

轉發就是我們要把有線的通道和無線的通道連接起來，稱為封包轉發(packet forwarding)

首先進到設定檔 `/etc/sysctl.conf`，找到有`net.ipv4.ip_forward`的特定行後進行些改

    net.ipv4.ip_forward=1

接著設定NAT

    sudo iptables -F
    sudo iptables -F -t nat
    sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
    sudo iptables -A FORWARD -i eth0 -o wlan0 -m state --state RELATED,ESTABLISHED -j ACCEPT
    sudo iptables -A FORWARD -i wlan0 -o eth0 -j ACCEPT

這裡也是很長一串，常會寫錯字，要稍微小心檢查一下。
如果之後想要讓下面的裝置能互通的的話，可以再增加一行

    sudo iptables -A Forward -i wlan0 -o wlan0 -j ACCEPT

接著，因為我們不想每次都要改一次設定，我們把這一串設定先存起來。

    sudo sh -c "iptables-save > /etc/iptables.ipv4.nat"

然後讓每次開機後都會去套用以上兩個設定，我們到` /etc/rc.local`，在`exit 0`之前將入下面兩行

    sudo sh -c "echo 1 > /proc/sys/net/ipv4/ip_forward"
    sudo iptables-restore < /etc/iptables.ipv4.nat

完成後存檔，這樣之後開機就會自動設定nat並開啟轉發功能了。

## 4.Hostapd設定

接下來就是重頭戲了，我們已經設定好無限、有線網卡，也已經做好分配及轉發，
但實際上無線網卡還不是個AP，而hostapd就是負責這個工作

大多數的文章都是說直接下載，但這邊我到是失敗了很多次，後來外國的論壇有提到，
直接去他們開發源找到最新版driver才能完美支援樹莓派3B的無線網卡。

[參考討論串](https://www.raspberrypi.org/forums/viewtopic.php?t=74193)
[參考網址](https://wireless.wiki.kernel.org/en/users/documentation/hostapd?s[]=hostapd)

首先我們先安裝最新版本，下載需要一小段時間。

    git clone git://w1.fi/srv/git/hostap.git
    cd hostap/hostapd

這邊通常會發現你沒有辦法直接`make`，所以`deconfig`

    cp defconfig .config
    sudo nano .config

編輯下列文字，將#字拿掉

    #CONFIG_DRIVER_NL80211=y

這時候應該是可以`make`了，但如果在`make`的過程中出現缺少某些文件，則需要再下載一個套件

    sudo apt-get install libnl-dev
    sudo apt-get install libnl-3-dev

如果怕不夠新，可以在`update`一下。
安裝好後在`make`應該就會成功了。

接下來我們要來製作一個AP的設定檔，

    sudo nano /etc/hostapd/hostapd.conf

開始進行設定:

    interface=wlan0
    ssid=RPI_AP #設邊就是你的AP名稱
    driver=nl80211 #網卡的driver 要住L和1的差別
    hw_mode=g
    channel=6 #這邊的通道可以隨意修改，但不是每台網卡都可以隨意設定，設成1應該是最好保險
    macaddr_acl=0
    ignore_broadcast_ssid=0
    wpa=2
    wpa_passphrase=thepassworf #這就是密碼，可自行更改
    wpa_key_mgmt=WPA-PSK
    wpa_pairwise=TKIP
    rsn_pairwise=CCMP
    wme_enable=1

設定完後存檔
然後要進到`/etc/default/hostpad`修改一下讓這個設定檔變成預設:

    DAEMON_CONF="/etc/hostapd/hostapd.conf"

最後我們要讓這個功能在一開機就生效，

    sudo systemctl enable hostapd

以上就大功告成了
最後測試以前，我們要將所有服務都重啟一次，畢竟都改過一輪了

    sudo service hostapd restart
    sudo service dnsmasq restart #如果你用的是dnsmasq


然後重啟`reboot`之後，你可以用手機或是筆電來連接看看
沒有意外的話就可以輸入你剛剛設定的密碼來上網了!

## 結語

其實過程中我花了快兩天的時間來setup，因為第一天還在摸索整個程序，然後卡在driver一直活不起來的狀態，
然後第二天才到最新的hostapd成功把IP發給我的手機，但是卻一直無法上網，
經過幾次調整轉發設定檔和hostspd.conf之後終於能上網了。

不過這也只是樹莓派3B的程序，基本上跟網路文章八九不離十，不過軟硬體不斷在更新，文章無法更新到的也只能去找一些Q&A
如果我有寫錯、或是用詞錯誤、抑或是我誤解的部分，也歡迎指教。
希望你也能成功設定。

[參考網址一](https://blog.gtwang.org/iot/setup-raspberry-pi-as-wireless-access-point/)
[參考網址二](http://atceiling.blogspot.com/2017/02/raspberry-pi-wireless-access-point.html)
[參考網址三](https://blog.csdn.net/weixin_41656968/article/details/79818033)
[參考網址四](http://blog.itist.tw/2016/03/using-raspberry-pi-3-as-wifi-ap-with-raspbian-jessie.html)

  
