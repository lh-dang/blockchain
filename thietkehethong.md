# 🎓 HỆ THỐNG QUẢN LÝ SINH VIÊN BLOCKCHAIN (Ethereum Demo)
## 1. Kiến trúc tổng thể

**Công nghệ:**

- Blockchain: Ethereum (Ganache / Sepolia)
- Smart Contract: Solidity (Truffle, OpenZeppelin)
- Off-chain Storage: IPFS (không mã hóa, tạm thời)
- Backend: Node.js + Express + Web3.js
- Frontend: React (MetaMask connect)
- Dữ liệu phụ trợ: CSV import/export (điểm, sinh viên, môn học)

# HỆ THỐNG QUẢN LÝ SINH VIÊN DỰA TRÊN BLOCKCHAIN ETHEREUM

## I. MỤC TIÊU HỆ THỐNG

Xây dựng hệ thống quản lý sinh viên dựa trên blockchain Ethereum, hỗ trợ:

- **Quản lý thông tin sinh viên** (ẩn danh trên blockchain, không để lộ dữ liệu nhạy cảm)
- **Lưu trữ điểm theo học kỳ** (1 học kỳ = 1 record on-chain) → **Tối ưu chi phí gas**
- **Lịch sử bất biến** (event log): mỗi lần submit = 1 event, không xóa bản cũ
- **Học cải thiện**: Giảng viên có thể cập nhật điểm cả học kỳ, hệ thống giữ phiên bản mới nhất
- **Xét tốt nghiệp tự động** dựa trên kết quả đã đạt
- **Cấp bằng tốt nghiệp** dưới dạng NFT duy nhất, chứa toàn bộ hành trình học tập (thông qua Merkle root)
- **Giảng viên** có trang nhập điểm theo học kỳ
- **Sinh viên** có trang xem timeline học tập theo học kỳ
- **Doanh nghiệp** có trang verify bằng và bảng điểm
- **Tối đa hóa** tính minh bạch, bất biến, bảo mật, **tối ưu chi phí**

---

## II. KIẾN TRÚC DỮ LIỆU & BẢO MẬT

### 1. Dữ liệu nhạy cảm

**Dữ liệu bao gồm**: Họ tên, MSSV, ngày sinh, môn học chi tiết, điểm gốc

> ⚠️ **Không bao giờ đưa lên blockchain dưới dạng plaintext**

**Phương pháp lưu trữ:**

1. JSON chi tiết điểm được **mã hóa bằng public key** của sinh viên
2. Upload bản mã hóa lên **IPFS**
3. Nhận về **CID** và **ciphertextHash**

**Lợi ích:**
- Blockchain chỉ chứa hash, không thể lần ra thông tin cá nhân
- Chỉ sinh viên mới giải mã được
- Doanh nghiệp có thể verify tính toàn vẹn (hash trùng đúng)

### 2. Dữ liệu công khai tối thiểu

**On-chain chỉ lưu (theo HỌC KỲ):**

- `studentAddress` (ví MetaMask)
- `semesterId` (mã học kỳ dạng bytes32, ví dụ: "2024_HK1", "2024_HK2")
- `ipfsCID` (link đến file JSON mã hóa chứa tất cả môn trong học kỳ)
- `hashCiphertext` (hash của file mã hóa)
- `version` (phiên bản cập nhật, để theo dõi nếu giảng viên sửa điểm)
- `timestamp` (thời gian submit)

**Lợi ích:**
- ✅ **Giảm 80-90% chi phí gas**: thay vì 10 môn = 10 giao dịch → chỉ cần 1 giao dịch/học kỳ
- ✅ **Đơn giản hóa**: một lần submit cho cả học kỳ
- ✅ **Linh hoạt**: Giảng viên có thể cập nhật toàn bộ học kỳ nếu có sai sót

> ❌ **KHÔNG có**: Họ tên, MSSV, lớp, khóa, ngành, ngày sinh, điểm chi tiết từng môn

---

## III. LUỒNG NGHIỆP VỤ CỦA HỆ THỐNG

### 1. Quản trị viên (Admin)

**Nhiệm vụ:**
- Deploy smart contract
- Tạo quyền: cấp role `lecturer`, `student`
- Cập nhật chương trình đào tạo (curriculum)
  - Danh sách môn bắt buộc cho từng ngành
  - Có thể version hóa (nếu thay đổi CTĐT)

---

### 2. Giảng viên (Lecturer)

**Quy trình nhập điểm THEO HỌC KỲ:**

