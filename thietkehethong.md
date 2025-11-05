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
**Triết lý thiết kế:**
- **Hybrid tối ưu chi phí:** on-chain chỉ giữ “dấu vân tay” (Merkle root + sự kiện), off-chain giữ chi tiết (IPFS + DB).
- **Bất biến nhưng linh hoạt:** mọi thay đổi không xóa mà version hóa, sửa điểm = bản mới + hai chữ ký.
- **Chứng minh được, không cần tin tưởng mù quáng:** mọi thứ có hash/Proof kiểm lại được từ blockchain.

## 2. Kiến trúc tổng thể

**Smart Contracts**

- `AccessControlRegistry` (vai trò: LECTURER/DEAN/RECTOR/ADMIN).

- `TranscriptRegistry` (UUPS + Timelock + Multisig): lưu semesterRoot và sự kiện (commit, amend).

- `DiplomaRegistry` (non-upgradeable): cấp/thu hồi bằng; dữ liệu bằng cực ổn định cho nhà tuyển dụng.

**Off-chain**

- **IPFS:** lưu `grade.json` (v1…vn), `semester.json`, `diploma_meta.json` (+ PDF); demo chưa mã hóa, prod bật AES-GCM + wrap key cho SV & Trường.
- **Database** (Postgres/Mongo): metadata, mapping `studentIdHash ↔ CIDs`, trạng thái kỳ, curriculum, audit log.
- **Indexer:** bám event on-chain để đồng bộ dữ liệu cho frontend.

**Frontend**

- Portal 4 vai trò (GV/Khoa/Hiệu trưởng/Admin) + **Student Dashboard + Trang Verify dành cho doanh nghiệp.**

**Backend (Node/Express)**
- CSV import, chuẩn hóa canonical JSON (JCS), sinh/kiểm Merkle-of-Merkles, gọi contract, cung cấp API verify.

3) Dữ liệu & băm (hash)

**Canonical JSON: khóa sắp xếp, số liệu chuẩn, không lệch hash.**

**Merkle-of-Merkles (mạnh nhất)**
- Lá 1 (per-course): keccak(courseId, courseCID, version).
- Root SV (một SV trong kỳ).
- Lá 2 (per-student): keccak(studentIdHash, semesterId, studentCourseRoot).
- semesterRoot (ghi on-chain).
→ Doanh nghiệp có thể verify đến từng môn bằng 2 lớp proof, không phải lộ hết kỳ.

## 4 Vai trò & quyền hạn (điểm mạnh)
- **Lecturer (GV):** nhập điểm (form hoặc CSV) → ký EIP-712 → tạo grade.json (v1, v2…) lên IPFS.
- **Dean (Trưởng khoa):** hệ thống tự gom → tính GPA/TC → ký EIP-712 semester.json (chứa “pointer latest” và version từng môn).
- **Rector (Hiệu trưởng/Issuer):** build Merkle cho toàn khoa/kỳ → commit on-chain (1 giao dịch/kỳ) → issue/revoke bằng trên DiplomaRegistry (ký bằng Ledger/Gnosis Safe).
- **Admin:** quản trị học kỳ, môn, CSV, phân quyền; không can thiệp nội dung học thuật.
- Student: xem timeline, đăng ký xét tốt nghiệp, tải proof; khi có bằng → xem NFT & QR verify.
## 5. Website Demo (3 giao diện)

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


## 6. Quy trình chuẩn (workflows)

1. **Grading (GV):** CSV → chuẩn hóa → ký EIP-712 → IPFS → DB cập nhật “latest”.
2. **Semester Sign (Khoa):** auto-gom → ký EIP-712 bundle (kèm map phiên bản “latest”) → IPFS.
3. **Commit (Hiệu trưởng):** build Merkle-of-Merkles → `setSemesterRoot(semesterId, root, cid)` on-chain.
4. **Amend (Sửa điểm):** GV ký bản mới + Dean/Admin đồng ký → emit GradeChanged (on-chain event), bundle mới → commit root mới (lưu lịch sử).
5. **Graduation:** SV bấm yêu cầu → backend kiểm curriculum (BB/TC + miễn AV nếu B1/IELTS) → Hiệu trưởng ký `issueDiploma` (NFT) → `DiplomaRegistry` lưu immutable + revoke khi cần.
6. **Verify (Doanh nghiệp):** quét QR → nhập `studentId`/`courseId` → gửi proof 2 lớp lên UI → so với `semesterRoot` on-chain + trạng thái `DiplomaRegistry` → Valid/Invalid + txHash/timestamp.

## 7. Lợi thế cạnh tranh (tóm tắt)
- Xác thực tới cấp môn (Merkle-of-Merkles) mà vẫn chỉ 1 giao dịch/kỳ.
- Sửa điểm minh bạch: version hóa + 2 chữ ký + event on-chain, không thể “đổi lén”.
- Không rò rỉ PII trên chain, dễ compliant.
- Không giữ private key trên server (an toàn thực chiến).
- Có đường nâng cấp (UUPS) nhưng bằng cấp vẫn bất biến (registry riêng).
- Triển khai nhanh: CSV → ký EIP-712 → IPFS → Merkle → commit.
