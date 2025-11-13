**⚙️ 1. Cài trong project hiện tại (khuyến nghị)**
- Vào thư mục project blockchain của bạn (ví dụ student-ledger hoặc truffle-demo) rồi chạy:
```
npm install @openzeppelin/contracts
# Lệnh này sẽ tạo thư mục:
# node_modules/@openzeppelin/contracts/
```

**🧩 2. Nếu bạn dùng Truffle, cần cài thêm dev tools:** Trong thư mục Truffle project:
```
npm install @openzeppelin/contracts @openzeppelin/test-helpers
```
- Tăng khả năng test, debug và triển khai smart contract bằng thư viện của OpenZeppelin.

# CODE DEWMO

### Test nhanh Ganache (để chắc chắn mọi thứ OK)
**Mở terminal 1:**
```
ganache -p 8545 -m "test test test test test test test test test test test junk"
```

### Tạo project & cài dependency

```
mkdir -p ~/student-ledger && cd ~/student-ledger
truffle init

npm init -y
npm install @openzeppelin/contracts dotenv
```

```
student-ledger/
├── contracts/
│   ├── AccessControlRegistry.sol
│   ├── TranscriptRegistry.sol
│   └── DiplomaRegistry.sol
├── migrations/
│   ├── 1_initial_migration.js
│   └── 2_deploy_core.js
├── truffle-config.js
└── package.json
```

### Compile & migrate (nhớ Ganache đang chạy)

**Trong thư mục ~/student-ledger:**
```
truffle compile
truffle migrate --reset --network development
```

### Chạy ipfs
```
ipfs daemon
```
### chạy ganache

```
ganache-cli --deterministic
```