1. **Giao diện web** của giảng viên thực hiện:
   - Chọn **học kỳ** (VD: HK1/2024, HK2/2024)
   - Nhập điểm **tất cả các môn** của sinh viên trong học kỳ đó
   - Tạo JSON chi tiết:
   ```json
   {
     "semesterId": "2024_HK1",
     "studentInfo": {
       "name": "Nguyễn Văn A",
       "studentId": "20210001",
       "major": "Computer Science"
     },
     "courses": [
       {"courseId": "CS101", "courseName": "Lập trình C", "credits": 3, "score": 8.5},
       {"courseId": "CS102", "courseName": "Cấu trúc dữ liệu", "credits": 4, "score": 7.0},
       {"courseId": "MATH201", "courseName": "Toán rời rạc", "credits": 3, "score": 9.0}
     ],
     "gpa": 8.1
   }
   ```
   - Mã hóa bằng **public key** của sinh viên
   - Upload lên **IPFS** → nhận **CID**
   - Tính `hashCiphertext = keccak256(ciphertext)`

2. **Gửi giao dịch** `submitSemesterGrades()` lên smart contract:

```solidity
submitSemesterGrades(
    studentAddress,
    semesterId,      // "2024_HK1"
    cid,             // IPFS CID
    hashCiphertext   // Hash của file mã hóa
)
```

**Smart contract thực hiện 2 việc:**

#### 2.1. Lưu lịch sử học kỳ (immutable event)

```solidity
event SemesterGradesRecorded(
    address indexed student,
    bytes32 indexed semesterId,
    uint256 version,          // Phiên bản (1, 2, 3...)
    string cid,
    bytes32 hashCiphertext,
    uint256 timestamp
);
```

→ Phục vụ timeline, audit log, doanh nghiệp verify

#### 2.2. Cập nhật điểm hiệu lực (currentSemester)

- Lưu **phiên bản mới nhất** của học kỳ
- Nếu giảng viên sửa điểm → `version++`
- Event cũ vẫn tồn tại → audit trail đầy đủ
- → Phục vụ xét tốt nghiệp

**Lợi ích chi phí:**
- ❌ **Cũ**: 10 môn × 100,000 gas = 1,000,000 gas/học kỳ
- ✅ **Mới**: 1 giao dịch ≈ 150,000 gas/học kỳ → **Tiết kiệm 85%**

---

### 3. Sinh viên (Student)

#### 3.1. Xem timeline học tập THEO HỌC KỲ

**Frontend thực hiện:**
- Query event `SemesterGradesRecorded(student, *)`
- Sort theo thời gian
- Render dòng thời gian:
  ```
  📅 HK1/2023 (Version 1) → GPA: 7.5
     📎 CID: Qm... → Click để tải & giải mã
     
  📅 HK2/2023 (Version 1) → GPA: 8.0
     📎 CID: Qm...
     
  📅 HK1/2024 (Version 2) → GPA: 8.5 ⚠️ Đã cập nhật
     📎 CID (v1): Qm... [Phiên bản cũ]
     📎 CID (v2): Qm... [Phiên bản mới - Hiệu lực]
  ```

**Chi tiết khi click vào một học kỳ:**
- Tải file IPFS
- Giải mã bằng private key
- Hiển thị bảng điểm chi tiết:
  - Danh sách môn học
  - Điểm từng môn
  - Số tín chỉ
  - GPA học kỳ

#### 3.2. Yêu cầu xét tốt nghiệp & Mint NFT (TRUSTLESS)

**Khi bấm nút "Mint Diploma":**

**Quy trình tự động 100% on-chain:**

```solidity
// Sinh viên tự gọi, không cần admin
function mintDiploma() external {
    require(!hasRole(STUDENT_ROLE, msg.sender) || msg.sender != address(0));
    require(!hasDiploma[msg.sender], "Already graduated");
    
    // 1. Kiểm tra điều kiện tốt nghiệp
    require(checkGraduation(msg.sender), "Not eligible");
    
    // 2. Tính Merkle root từ tất cả học kỳ
    bytes32 merkleRoot = calculateMerkleRoot(msg.sender);
    
    // 3. Mint NFT tự động
    uint256 tokenId = _tokenIdCounter++;
    _safeMint(msg.sender, tokenId);
    hasDiploma[msg.sender] = true;
    
    // 4. Emit event
    emit DiplomaIssued(tokenId, msg.sender, merkleRoot, block.timestamp);
}
```

**Contract tự kiểm tra:**
1. Danh sách **học kỳ bắt buộc** từ `curriculum[major]`
2. `semesterGrades[student][semesterId]` cho tất cả học kỳ phải `.exists == true`
3. Chưa từng mint diploma (`hasDiploma[student] == false`)
4. Tính Merkle root từ hash các học kỳ

