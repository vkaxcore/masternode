# Setting Up a VKAX Masternode

This guide provides a comprehensive, step-by-step process to set up a VKAX masternode using a VPS. It includes all necessary commands and explanations to help even first-time users succeed. 

---

## **Overview**

You’ll need:

1. **10,000,000 VKAX** as collateral.
2. A **VKAX Core Wallet** for managing your funds and setting up the masternode.
3. A **Virtual Private Server (VPS)**, such as one from Oracle Cloud (free tier) or Vultr.

> **Tip:** If hosting from home, ensure your router allows port forwarding and you have a static public IP address. VPS hosting is recommended for stability.

---

## **Step 1: Set Up the Local Wallet**

### 1.1 Install the VKAX Wallet

1. Download the VKAX Core Wallet from [VKAX Releases](https://github.com/vkaxcore/VKAX/releases).
2. Install the wallet and let it fully sync with the blockchain. This may take hours depending on your connection.

### 1.2 Create the Collateral Address

1. Go to **Tools > Debug Console** in the wallet and generate a new address:
   ```bash
   getnewaddress
   ```
2. Save the generated address (e.g., `XmdhxVWeyNEc9pjeNW9reTwcDMCwGLi2H`). This is your **collateral address**.

3. Send exactly **10,000,000 VKAX** to the collateral address and wait for at least **1 confirmation**.

---

### 1.3 Prepare the `protx.txt` File

Create a text file (`protx.txt`) on your computer with the following structure:
```bash
protx register <collateral-hash> <collateral-index> <server_ip>:11110 <owner> <bls-public-key> <voting> 0 <payout> <some-change-address>
```

#### 1.4 Generate Three Addresses

In the wallet, run these commands:
```bash
getnewaddress "ownerKeyAddr"
getnewaddress "votingKeyAddr"
getnewaddress "payoutAddress"
```

Update `protx.txt`:
- `<owner>`: Replace with `ownerKeyAddr`.
- `<voting>`: Replace with `votingKeyAddr`.
- `<payout>`: Replace with `payoutAddress`.

Save `protx.txt`. You will add more details later.

---

## **Step 2: Prepare the VPS**

### 2.1 Set Up the VPS Instance

1. **Choose a VPS provider:** Oracle Cloud offers free-tier instances, and Vultr provides reliable and affordable VPS.
2. **Install Ubuntu 22.04** or **20.04** for compatibility.
3. **Connect to the VPS:** Use SSH:
   ```bash
   ssh root@<server_ip>
   ```
   Replace `<server_ip>` with your VPS IP.

4. Set a strong root password:
   ```bash
   passwd root
   ```

---

### 2.2 Install Dependencies

1. Update and upgrade the VPS:
   ```bash
   sudo apt-get update -y && sudo apt-get upgrade -y
   ```
2. Install required tools:
   ```bash
   sudo apt-get install curl build-essential libtool autotools-dev automake pkg-config python3 bsdmainutils bison libssl-dev libevent-dev libboost-system-dev libboost-filesystem-dev libboost-chrono-dev libboost-program-options-dev libboost-test-dev libboost-thread-dev g++-aarch64-linux-gnu g++-mingw-w64-x86-64 mingw-w64-x86-64-dev gperf zip git curl ufw -y
   ```

---

### 2.3 Open the Firewall

1. Allow necessary ports, including port **11110** for VKAX communication:
   ```bash
   sudo ufw allow 22/tcp
   sudo ufw allow 11110/tcp
   sudo ufw enable
   ```

---

### 2.4 Create the `vkax` User

1. Create a non-sudo user:
   ```bash
   adduser vkax
   ```
2. Assign a password but **do not grant sudo privileges**.

---

## **Step 3: Build the VKAX Daemon**

1. Clone the VKAX repository:
   ```bash
   git clone https://github.com/vkaxcore/VKAX.git
   cd VKAX
   ```
2. Build dependencies:
   ```bash
   cd depends
   chmod +x conf*
   make
   cd ..
   ```
3. Generate configuration files:
   ```bash
   ./autogen.sh
   ```
4. Configure and compile the daemon:
   ```bash
   ./configure --prefix=$PWD/vkax-build/ --disable-wallet --without-gui
   make
   ```
5. Install the binaries:
   ```bash
   mkdir -p /home/vkax/.vkaxcore
   cp vkax-build/vkaxd vkax-build/vkax-cli /home/vkax/.vkaxcore/
   ```

---

## **Step 4: Configure the VKAX Node**

1. Switch to the `vkax` user:
   ```bash
   su - vkax
   ```
2. Create the `vkax.conf` file:
   ```bash
   nano ~/.vkaxcore/vkax.conf
   ```
   Add:
   ```
   rpcuser=<rpcuser>
   rpcpassword=<rpcpassword>
   rpcallowip=127.0.0.1
   listen=1
   server=1
   daemon=1
   masternodeblsprivkey=<masternodeblsprivkey>
   externalip=<server_ip>
   ```
   Replace placeholders with your details.

3. Start the daemon:
   ```bash
   ~/.vkaxcore/vkaxd
   ```

4. Check synchronization:
   ```bash
   ~/.vkaxcore/vkax-cli mnsync status
   ```

---

## **Step 5: Complete and Submit `protx.txt`**

### 5.1 Add Final Details to `protx.txt`

1. Get your collateral transaction details:
   ```bash
   masternode outputs
   ```
2. Add:
   - `<collateral-hash>`: From `masternode outputs`.
   - `<collateral-index>`: From `masternode outputs`.
   - `<server_ip>`: Your VPS IP.

---

### 5.2 Register the Masternode

1. Open the wallet console on Windows.
2. Submit the ProTx:
   ```bash
   protx register <contents of protx.txt>
   ```
3. Wait for **1 confirmation** to see the masternode in the list.

---

## **Step 6: Create a Systemd Service**

1. Exit the `vkax` user:
   ```bash
   exit
   ```

2. Create a service file:
   ```bash
   sudo nano /etc/systemd/system/vkax.service
   ```

3. Add:
   ```
   [Unit]
   Description=VKAX Masternode Service
   After=network.target

   [Service]
   Type=forking
   User=vkax
   WorkingDirectory=/home/vkax/.vkaxcore/
   ExecStart=/home/vkax/.vkaxcore/vkaxd
   ExecStop=/home/vkax/.vkaxcore/vkax-cli stop
   Restart=on-failure
   RestartSec=10s

   [Install]
   WantedBy=multi-user.target
   ```

4. Enable and start the service:
   ```bash
   sudo systemctl enable vkax
   sudo systemctl start vkax
   ```

---

## **Step 7: Verify Masternode Status**

1. Check synchronization:
   ```bash
   ~/.vkaxcore/vkax-cli mnsync status
   ```
   Ensure `"IsBlockchainSynced": true`.

2. Verify masternode status:
   ```bash
   ~/.vkaxcore/vkax-cli masternode status
   ```
   You should see `"status": "READY"`.

3. Check the masternode list:
   ```bash
   ~/.vkaxcore/vkax-cli masternode list
   ```

---

## **Conclusion**

Your VKAX masternode is now operational. Monitor it regularly, update software as needed, and ensure your collateral remains untouched. By following this guide, you are contributing to the network while earning rewards. 🎉





# 设置 VKAX 主节点

本指南提供了使用 VPS 设置 VKAX 主节点的详细步骤，即使是首次操作的用户也能轻松完成。以下包括所有必要命令和说明。

---

## **概述**

您需要准备：

1. **10,000,000 VKAX** 作为抵押。
2. 一个用于管理资金和设置主节点的 **VKAX Core 钱包**。
3. 一台 **虚拟专用服务器 (VPS)**，推荐使用 Oracle Cloud 免费套餐或 Vultr。

> **提示：** 如果在家中托管，需确保路由器支持端口转发，并使用固定公网 IP 地址。推荐使用 VPS 托管以确保稳定性。

---

## **第一步：设置本地钱包**

### 1.1 安装 VKAX 钱包

1. 从 [VKAX Releases](https://github.com/vkaxcore/VKAX/releases) 下载 VKAX Core 钱包。
2. 安装钱包并等待区块链同步完成（可能需要数小时）。

### 1.2 创建抵押地址

1. 在钱包中选择 **工具 > 调试控制台**，生成新地址：
   ```bash
   getnewaddress
   ```
2. 保存生成的地址（例如 `XmdhxVWeyNEc9pjeNW9reTwcDMCwGLi2H`），此地址即为 **抵押地址**。

3. 向该地址发送正好 **10,000,000 VKAX**，并等待至少 **1 次确认**。

---

### 1.3 准备 `protx.txt` 文件

在计算机上创建一个名为 `protx.txt` 的文本文件，内容如下：
```bash
protx register <collateral-hash> <collateral-index> <server_ip>:11110 <owner> <bls-public-key> <voting> 0 <payout> <some-change-address>
```

#### 1.4 生成三个地址

在钱包中运行以下命令：
```bash
getnewaddress "ownerKeyAddr"
getnewaddress "votingKeyAddr"
getnewaddress "payoutAddress"
```

更新 `protx.txt`：
- `<owner>`：替换为 `ownerKeyAddr`。
- `<voting>`：替换为 `votingKeyAddr`。
- `<payout>`：替换为 `payoutAddress`。

保存 `protx.txt` 文件，稍后将补充更多内容。

---

## **第二步：准备 VPS**

### 2.1 设置 VPS 实例

1. **选择 VPS 提供商：** Oracle Cloud 提供免费套餐，Vultr 提供可靠且经济实惠的 VPS。
2. **安装 Ubuntu 22.04** 或 **20.04** 系统。
3. **连接到 VPS：** 使用 SSH：
   ```bash
   ssh root@<server_ip>
   ```
   将 `<server_ip>` 替换为您的 VPS IP 地址。

4. 设置强密码：
   ```bash
   passwd root
   ```

---

### 2.2 安装依赖

1. 更新和升级 VPS 系统：
   ```bash
   sudo apt-get update -y && sudo apt-get upgrade -y
   ```
2. 安装所需工具：
   ```bash
   sudo apt-get install curl build-essential libtool autotools-dev automake pkg-config python3 bsdmainutils bison libssl-dev libevent-dev libboost-system-dev libboost-filesystem-dev libboost-chrono-dev libboost-program-options-dev libboost-test-dev libboost-thread-dev g++-aarch64-linux-gnu g++-mingw-w64-x86-64 mingw-w64-x86-64-dev gperf zip git curl ufw -y
   ```

---

### 2.3 开启防火墙

1. 允许必要端口，包括 VKAX 的 **11110** 端口：
   ```bash
   sudo ufw allow 22/tcp
   sudo ufw allow 11110/tcp
   sudo ufw enable
   ```

---

### 2.4 创建 `vkax` 用户

1. 创建非超级用户：
   ```bash
   adduser vkax
   ```
2. 设置密码，但 **不要授予 sudo 权限**。

---

## **第三步：构建 VKAX 守护进程**

1. 克隆 VKAX 仓库：
   ```bash
   git clone https://github.com/vkaxcore/VKAX.git
   cd VKAX
   ```
2. 构建依赖项：
   ```bash
   cd depends
   chmod +x conf*
   make
   cd ..
   ```
3. 生成配置文件：
   ```bash
   ./autogen.sh
   ```
4. 配置并编译守护进程：
   ```bash
   ./configure --prefix=$PWD/vkax-build/ --disable-wallet --without-gui
   make
   ```
5. 安装二进制文件：
   ```bash
   mkdir -p /home/vkax/.vkaxcore
   cp vkax-build/vkaxd vkax-build/vkax-cli /home/vkax/.vkaxcore/
   ```

---

## **第四步：配置 VKAX 节点**

1. 切换到 `vkax` 用户：
   ```bash
   su - vkax
   ```
2. 创建 `vkax.conf` 文件：
   ```bash
   nano ~/.vkaxcore/vkax.conf
   ```
   添加以下内容：
   ```
   rpcuser=<rpcuser>
   rpcpassword=<rpcpassword>
   rpcallowip=127.0.0.1
   listen=1
   server=1
   daemon=1
   masternodeblsprivkey=<masternodeblsprivkey>
   externalip=<server_ip>
   ```
   替换占位符为您的信息。

3. 启动守护进程：
   ```bash
   ~/.vkaxcore/vkaxd
   ```

4. 检查同步状态：
   ```bash
   ~/.vkaxcore/vkax-cli mnsync status
   ```

---

## **第五步：完成并提交 `protx.txt`**

### 5.1 添加最终细节

1. 获取抵押交易详情：
   ```bash
   masternode outputs
   ```
2. 更新以下内容：
   - `<collateral-hash>`：来自 `masternode outputs`。
   - `<collateral-index>`：来自 `masternode outputs`。
   - `<server_ip>`：您的 VPS IP 地址。

---

### 5.2 注册主节点

1. 打开 Windows 钱包控制台。
2. 提交 ProTx：
   ```bash
   protx register <contents of protx.txt>
   ```
3. 等待 **1 次确认**，主节点将显示在列表中。

---

## **第六步：创建 Systemd 服务**

1. 退出 `vkax` 用户：
   ```bash
   exit
   ```

2. 创建服务文件：
   ```bash
   sudo nano /etc/systemd/system/vkax.service
   ```

3. 添加以下内容：
   ```
   [Unit]
   Description=VKAX Masternode Service
   After=network.target

   [Service]
   Type=forking
   User=vkax
   WorkingDirectory=/home/vkax/.vkaxcore/
   ExecStart=/home/vkax/.vkaxcore/vkaxd
   ExecStop=/home/vkax/.vkaxcore/vkax-cli stop
   Restart=on-failure
   RestartSec=10s

   [Install]
   WantedBy=multi-user.target
   ```

4. 启用并启动服务：
   ```bash
   sudo systemctl enable vkax
   sudo systemctl start vkax
   ```

---

## **第七步：验证主节点状态**

1. 检查同步状态：
   ```bash
   ~/.vkaxcore/vkax-cli mnsync status
   ```
   确保 `"IsBlockchainSynced": true`。

2. 验证主节点状态：
   ```bash
   ~/.vkaxcore/vkax-cli masternode status
   ```
   应显示 `"status": "READY"`。

3. 检查主节点列表：
   ```bash
   ~/.vkaxcore/vkax-cli masternode list
   ```

---

## **总结**

您的 VKAX 主节点现已成功运行。定期监控节点状态，按需更新软件，确保抵押资金未被动用。通过此指南，您不仅支持了网络，还能获得奖励。🎉





# VKAX मास्टरनोड सेटअप करें 

यह गाइड VPS का उपयोग करके VKAX मास्टरनोड सेटअप करने के लिए एक विस्तृत चरण-दर-चरण प्रक्रिया प्रदान करता है। यह गाइड पहली बार उपयोगकर्ताओं के लिए भी इसे आसान बनाता है। 

---

## **परिचय**

आपको आवश्यकता होगी:

1. **10,000,000 VKAX** गारंटी के रूप में।
2. अपने फंड को प्रबंधित करने और मास्टरनोड सेटअप के लिए एक **VKAX कोर वॉलेट**।
3. एक **वर्चुअल प्राइवेट सर्वर (VPS)**, जैसे Oracle Cloud (फ्री टियर) या Vultr।

> **टिप:** यदि आप होम नेटवर्क से होस्ट कर रहे हैं, तो सुनिश्चित करें कि आपका राउटर पोर्ट फॉरवर्डिंग को अनुमति देता है और आपके पास एक स्थिर सार्वजनिक IP पता है। VPS होस्टिंग स्थिरता के लिए अनुशंसित है।

---

## **चरण 1: लोकल वॉलेट सेट करें**

### 1.1 VKAX वॉलेट इंस्टॉल करें

1. [VKAX रिलीज़](https://github.com/vkaxcore/VKAX/releases) से VKAX कोर वॉलेट डाउनलोड करें।
2. वॉलेट इंस्टॉल करें और इसे ब्लॉकचेन के साथ पूरी तरह से सिंक होने दें। इसमें कुछ घंटे लग सकते हैं।

### 1.2 गारंटी पता (Collateral Address) बनाएं

1. वॉलेट में **टूल्स > डिबग कंसोल** पर जाएं और नया पता जनरेट करें:
   ```bash
   getnewaddress
   ```
2. जनरेट किया गया पता सेव करें (उदाहरण: `XmdhxVWeyNEc9pjeNW9reTwcDMCwGLi2H`)। यह आपका **गारंटी पता** होगा।

3. इस पते पर ठीक **10,000,000 VKAX** भेजें और कम से कम **1 पुष्टि** (confirmation) की प्रतीक्षा करें।

---

### 1.3 `protx.txt` फ़ाइल तैयार करें

अपने कंप्यूटर पर एक टेक्स्ट फ़ाइल (`protx.txt`) बनाएं और इसमें निम्नलिखित लिखें:
```bash
protx register <collateral-hash> <collateral-index> <server_ip>:11110 <owner> <bls-public-key> <voting> 0 <payout> <some-change-address>
```

#### 1.4 तीन नए पते जनरेट करें

वॉलेट में निम्नलिखित कमांड चलाएं:
```bash
getnewaddress "ownerKeyAddr"
getnewaddress "votingKeyAddr"
getnewaddress "payoutAddress"
```

`protx.txt` अपडेट करें:
- `<owner>`: `ownerKeyAddr` से बदलें।
- `<voting>`: `votingKeyAddr` से बदलें।
- `<payout>`: `payoutAddress` से बदलें।

`protx.txt` सेव करें। इसे बाद में और जानकारी से भरें।

---

## **चरण 2: VPS तैयार करें**

### 2.1 VPS इंस्टेंस सेट करें

1. **VPS प्रदाता चुनें:** Oracle Cloud मुफ्त टियर प्रदान करता है और Vultr विश्वसनीय और किफायती VPS प्रदान करता है।
2. **Ubuntu 22.04** या **20.04** इंस्टॉल करें।
3. **VPS से कनेक्ट करें:** SSH का उपयोग करें:
   ```bash
   ssh root@<server_ip>
   ```
   `<server_ip>` को अपने VPS के IP पते से बदलें।

4. मजबूत पासवर्ड सेट करें:
   ```bash
   passwd root
   ```

---

### 2.2 आवश्यक पैकेज इंस्टॉल करें

1. VPS को अपडेट और अपग्रेड करें:
   ```bash
   sudo apt-get update -y && sudo apt-get upgrade -y
   ```
2. आवश्यक टूल इंस्टॉल करें:
   ```bash
   sudo apt-get install curl build-essential libtool autotools-dev automake pkg-config python3 bsdmainutils bison libssl-dev libevent-dev libboost-system-dev libboost-filesystem-dev libboost-chrono-dev libboost-program-options-dev libboost-test-dev libboost-thread-dev g++-aarch64-linux-gnu g++-mingw-w64-x86-64 mingw-w64-x86-64-dev gperf zip git curl ufw -y
   ```

---

### 2.3 फ़ायरवॉल चालू करें

1. आवश्यक पोर्ट खोलें, जिसमें VKAX के लिए **11110** पोर्ट शामिल है:
   ```bash
   sudo ufw allow 22/tcp
   sudo ufw allow 11110/tcp
   sudo ufw enable
   ```

---

### 2.4 `vkax` उपयोगकर्ता बनाएं

1. एक गैर-सुपर उपयोगकर्ता बनाएं:
   ```bash
   adduser vkax
   ```
2. पासवर्ड सेट करें लेकिन **sudo अनुमति न दें**।

---

## **चरण 3: VKAX डेमन बनाएं**

1. VKAX रिपॉजिटरी क्लोन करें:
   ```bash
   git clone https://github.com/vkaxcore/VKAX.git
   cd VKAX
   ```
2. डिपेंडेंसी बनाएं:
   ```bash
   cd depends
   chmod +x conf*
   make
   cd ..
   ```
3. कॉन्फ़िगरेशन फाइलें तैयार करें:
   ```bash
   ./autogen.sh
   ```
4. डेमन को कॉन्फ़िगर और कंपाइल करें:
   ```bash
   ./configure --prefix=$PWD/vkax-build/ --disable-wallet --without-gui
   make
   ```
5. बाइनरी इंस्टॉल करें:
   ```bash
   mkdir -p /home/vkax/.vkaxcore
   cp vkax-build/vkaxd vkax-build/vkax-cli /home/vkax/.vkaxcore/
   ```

---

## **चरण 4: VKAX नोड को कॉन्फ़िगर करें**

1. `vkax` उपयोगकर्ता पर स्विच करें:
   ```bash
   su - vkax
   ```
2. `vkax.conf` फ़ाइल बनाएं:
   ```bash
   nano ~/.vkaxcore/vkax.conf
   ```
   इसमें निम्नलिखित जोड़ें:
   ```
   rpcuser=<rpcuser>
   rpcpassword=<rpcpassword>
   rpcallowip=127.0.0.1
   listen=1
   server=1
   daemon=1
   masternodeblsprivkey=<masternodeblsprivkey>
   externalip=<server_ip>
   ```
   प्लेसहोल्डर को अपनी जानकारी से बदलें।

3. डेमन शुरू करें:
   ```bash
   ~/.vkaxcore/vkaxd
   ```

4. सिंक स्थिति जांचें:
   ```bash
   ~/.vkaxcore/vkax-cli mnsync status
   ```

---

## **चरण 5: `protx.txt` फाइल पूरी करें और सबमिट करें**

### 5.1 अंतिम विवरण जोड़ें

1. गारंटी लेन-देन की जानकारी प्राप्त करें:
   ```bash
   masternode outputs
   ```
2. अद्यतन करें:
   - `<collateral-hash>`: `masternode outputs` से प्राप्त करें।
   - `<collateral-index>`: `masternode outputs` से प्राप्त करें।
   - `<server_ip>`: अपना VPS IP दर्ज करें।

---

### 5.2 मास्टरनोड रजिस्टर करें

1. विंडोज वॉलेट कंसोल खोलें।
2. ProTx सबमिट करें:
   ```bash
   protx register <contents of protx.txt>
   ```
3. **1 पुष्टि** का इंतजार करें, मास्टरनोड सूची में दिखेगा।

---

## **चरण 6: सिस्टमड सेवा बनाएं**

1. `vkax` उपयोगकर्ता से बाहर निकलें:
   ```bash
   exit
   ```

2. एक सेवा फ़ाइल बनाएं:
   ```bash
   sudo nano /etc/systemd/system/vkax.service
   ```

3. निम्नलिखित जोड़ें:
   ```
   [Unit]
   Description=VKAX Masternode Service
   After=network.target

   [Service]
   Type=forking
   User=vkax
   WorkingDirectory=/home/vkax/.vkaxcore/
   ExecStart=/home/vkax/.vkaxcore/vkaxd
   ExecStop=/home/vkax/.vkaxcore/vkax-cli stop
   Restart=on-failure
   RestartSec=10s

   [Install]
   WantedBy=multi-user.target
   ```

4. सेवा सक्षम करें और शुरू करें:
   ```bash
   sudo systemctl enable vkax
   sudo systemctl start vkax
   ```

---

## **चरण 7: मास्टरनोड स्थिति सत्यापित करें**

1. सिंक स्थिति जांचें:
   ```bash
   ~/.vkaxcore/vkax-cli mnsync status
   ```
   सुनिश्चित करें कि `"IsBlockchainSynced": true` है।

2. मास्टरनोड स्थिति सत्यापित करें:
   ```bash
   ~/.vkaxcore/vkax-cli masternode status
   ```
   आपको `"status": "READY"` देखना चाहिए।

3. मास्टरनोड सूची जांचें:
   ```bash
   ~/.vkaxcore/vkax-cli masternode list
   ```

---

## **निष्कर्ष**

आपका VKAX मास्टरनोड अब चालू है। नियमित रूप से निगरानी करें, सॉफ़्टवेयर अपडेट करें, और सुनिश्चित करें कि आपकी गारंटी अछूती है। इस गाइड का पालन करके, आप नेटवर्क का समर्थन करते हैं और पुरस्कार अर्जित करते हैं। 🎉

