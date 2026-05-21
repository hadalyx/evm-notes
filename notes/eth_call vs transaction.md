'eth_call로 호출되는 거의 모든 함수가 트랜잭션으로도 호출 가능한가?'라는 의문에서 시작


EVM 관점에서 보면 함수 호출은 그냥 bytecode 실행. 
호출 채널이 RPC eth_call인지 트랜잭션인지는 EVM 입장에서 거의 동일 
그래서 — view, pure, 일반 함수 모두 두 방식으로 호출 가능합니다.

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

