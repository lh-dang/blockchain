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
- **Lưu trữ điểm theo dạng lịch sử bất biến** (event log)
- **Học cải thiện**: mỗi lần thi = 1 record, không xoá bản cũ
- **Xét tốt nghiệp tự động** dựa trên kết quả đã đạt
- **Cấp bằng tốt nghiệp** dưới dạng NFT duy nhất, chứa toàn bộ hành trình học tập (thông qua Merkle root hoặc metadata tổng hợp)
- **Giảng viên** có trang nhập điểm
- **Sinh viên** có trang xem timeline học tập
- **Doanh nghiệp** có trang verify bằng và bảng điểm
- **Tối đa hóa** tính minh bạch, bất biến, bảo mật

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

**On-chain chỉ lưu:**

- `studentAddress` (ví MetaMask)
- `courseId` (mã môn dạng bytes32)
- `scorePublic` (điểm rút gọn → ví dụ 85 = 8.5 điểm, hoặc pass/fail)
- `exists` flag
- `attemptId` hiện hành

> ❌ **KHÔNG có**: Họ tên, MSSV, lớp, khóa, ngành, ngày sinh

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

**Quy trình nhập điểm:**

1. **Giao diện web** của giảng viên thực hiện:
   - Nhập điểm cho sinh viên
   - Tạo JSON chi tiết
   - Mã hóa bằng public key của sinh viên
   - Upload lên IPFS → nhận **CID**
   - Tính `hashCiphertext = keccak256(ciphertext)`

2. **Gửi giao dịch** `submitGrade()` lên smart contract:

```solidity
submitGrade(
    studentAddress,
    courseId,
    scorePublic,
    cid,
    hashCiphertext
)
```

**Smart contract thực hiện 2 việc:**

#### 2.1. Lưu lịch sử điểm (immutable event)

```solidity
event GradeAttemptRecorded(
    address indexed student,
    bytes32 indexed courseId,
    uint256 attemptId,
    uint256 scorePublic,
    string cid,
    bytes32 hashCiphertext,
    uint256 timestamp
);
```

→ Phục vụ timeline, audit log, doanh nghiệp verify

#### 2.2. Cập nhật điểm hiệu lực (currentGrade)

- Nếu điểm mới **cao hơn** điểm cũ → cập nhật
- Nếu **thấp hơn** → giữ điểm cũ
- → Phục vụ xét tốt nghiệp

---

### 3. Sinh viên (Student)

#### 3.1. Xem timeline học tập

**Frontend thực hiện:**
- Query event `GradeAttemptRecorded(student, *)`
- Sort theo thời gian
- Render dòng thời gian:
  ```
  📅 Lần thi 1 → điểm 6.5
  📅 Lần thi cải thiện → 8.0
  📎 CID → có thể click tải về và giải mã bằng private key
  ```

#### 3.2. Yêu cầu xét tốt nghiệp

**Khi bấm nút "Xét tốt nghiệp":**

Contract đọc:
1. Danh sách môn bắt buộc từ `curriculum`
2. `currentGrade[student][courseId]`

**Điều kiện:**
- Nếu mọi môn đều có điểm **≥ 5.0** → cho phép mint NFT

---

### 4. Cấp bằng tốt nghiệp (Mint NFT)

#### 4.1. Khi điều kiện tốt nghiệp thỏa

**Contract sẽ:**
1. Tính **Merkle root** hoặc metadata tổng hợp từ danh sách môn & điểm hiệu lực
2. Mint **1 NFT duy nhất** cho sinh viên
3. Metadata NFT chứa:
   - Merkle root → đại diện toàn bộ quá trình học
   - Link verify
   - Thời gian tốt nghiệp

#### 4.2. NFT = Bằng tốt nghiệp

**Doanh nghiệp chỉ cần:**
- `tokenId` → verify tồn tại
- Metadata → verify Merkle root
- Hash on-chain = hash của IPFS ciphertext → xác thực file bảng điểm sinh viên cung cấp

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
| 🔒 **Immutable** | Giảng viên không thể sửa điểm cũ → chỉ tạo attempt mới |
| ✅ **Chống giả mạo** | Sinh viên không thể giả mạo điểm vì: CID mã hóa, Hash on-chain, Mint NFT final |
| 🏢 **Verify độc lập** | Doanh nghiệp verify độc lập, không phụ thuộc nhà trường |

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

| Thành phần | Cách thức |
|------------|-----------|
| 📚 **Môn học** | Event log + IPFS mã hóa |
| 📊 **Điểm sử dụng** | Mapping `currentGrade` |
| 🎓 **Xét tốt nghiệp** | So môn theo curriculum |
| 📝 **Mỗi lần thi** | Attempt mới, không xóa |
| 🏆 **NFT bằng** | Mint 1 NFT bằng tốt nghiệp cuối cùng |

### Công thức thành công

```
Blockchain = Sự thật
IPFS = Kho dữ liệu mã hóa
Frontend = Trải nghiệm
```

> ⚠️ **Nguyên tắc vàng**: Không có thông tin nhạy cảm on-chain

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

### Flow 1: Nhập điểm

