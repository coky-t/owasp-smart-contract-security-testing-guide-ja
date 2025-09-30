# アクセス制御と認証のテスト (Testing Access Control and Authentication Vulnerabilties)

### **説明**

不適切なアクセス制御はスマートコントラクトにおける重大なセキュリティ脆弱性であり、認可されていないユーザーが機密性の高い機能やデータにアクセスしたり、変更できる場合に発生します。この問題は一般的に、コードがユーザーのパーミッションに基づいた厳格なアクセス制御を適用していない場合に発生します。

アクセス制御の脆弱性は、以下のような、ガバナンスや重要な操作に関わるシナリオにおいて特に深刻です。

- トークンの発行
- 提案への投票
- 資金の引き落とし
- コントラクトの一時停止またはアップグレード
- コントラクトの所有者の変更

### **例: 適切なアクセス制御のないコード**


```solidity
function burn(address account, uint256 amount) public 
{
    // No access control is implemented for the burn function
    _burn(account, amount); 
}
```
### **影響**

- 攻撃者はコントラクト内の重要な機能やデータに不正アクセスし、その完全性とセキュリティを侵害する可能性があります。
- 脆弱性はコントラクトで管理されている資金や資産の盗難につながり、ユーザーや利害関係者に重大な経済的損害をもたらす可能性があります。


### **対策**


- 初期化関数は認可されたエンティティによってのみ一度だけ呼び出されるようにします。
- コントラクトにおいて Ownable や RBAC (Role-Based Access Control) などの確立されたアクセス制御パターンを使用してパーミッションを管理し、認可されたユーザーのみが特定の機能にアクセスできるようにします。これは onlyOwner などの適切なアクセス制御修飾子や、機密性の高い機能へのカスタムロールを追加することで実現できます。

### 不適切なアクセス制御攻撃の被害を受けたスマートコントラクトの事例:

- [HospoWise Hack Analysis](https://blog.solidityscan.com/access-control-vulnerabilities-in-smart-contracts-a31757f5d707)
- [LAND NFT Hack Analysis](https://blog.solidityscan.com/land-hack-analysis-missing-access-control-66fb9555a3e3)

## マルチ署名スキームのテスト (Testing Multi-Signature Schemes)


重要な操作にはマルチ署名スキームが実装されていることを確認し、複数の承認者からの承認を要求することで、セキュリティを強化し、不正なアクションのリスクを軽減します。

- 機密性の高い操作を実行する前に、マルチ署名ロジックが構成可能な承認の閾値を確保していることを検証します。
- すべての署名者が認証されており、重複した署名が拒否されることを確認します。
- 不十分な承認や改竄されたデータなど、エッジケースをテストします。

```solidity
// Example of multi-signature scheme
pragma solidity ^0.8.0;

contract MultiSigWallet {
    address[] public owners;
    uint public requiredApprovals;
    mapping(address => bool) public isOwner;
    mapping(uint => mapping(address => bool)) public approvals;

    struct Transaction {
        address to;
        uint value;
        bool executed;
    }

    Transaction[] public transactions;

    constructor(address[] memory _owners, uint _requiredApprovals) {
        require(_owners.length > 0, "Owners required");
        require(_requiredApprovals > 0 && _requiredApprovals <= _owners.length, "Invalid approval count");

        for (uint i = 0; i < _owners.length; i++) {
            isOwner[_owners[i]] = true;
        }
        owners = _owners;
        requiredApprovals = _requiredApprovals;
    }

    function submitTransaction(address _to, uint _value) public {
        require(isOwner[msg.sender], "Not an owner");
        transactions.push(Transaction({to: _to, value: _value, executed: false}));
    }

    function approveTransaction(uint _txIndex) public {
        require(isOwner[msg.sender], "Not an owner");
        require(!approvals[_txIndex][msg.sender], "Already approved");

        approvals[_txIndex][msg.sender] = true;
    }

    function executeTransaction(uint _txIndex) public {
        require(transactions[_txIndex].executed == false, "Already executed");

        uint approvalCount = 0;
        for (uint i = 0; i < owners.length; i++) {
            if (approvals[_txIndex][owners[i]]) {
                approvalCount++;
            }
        }

        require(approvalCount >= requiredApprovals, "Not enough approvals");

        transactions[_txIndex].executed = true;
        payable(transactions[_txIndex].to).transfer(transactions[_txIndex].value);
    }

    receive() external payable {}
}
```
ロジックをレビューして、すべての承認が実行前に検証され、重複する承認が防止され、トランザクションデータの完全性が維持されていることを確認します。さまざまな承認シナリオでテストを行い、エッジケースの適切な処理を検証します。

___

## アイデンティティ検証のテスト (Identity Verification Test)


予期しないアドレスが、特に、これらのアドレスが同じプロトコル内のコントラクトを参照している場合、意図しない動作を引き起こさないことを検証します。

- コントラクトとやり取りする際、機密性の高い操作を実行する前に予期しないアドレスが適切に検証されていることを確認します。
```solidity
require(address(contract) != address(0), "Invalid address");
```

予期しない ecrecover 出力から生じる脆弱性を回避するために、ecrecover などの関数がすべての潜在的な null アドレスを適切に処理することを検証します。

- ecrecover が空または null アドレスを処理しないことを確認します。
```solidity
address recovered = ecrecover(messageHash, v, r, s);
require(recovered != address(0), "Invalid signature");
```

___

## 最小権限の原則のテスト (Least Privilege Principle Test)


悪意のあるコントラクトからの潜在的な悪用を避けるため、認可には tx.origin ではなく msg.sender を使用します。送信者が EDA であることを確認するために require(tx.origin == msg.sender) などのチェックを含めます。

- tx.origin は悪意のあるコントラクトによって悪用され、システムを騙して、何も知らないユーザーの代わりにアクションを実行する可能性があります。メッセージの直接の送信者を参照するため、msg.sender が推奨されます。
```solidity
require(msg.sender == owner, "Not the owner");
require(tx.origin == msg.sender, "Only EOA can execute");
```

特定のアドレス (LUSD など) はトークンの受信からブロックまたは制限されているかもしれません。アドレス制限が適切に管理および検証されていることを確認します。

- 特定のアドレス (LUSD など) がトークンの受信からブロックされる必要がある場合、これらのアドレスを制限するために実施されるチェックがあることを確認します。
```solidity
address restrictedAddress = 0x123...;  // Example of a restricted address
require(msg.sender != restrictedAddress, "Restricted address cannot perform this operation");
```


重要なセキュリティチェックを実施するために Guard のフック (checkTransaction(), checkAfterExecution() など) が実行されていることを確認します。

- Guard コントラクトを使用している場合、checkTransaction() や checkAfterExecution() などのフックが適切に実装され、セキュリティ条件が適用されるようにします。
```solidity
function checkTransaction() internal {
    // Add conditions to verify transaction before execution
}

function checkAfterExecution() internal {
    // Add conditions to verify transaction after execution
}
```


---

## 重要な機能でのアクセス制御のテスト (Test Access Control on Critical Functions)


Ensure that access controls are implemented correctly to determine who can use certain functions, and avoid unauthorized changes or withdrawals.

- Ensure that functions requiring specific roles or permissions are restricted properly using onlyOwner or role-based checks.
```solidity
    modifier onlyOwner() {
        require(msg.sender == owner, "Caller is not the owner");
        _;
    }

    function withdraw() external onlyOwner {
        // Only the owner can withdraw
    }
```

---

## 期限付きパーミッションのテスト (Test Timed Permissions)



Ensure that access controls are implemented correctly to determine who can use certain functions, and avoid unauthorized changes or withdrawals.

- Ensure that functions requiring specific roles or permissions are restricted properly using onlyOwner or role-based checks.

```solidity
    modifier onlyOwner() {
        require(msg.sender == owner, "Caller is not the owner");
        _;
    }

    function withdraw() external onlyOwner {
        // Only the owner can withdraw
    }
```

---

## マークルツリーを使用するアクセス制御のテスト (Test Access Control Using Merkle Trees)


Ensure that `msg.sender` validation is properly implemented when using Merkle trees to maintain security and prevent unauthorized access.

- When using Merkle trees to authenticate users or grant permissions, ensure that the contract verifies that `msg.sender` matches the expected address and Merkle proof. This prevents unauthorized actors from bypassing security by submitting incorrect proofs.

```solidity
    require(verifyMerkleProof(msg.sender, merkleProof), "Invalid Merkle proof");
```

- Use whitelisting to restrict interactions to a specific set of addresses, providing additional security against malicious actors.

- Implement a whitelisting mechanism that allows only approved addresses to interact with specific functions. Ensure that only addresses explicitly added to the whitelist are able to execute sensitive operations.

```solidity
    address[] public whitelist;

    modifier onlyWhitelisted() {
        bool isWhitelisted = false;
        for (uint i = 0; i < whitelist.length; i++) {
            if (msg.sender == whitelist[i]) {
                isWhitelisted = true;
                break;
            }
        }
        require(isWhitelisted, "Address not whitelisted");
        _;
    }

    function addToWhitelist(address _address) external onlyOwner {
        whitelist.push(_address);
    }
```
- Ensure that functions modifying the contract state or accessing sensitive operations have proper access controls implemented.

- Critical functions, such as those that modify contract state or handle sensitive information, should only be callable by authorized addresses (e.g., the owner or an admin). Use modifiers to enforce access controls for these functions.

```solidity
    modifier onlyOwner() {
        require(msg.sender == owner, "Not the contract owner");
        _;
    }

    function modifyContractState() external onlyOwner {
        // Logic to modify contract state
    }
```
