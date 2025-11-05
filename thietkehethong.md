# 🎓 HỆ THỐNG QUẢN LÝ SINH VIÊN BLOCKCHAIN (Ethereum Demo)
## 1. Kiến trúc tổng thể

**Công nghệ:**

- Blockchain: Ethereum (Ganache / Sepolia)
- Smart Contract: Solidity (Truffle, OpenZeppelin)
- Off-chain Storage: IPFS (không mã hóa, tạm thời)
- Backend: Node.js + Express + Web3.js
- Frontend: React (MetaMask connect)
- Dữ liệu phụ trợ: CSV import/export (điểm, sinh viên, môn học)


**Mô hình tổng quan:**

```
Giảng viên nhập điểm → Ký → IPFS → Trưởng khoa ký tổng hợp → Hiệu trưởng ký Merkle → Blockchain → NFT bằng tốt nghiệp
```

## 2. Vai trò và quyền hạn
| Vai trò               | Quyền hạn                                                                                         | Ghi chú                               |
| --------------------- | ------------------------------------------------------------------------------------------------- | ------------------------------------- |
| **Admin**             | Tạo tài khoản sinh viên, giảng viên, khóa học, phân quyền; thêm môn, học kỳ                       | Không ký học thuật                    |
| **Giảng viên bộ môn** | Nhập điểm (CSV hoặc web form), ký bằng MetaMask (EIP-712), lưu IPFS                               | Mọi lần chỉnh điểm đều cần chữ ký lại |
| **Trưởng khoa**       | Xác nhận và ký học kỳ (tự động gộp từ hệ thống), lưu lên IPFS                                     | Cũng cần ký lại khi điểm chỉnh        |
| **Hiệu trưởng**       | Xây Merkle Tree cho tất cả sinh viên trong học kỳ, ký và ghi root on-chain; có quyền thu hồi bằng | Chịu trách nhiệm cuối cùng            |
| **Sinh viên**         | Xem tiến trình học tập, đăng ký xét tốt nghiệp, nhận NFT bằng tốt nghiệp                          | Có thể xem toàn bộ chain học tập      |
| **Doanh nghiệp**      | Xác thực NFT và dữ liệu học tập qua QR code hoặc trang web verify                                 | Không cần liên hệ trường              |

## 3. Website Demo (3 giao diện)

**Giảng viên:**
- Trang “Nhập điểm”
- Nút “Import CSV” hoặc nhập tay
- Nút “Ký & Lưu”
- Hiển thị CID, hash, trạng thái “Chờ trưởng khoa”

**Trưởng khoa:**
- Trang “Xác nhận học kỳ”
- Hệ thống tự gom môn theo sinh viên
- Nút “Ký học kỳ”
- Hiển thị GPA, tín chỉ

**Hiệu trưởng:**
- Trang “Tổng hợp & Đưa lên Blockchain”
- Hiển thị root, số sinh viên, CID
- Nút “Ký & Commit”
- Có tab “Thu hồi bằng” với lý do

**Sinh viên:**
- Trang “Tiến trình học tập” (timeline)
- Nút “Đăng ký xét tốt nghiệp”
- Khi có NFT → xem thông tin, QR verify

**Admin Portal (web)**