**Điều kiện on-chain:**
- ✅ Tất cả học kỳ bắt buộc đã submit
- ✅ Mỗi học kỳ có CID và hash hợp lệ
- ✅ Chưa tốt nghiệp trước đó

> 💡 **Trustless 100%**: Không cần admin, không cần ai phê duyệt. Smart contract tự quyết định dựa trên dữ liệu on-chain.

---

### 4. Cấp bằng tốt nghiệp (Mint NFT) - TRUSTLESS

#### 4.1. Quy trình tự động hoàn toàn

**Sinh viên tự mint khi đủ điều kiện:**

```javascript
// Frontend - Sinh viên click "Mint Diploma"
await contract.mintDiploma();
// ✅ Thành công → NFT được mint
// ❌ Thất bại → Hiện lý do: thiếu học kỳ X, đã tốt nghiệp, etc.
```

**Contract tự động:**
1. Kiểm tra `curriculum[studentMajor[msg.sender]].requiredSemesters[]`
2. Verify tất cả học kỳ đã submit (`exists == true`)
3. Tính **Merkle root** = `hash(HK1_hash + HK2_hash + ... + HK8_hash)`
4. Mint **1 NFT duy nhất** cho sinh viên
5. Đánh dấu `hasDiploma[student] = true` → không mint lại được

**Metadata NFT (lưu on-chain hoặc IPFS):**
```json
{
  "name": "University Diploma #123",
  "description": "Blockchain-verified graduation certificate",
  "merkleRoot": "0xabc123...",
  "graduationDate": 1699920000,
  "student": "0x1234...",
  "major": "Computer Science"
}
```

#### 4.2. NFT = Bằng tốt nghiệp (Tamper-proof)

**Doanh nghiệp verify:**
1. Kiểm tra `ownerOf(tokenId) == studentAddress`
2. Lấy `merkleRoot` từ NFT metadata
3. Lấy hash từng học kỳ: `semesterGrades[student][semesterId].hashCiphertext`
4. Tính lại merkleRoot và so sánh
5. ✅ Khớp → Bằng thật, dữ liệu đầy đủ

> 🔒 **Không thể giả mạo**: NFT chỉ mint được khi contract verify đủ điều kiện. Không ai (kể cả admin) can thiệp được.

---

### 5. Doanh nghiệp (Verifier)

**Quy trình verify:**

1. Doanh nghiệp truy cập trang verify
2. Upload file điểm/bằng mà sinh viên gửi (plaintext đã giải mã)
3. Hệ thống tính hash
4. So sánh hash với `hashCiphertext` on-chain

**Kết quả:**
- ✅ **Khớp** → tài liệu chuẩn, không chỉnh sửa
- ❌ **Không khớp** → có dấu hiệu giả mạo

> 💡 **Không cần tin sinh viên, không cần tin nhà trường → trust blockchain**

---

## IV. KIẾN TRÚC BLOCKCHAIN & OFF-CHAIN

### 1. On-chain (Ethereum)

**Chỉ lưu:**
- ✅ Event lịch sử điểm (immutable)
- ✅ Mapping `currentGrade`
- ✅ Chương trình đào tạo
- ✅ NFT bằng tốt nghiệp
- ✅ Hash / CID chứng chỉ môn học
- ✅ Role access (student, lecturer, admin)

### 2. Off-chain

#### IPFS
- Lưu JSON mã hóa từng môn học
- Lưu metadata NFT

#### Frontend
- **Giảng viên**: upload điểm
- **Sinh viên**: xem timeline + decrypt file
- **Doanh nghiệp**: verify

#### Backend (optional)
- Không bắt buộc, demo có thể chạy 100% frontend + web3.js
- Chỉ cần nếu muốn:
  - Caching event
  - Indexing
  - Phân quyền phức tạp

---

## V. TÍNH NĂNG BẢO MẬT CHÍNH

| Tính năng | Mô tả |
|-----------|-------|
| 🔐 **Mã hóa end-to-end** | Dữ liệu sinh viên được mã hóa bằng public key |
| 🕵️ **Ẩn danh** | Blockchain chỉ giữ hash, không lộ danh tính |
| 🔒 **Immutable** | Giảng viên không thể sửa điểm cũ → chỉ tạo version mới |
| ✅ **Chống giả mạo** | Sinh viên không thể giả mạo điểm vì: CID mã hóa, Hash on-chain, Merkle proof |
| 🏢 **Verify độc lập** | Doanh nghiệp verify độc lập, không phụ thuộc nhà trường |
| 🚫 **Trustless 100%** | **Sinh viên tự mint diploma khi đủ điều kiện, không cần admin phê duyệt** |
| 🛡️ **Chống mint lại** | Mapping `hasDiploma[]` ngăn mint trùng lặp |

