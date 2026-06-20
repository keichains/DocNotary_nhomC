# CertiChain — dApp Công chứng & Cấp chứng chỉ phi tập trung

Hệ thống gồm 2 smart contract độc lập trên Ethereum Sepolia:

- **`CertificateRegistry`** — cấp / thu hồi / xác minh chứng chỉ cá nhân, có phân quyền Admin/Issuer/Recipient, lưu file gốc qua IPFS (mã hoá).
- **`DocumentNotary`** — công chứng hàng loạt tài liệu (batch) bằng Merkle Tree, không phân quyền, tối ưu gas (N file = 1 transaction).

---

## 1. Kiến trúc dự án

```text
dApp_FileStoring/
├── contracts/                  # Solidity: CertificateRegistry.sol, DocumentNotary.sol
├── scripts/deploy.ts           # Script deploy cả 2 contract
├── test/                       # Hardhat + Chai unit test (TypeScript)
├── merkle_tree/                # Rust core engine (PyO3 → module Python "merkle_rs")
│   ├── Cargo.toml
│   └── src/lib.rs              # sha256, build Merkle Tree, generate/verify proof
├── backend_api/                # FastAPI: hash file, mã hoá AES, pin IPFS, phân quyền truy cập
│   ├── main.py
│   ├── ipfs_service.py
│   └── .env                    # (tự tạo, không commit)
├── certichain-frontend/        # React + Vite + ethers.js
├── requirements.txt            # Dependency Python (backend)
├── package.json                # Dependency Node (Hardhat/Solidity)
└── .env                        # (tự tạo, không commit) — cấu hình Hardhat
```

**Luồng dữ liệu tổng quát:**
```text
[React/Vite Frontend] <-> ethers.js <-> [MetaMask] <-> [Sepolia Testnet]
        |                                          (CertificateRegistry / DocumentNotary)
        v  HTTP/fetch
[FastAPI Backend] --PyO3--> [Rust merkle_rs]  (hash, Merkle Tree, Merkle Proof)
        |
        +--AES-256-GCM encrypt/decrypt--> [Pinata API] --> [IPFS Network]
```

---

## 2. Yêu cầu môi trường

