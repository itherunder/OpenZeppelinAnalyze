## 工具
**用于实现功能的一些工具库合约**

### 0x00 Address.sol
---
**地址相关的工具库合约，library合约，无依赖**

#### 0. 函数
```solidity
// 判断地址是否为合约地址
function isContract(address account) internal view returns (bool) {
    // This method relies on extcodesize/address.code.length, which returns 0
    // for contracts in construction, since the code is only stored at the end
    // of the constructor execution.
    // 以前还需要extcodesize 内联实现，现在不需要了，详情查看https://github.com/ethereum/solidity/blob/develop/Changelog.md
    return account.code.length > 0;
}

// 发送以太
function sendValue(address payable recipient, uint256 amount) internal {
    require(address(this).balance >= amount, "Address: insufficient balance");

    // 可以把所有的gas 都发过去执行，而transfer 只有2300 gas可以供给调用的合约使用
    (bool success, ) = recipient.call{value: amount}("");
    // 如果报错，直接revert
    require(success, "Address: unable to send value, recipient may have reverted");
}

// 函数调用，封装调用合约的函数
function functionCall(address target, bytes memory data) internal returns (bytes memory) {
    return functionCall(target, data, "Address: low-level call failed");
}
// 继续封装，加了个错误信息参数
function functionCall(
    address target,
    bytes memory data,
    string memory errorMessage
) internal returns (bytes memory) {
    return functionCallWithValue(target, data, 0, errorMessage);
}
// 继续封装，加了个value参数，可以发送以太
function functionCallWithValue(
    address target,
    bytes memory data,
    uint256 value,
    string memory errorMessage
) internal returns (bytes memory) {
    // 如果没有足够的以太，直接revert
    require(address(this).balance >= value, "Address: insufficient balance for call");
    // 调用的必须是合约地址
    require(isContract(target), "Address: call to non-contract");

    // 用底层call，发送以太并携带执行参数data
    (bool success, bytes memory returndata) = target.call{value: value}(data);
    // 验证结果
    return verifyCallResult(success, returndata, errorMessage);
}
// 验证结果
function verifyCallResult(
    bool success,
    bytes memory returndata,
    string memory errorMessage
) internal pure returns (bytes memory) {
    if (success) {
        return returndata;
    } else {
        // Look for revert reason and bubble it up if present
        if (returndata.length > 0) {
            // The easiest way to bubble the revert reason is using memory via assembly

            // 利用内联汇编revert并且将调用合约的结果放到内存中
            assembly {
                let returndata_size := mload(returndata)
                revert(add(32, returndata), returndata_size)
            }
        } else {
            // 如果没有返回内容的话，直接revert错误信息
            revert(errorMessage);
        }
    }
}

// 调用上面的functionCallWithValue，携带异常信息
function functionCallWithValue(
    address target,
    bytes memory data,
    uint256 value
) internal returns (bytes memory) {
    return functionCallWithValue(target, data, value, "Address: low-level call with value failed");
}

// 封装staticcall 函数，以静态调用的方式调用合约的函数
function functionStaticCall(address target, bytes memory data) internal view returns (bytes memory) {
    return functionStaticCall(target, data, "Address: low-level static call failed");
}
// 继续封装，加了个错误信息参数
function functionStaticCall(
    address target,
    bytes memory data,
    string memory errorMessage
) internal view returns (bytes memory) {
    // 由于staticcall不允许所调用的合约状态发生改变，因此也不能发送以太
    require(isContract(target), "Address: static call to non-contract");

    // staticcall，和call 一样，不过不允许target 合约修改它自身的状态
    (bool success, bytes memory returndata) = target.staticcall(data);
    // 验证结果
    return verifyCallResult(success, returndata, errorMessage);
}

// 封装delegatecall 函数，以delegatecall 调用的方式调用合约的函数
function functionDelegateCall(address target, bytes memory data) internal returns (bytes memory) {
    return functionDelegateCall(target, data, "Address: low-level delegate call failed");
}
// 继续封装，加了个错误信息参数
function functionDelegateCall(
    address target,
    bytes memory data,
    string memory errorMessage
) internal returns (bytes memory) {
    require(isContract(target), "Address: delegate call to non-contract");

    // delegatecall，仅使用target 合约的代码，其他状态、内存都是使用调用者的，目的是为了使用library 合约的代码，因此当代码中使用了library 时都会使用到这个delegatecall 调用
    (bool success, bytes memory returndata) = target.delegatecall(data);
    // 验证结果
    return verifyCallResult(success, returndata, errorMessage);
}
```

