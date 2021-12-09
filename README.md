# simple-exchange (solidity + truffle + react)
一个简单的去中心化交易所，初步实现了质押ETH增加流动性，赎回ETH获取流动性收益；以及抵押ETH用于借贷Token代币，偿还时归还Token本金和利息（ETH）。

**步骤**

1. 下载代码，搭建开发环境：
    - 运行npm install，为项目添加依赖，包括truffle开发环境，前端React组件，web3js以及测试环境chai等，
    - Chrome浏览器安装metamask钱包插件，
    - 安装ganache客户端并启动本地节点，注意ganache本地节点的netwokid是1337，端口是7545，如果端口号变动需要在truffle-configure.js文件中更新；
2. 开启终端，cd到项目目录，运行truffle complie，为合约生成abi文件；
3. 运行truffle test可以运行测试；
4. 运行truffle migrate --reset，第一次部署可以省略--reset
5. 运行truffle console打开truffle控制台，可以键入web3命令来获取状态数据，比如：
    - accounts = await web3.eth.getAccounts(),获取全部账户，
    - balance = await web3.eth.getBalance(account)获取指定账户余额，
6. 以上步骤完成合约部署，运行yarn start，启动react服务，浏览器呈现dapp页面

**要点**

1. 运行truffle migrate部署合约时，自定义部署脚本2_deploy.js中 
```javascript 
await token.passMinterRole(dbank.address)
```
把minter token权限赋给dbank合约地址，是为了让dbank合约能在用户赎回的时候付给用户流动性收益；

2. Token合约继承自solidiy开源库openzeppelin的ERC20合约，这些库已经过审核以实现高标准的安全性，所以依赖于它们的合约在正确使用时不易受到黑客攻击。

3. 用户赎回ETH的时候，dbank合约会根据质押ETH的时间来计算流动性收益，年化收益是10%，质押和赎回的时候dbank会记录当时的block.timestamp，时长就是两个timestamp的差值，然后根据时长的秒数在一年中所占比例计算流动性收益。
```javascript 
withdraw() public {
    //判断用户是否质押了ETH，没有质押则不能赎回
    require(isDeposited[msg.sender]==true, 'Error, no previous deposit');
    //获取用户质押的ETH数额
    uint userBalance = etherBalanceOf[msg.sender]; 

    //计算用户质押的时长
    uint depositTime = block.timestamp - depositStart[msg.sender];
    
    //根据时长计算用户获得的流动性收益
    uint totalSecondsOneYear = 60 * 60 * 24 * 365;
    uint interest = (etherBalanceOf[msg.sender] * depositTime) / (totalSecondsOneYear * 10);

    //给用户转给质押的ETH
    msg.sender.transfer(etherBalanceOf[msg.sender]); 
    //付给用户流动性收益，用的时token代币
    token.mint(msg.sender, interest); 

    //重置质押的状态
    depositStart[msg.sender] = 0;
    etherBalanceOf[msg.sender] = 0;
    isDeposited[msg.sender] = false;
    //触发Withdraw事件，通知前端
    emit Withdraw(msg.sender, userBalance, depositTime, interest);
  }
  ```
  
  4. 用户借贷的时候会抵押ETH，可以借到token，数额是抵押的ETH的一半，偿还的时候除了要归还借出的token外，还要偿还一定数额（10%的抵押ETH）的ETH作为借贷利息。
  ```javascript 
  function payOff() public {
    //判断用户是否已经抵押ETH用于借贷，没有则不能偿还
    require(isBorrowed[msg.sender] == true, 'Error, loan not active');
    //偿还借出的token代币，成功则继续
    require(token.transferFrom(msg.sender, address(this), collateralEther[msg.sender]/2), "Error, can't receive tokens"); 
    //计算借贷利息，数额是抵押ETH的10%
    uint fee = collateralEther[msg.sender]/10; 
    feeEther[msg.sender] = fee;
    //转给用户抵押的ETH，但是要减掉借贷利息，相当于是用ETH来偿还利息
    msg.sender.transfer(collateralEther[msg.sender]-fee);
    //充值借贷状态
    collateralEther[msg.sender] = 0;
    isBorrowed[msg.sender] = false;
    //触发偿还事件
    emit PayOff(msg.sender, fee);
  }
  ```
  
  5. 前端React监听Solidity emit 的 PayOff事件，更新Fee：
  ```javascript 
  dbank.events.PayOff((error, event) => {
          if(!error){
            console.log(event)
            let fee = web3.utils.fromWei(event.returnValues.fee, 'ether')
            this.setState({fee: fee})
          }else{
            console.log(error)
          }
        })
   ```
