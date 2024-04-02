### 初始化嵌入式硬件
```rust
    let p = embassy_nrf::init(Default::default());
```
### 设置输出引脚
```rust
    let mut led = Output::new(p.P0_13, Level::Low, OutputDrive::Standard);
```
- `p.P0_13` 是 LED 灯连接的 GPIO 引脚。
- `Level::Low` 设置了初始状态为低电平。
- `OutputDrive::Standard` 设置了引脚的驱动模式为标准驱动模式。
### 创建UART通信
```rust
    let mut u = BufferedUarte::new(
        p.UARTE0,
        p.TIMER0,
        p.PPI_CH0,
        p.PPI_CH1,
        p.PPI_GROUP0,
        Irqs,
        p.P0_08,
        p.P0_06,
        config,
        &mut rx_buffer,
        &mut tx_buffer,
    );
```
传入UART控制器`p.UARTE0`，定时器`p.TIMER0`，PPI通道`p.PPI_CH0` `p.PPI_CH1`，PPI组`p.PPI_GROUP0`，中断处理程序`Irqs`，GPIO引脚`p.P0_08`  `p.P0_06`，配置`config`，缓冲区。
### 异步调度
```rust
unwrap!(spawner.spawn(send_task(channel.sender())));
unwrap!(spawner.spawn(recv_task(p.P0_13.degrade(), channel.receiver())));

async fn send_task(sender: Sender<'static, NoopRawMutex, LedState, 1>){}
async fn recv_task(led: AnyPin, receiver: Receiver<'static, NoopRawMutex, LedState, 1>){}
```

```rust
    unwrap!(spawner.spawn(run1()));
    unwrap!(spawner.spawn(run2()));
    unwrap!(spawner.spawn(run3()));
```
### 绑定输入通道和异步任务
```rust
	let ch1 = InputChannel::new(
        p.GPIOTE_CH0,
        Input::new(p.P0_11, Pull::Up),
        InputChannelPolarity::HiToLo,
    );
    let button1 = async {
        loop {
            ch1.wait().await;
            info!("Button 1 pressed")
        }
    };
    ……
    futures::join!(button1, button2, button3, button4);
    
```

```rust
async fn button_task(n: usize, mut pin: Input<'static>) {
    loop {
        pin.wait_for_low().await;
        info!("Button {:?} pressed!", n);
        pin.wait_for_high().await;
        info!("Button {:?} released!", n);
    }
}

let btn1 = Input::new(p.P0_11, Pull::Up);
unwrap!(spawner.spawn(button_task(1, btn1)));
……

```
### 互斥锁
```rust
static MUTEX: Mutex<ThreadModeRawMutex, u32> = Mutex::new(0);
let mut m = MUTEX.lock().await;
*m += 1;
```
### PPI实现事件响应
```rust
	let button1 = InputChannel::new(
        p.GPIOTE_CH0,
        Input::new(p.P0_11, Pull::Up),
        InputChannelPolarity::HiToLo,
    );
	let led1 = OutputChannel::new(
        p.GPIOTE_CH4,
        Output::new(p.P0_13, Level::Low, OutputDrive::Standard),
        OutputChannelPolarity::Toggle,
    );
    ……
    let mut ppi = Ppi::new_one_to_one(p.PPI_CH0, button1.event_in(), led1.task_out());
    ppi.enable();
```
### 绑定中断处理程序
```rust
bind_interrupts!(struct Irqs {
    QDEC => qdec::InterruptHandler<peripherals::QDEC>;
});
```
