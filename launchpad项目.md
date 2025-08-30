### Farm（质押挖矿,流动性挖矿）

质押过程：用户质押，然后等收益



前端调用合约方法的方式

```
const contract = new Contract(airdropAddress, AirdropAbi.abi, signer);
const res = await airdropContract.withdrawTokens()
airdropAddress：合约地址
AirdropAbi.abi合约部署完成的二进制代码
signer:调用者
```