---

## VI. TÍNH NĂNG BẤT BIẾN & MINH BẠCH

- ✅ **Tất cả lần thi** được log trong event → không xóa được
- ✅ **currentGrade** giúp hệ thống lấy điểm cao nhất
- ✅ **NFT tốt nghiệp** đảm bảo bằng cấp là độc nhất
- ✅ **Dấu vân tay dữ liệu** (hash) không thay đổi qua năm tháng

---

## VII. MÔ HÌNH RÚT GỌN CHO DEMO

### 1 Contract chính

```
StudentManagement.sol
├── Role Management
├── Submit Grade
├── Grade History (Events)
├── Current Grade (Mapping)
├── Curriculum
├── Graduation Check
└── NFT Graduation Certificate
```

### 2 Web UI

1. **Lecturer Portal** – nhập điểm & upload
2. **Student Portal** – xem timeline & bấm xét tốt nghiệp
3. **(Optional) Verifier Portal** – xác thực bằng cấp

---

## VIII. TÓM TẮT THEO PHONG CÁCH "DỒN TRỌNG TÂM"

| Thành phần | Cách thức | Tối ưu chi phí | Trustless |
|------------|-----------|----------------|----------|
| 📚 **Học kỳ** | 1 học kỳ = 1 record on-chain | ✅ Giảm 85% gas | ✅ |
| 📊 **Điểm chi tiết** | JSON mã hóa trên IPFS | ✅ Không tốn gas | ✅ |
| 🎓 **Xét tốt nghiệp** | Contract tự kiểm tra on-chain | ✅ Tự động hóa | ✅ 100% |
| 📝 **Cập nhật điểm** | Version mới, event cũ vẫn tồn tại | ✅ Audit trail đầy đủ | ✅ |
| 🏆 **NFT bằng** | **Sinh viên tự mint**, không cần admin | ✅ Bất biến | ✅ **TRUSTLESS** |

### Công thức thành công

```
Blockchain = Sự thật (minimal data)
IPFS = Kho dữ liệu mã hóa (bulk data)
Frontend = Trải nghiệm
Tối ưu = 1 học kỳ = 1 transaction
```

> ⚠️ **Nguyên tắc vàng**: 
> - Không có thông tin nhạy cảm on-chain
> - **Tối thiểu hóa số lượng transaction** → Nhóm dữ liệu theo học kỳ

---

## IX. SƠ ĐỒ KIẾN TRÚC TỔNG THỂ

```
┌─────────────────────────────────────────────────────────┐
│                    FRONTEND LAYER                        │
├─────────────────────────────────────────────────────────┤
│  Lecturer Portal  │  Student Portal  │  Verifier Portal │
│   - Input Grade   │   - Timeline     │   - Verify Cert  │
│   - Encrypt Data  │   - Decrypt      │   - Hash Check   │
│   - Upload IPFS   │   - Request Grad │                  │
└──────────┬────────────────┬────────────────┬────────────┘
           │                │                │
           ▼                ▼                ▼
┌─────────────────────────────────────────────────────────┐
│                   WEB3.JS / ETHERS.JS                    │
└──────────┬────────────────┬────────────────┬────────────┘
           │                │                │
           ▼                ▼                ▼
┌─────────────────────────────────────────────────────────┐
│              ETHEREUM BLOCKCHAIN LAYER                   │
├─────────────────────────────────────────────────────────┤
│         StudentManagement Smart Contract                 │
│  ┌───────────────────────────────────────────────┐      │
│  │ - Role Management (AccessControl)             │      │
│  │ - submitGrade(student, course, score, cid)    │      │
│  │ - currentGrade[student][course]               │      │
│  │ - Curriculum Management                       │      │
│  │ - checkGraduation(student)                    │      │
│  │ - mintDiplomaNFT(student, metadata)           │      │
│  └───────────────────────────────────────────────┘      │
│                                                          │
│  Events (Immutable Log):                                │
│  - GradeAttemptRecorded(...)                            │
│  - DiplomaIssued(...)                                   │
└──────────┬───────────────────────────────────┬──────────┘
           │                                   │
           ▼                                   ▼
┌──────────────────────┐          ┌──────────────────────┐
│   IPFS STORAGE       │          │   NFT METADATA       │
│  - Encrypted JSON    │          │  - Merkle Root       │
│  - Student Data      │          │  - Diploma Info      │
│  - Grade Details     │          │  - Verification Link │
└──────────────────────┘          └──────────────────────┘
```

