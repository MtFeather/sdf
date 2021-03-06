# 虛擬機器的服務、安裝與啟用
[參考資料:http://dic.vbird.tw/network_project/unit03.php](http://dic.vbird.tw/network_project/unit03.php)

## 1. 虛擬機器系統大致上需要什麼服務與軟體
- 基本上，在母系統上面會有一組服務來協助虛擬機器的運行，這組程式通常稱為虛擬機器監督器 (VMM, Virtual Machine Mointor)。 這組程式必需要能夠分配系統資源給 VM，也需要讓虛擬機器的指令可以傳給 CPU 執行，因此是一個相當重要的服務。VMM 主要的工作就是模擬出一個硬體， 並且這組硬體在邏輯上，與其他硬體是獨立存在的。
- 上述的 VMM 根據發展的不同而有多種類型，你可以參考 wiki 查詢 full virtualization 以及 paravirtualization 等， 也可以直接查詢 http://benjr.tw/3383 的內容，理解一下不同的 VMM 的作用。 目前我們使用的大概都是硬體支援虛擬化的功能！VM 的 CPU 指令，都可以透過母機器的 CPU 虛擬化指令集來達成直接傳達的功能！
- 常見的虛擬化術語：
  - Host： 就是虛擬機器所在的那部實體主機
  - VM： 就是虛擬機器硬體的意思
  - Guest OS： 在 VM 上面所安裝的獨立的作業系統
- 經常使用到的軟體：
  - KVM： 整合到 Linux 核心，是最重要的虛擬化技術，可以虛擬出 CPU 的硬體
  - qemu： 相對於 KVM，qemu 則主要在虛擬出各項週邊設備，包括磁碟、網卡、USB、顯卡、音效等
  - libvirtd： 提供使用者一個管理 VM 的服務
  - virt-manager： 有點類似圖形界面，可以搭配 libvirtd 進行虛擬機器的管理。
## 2. 虛擬機器管理員，使用 libvirtd 以及 virsh 指令的管理
- 觀察 libvirtd 是否執行中
```bash
systemctl status libvirtd
```
- 觀察目前的虛擬機器以及虛擬網路環境
```bash
virsh list
virsh net-list
```
- 關閉與取消定義的機器
```bash
virsh [shutdown|destroy|undefine] domain
virsh [net-destroy|net-undefine] netdomain
```
- 你的系統目前會有很奇特的 port，原因之一，就是因為預設的網路系統會啟動一個虛擬橋接器，這個 virbr0 會自動的加入一個 192.168.122.X/24 的網段給你的虛擬機器使用，並且使用的是 NAT 的機制，因此使用這個 virbr0 橋接器連結的系統， 就可以自動的透過你的 host 上網了。你可以先去底下的檔案內去瞧一瞧網路設定值
```bash
vim /etc/libvirt/qemu/networks/default.xml
ip addr show
```
- 你可以查詢、關閉與取消網路橋接器的定義！透過的方法如上所示
```bash
virsh net-list
virsh net-destroy default
virsh net-list --all
virsh net-start default
virsh net-destroy default
# virsn net-undefine default (這個指令暫時不要進行！)
```
你的系統可能由於近期內有升級過，或者是其他函式庫有更新，可能會導致你的 libvirtd 有點奇怪，有時候會出現如下的訊息
```bash
# 1. 指令操作時，出現如下的奇怪錯誤！明明沒啥問題！
virsh net-start default
錯誤：無法開啟網路 default
錯誤：The name org.fedoraproject.FirewallD1 was not provided by any .service files

# 2. 查看 messages 時，出現如下奇怪的錯誤：
Mar 11 20:11:41 120-114-142-27 kernel: virbr0: port 1(virbr0-nic) entered disabled state
Mar 11 20:11:41 120-114-142-27 libvirtd: 2019-03-11 12:11:41.956+0000: 4955: error : virNetDevSendEthtoolIoctl:3072 : ethtool ioctl error: 沒有此一裝置
Mar 11 20:11:41 120-114-142-27 NetworkManager[4186]:   [1552306301.9576] device (virbr0-nic): released from master device virbr0
Mar 11 20:11:41 120-114-142-27 libvirtd: 2019-03-11 12:11:41.958+0000: 4955: error : virNetDevSendEthtoolIoctl:3072 : ethtool ioctl error: 沒有此一裝置
Mar 11 20:11:41 120-114-142-27 libvirtd: 2019-03-11 12:11:41.960+0000: 4955: error : virNetDevSendEthtoolIoctl:3072 : ethtool ioctl error: 沒有此一裝置
```
處理的方法其實很簡單，完整的重新開機是一個方式，另一個則是透過重新啟動 libvirtd 來處理即可
```bash
systemctl restart libvirtd
```
- 完成底下的實做
a. 將 /etc/libvirt/qemu/networks/default.xml 備份到 /root/virtual/ 目錄內
```bash
mkdir ~/virtual
cd /root/virtual
cp /etc/libvirt/qemu/networks/default.xml /root/virtual
```
b. 列出目前所有的虛擬網路橋接器
```bash
virsh net-list --all
```
## 3.  虛擬網路橋接器的設計
- 我們的系統要連接上網路，可以透過 NAT 也可以透過將網卡直接接在小烏龜上面來撥接， 因此，可以是透過 bridge 也可以透過 NAT 的意思。
- 虛擬橋接器 (virtual bridge) 的界面設計，除了可以透過實體的網卡模擬橋接器之外， 事實上，qemu 也提供了兩種基本的橋接器給我們使用的：
  - 透過 NAT，例如剛剛的 default 網路界面
  - 透過直接 forward 到外部實體網卡上！就是直接 bridge 功能！
- 透過 NAT 的方式處理，例如底下這樣的設計
```bash
vim qnet.xml
<network>
  <name>qnet</name>
  <forward mode='nat'/>
  <bridge name='virbr1' stp='on' delay='0'/>
  <mac address='52:54:00:66:ff:0c'/>
  <ip address='192.168.10.254' netmask='255.255.255.0'>
    <dhcp>
      <range start='192.168.10.1' end='192.168.10.100'/>
    </dhcp>
  </ip>
</network>

virsh net-create qnet.xml
virsh net-list
```
這裡要說明一下，兩種啟動的方式:
```bash
virsh net-start  netdomain		# 這個網路界面已經被記載到 libvirtd 的管理中
virsh net-create netdomain.xml		# 這個網路界面沒有被記載，但是，是實際存在的一個 xml 檔案格式
```
- 透過介接到實體網卡來處理的 forward 模式
```bash
vim qforward.xml
<network>
  <name>qforward</name>
  <forward dev='eno1' mode='bridge'>
    <interface dev='eno1'/>
  </forward>
</network>

virsh net-create qforward.xml
virsh net-list
```
但是由於 forward 模式直接取用實體網卡來作為橋接，因此，使用 ip addr show 的時候，並不需要額外的橋接界面， 所以你只會看到我們剛剛設計的 virbr1 那張界面橋接器而已。
## 4. 設計虛擬磁碟
- 虛擬磁碟可以是實體磁碟、可以是檔案、可以是 LVM 裝置等等
- 虛擬磁碟可以使用 qemu-img 來建置
- 常見的虛擬磁碟格式，主要為 qcow2 與 raw ，其餘不要考慮
- raw 格式最快，但是得先預留出磁碟容量，因此不建議
- qcow2 還在持續發展中，速度與 raw 已經差不多，而且檔案系統用多少，算多少。
- 觀察與建立的方法
```bash
qemu-img --help
qemu-img create -f qcow2 -o cluster_size=[512,1K,..2M] /vmdisk/your_image_filename.img sizeG 
qemu-img info yourfile.img
```
請實做一個 40G 的虛擬磁碟，檔名就設定為 /vmdisk/centos7.raw.img
```bash
qemu-img create -f qcow2 -o cluster_size=1M /vmdisk/centos7.raw.img 40G
```
## 5. 由系統預設建立 XML 檔案資訊
- 安裝 virt-install 軟體，使用 virt-install 去建立一個 XML 檔案之後，再來修改
```bash
yum install virt-install
man virt-install
--dry-run
--print-xml
virt-install \
	--name demo1 \
	--cpu host --vcpus 4 --memory 2048 --memballoon virtio \
	--clock offset=localtime \
	--controller virtio-scsi \
	--disk /vmdisk/centos7.raw.img,cache=writeback,io=threads,device=disk,bus=virtio \
	--network network=qnet,model=virtio \
	--graphics spice,port=5911,listen=0.0.0.0,password=centos7 \
	--cdrom /vmdisk/iso/CentOS-7-x86_64-DVD-1810.iso \
	--video qxl \
	--dry-run --print-xml \
```
還有其他參數，請自行前往查詢
- 更多的 XML 參數，可以參考： https://libvirt.org/formatdomain.html
- 修改 xml 檔案，讓該檔案的內容符合你的環境規劃！
- 使用 xml 檔案啟動虛擬機，透過虛擬機的內容，開始進行 CentOS 虛擬機器的安裝流程！
(務必使用最小安裝！)
- 使用 virsh list 查閱虛擬機器的狀態
- 搭配你的 VM spice port，可能需要開啟防火牆設置！
- 注意，取得虛擬機器的終端界面，務必要小心在意！尤其是取得時，最好輸入帳號與密碼才好喔！

# 虛擬機器的操作
## 1. 一開始，只好使用 spice 或 VNC 的連線狀態：
- 因為一開始你的 VM 並沒有系統的關係！
- 透過 remote-viewer 的支援，直接從遠端抓 spice 連線過來操作！
- 記得，還是需要設定密碼啦！而且最好不要有圖形界面為宜！
- 還是需要設定好你虛擬機的 IP 喔！
- 最重要的就是設定好虛擬機器的防火牆，以及設定好 iptables 的 DNAT 轉埠口功能，方便你未來直接操作。
## 2. 經常性的操作：
- 全系統升級
- 安裝 net-tools、 bash-completion 等常用軟體
- 安裝 iptables-servers ，處理防火牆連線事宜
- 取消不要的埠口服務，整理好所需要的環境
