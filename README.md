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