---

## X. DATAFLOW DIAGRAM

### Flow 1: Nhập điểm THEO HỌC KỲ (Tối ưu chi phí)

```
Lecturer
   │
   ├─► 1. Select semester (e.g., "2024_HK1")
   │
   ├─► 2. Input ALL course grades for the semester
   │       - CS101: 8.5
   │       - CS102: 7.0
   │       - MATH201: 9.0
   │       - ...
   │
   ├─► 3. Create JSON for ENTIRE SEMESTER
   │       {
   │         "semesterId": "2024_HK1",
   │         "courses": [...],
   │         "gpa": 8.1
   │       }
   │
   ├─► 4. Encrypt with student's public key
   │       ciphertext = encrypt(JSON, studentPubKey)
   │
   ├─► 5. Upload to IPFS
   │       → CID: "Qm..."
   │
   ├─► 6. Calculate hash
   │       hashCiphertext = keccak256(ciphertext)
   │
   └─► 7. Submit to blockchain (1 transaction for WHOLE semester)
           submitSemesterGrades(studentAddr, "2024_HK1", CID, hash)
                │
                ├─► Emit: SemesterGradesRecorded(...)
                │
                └─► Update: semesterGrades[student][semesterId]

💰 Gas saved: ~85% compared to per-course submission
```

### Flow 2: Mint Diploma (TRUSTLESS - Sinh viên tự mint)

```
Student
   │
   ├─► 1. Click "Mint Diploma" (Tự gọi contract)
   │
   └─► 2. Contract.mintDiploma() checks:
           │
           ├─► hasDiploma[msg.sender] == false ?
           │   └─► Revert "Already graduated"
           │
           ├─► Read studentMajor[msg.sender] → get major
           │
           ├─► Read curriculum[major].requiredSemesters[]
           │   (e.g., ["2021_HK1", "2021_HK2", ..., "2024_HK2"])
           │
           ├─► For each required semester:
           │   │
           │   ├─► semesterGrades[msg.sender][semesterId].exists ?
           │   │   └─► NO → Revert "Missing semester: 2023_HK1"
           │   │
           │   └─► YES → Continue
           │
           ├─► All semesters exist?
           │   │
           │   ├─► YES → Calculate Merkle Root
           │   │          │
           │   │          ├─► merkleRoot = hash(
           │   │          │      semesterGrades[student][HK1].hashCiphertext +
           │   │          │      semesterGrades[student][HK2].hashCiphertext +
           │   │          │      ...
           │   │          │   )
           │   │          │
           │   │          ├─► _safeMint(msg.sender, tokenId)
           │   │          │
           │   │          ├─► hasDiploma[msg.sender] = true
           │   │          │
           │   │          └─► Emit: DiplomaIssued(tokenId, msg.sender, merkleRoot)
           │   │
           │   └─► NO → Revert "Not eligible"

💡 Không cần admin, không cần phê duyệt
✅ Trustless 100%
```

### Flow 3: Verify bằng cấp

```
Company
   │
   ├─► 1. Receive diploma file from student
   │       (plaintext, decrypted by student)
   │
   ├─► 2. Upload to verification portal
   │
   ├─► 3. System calculates hash
   │       calculatedHash = keccak256(file)
   │
   ├─► 4. Query blockchain
   │       onChainHash = getGradeHash(student, course)
   │
   └─► 5. Compare
           │
           ├─► calculatedHash == onChainHash
           │   └─► ✅ VERIFIED
           │
           └─► calculatedHash != onChainHash
               └─► ❌ TAMPERED
```

---

## XI. SEQUENCE DIAGRAM

### Sequence: Submit Grade

```
Lecturer    Frontend    IPFS    Smart Contract    Blockchain
    │            │         │            │              │
    ├─ Input ───►│         │            │              │
    │            │         │            │              │
    │            ├─Encrypt─┤            │              │
    │            │         │            │              │
    │            ├─Upload─►│            │              │
    │            │◄──CID───┤            │              │
    │            │         │            │              │
    │            ├─submitGrade(...)────►│              │
    │            │         │            ├─Validate────►│
    │            │         │            │              │
    │            │         │            ├─Store────────┤
    │            │         │            │              │
    │            │         │            ├─Emit Event───┤
    │            │         │            │              │
    │            │◄────────Success──────┤              │
    │◄─Success───┤         │            │              │
```

### Sequence: Mint Diploma (TRUSTLESS)

