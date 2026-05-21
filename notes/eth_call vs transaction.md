## 'eth_call로 호출되는 거의 모든 함수가 트랜잭션으로도 호출 가능한가?'라는 의문에서 시작


EVM 관점에서 보면 함수 호출은 그냥 bytecode 실행. 
호출 채널이 RPC eth_call인지 트랜잭션인지는 EVM 입장에서 거의 동일 
그래서 — view, pure, 일반 함수 모두 두 방식으로 호출 가능.

### EVM에는 트랜잭션 컨텍스트와 eth_call 컨텍스트에서 다르게 동작하는 opcode들이 존재
1. BLOCK.TIMESTAMP, BLOCK.NUMBER, BLOCK.COINBASE 등
   트랜잭션: 자기가 포함된 블록의 값
   eth_call: 호출 시점의 latest block 값 (또는 사용자가 지정한 block의 값)
   만약 다음 블록 직전에 호출한다면? 

공격 예시)
```
function trade() external {
    if (block.number > simulationBlock) {
        // 실제 트랜잭션
        rugpull();
    } else {
        // 시뮬레이션
        normalTrade();
    }
}
```

2. MSG.SENDER
   트랜잭션: 서명자의 주소 (변조 불가)
   eth_call: from 파라미터로 임의 지정 가능 (변조 가능)

공격 예시)
```
function attack() external {
    if (msg.sender == tx.origin) {  
        // eth_call로 시뮬레이션할 때는 EOA처럼 보이게 함
        return goodLooking();
    } else {
        // 실제 트랜잭션에서는 다르게 동작
        return drainFunds();
    }
}
```

3. MSG.VALUE, TX.GASPRICE
   트랜잭션: 실제 첨부된 값과 가스 가격
   eth_call: 사용자가 임의 지정 가능

물론 실제로 시뮬레이션 공격은 더 복잡한 패턴으로 들어가야 작동함. 
저걸로는 분기가 명확하지 않기에, gas, Storage state 변화등의 기법을 넣어야 작동
