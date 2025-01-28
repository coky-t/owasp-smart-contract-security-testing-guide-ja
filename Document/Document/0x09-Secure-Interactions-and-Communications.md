# スマートコントラクトのチェックされていない外部呼び出しのテスト (Testing Unchecked External Calls in Smart Contracts)


### **説明**

Unchecked external calls occur when a smart contract makes an external call to another contract or address without verifying the call's outcome. In Ethereum, external calls may fail silently, and the calling contract may mistakenly proceed as if the call succeeded. This leads to state inconsistencies and potential exploitation. The issue is particularly risky in functions like delegatecall, send, or call, where the outcome must be explicitly checked.

### **例: 適切なアクセス制御のないコード**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.4.24;

contract Proxy {
    address public owner;

    constructor() public {
        owner = msg.sender;
    }

    function forward(address callee, bytes _data) public {
        require(callee.delegatecall(_data)); // Unchecked external call vulnerability
    }
}
```
### **影響**

- When external calls fail and their results are unchecked, the contract can proceed under incorrect assumptions, leading to potential loss of funds or other unexpected behaviors.
- Unverified external calls can lead to incorrect updates to the contract state, making it vulnerable to exploits and logical inconsistencies.
- Attackers can manipulate such vulnerabilities to execute malicious code or withdraw funds multiple times.


### **対策**


- Use safer methods such as transfer() over send() when transferring Ether. The transfer() function reverts automatically if the call fails.
- For low-level functions like call and delegatecall, always check the return value and handle failures appropriately.
- Limit interactions with untrusted contracts and ensure robust validation before performing critical operations.

---

### **テスト 1: 安全なコントラクトインタラクションを確認する**

#### 脆弱なコード:

```solidity
pragma solidity ^0.5.0;

contract VulnerableContract {
    address public trustedAddress;

    constructor(address _trustedAddress) public {
        trustedAddress = _trustedAddress;
    }

    function callExternalContract(address _to, uint256 _value) public {
        _to.call{value: _value}(""); // Vulnerable to reentrancy or other attacks
    }
}
```

#### **なぜ脆弱なのか**
- The contract uses `.call` to make external calls, which is generally unsafe and susceptible to reentrancy attacks.
- The function allows anyone to trigger the call and transfer funds, exposing the contract to potential attacks.


#### 修正されたコード:

```solidity
pragma solidity ^0.8.0;

contract SecureContract {
    address public trustedAddress;

    constructor(address _trustedAddress) {
        trustedAddress = _trustedAddress;
    }

    function callExternalContract(address _to, uint256 _value) public {
        require(_to != address(0), "Invalid address");
        (bool success, ) = _to.call{value: _value}("");
        require(success, "External call failed");
    }
}
```
#### **チェック方法**
- **Code Review:** Look for external calls in the contract and ensure that they are using safe methods such as `transfer` or `send` where appropriate, or implementing reentrancy protection.
- **Static Analysis:** Use static analysis tools like SolidityScan, MythX or Slither to detect unsafe calls in the code.
- **Dynamic Testing:** Simulate a reentrancy attack by deploying a contract that calls the vulnerable contract and tries to drain funds.