```
Student    Frontend    Smart Contract (StudentManagement)    Blockchain
   │           │                      │                           │
   ├─Click─────►│                     │                           │
   │  "Mint"    │                     │                           │
   │           │                      │                           │
   │           ├─mintDiploma()───────►│                           │
   │           │  (msg.sender)        │                           │
   │           │                      │                           │
   │           │              ┌───────┴────────┐                  │
   │           │              │ 1. Check:      │                  │
   │           │              │  hasDiploma?   │                  │
   │           │              │                │                  │
   │           │              │ 2. Check:      │                  │
   │           │              │  All semesters │                  │
   │           │              │  exist?        │                  │
   │           │              │                │                  │
   │           │              │ 3. Calculate:  │                  │
   │           │              │  merkleRoot    │                  │
   │           │              └───────┬────────┘                  │
   │           │                      │                           │
   │           │                      ├─_safeMint(student)───────►│
   │           │                      │                           │
   │           │                      ├─hasDiploma[]=true────────►│
   │           │                      │                           │
   │           │                      ├─Emit DiplomaIssued───────►│
   │           │                      │                           │
   │           │◄─────Success─────────┤                           │
   │           │  (tokenId, merkleRoot)                           │
   │◄─Success──┤                      │                           │
   │  "Congrats!"                     │                           │

💡 NO ADMIN INVOLVED - Fully Trustless
```

---

## XII. SMART CONTRACT STRUCTURE (Tối ưu chi phí)

### StudentManagement.sol

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/access/AccessControl.sol";
import "@openzeppelin/contracts/token/ERC721/ERC721.sol";