### 0x01 Arrays.sol
---
**数组操作的library合约**
#### 0. 依赖
##### a. [math](./utils_math.md)

#### 1. 函数
```solidity
// 给定一个数组和element，二分查找第一个大于等于element的索引
function findUpperBound(uint256[] storage array, uint256 element) internal view returns (uint256) {
    if (array.length == 0) {
        return 0;
    }

    uint256 low = 0;
    uint256 high = array.length;

    while (low < high) {
        uint256 mid = Math.average(low, high);

        // Note that mid will always be strictly less than high (i.e. it will be a valid array index)
        // because Math.average rounds down (it does integer division with truncation).
        if (array[mid] > element) {
            high = mid;
        } else {
            low = mid + 1;
        }
    }

    // At this point `low` is the exclusive upper bound. We will return the inclusive upper bound.
    if (low > 0 && array[low - 1] == element) {
        return low - 1;
    } else {
        return low;
    }
}
```

### 0x02 Base64.sol
---
**Base64编码库合约，没有依赖**
#### 0. 函数
```solidity
// 看不懂，就是Base64编码，byte数组转成Base64编码的字符串
function encode(bytes memory data) internal pure returns (string memory) {
    /**
        * Inspired by Brecht Devos (Brechtpd) implementation - MIT licence
        * https://github.com/Brechtpd/base64/blob/e78d9fd951e7b0977ddca77d92dc85183770daf4/base64.sol
        */
    if (data.length == 0) return "";

    // Loads the table into memory
    string memory table = _TABLE;

    // Encoding takes 3 bytes chunks of binary data from `bytes` data parameter
    // and split into 4 numbers of 6 bits.
    // The final Base64 length should be `bytes` data length multiplied by 4/3 rounded up
    // - `data.length + 2`  -> Round up
    // - `/ 3`              -> Number of 3-bytes chunks
    // - `4 *`              -> 4 characters for each chunk
    string memory result = new string(4 * ((data.length + 2) / 3));

    assembly {
        // Prepare the lookup table (skip the first "length" byte)
        let tablePtr := add(table, 1)

        // Prepare result pointer, jump over length
        let resultPtr := add(result, 32)

        // Run over the input, 3 bytes at a time
        for {
            let dataPtr := data
            let endPtr := add(data, mload(data))
        } lt(dataPtr, endPtr) {

        } {
            // Advance 3 bytes
            dataPtr := add(dataPtr, 3)
            let input := mload(dataPtr)

            // To write each character, shift the 3 bytes (18 bits) chunk
            // 4 times in blocks of 6 bits for each character (18, 12, 6, 0)
            // and apply logical AND with 0x3F which is the number of
            // the previous character in the ASCII table prior to the Base64 Table
            // The result is then added to the table to get the character to write,
            // and finally write it in the result pointer but with a left shift
            // of 256 (1 byte) - 8 (1 ASCII char) = 248 bits

            mstore8(resultPtr, mload(add(tablePtr, and(shr(18, input), 0x3F))))
            resultPtr := add(resultPtr, 1) // Advance

            mstore8(resultPtr, mload(add(tablePtr, and(shr(12, input), 0x3F))))
            resultPtr := add(resultPtr, 1) // Advance

            mstore8(resultPtr, mload(add(tablePtr, and(shr(6, input), 0x3F))))
            resultPtr := add(resultPtr, 1) // Advance

            mstore8(resultPtr, mload(add(tablePtr, and(input, 0x3F))))
            resultPtr := add(resultPtr, 1) // Advance
        }

        // When data `bytes` is not exactly 3 bytes long
        // it is padded with `=` characters at the end
        switch mod(mload(data), 3)
        case 1 {
            mstore8(sub(resultPtr, 1), 0x3d)
            mstore8(sub(resultPtr, 2), 0x3d)
        }
        case 2 {
            mstore8(sub(resultPtr, 1), 0x3d)
        }
    }

    return result;
}
```

