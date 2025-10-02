# スマートコントラクトのチェックされていない外部呼び出しのテスト (Testing Unchecked External Calls in Smart Contracts)


### **説明**

チェックされていない外部呼び出しは、スマートコントラクトが呼び出しの結果を検証せずに別のコントラクトやアドレスへの外部呼び出しを行う場合に発生します。Ethereum では、外部呼び出しがサイレントに失敗し、呼び出し元のコントラクトが誤って呼び出しが成功したかのように処理を進める可能性があります。これは状態の不整合や潜在的な悪用につながります。この問題は delegatecall, send, call などの関数において、結果を明示的にチェックする必要があるため、特に危険です。

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

- 外部呼び出しが失敗し、その結果がチェックされていない場合、コントラクトは誤った仮定に基づいて処理を進める可能性があり、資金の損失やその他の予期しない動作につながる可能性があります。
- 検証されていない外部呼び出しはコントラクトの状態の誤った更新につながる可能性があり、エクスプロイトや論理的な不整合に対して脆弱になります。
- 攻撃者はこのような脆弱性を操作して、悪意のあるコードを実行したり、資金を複数回引き落としできます。


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