contract StudentManagement is AccessControl, ERC721 {
    
    bytes32 public constant ADMIN_ROLE = keccak256("ADMIN_ROLE");
    bytes32 public constant LECTURER_ROLE = keccak256("LECTURER_ROLE");
    bytes32 public constant STUDENT_ROLE = keccak256("STUDENT_ROLE");
    
    // Semester data structure (1 semester = 1 record)
    struct SemesterGrade {
        string cid;               // IPFS CID chứa JSON tất cả môn
        bytes32 hashCiphertext;   // Hash của file mã hóa
        uint256 version;          // Phiên bản (cho phép cập nhật)
        uint256 timestamp;        // Thời gian submit
        bool exists;
    }
    
    // Curriculum structure
    struct Curriculum {
        bytes32[] requiredSemesters;  // Danh sách học kỳ bắt buộc
        uint256 minPassScore;         // e.g., 50 = 5.0
        uint256 minCredits;           // Tổng tín chỉ tối thiểu
    }
    
    // Storage - MỖI HỌC KỲ = 1 RECORD
    mapping(address => mapping(bytes32 => SemesterGrade)) public semesterGrades;
    mapping(bytes32 => Curriculum) public curriculums; // major => curriculum
    mapping(address => bytes32) public studentMajor;
    
    uint256 private _tokenIdCounter;
    
    // Events
    event SemesterGradesRecorded(
        address indexed student,
        bytes32 indexed semesterId,  // e.g., "2024_HK1"
        uint256 version,
        string cid,
        bytes32 hashCiphertext,
        uint256 timestamp
    );
    
    event DiplomaIssued(
        uint256 indexed tokenId,
        address indexed student,
        bytes32 merkleRoot,
        uint256 timestamp
    );
    
    constructor() ERC721("Diploma", "DIP") {
        _grantRole(DEFAULT_ADMIN_ROLE, msg.sender);
        _grantRole(ADMIN_ROLE, msg.sender);
    }
    
    // Submit semester grades (1 transaction cho cả học kỳ)
    function submitSemesterGrades(
        address student,
        bytes32 semesterId,      // "2024_HK1"
        string memory cid,
        bytes32 hashCiphertext
    ) external onlyRole(LECTURER_ROLE) {
        SemesterGrade storage semester = semesterGrades[student][semesterId];
        
        uint256 newVersion = semester.version + 1;
        
        // Emit immutable event
        emit SemesterGradesRecorded(
            student,
            semesterId,
            newVersion,
            cid,
            hashCiphertext,
            block.timestamp
        );
        
        // Update current semester data
        semester.cid = cid;
        semester.hashCiphertext = hashCiphertext;
        semester.version = newVersion;
        semester.timestamp = block.timestamp;
        semester.exists = true;
    }
    
    // Check graduation eligibility
    function checkGraduation(address student) public view returns (bool) {
        bytes32 major = studentMajor[student];
        Curriculum storage curriculum = curriculums[major];
        
        // Check all required semesters exist
        for (uint i = 0; i < curriculum.requiredSemesters.length; i++) {
            bytes32 semesterId = curriculum.requiredSemesters[i];
            SemesterGrade storage semester = semesterGrades[student][semesterId];
            
            if (!semester.exists) {
                return false;
            }
            
            // Note: Detailed course validation done off-chain or via oracle
            // because parsing IPFS JSON on-chain is too expensive
        }
        
        return true;
    }
    
    // Calculate Merkle root from all semesters
    function calculateMerkleRoot(address student) public view returns (bytes32) {
        bytes32 major = studentMajor[student];
        Curriculum storage curriculum = curriculums[major];
        bytes32[] memory hashes = new bytes32[](curriculum.requiredSemesters.length);
        
        for (uint i = 0; i < curriculum.requiredSemesters.length; i++) {
            bytes32 semesterId = curriculum.requiredSemesters[i];
            hashes[i] = semesterGrades[student][semesterId].hashCiphertext;
        }
        
        // Simple merkle root: hash of all semester hashes
        return keccak256(abi.encodePacked(hashes));
    }
    
    // Mint diploma NFT - TRUSTLESS (Student calls directly)
    mapping(address => bool) public hasDiploma;
    
    function mintDiploma() external {
        address student = msg.sender;
        
        // Prevent double minting
        require(!hasDiploma[student], "Already graduated");
        
        // Check eligibility (all semesters completed)
        require(checkGraduation(student), "Not eligible: missing required semesters");
        
        // Calculate merkle root
        bytes32 merkleRoot = calculateMerkleRoot(student);
        
        // Mint NFT
        uint256 tokenId = _tokenIdCounter++;
        _safeMint(student, tokenId);
        hasDiploma[student] = true;
        
        emit DiplomaIssued(tokenId, student, merkleRoot, block.timestamp);
    }
    
    // Admin can revoke diploma in case of fraud (optional, with governance)
    function revokeDiploma(address student, uint256 tokenId) 
        external 
        onlyRole(ADMIN_ROLE) 
    {
        require(ownerOf(tokenId) == student, "Invalid token");
        _burn(tokenId);
        hasDiploma[student] = false;
    }
    
    // Get semester data
    function getSemesterGrade(address student, bytes32 semesterId) 
        external 
        view 
        returns (string memory cid, bytes32 hash, uint256 version, uint256 timestamp) 
    {
        SemesterGrade storage semester = semesterGrades[student][semesterId];
        require(semester.exists, "Semester not found");
        
        return (semester.cid, semester.hashCiphertext, semester.version, semester.timestamp);
    }
}
```

### So sánh chi phí Gas:

| Phương pháp | Giao dịch/học kỳ | Gas estimate | Chi phí (20 Gwei) |
|-------------|------------------|--------------|-------------------|
| **Mỗi môn = 1 record** | 10 giao dịch | ~1,000,000 gas | ~$0.60 |
| **Mỗi học kỳ = 1 record** ✅ | 1 giao dịch | ~150,000 gas | ~$0.09 |
| **Tiết kiệm** | **90%** | **85%** | **$0.51/học kỳ** |

---

## XIII. IMPLEMENTATION CHECKLIST

### Phase 1: Smart Contract Development
- [ ] Set up Hardhat project
- [ ] Implement role-based access control
- [ ] Implement grade submission function
- [ ] Implement event logging
- [ ] Implement currentGrade logic
- [ ] Implement curriculum management
- [ ] Implement graduation check
- [ ] Implement NFT minting
- [ ] Write unit tests (>80% coverage)

### Phase 2: IPFS Integration
- [ ] Set up IPFS node (or use Pinata/Infura)
- [ ] Implement encryption module (Web3.js/CryptoJS)
- [ ] Implement upload function
- [ ] Implement download & decrypt function
- [ ] Test end-to-end encryption

### Phase 3: Frontend Development
- [ ] **Lecturer Portal**
  - [ ] Connect wallet (MetaMask)
  - [ ] Grade input form
  - [ ] Encryption & IPFS upload
  - [ ] Submit transaction
  - [ ] Transaction status display
  
- [ ] **Student Portal**
  - [ ] Connect wallet
  - [ ] Timeline view (query events)
  - [ ] Decrypt & download grades
  - [ ] Request graduation button
  - [ ] View NFT diploma
  
- [ ] **Verifier Portal**
  - [ ] Upload file interface
  - [ ] Hash calculation
  - [ ] Blockchain query
  - [ ] Verification result display

### Phase 4: Testing & Deployment
- [ ] Local testing (Hardhat network)
- [ ] Testnet deployment (Sepolia/Goerli)
- [ ] Integration testing
- [ ] Security audit
- [ ] Mainnet deployment (if applicable)

---

## XIV. PHÂN TÍCH CHI PHÍ CHI TIẾT

### So sánh 2 phương pháp:

#### ❌ Phương pháp 1: Mỗi môn = 1 record (KHÔNG khuyến nghị)

**Kịch bản:** Sinh viên học 10 môn/học kỳ, 8 học kỳ

```
Giao dịch cần thiết:
- 10 môn × 8 học kỳ = 80 giao dịch
- Mỗi giao dịch: ~100,000 gas
- Tổng gas: 8,000,000 gas

