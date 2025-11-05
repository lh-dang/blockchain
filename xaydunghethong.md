## XÓA SẠCH HỆ THỐNG
### 🚮 1. Gỡ Node.js và npm

**Kiểm tra Node đang cài ở đâu:**
```
which node
which npm
```

**Rồi gỡ hoàn toàn (tùy cách cài):**
```
sudo apt remove nodejs npm -y
sudo apt purge nodejs npm -y
```

**Nếu cài qua NodeSource (curl script):**
```
sudo rm -rf /etc/apt/sources.list.d/nodesource.list
sudo apt autoremove -y
```

**Nếu dùng nvm (Node Version Manager):**
```
rm -rf ~/.nvm
```
### 🧩 2. Xóa Truffle và toàn bộ thư mục liên quan
```
sudo npm uninstall -g truffle
sudo rm -rf ~/.config/truffle ~/.truffle ~/.cache/truffle
```

### 🧹 3. Xóa cache npm & module rác
```
npm cache clean --force
sudo rm -rf ~/.npm
sudo rm -rf ~/.node-gyp
sudo rm -rf /usr/local/lib/node_modules
```

### 🔁 4. Làm sạch hệ thống
```
sudo apt autoremove -y
sudo apt autoclean
```

### 🔁 5. Xóa Ganache

```
sudo npm uninstall -g ganache
sudo npm uninstall -g ganache-cli
```

```
npm cache clean --force
ganache --version
```
### 📦 6. Gỡ IPFS bản cài qua npm
```
sudo rm -f /usr/local/bin/ipfs
sudo rm -rf ~/.ipfs
sudo rm -rf ~/.cache/ipfs
sudo rm -rf ~/.config/ipfs
```

```
sudo apt remove ipfs -y
sudo apt purge ipfs -y
```

```
sudo npm uninstall -g ipfs
npm cache clean --force
ipfs version
```

## CÀI LẠI HỆ THỐNG
### 🌱 1. Cài Node.js 20 qua NodeSource
```
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
```

```
node -v
npm -v
```
### 🌱 2. Cài lại truffle ganache ipfs  bản mới nhất
```
sudo npm install -g truffle ganache ipfs
```

###  🌱 3. OpenZeppelin

- OpenZeppelin không phải phần mềm độc lập, mà là thư viện npm (dạng package JavaScript) dành cho Solidity/Truffle.
- Nó giúp bạn có sẵn các smart contract chuẩn bảo mật (ERC20, ERC721, AccessControl, Ownable, v.v.) để khỏi phải tự code lại từ đầu.

**⚙️ 1. Cài trong project hiện tại (khuyến nghị)**
- Vào thư mục project blockchain của bạn (ví dụ student-ledger hoặc truffle-demo) rồi chạy:
```
npm install @openzeppelin/contracts
# Lệnh này sẽ tạo thư mục:
# node_modules/@openzeppelin/contracts/
```

**🧩 3. Nếu bạn dùng Truffle, cần cài thêm dev tools:** Trong thư mục Truffle project:
```
npm install @openzeppelin/contracts @openzeppelin/test-helpers
```
- Tăng khả năng test, debug và triển khai smart contract bằng thư viện của OpenZeppelin.