```
Lecturer
   │
   ├─► 1. Input grade data
   │
   ├─► 2. Create JSON
   │       {student: "0x...", course: "CS101", score: 8.5, ...}
   │
   ├─► 3. Encrypt with student's public key
   │       ciphertext = encrypt(JSON, studentPubKey)
   │
   ├─► 4. Upload to IPFS
   │       → CID: "Qm..."
   │
   ├─► 5. Calculate hash
   │       hashCiphertext = keccak256(ciphertext)
   │
   └─► 6. Submit to blockchain
           submitGrade(studentAddr, courseId, 85, CID, hash)
                │
                ├─► Emit: GradeAttemptRecorded(...)
                │
                └─► Update: currentGrade[student][course]
```

### Flow 2: Xét tốt nghiệp

```
Student
   │
   ├─► 1. Click "Request Graduation"
   │
   └─► 2. Contract checks:
           │
           ├─► Read curriculum.requiredCourses[]
           │
           ├─► For each course:
           │   └─► currentGrade[student][course] >= 5.0 ?
           │
           ├─► All passed?
           │   │
           │   ├─► YES → Calculate Merkle Root
           │   │          │
           │   │          └─► mintDiplomaNFT(student, metadata)
           │   │                   │
           │   │                   └─► Emit: DiplomaIssued(tokenId, student)
           │   │
           │   └─► NO → Revert "Not eligible"
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

### Sequence: Check Graduation & Mint NFT

```
Student    Frontend    Smart Contract    NFT Contract    Blockchain
   │           │              │                 │             │
   ├─Request──►│              │                 │             │
   │           │              │                 │             │
   │           ├─checkGraduation()─────►│       │             │
   │           │              │         │       │             │
   │           │              ├─Read curriculum─┤             │
   │           │              │         │       │             │
   │           │              ├─Check all grades┤             │
   │           │              │         │       │             │
   │           │              ├─Calculate Merkle│             │
   │           │              │         │       │             │
   │           │              ├─mintNFT()──────►│             │
   │           │              │         │       ├─Mint NFT───►│
   │           │              │         │       │             │
   │           │              │         │◄──tokenId───────────┤
   │           │              │         │       │             │
   │           │◄─────────Success───────┤       │             │
   │◄─Success──┤              │         │       │             │
```

---

## XII. SMART CONTRACT STRUCTURE

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
    
    // Grade data structure
    struct Grade {
        uint256 scorePublic;      // e.g., 85 = 8.5
        string cid;               // IPFS CID
        bytes32 hashCiphertext;   // Hash of encrypted data
        uint256 attemptId;        // Current attempt number
        bool exists;
    }
    
    // Curriculum structure
    struct Curriculum {
        bytes32[] requiredCourses;
        uint256 minPassScore;     // e.g., 50 = 5.0
    }
    
    // Storage
    mapping(address => mapping(bytes32 => Grade)) public currentGrade;
    mapping(bytes32 => Curriculum) public curriculums; // major => curriculum
    mapping(address => bytes32) public studentMajor;
    
    uint256 private _tokenIdCounter;
    
    // Events
    event GradeAttemptRecorded(
        address indexed student,
        bytes32 indexed courseId,
        uint256 attemptId,
        uint256 scorePublic,
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
    
    // Submit grade
    function submitGrade(
        address student,
        bytes32 courseId,
        uint256 scorePublic,
        string memory cid,
        bytes32 hashCiphertext
    ) external onlyRole(LECTURER_ROLE) {
        Grade storage grade = currentGrade[student][courseId];
        
        uint256 newAttemptId = grade.attemptId + 1;
        
        // Emit immutable event
        emit GradeAttemptRecorded(
            student,
            courseId,
            newAttemptId,
            scorePublic,
            cid,
            hashCiphertext,
            block.timestamp
        );
        
        // Update current grade (only if better)
        if (!grade.exists || scorePublic > grade.scorePublic) {
            grade.scorePublic = scorePublic;
            grade.cid = cid;
            grade.hashCiphertext = hashCiphertext;
        }
        
        grade.attemptId = newAttemptId;
        grade.exists = true;
    }
    
    // Check graduation eligibility
    function checkGraduation(address student) external view returns (bool) {
        bytes32 major = studentMajor[student];
        Curriculum storage curriculum = curriculums[major];
        
        for (uint i = 0; i < curriculum.requiredCourses.length; i++) {
            bytes32 courseId = curriculum.requiredCourses[i];
            Grade storage grade = currentGrade[student][courseId];
            
            if (!grade.exists || grade.scorePublic < curriculum.minPassScore) {
                return false;
            }
        }
        
        return true;
    }
    
    // Mint diploma NFT
    function mintDiploma(address student, bytes32 merkleRoot) 
        external 
        onlyRole(ADMIN_ROLE) 
    {
        require(this.checkGraduation(student), "Not eligible");
        
        uint256 tokenId = _tokenIdCounter++;
        _safeMint(student, tokenId);
        
        emit DiplomaIssued(tokenId, student, merkleRoot, block.timestamp);
    }
}
```

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

## XIV. SECURITY CONSIDERATIONS

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
