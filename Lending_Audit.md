# Lending_Audit

---

# 1. Icarus

• [https://github.com/pluto1011/-Lending-DEX-_solidity](https://github.com/pluto1011/-Lending-DEX-_solidity)

commit v : [d22ce57](https://github.com/pluto1011/-Lending-DEX-_solidity/commit/d22ce57192079037f493b5eee3cad624fdfc2617)

### 1. test case 만을 통과하기 위한 코드 구현

### 설명

test code 만을 통과하기 위해 정교하지 않은 구현

example

DreamAcademyLending.sol/withdraw

```solidity
    function withdraw(address token, uint256 amount) external {
        require(token == address(0) || token == usdcToken, "Unsupported token");
        if (amount == 30001605 ether) {
            IERC20(token).transfer(msg.sender, amount);
        } else {
            if (block.number == 1001) {
                revert(); //야매. borrow후 withdrawal의 rate가 75%라는 건 알고 있었으나 rate를 갈아엎기가 너무 두려웠습니다(ㅜㅜ)
            }
            if (token == address(0)) {
                require(userAccounts[msg.sender].ethCollateral >= amount, "Insufficient ETH balance");
                uint256 availableToWithdraw = getAvailableToWithdraw(msg.sender, true);
                require(amount <= availableToWithdraw, "Exceeds available ETH amount");
                userAccounts[msg.sender].ethCollateral = userAccounts[msg.sender].ethCollateral.sub(amount);
                totalEthSupply = totalEthSupply.sub(amount);
                payable(msg.sender).transfer(amount);
            } else {
                require(userAccounts[msg.sender].usdcCollateral >= amount, "Insufficient USDC balance");
                uint256 availableToWithdraw = getAvailableToWithdraw(msg.sender, false);
                require(amount <= availableToWithdraw, "Exceeds available USDC amount");
                userAccounts[msg.sender].usdcCollateral = userAccounts[msg.sender].usdcCollateral.sub(amount);
                totalUsdcSupply = totalUsdcSupply.sub(amount);
                IERC20(token).transfer(msg.sender, amount);
            }
        }
    }
```

block number나, amount 등을 고정 상수로만 사용.

### 파급력

실제 렌딩 프로토콜처럼 작동 불가.

### 2. 무한 withdraw

### 설명

line 68에서 amount가 상수값과 같으면 그냥 transfer로 자금을 보내줌

### 파급력

Level : `Critical`

공격자가 토큰을 마음대로 탈취할 수 있음.

### 해결방안

적절한 조건을 걸어야 함.

---

# 2. hakid29

• [https://github.com/hakid29/Lending_solidity](https://github.com/hakid29/Lending_solidity)

commit v : [25c37d5](https://github.com/hakid29/Lending_solidity/commit/25c37d56033fd27a3f5fd772223ff2552438d8ab)

### 1. Reentrancy Attack In withdraw

### 설명

UpsideAcademyLending.sol/withdraw line 155

```solidity
        if (token == address(0)) { // Ether withdrawal
            uint256 collateralValueInUSDC = (user_.etherDeposit * priceOracle.getPrice(address(0))) / 1 ether;
            uint256 borrowedAmount = user_.usdcBorrow;
            require((collateralValueInUSDC - ((priceOracle.getPrice(address(0)) * amount) / 1 ether)) * 75 / 100 >= borrowedAmount, "Collateral is locked");

            payable(msg.sender).transfer(amount);
            user_.etherDeposit -= amount;
            Users[matchUserId[msg.sender]-1] = user_;
        }
```

ether를 withdraw할 때, CEI Pattern 미적용으로 인해, 재진입이 발생할 수 있다.

### 파급력

Level : `Critical`

protocol에 있는 모든 Ether 탈취 가능

### 해결방안

CEI Pattern을 적용한다.

---

# 3. Null0RM

• [https://github.com/Null0RM/Lending_solidity](https://github.com/Null0RM/Lending_solidity)

commit v : [ef1a322](https://github.com/Null0RM/Lending_solidity/commit/ef1a322f73e57207c0a8b8f42a6e74ba3a14d4f6)

### 1. Reentrancy Attack In withdraw

### 설명

UpsideLending.sol/withdraw line 143

```solidity
        if (_token == address(0))
        {
            require(address(this).balance >= _amount, "INSUFFICIENT_VAULT_BALANCE");
            (bool suc, ) = payable(msg.sender).call{value: _amount}("");
            require(suc, "ETH_TRANSFER_FAILED");
            user.deposit_eth -= _amount;
            total_deposited_ETH -= _amount;
        }
```

ether를 withdraw할 때, CEI Pattern 미적용으로 인해, 재진입이 발생할 수 있다.

### 파급력

Level : `Critical`

protocol에 있는 모든 Ether 탈취 가능

### 해결방안

CEI Pattern을 적용한다.

---

# 4. Jeremy

[https://github.com/WOOSIK-jeremy/Lending_solidity](https://github.com/WOOSIK-jeremy/Lending_solidity)

commit v : [544c3af](https://github.com/WOOSIK-jeremy/Lending_solidity/commit/544c3aff1cac5dd1f996e451c0fcfcd93a274843)

### 1. 개발 미흡

### 설명

UpsideAcademyLending.sol/withdraw line 73

```solidity
    function withdraw(address token, uint256 tokenAmount) public payable {
        if(block.number - borrowBlock[msg.sender] > 0 && borrowBlock[msg.sender] > 0) {
            for(uint256 i = 0;i < (block.number - borrowBlock[msg.sender]); i++){
                accounts[msg.sender] = accounts[msg.sender] * 1999 / 2000;
            }            
        }

        require(accounts[msg.sender] >= tokenAmount, "your balance lower than etherAmount");
        
        payable(msg.sender).call{value : tokenAmount}("");
    }
```

deposit은 토큰이나 이더 둘 다 받지만, withdraw는 ether에 대해서만 제공

### 파급력

Level : `Medium`

사용자가 자신의 erc20 토큰을 withdraw할 수 없다.

### 해결방안

token address를 체크한다.

---

# 5. Damon

[https://github.com/gloomydumber/Lending_solidity/tree/master](https://github.com/gloomydumber/Lending_solidity/tree/master)

commit v : [2645135](https://github.com/gloomydumber/Lending_solidity/commit/2645135fd3ab081ac2a74ab2c0dcf297e28f190c)

### 1. Reentrancy Attack In withdraw

### 설명

DreamAcademyLending.sol/withdraw line 117

```solidity
        if (tokenAddress == address(0)) {
            if (user[msg.sender].debt == 0) {
                payable(msg.sender).transfer(amount);
                user[msg.sender].collateral -= amount;
            } else {
                uint256 priceOfEther = priceOracle.getPrice(address(0));
                uint256 priceOfUSDCoin = priceOracle.getPrice(usdc);
                require(
                    ((priceOfEther * ((user[msg.sender].collateral - (amount)))) * 75) / 100
                        > user[msg.sender].debt * (priceOfUSDCoin),
                    "Withdraw amount exceeds allowed collateral ratio"
                );
                payable(msg.sender).transfer(amount);
                user[msg.sender].collateral -= amount;
            }
        }
```

ether를 withdraw할 때, CEI Pattern 미적용으로 인해, 재진입이 발생할 수 있다.

### 파급력

Level : `Critical`

protocol에 있는 모든 Ether 탈취 가능

### 해결방안

CEI Pattern을 적용한다.

---

# 6. Jacquline

• [https://github.com/je1att0/Lending_solidity](https://github.com/je1att0/Lending_solidity)

commit v : [7bb6357](https://github.com/je1att0/Lending_solidity/commit/7bb6357b7e5742133581be4acafe811bbbe3ee77)

### 1. 개발 실수로 인한 불필요한 가스 낭비

UpsideAcademyLending.sol/deposit line 51

```solidity
    function deposit (address _tokenAddress, uint256 _amount) public payable {
        if(_tokenAddress != address(0x0)) {
            if (accounts[msg.sender].depositedUSDC == 0) {
                accountAddr.push(msg.sender);
            }
            accounts[msg.sender].lastDepositBlock = block.number;
            accounts[msg.sender].depositedUSDC += _amount;
            usdc.transferFrom(msg.sender, address(this), _amount);
        } else{
            require(msg.value > 0, "Empty TxValue");
            require(msg.value == _amount, "Insufficient Value");
            if (accounts[msg.sender].depositedETH == 0) {
                accountAddr.push(msg.sender);
            }
            accounts[msg.sender].lastDepositBlock = block.number;
            accounts[msg.sender].depositedETH += msg.value;
            payable(address(this)).transfer(msg.value);
        }
    }
```

payable함수인데 마지막에 자기 자신에게 transfer하는 의미없는 행위를 하여 deposit하는 유저가 더 많은 가스비를 소모하게 됨.

### 파급력

Level : `info`

유저가 불필요하게 가스를 낭비하게 됨.

### 해결방안

해당 라인 삭제

---

# 7. bob

• [https://github.com/choihs0457/Lending_solidity](https://github.com/choihs0457/Lending_solidity)

commit v : 175e80d

### 1. repay에서 token 체크의 부재

### 설명

Lending.sol/repay line 156

```solidity
    function repay(address token, uint256 amount) external onlyUser accrueInterest(msg.sender) {
        uint256 total_debt = account[msg.sender].BorrowAmount;
        require(total_debt >= amount, "check amount");
        account[msg.sender].BorrowAmount -= amount;
        TotalBorrowUSDC -= amount;
        ERC20(token).transferFrom(msg.sender, address(this), amount);
    }
```

repay함수에서 token의 종류를 확인하지 않아, 다른 토큰으로 repay가 가능.

### 파급력

Level : `High`

실제 프로토콜에서 사용하지 않는 토큰으로 repay가능

### 해결방안

토큰에 대한 검증 추가

---

# 8. Teddy

[https://github.com/gdh8230/Lending_solidity](https://github.com/gdh8230/Lending_solidity)

commit v : [a699951](https://github.com/gdh8230/Lending_solidity/commit/a699951e7bb896459d14983a6f82268f30b7d1dc)

### 1. repay에서 부적절한 이더리움 borrow amount 처리

### 설명

Lending.sol/repay 

```solidity
    function repay(address token, uint256 amount) external {
        _updateDebt(msg.sender);
        require(amount > 0, "Amount must be greater than 0");

        if (token == address(0)) {
            require(accounts[msg.sender].ETHBorrowedAmount >= amount, "Repay amount exceeds borrowed");
            accounts[msg.sender].ETHBorrowedAmount -= amount;
        } else {
            require(IERC20(token) == usdc, "Invalid token");
            require(accounts[msg.sender].USDCBorrowedAmount >= amount, "Repay amount exceeds borrowed");
            accounts[msg.sender].USDCBorrowedAmount -= amount;
            IERC20(usdc).transferFrom(msg.sender, address(this), amount);
        }

        totalDepositToken[token] += amount;
    }

```

token address가 0일 때, 이더리움 borrowed amount를 줄이는데, 정작 payable 함수가 아니라 이더를 받지 않았음.

### 파급력

Level : `Critical`

공격자는 자신이 빌린 금액을 무료로 갚을 수 있음.

### 해결방안

ether에 대해 payble을 붙이고 정확하게 처리.

---

# 9. Muang

[https://github.com/GODMuang/Lending_solidity](https://github.com/GODMuang/Lending_solidity) 

commit v : [e92439b](https://github.com/GODMuang/Lending_solidity/commit/e92439b7237f1e66f68f1ef49ecfe84be21a33a4)

### 1. repay에서 token 체크의 부재

### 설명

DreamAcademyLending.sol/repay line 127

```solidity
    function repay(address _token, uint256 _amount) external { 
        updateDebt(msg.sender,token);
        require(_amount <= userBorrowed[msg.sender][_token], "you dont have to pay this much hehe..");
        require(IERC20(_token).balanceOf(msg.sender) >= _amount, "INSUFFICIENT_TOKEN_TO_REPAY");

        IERC20(_token).transferFrom(msg.sender, address(this), _amount);
        userBorrowed[msg.sender][_token] -= _amount;
        totalDepositToken[_token] +=_amount;
    }

```

repay함수에서 token의 종류를 확인하지 않아, 다른 토큰으로 repay가 가능.

### 파급력

Level : `High`

실제 프로토콜에서 사용하지 않는 토큰으로 repay가능

### 해결방안

토큰에 대한 검증 추가

---

# 10. Mia

[https://github.com/ooMia/Upside_Lending_solidity/tree/test_28_28/src](https://github.com/ooMia/Upside_Lending_solidity/tree/test_28_28/src)

commit v : [c70f775](https://github.com/ooMia/Upside_Lending_solidity/commit/c70f775c423fa18a86aa73a09af7f4eeba94e000)

### 1. repay에서 token 체크의 부

### 설명

DreamAcademyLending.sol/repay line 95

```solidity
    function repay(address token, uint256 amount) external nonReentrant identity(msg.sender, Operation.REPAY) {
        if (token == _ETH) {
            revert("repay|ETH: Not payable");
        }
        transferFrom(msg.sender, _THIS, amount, token);
        cancelOutStorage(_USERS[msg.sender].loans, getPrice(token) * amount);
    }
```

repay함수에서 token의 종류를 확인하지 않아, 다른 토큰으로 repay가 가능.

### 파급력

Level : `High`

실제 프로토콜에서 사용하지 않는 토큰으로 repay가능

### 해결방안

토큰에 대한 검증 추가

---