## Coordinator

1. 分发任务
2. 发送任务后，将Worker ID和发送时间注册一个回调函数，当超时时，调用函数将任务重新放回到任务List中

## Worker

1. `Map Worker`
2. `Reduce Worker`