Chi phí (giá gas = 20 Gwei, ETH = $2,000):
- 8,000,000 × 20 Gwei = 0.16 ETH
- 0.16 ETH × $2,000 = $320
```

**Nhược điểm:**
- 💸 Chi phí cao
- 🐌 Nhiều giao dịch → chậm
- 😰 Trải nghiệm người dùng kém

---

#### ✅ Phương pháp 2: Mỗi học kỳ = 1 record (KHUYẾN NGHỊ)

**Kịch bản:** Sinh viên học 10 môn/học kỳ, 8 học kỳ

```
Giao dịch cần thiết:
- 8 học kỳ = 8 giao dịch
- Mỗi giao dịch: ~150,000 gas (lớn hơn một chút vì lưu CID)
- Tổng gas: 1,200,000 gas

Chi phí (giá gas = 20 Gwei, ETH = $2,000):
- 1,200,000 × 20 Gwei = 0.024 ETH
- 0.024 ETH × $2,000 = $48
```

**Ưu điểm:**
- ✅ **Tiết kiệm 85%** ($320 → $48)
- ✅ Ít giao dịch hơn (80 → 8)
- ✅ Trải nghiệm tốt hơn
- ✅ Giảng viên nhập 1 lần cho cả học kỳ

---

### Bảng so sánh tổng quan:

| Tiêu chí | Mỗi môn | Mỗi học kỳ | Tiết kiệm |
|----------|---------|------------|----------|
| **Số giao dịch** | 80 | 8 | 90% |
| **Gas total** | 8M | 1.2M | 85% |
| **Chi phí ($)** | $320 | $48 | $272 |
| **Thời gian chờ** | Cao | Thấp | - |
| **UX giảng viên** | Nhập 80 lần | Nhập 8 lần | - |
| **Storage on-chain** | 80 records | 8 records | 90% |

---

### Chi phí cho Layer 2 (Polygon, Arbitrum):

**Phương pháp theo học kỳ trên Polygon:**
```
Gas price: ~30 Gwei (Polygon)
ETH price: Không áp dụng (dùng MATIC ~$0.8)

8 giao dịch × 150,000 gas × 30 Gwei = 36,000,000 Gwei
= 0.036 MATIC × $0.8 = $0.03
```

**Kết luận:** Layer 2 giảm chi phí xuống **gần như $0**

---

## XV. SECURITY CONSIDERATIONS

| Risk | Mitigation |
|------|-----------|
| **Private key exposure** | Use hardware wallet for admin/lecturer |
| **IPFS data availability** | Use pinning service (Pinata) |
| **Front-running** | Use commit-reveal pattern (if needed) |
| **Reentrancy** | Follow CEI pattern, use ReentrancyGuard |
| **Access control** | Use OpenZeppelin AccessControl |
| **Integer overflow** | Solidity 0.8+ has built-in protection |
| **Gas limit** | Optimize loops, batch operations |

---

## XV. FUTURE ENHANCEMENTS

- 🔄 **Multi-signature approval** for grade submission
- 🌐 **Multiple language support** for internationalization
- 📊 **Analytics dashboard** for administrators
- 🔔 **Notification system** via WebSocket
- 🎨 **Customizable NFT design** for different majors
- 🔗 **Integration with university ERP** systems
- ⚡ **Layer 2 solution** (Polygon/Arbitrum) for lower gas fees
- 🤖 **AI-powered fraud detection** on grade patterns

---

## XVI. REFERENCES

- [Ethereum Documentation](https://ethereum.org/developers)
- [OpenZeppelin Contracts](https://docs.openzeppelin.com/contracts)
- [IPFS Documentation](https://docs.ipfs.tech/)
- [Hardhat Documentation](https://hardhat.org/docs)
- [Web3.js Documentation](https://web3js.readthedocs.io/)
- [ERC-721 NFT Standard](https://eips.ethereum.org/EIPS/eip-721)
