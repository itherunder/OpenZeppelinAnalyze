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