### 0x03 Checkpoints.sol
---
**该库合约可以用来记录历史记录，例如vote合约**
#### 0. 依赖
##### a. [math](./utils_math.md)

#### 1. 结构体&全局变量
```solidity
// 存储一个checkpoint
struct Checkpoint {
    uint32 _blockNumber;
    uint224 _value;
}

// 存储一个checkpoint的数组
struct History {
    Checkpoint[] _checkpoints;
}
```

#### 2. 函数
```solidity
// 获取最近的一个checkpoint的值_value
function latest(History storage self) internal view returns (uint256) {
    uint256 pos = self._checkpoints.length;
    // 没有的checkpoint话，返回0
    return pos == 0 ? 0 : self._checkpoints[pos - 1]._value;
}

// 获取某个块高度的checkpoint的值_value
function getAtBlock(History storage self, uint256 blockNumber) internal view returns (uint256) {
    // 需要当前块高度大于等于blockNumber
    require(blockNumber < block.number, "Checkpoints: block not yet mined");

    // 二分查找找到blockNumber对应的checkpoint
    uint256 high = self._checkpoints.length;
    uint256 low = 0;
    while (low < high) {
        uint256 mid = Math.average(low, high);
        if (self._checkpoints[mid]._blockNumber > blockNumber) {
            high = mid;
        } else {
            low = mid + 1;
        }
    }
    return high == 0 ? 0 : self._checkpoints[high - 1]._value;
}

// 添加一个checkpoint
function push(History storage self, uint256 value) internal returns (uint256, uint256) {
    uint256 pos = self._checkpoints.length;
    uint256 old = latest(self);
    // 如果当前最后一个checkpoint 的块高和现在执行环境的块高一致，就直接替换这个checkpoint里面的_value了
    if (pos > 0 && self._checkpoints[pos - 1]._blockNumber == block.number) {
        self._checkpoints[pos - 1]._value = SafeCast.toUint224(value);
    } else { // 否则就添加一个新的checkpoint
        self._checkpoints.push(
            Checkpoint({_blockNumber: SafeCast.toUint32(block.number), _value: SafeCast.toUint224(value)})
        );
    }
    return (old, value);
}

// 对最近的数据使用一个函数op进行处理，然后再调用上面的push 函数添加这个新的checkpoint
function push(
    History storage self,
    function(uint256, uint256) view returns (uint256) op,
    uint256 delta
) internal returns (uint256, uint256) {
    return push(self, op(latest(self), delta));
}
```

### 0x04 Context.sol
---
**提供当前执行的上下文环境的合约，虚合约，没有依赖**

#### 0. 函数
```solidity
// 获取当前执行环境的合约调用者
function _msgSender() internal view virtual returns (address) {
    return msg.sender;
}

// 获取当前执行环境的携带的数据
function _msgData() internal view virtual returns (bytes calldata) {
    return msg.data;
}
```

### 0x05. Counters.sol
---
**计数器库合约，没有依赖**

#### 0. 结构体
```solidity
// 计数器结构体，就一个_value属性，uint256
struct Counter {
    uint256 _value; // default: 0
}
```