| Công cụ | Phiên bản gợi ý | Dùng cho |
|---|---|---|
| Node.js + npm | ≥ 18 | Hardhat, frontend |
| Python | ≥ 3.11 | Backend FastAPI |
| Rust + Cargo | ổn định mới nhất | Build module `merkle_rs` |
| [maturin](https://www.maturin.rs/) | mới nhất | Biên dịch Rust → Python extension |
| MetaMask | — | Ký giao dịch & message |
| Ví có Sepolia ETH | — | Deploy contract, gọi `issueCertificate`/`notarize` |
| RPC Sepolia | Alchemy / Infura | Kết nối Hardhat + backend tới Sepolia |
| Tài khoản Pinata | gói Free đủ dùng | Pin file mã hoá lên IPFS |

---

## 3. Cài đặt smart contract (Hardhat)

```bash
npm install --legacy-peer-deps
```

> ⚠️ **Bắt buộc dùng `--legacy-peer-deps`**: `package.json` hiện khai `chai@^6` nhưng `@nomicfoundation/hardhat-chai-matchers@2.1.2` (phụ thuộc của `hardhat-toolbox`) chỉ tương thích `chai@^4.x`. Nếu chạy `npm install` thường sẽ báo lỗi `ERESOLVE` ngay từ đầu. Cách sửa triệt để hơn (khuyến nghị làm sau): hạ `chai` về `^4.5.0` trong `package.json` rồi `npm install` lại bình thường.

Biên dịch contract:
```bash
npx hardhat compile
```

Chạy unit test:
```bash
npx hardhat test
```

---

## 4. Cấu hình biến môi trường cho Hardhat

Tạo file `.env` tại thư mục gốc:
```env
SEPOLIA_RPC_URL=your_sepolia_rpc_url
PRIVATE_KEY=your_wallet_private_key
```
- `SEPOLIA_RPC_URL`: RPC URL của mạng Sepolia (Alchemy/Infura).
- `PRIVATE_KEY`: private key ví dùng để deploy contract (**không** thêm tiền tố `0x`).

⚠️ Không commit file `.env` lên GitHub — dùng `.env.example` (mục 9) để chia sẻ template.

---

## 5. Deploy smart contract lên Sepolia

```bash
npx hardhat run scripts/deploy.ts --network sepolia
```

Terminal sẽ in ra địa chỉ 2 contract, ví dụ:
```text
CertificateRegistry : 0xd11eF4f5A843211C3a22B0Ad1B16fEee7214feC0
DocumentNotary       : 0x0F4d6F11aAa73Cb2bca320Fb9Ed4f05b77217355
```
Giữ lại 2 địa chỉ này — cần điền vào cả `backend_api/.env` (mục 6) và `certichain-frontend/.env` (mục 7).

---

## 6. Cài đặt & cấu hình Backend (FastAPI)

### 6.1 Build module Rust `merkle_rs`
```bash
cd merkle_tree
maturin develop --features extension-module
cd ..
```
Lệnh này biên dịch `merkle_tree/src/lib.rs` thành module Python `merkle_rs`, cài thẳng vào virtualenv đang active. Cần chạy lại mỗi khi sửa code Rust.

### 6.2 Cài dependency Python
```bash
python -m venv venv
venv\Scripts\activate          # Windows
# source venv/bin/activate     # macOS/Linux

pip install -r requirements.txt --break-system-packages
```

### 6.3 Tạo `backend_api/.env`
```env
SEPOLIA_RPC_URL="https://eth-sepolia.g.alchemy.com/v2/YOUR_ALCHEMY_KEY"
PRIVATE_KEY="your_wallet_private_key"
PINATA_JWT="your_pinata_jwt_token"
CONTRACT_ADDRESS="0x...địa_chỉ_CertificateRegistry_vừa_deploy"
FILE_ENCRYPTION_KEY="..."
```

**Tạo Pinata API Key (PINATA_JWT):**
1. Vào https://app.pinata.cloud/developers/api-keys → **New Key**
2. Bật quyền `pinFileToIPFS` (Pinning Services API) → **Create Key**
3. Copy JWT (chỉ hiện 1 lần) vào `.env`. Không dùng chung JWT giữa các thành viên.

**Tạo `FILE_ENCRYPTION_KEY` (khoá mã hoá AES-256 cho file pin lên IPFS):**
```bash
python -c "import os,base64;print(base64.b64encode(os.urandom(32)).decode())"
```
⚠️ Nếu nhiều người cùng chạy chung 1 backend thật, **phải dùng chung một giá trị này** — backend nào mã hoá file thì backend đó (và chỉ backend đó) mới giải mã được. Đừng tự generate riêng mỗi máy nếu deploy backend dùng chung.

### 6.4 Chạy backend
```bash
python -m uvicorn backend_api.main:app --host 127.0.0.1 --port 8000 --reload
```
Kiểm tra: mở `http://127.0.0.1:8000/api/health` phải trả `200 OK`.

---

## 7. Cài đặt Frontend

```bash
cd certichain-frontend
npm install
```

Tạo file `.env` trong `certichain-frontend/`:
```env
VITE_BACKEND_URL=http://127.0.0.1:8000
VITE_CONTRACT_ADDRESS=0x...địa_chỉ_CertificateRegistry
VITE_NOTARY_ADDRESS=0x...địa_chỉ_DocumentNotary
```

---

## 8. Chạy frontend

```bash
npm run dev
```
Mở `http://localhost:5173`, kết nối MetaMask, chuyển sang:
```text
Sepolia Testnet — Chain ID: 11155111
```

**Quy trình chạy đầy đủ (3 process song song):**
```bash
# Terminal 1 — Backend
python -m uvicorn backend_api.main:app --host 127.0.0.1 --port 8000 --reload

# Terminal 2 — Frontend
cd certichain-frontend && npm run dev

# (Smart contract đã deploy sẵn lên Sepolia ở bước 5, không cần chạy thêm gì)
```

---

## 9. Các chức năng chính

### `CertificateRegistry` — quản lý chứng chỉ cá nhân
- Cấp chứng chỉ (`issueCertificate`, chỉ Issuer)
- Xem danh sách / xem chứng chỉ của ví hiện tại
- Xác minh chứng chỉ (`verifyCertificate`, không tốn gas)
- Thu hồi chứng chỉ (`revokeCertificate`, Admin hoặc đúng Issuer)
- Quản lý Issuer/Admin (`grantIssuer`, `revokeIssuer`)
- **Xem file gốc có kiểm soát**: ký message off-chain (`personal_sign`) → backend xác thực vai trò (Admin/Issuer/Recipient) qua `eth_call` → giải mã file đã mã hoá AES-256-GCM trên IPFS và trả về. Không tốn gas, không trả link IPFS công khai.

### `DocumentNotary` — công chứng batch tài liệu
- Chọn nhiều file → backend (Rust `merkle_rs`) hash từng file
- Tạo Merkle Root đại diện cho cả batch
- Ghi Merkle Root lên blockchain (`notarize`, không phân quyền, 1 transaction cho cả batch)
- Verify từng file bằng Merkle Proof ngay on-chain (`verifyDocument`, dùng `sha256` + domain-separation prefix `0x00`/`0x01` đồng bộ với cách Rust tính hash)

```text
CertificateRegistry = quản lý nghiệp vụ chứng chỉ cá nhân + lưu trữ file có kiểm soát
DocumentNotary       = chứng minh hàng loạt tài liệu/hồ sơ gốc tồn tại và không bị sửa đổi
```

---

## 10. Bảo mật file IPFS — lưu ý quan trọng

Gói Pinata Free/Picnic **không** hỗ trợ Private IPFS Network (chỉ Enterprise). Vì `getCertificate()` là hàm `public view`, CID của file luôn đọc được bởi bất kỳ ai trên Ethereum. Để tránh lộ file qua public gateway, hệ thống:
1. **Mã hoá** file bằng AES-256-GCM trước khi pin (khoá `FILE_ENCRYPTION_KEY`, chỉ backend giữ).
2. **Không bao giờ** trả URL gateway công khai cho frontend.
3. Chỉ giải mã file trong bộ nhớ backend, sau khi xác thực chữ ký + vai trò (Admin/Issuer/Recipient).

→ Nếu cần nâng cấp bảo mật hơn nữa: chuyển sang Pinata Enterprise (Private IPFS) hoặc dùng KMS để quản lý `FILE_ENCRYPTION_KEY` thay vì lưu trong `.env`.

---

## 11. Ghi chú khắc phục lỗi thường gặp

| Lỗi | Nguyên nhân | Cách xử lý |
|---|---|---|
| `GET /api/health net::ERR_CONNECTION_REFUSED` | Backend chưa chạy / sai port | Chạy lại bước 6.4. Không ảnh hưởng các chức năng frontend không phụ thuộc backend (kết nối ví, gọi contract trực tiếp). |
| `npm install` báo `ERESOLVE` / peer dependency conflict | `chai@^6` xung đột `hardhat-chai-matchers` | Dùng `npm install --legacy-peer-deps`, hoặc hạ `chai` về `^4.5.0`. |
| `UnicodeEncodeError: 'latin-1' codec can't encode...` khi xem file có tên tiếng Việt | HTTP header không hỗ trợ Unicode trực tiếp | Đã xử lý ở `main.py` bằng `quote()` + `filename*=UTF-8''...` (RFC 5987) — đảm bảo đang chạy đúng bản code mới nhất. |
| Holder (không liên quan chứng chỉ) vẫn xem được file qua link dán trực tiếp | Dùng public gateway IPFS không mã hoá | Đảm bảo `ipfs_service.py` đang mã hoá AES trước khi pin (mục 10), không trả `ipfsURL` thô. |
| `maturin develop` báo thiếu Rust/Cargo | Chưa cài Rust toolchain | Cài qua https://rustup.rs trước khi chạy lại. |

---

## 12. Không commit các file/thư mục sau

```text
node_modules/
certichain-frontend/node_modules/
.env
.env.local
backend_api/.env
cache/
artifacts/
dist/
build/
__pycache__/
*.pyc
venv/
target/
```

Dùng file `.gitignore` đã có sẵn ở root. Nếu trước đó lỡ commit `.env` hoặc `__pycache__/`, gỡ khỏi tracking (không xoá file thật):
```bash
git rm --cached -r backend_api/__pycache__ python_blockchain/__pycache__ 2>/dev/null
git rm --cached .env backend_api/.env 2>/dev/null
git commit -m "chore: gỡ secrets/pycache khỏi git tracking"
```

Tạo `.env.example` để mô tả biến môi trường cần thiết, **không chứa giá trị thật**:

**`.env.example`** (root):
```env
SEPOLIA_RPC_URL=
PRIVATE_KEY=
```

**`backend_api/.env.example`**:
```env
SEPOLIA_RPC_URL=
PRIVATE_KEY=
PINATA_JWT=
CONTRACT_ADDRESS=
FILE_ENCRYPTION_KEY=
```

**`certichain-frontend/.env.example`**:
```env
VITE_BACKEND_URL=http://127.0.0.1:8000
VITE_CONTRACT_ADDRESS=
VITE_NOTARY_ADDRESS=
```

---

## 13. Quy trình chạy nhanh (tóm tắt toàn bộ)

```bash
# 1. Cài & build phần smart contract + Rust
npm install --legacy-peer-deps
npx hardhat compile
cd merkle_tree && maturin develop --features extension-module && cd ..

# 2. Deploy contract lên Sepolia
npx hardhat run scripts/deploy.ts --network sepolia
# → copy 2 địa chỉ vừa in ra vào backend_api/.env và certichain-frontend/.env

# 3. Cài & chạy backend (terminal riêng)
python -m venv venv && venv\Scripts\activate
pip install -r requirements.txt --break-system-packages
python -m uvicorn backend_api.main:app --host 127.0.0.1 --port 8000 --reload

# 4. Cài & chạy frontend (terminal riêng)
cd certichain-frontend
npm install
npm run dev
```

Mở `http://localhost:5173`, kết nối MetaMask trên Sepolia Testnet để bắt đầu sử dụng.