#### 1. 函数
```solidity
// 计数器当前值
function current(Counter storage counter) internal view returns (uint256) {
    return counter._value;
}

// 计数器加1
function increment(Counter storage counter) internal {
    // 这个unchecked 是0.8.0 之后的新特性
    // https://docs.soliditylang.org/en/v0.8.0/control-structures.html#checked-or-unchecked-arithmetic
    // 由于0.8.0 后overflow 会直接revert，但有时候我们想要溢出的话，可以使用unchecked，这样这个块里面的溢出就不会revert了
    unchecked {
        counter._value += 1;
    }
}

// 计数器减1
function decrement(Counter storage counter) internal {
    uint256 value = counter._value;
    require(value > 0, "Counter: decrement overflow");
    unchecked {
        counter._value = value - 1;
    }
}

// 重置计数器
function reset(Counter storage counter) internal {
    counter._value = 0;
}
```

### 0x06. Create2.sol
---
**用于创建合约的库，无任何依赖**

#### 0. 函数
```solidity
// 使用create2 创建一个合约
function deploy(
    uint256 amount,
    bytes32 salt,
    bytes memory bytecode
) internal returns (address) {
    address addr;
    require(address(this).balance >= amount, "Create2: insufficient balance");
    require(bytecode.length != 0, "Create2: bytecode length is zero");
    assembly {
        // `amount`: 初始发送的金额，`salt`: 加盐，`bytecode`: 合约的字节码，`add(bytecode, 0x20)`: 合约的字节码的起始内存位置，`mload(bytecode)`: 合约的字节码的长度
        addr := create2(amount, add(bytecode, 0x20), mload(bytecode), salt)
    }
    require(addr != address(0), "Create2: Failed on deploy");
    return addr;
}

// 计算地址
function computeAddress(
    bytes32 salt,
    bytes32 bytecodeHash,
    address deployer
) internal pure returns (address) {
    // create2 生成合约地址的计算规则
    bytes32 _data = keccak256(abi.encodePacked(bytes1(0xff), deployer, salt, bytecodeHash));
    return address(uint160(uint256(_data)));
}
```

#### 1. `create`和`create2`的区别
- 生成合约地址的规则不同
  - `create(val, ost, len)`: `keccak256(rlp([address(this), this.nonce]))[12:]`
  - `create2(val, ost, len, salt)`: `keccak256(0xff ++ address(this) ++ salt ++ keccak256(mem[ost:ost+len]))[12:]`
- 前者是可预测的（生成地址的参数中所有均是可知的），后者在加上salt后是不可预测的
[ref0](https://ethereum.org/en/developers/docs/evm/opcodes/), [ref1](https://ethereum.stackexchange.com/questions/101336/what-is-the-benefit-of-using-create2-to-create-a-smart-contract)

### 0x07. Multicall.sol
---
**一次外部调用中调用合约的多个其他函数**

#### 0. 依赖
##### a. [Address](#0x00-addresssol)

#### 1. 函数
```solidity
// 虽然是抽象合约，但是实现了这个多调用函数，子合约可以`override`该函数以实现自己的功能
function multicall(bytes[] calldata data) external virtual returns (bytes[] memory results) {
    results = new bytes[](data.length);
    for (uint256 i = 0; i < data.length; i++) {
        // 这里是直接调用自己，也可以调用其他地址
        results[i] = Address.functionDelegateCall(address(this), data[i]);
    }
    return results;
}
```

#### 2. 抽象合约和虚函数
- 如下有两个合约`a`，`b`以及`c`，其中`a`是抽象合约，`b`和`c`是其子合约，其中`c`也是抽象合约
- 当`a`合约中**存在未实现函数**时：
  - `a`无法被实例化，即**无法部署**（均实现时可以部署）
  - `b`必须实现`a`中未实现的函数，`c`同样作为**抽象合约**可以不实现`a`中的函数
  - `a`中未实现的函数**必须**声明为`virtual`
- `b`中重写的函数**必须**声明为`override`

```solidity
abstract contract a {
    function f() public returns(uint256) { return 5; }
    function f1() virtual public returns(uint256);
}

contract b is a {
    function f1() override public returns(uint256) { return 6; }
}

abstract contract c is a {
}
```