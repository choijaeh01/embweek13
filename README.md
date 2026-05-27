# I/O Device Driver

Achro-EM Kit I/O Device Driver Lab

임베디드 시스템 설계 실습 11장 I/O Device Driver 실습용 자료입니다.

이번 실습은 Achro-EM Kit에서 커널에 맞게 미리 빌드된 `.ko` 파일을 사용하여 `insmod`, `mknod`, `/dev/fpga_*`, 사용자 테스트 앱의 흐름을 직접 확인하는 실습입니다.

## 실습 핵심

사용자 프로그램은 하드웨어를 직접 제어하지 않습니다. 사용자 프로그램은 `/dev/fpga_*` 디바이스 파일을 열고 `read()` 또는 `write()`를 호출합니다. 이 호출은 커널에 적재된 디바이스 드라이버의 `file_operations`에 연결되고, 드라이버는 `fpga_interface_driver`가 제공하는 FPGA I/O 함수를 통해 실제 I/O 보드에 접근합니다.

```text
사용자 앱
  open(), read(), write(), close()
        |
        v
/dev/fpga_led, /dev/fpga_fnd, ...
        |
        v
개별 디바이스 드라이버 .ko
  register_chrdev()
  file_operations
        |
        v
fpga_interface_driver.ko
  iom_fpga_itf_read()
  iom_fpga_itf_write()
        |
        v
Achro-EM Kit I/O Board
```

## 개념 정리

### `/dev` 디렉토리

`/dev`는 리눅스에서 디바이스 파일들이 위치하는 표준 디렉토리입니다.

리눅스에서는 많은 장치를 파일처럼 다룹니다. 예를 들어 `/dev/null`, `/dev/tty`, `/dev/mmcblk0`, `/dev/i2c-1` 같은 파일들은 일반 텍스트 파일이 아니라 커널 드라이버와 연결되는 디바이스 파일입니다.

이번 실습의 `/dev/fpga_led`, `/dev/fpga_fnd`, `/dev/fpga_push_switch`도 같은 개념입니다. 단지 이름이 Achro-EM Kit의 FPGA I/O 장치에 맞게 붙은 것입니다.

```text
/dev
  리눅스에서 디바이스 파일이 위치하는 표준 디렉토리

/dev/fpga_led
  Achro-EM LED 드라이버에 접근하기 위해 만드는 디바이스 파일
```

### 노드 파일, 스페셜 파일, 디바이스 파일

`mknod`로 만드는 `/dev/fpga_led`는 일반 파일이 아닙니다. 이 파일 안에 LED 값이 저장되는 것이 아니라, 사용자 프로그램이 커널 드라이버로 들어가기 위한 입구 역할을 합니다.

```bash
sudo mknod /dev/fpga_led c 260 0
ls -l /dev/fpga_led
```

출력은 보통 다음과 비슷합니다.

```text
crw-r--r-- 1 root root 260, 0 ... /dev/fpga_led
```

여기서 앞의 `c`는 character device를 의미합니다.

```text
c    character device
260  major number
0    minor number
```

major number는 이 디바이스 파일이 어떤 커널 드라이버로 연결될지를 결정하는 번호입니다. minor number는 같은 드라이버 안에서 여러 장치를 구분할 때 사용하는 번호입니다. 이번 LED 실습에서는 minor number를 `0`으로 사용합니다.

### `insmod`와 `mknod`


```text
insmod
  .ko 파일을 커널에 적재한다.
  드라이버 코드가 커널 안에 등록된다.

mknod
  /dev/fpga_led 같은 디바이스 파일을 만든다.
  사용자 프로그램이 드라이버에 접근할 입구를 만든다.
```

LED 드라이버를 예로 들면, 먼저 다음 명령으로 드라이버를 커널에 적재합니다.

```bash
sudo insmod prebuilt/drivers/fpga_led_driver.ko
```

드라이버 내부에서는 major number를 커널에 등록합니다.

```c
register_chrdev(260, "fpga_led", &iom_led_fops);
```

이 코드는 커널에게 다음과 같이 알려주는 역할을 합니다.

```text
major number 260으로 들어오는 character device 요청은
fpga_led_driver의 file_operations로 연결한다.
```

그 다음 `mknod`로 같은 major number를 가진 디바이스 파일을 만듭니다.

```bash
sudo mknod /dev/fpga_led c 260 0
```

이제 `/dev/fpga_led`는 major number 260을 가리키는 입구가 됩니다. 따라서 사용자 프로그램이 `/dev/fpga_led`를 열면 커널은 major number 260에 등록된 LED 드라이버를 찾아갑니다.

```text
사용자 프로그램
  open("/dev/fpga_led")
  write()
  read()
        |
        v
/dev/fpga_led
  character device, major 260, minor 0
        |
        v
커널의 major 260 등록 정보
        |
        v
fpga_led_driver의 file_operations
        |
        v
iom_led_open(), iom_led_write(), iom_led_read()
```

즉, `mknod`는 응용프로그램과 디바이스 드라이버를 연결하는 사용자 공간의 접점을 만드는 명령입니다. 하지만 이 연결이 동작하려면 드라이버가 먼저 `insmod`로 커널에 올라가 있어야 하고, `mknod`에서 지정한 major number가 드라이버가 등록한 major number와 같아야 합니다.

```text
insmod만 한 경우
  드라이버는 커널에 있지만 사용자가 접근할 /dev 입구가 없다.

mknod만 한 경우
  /dev 파일은 있지만 해당 major number를 처리할 드라이버가 없을 수 있다.

insmod와 mknod를 모두 올바르게 한 경우
  사용자 프로그램의 open/read/write가 해당 드라이버 함수로 전달된다.
```

### 사용자 프로그램과 드라이버의 연결

사용자 프로그램은 하드웨어를 직접 제어하지 않고 `/dev/fpga_led`를 일반 파일처럼 엽니다.

```c
dev = open("/dev/fpga_led", O_RDWR);
write(dev, &data, 1);
read(dev, &data, 1);
close(dev);
```

하지만 커널 내부에서는 이 호출들이 드라이버의 `file_operations`에 등록된 함수로 연결됩니다.

```c
struct file_operations iom_led_fops = {
    .open = iom_led_open,
    .write = iom_led_write,
    .read = iom_led_read,
    .release = iom_led_release,
};
```

따라서 이번 실습의 핵심은 단순히 명령어를 입력하는 것이 아니라, 다음 연결 관계를 이해하는 것입니다.

```text
fpga_test_led.c
  open(), write(), read(), close()
        |
        v
/dev/fpga_led
  mknod로 만든 character device node
        |
        v
fpga_led_driver.c
  file_operations에 등록된 함수
        |
        v
Achro-EM Kit LED 하드웨어
```

## 중요한 제한

현재 실습 키트에서는 다음 경로가 깨져 있어 커널 모듈을 새로 빌드하기 어렵습니다.

```bash
/lib/modules/$(uname -r)/build
```

따라서 `source/` 디렉토리의 `.c`와 `Makefile`은 코드 구조를 읽고 이해하기 위한 참고 자료입니다. 실습 실행에는 `prebuilt/` 안의 `.ko`와 테스트 앱을 사용합니다.

실습 기준 커널은 다음과 같습니다.

```bash
uname -r
# 4.19.106-v7l+
```

커널 버전이 다르면 `.ko` 적재가 실패할 수 있습니다.

## 디렉토리 구성

```text
.
├── prebuilt/
│   ├── drivers/        # 실습에서 사용하는 미리 빌드된 커널 모듈
│   └── apps/           # 실습에서 사용하는 사용자 테스트 앱
└── source/             # 코드 구조 분석용 소스와 Makefile
```

## 디바이스 목록

| 장치 | Device node | Major | Driver | Test app | 예시 |
|---|---:|---:|---|---|---|
| LED | `/dev/fpga_led` | 260 | `fpga_led_driver.ko` | `fpga_test_led` | `255` |
| FND | `/dev/fpga_fnd` | 261 | `fpga_fnd_driver.ko` | `fpga_test_fnd` | `1234` |
| Dot Matrix | `/dev/fpga_dot` | 262 | `fpga_dot_driver.ko` | `fpga_test_dot` | `7` |
| Text LCD | `/dev/fpga_text_lcd` | 263 | `fpga_text_lcd_driver.ko` | `fpga_test_text_lcd` | `hello world` |
| Buzzer | `/dev/fpga_buzzer` | 264 | `fpga_buzzer_driver.ko` | `fpga_test_buzzer` | `Ctrl+C`로 종료 |
| Push Switch | `/dev/fpga_push_switch` | 265 | `fpga_push_switch_driver.ko` | `fpga_test_push_switch` | `Ctrl+C`로 종료 |
| Dip Switch | `/dev/fpga_dip_switch` | 266 | `fpga_dip_switch_driver.ko` | `fpga_test_dip_switch` | `Ctrl+C`로 종료 |
| Step Motor | `/dev/fpga_step_motor` | 267 | `fpga_step_motor_driver.ko` | `fpga_test_step_motor` | `1 0 10` |

## 실습 준비

Achro-EM Kit의 I/O Device 보드가 연결되어 있어야 합니다. 교재 안내처럼 I/O Device 보드 중앙 하단부의 SW7 스위치 1번을 ON 상태로 둡니다.

키트에 접속한 뒤 저장소 디렉토리로 이동합니다. 실습 파일은 `/home/pi/Modules`로 복사하지 않고 저장소 안에서 바로 사용합니다.

```bash
cd ~/embweek13
chmod +x prebuilt/apps/fpga_test_*
```

환경을 확인합니다.

```bash
uname -r
ls -l prebuilt/drivers/*.ko prebuilt/apps/fpga_test_*
strings prebuilt/drivers/fpga_interface_driver.ko | grep '^vermagic='
```

## 공통 실습 순서

모든 디바이스 실습은 같은 순서로 진행합니다.

1. 인터페이스 드라이버를 먼저 적재합니다.
2. 개별 디바이스 드라이버를 적재합니다.
3. `mknod`로 `/dev/fpga_*` 디바이스 파일을 만듭니다.
4. 테스트 앱을 실행합니다.
5. 디바이스 노드를 지우고, 개별 드라이버를 제거합니다.
6. 모든 실습이 끝나면 인터페이스 드라이버를 제거합니다.

인터페이스 드라이버는 개별 디바이스 드라이버들이 공통으로 사용하는 하위 드라이버입니다. 따라서 가장 먼저 적재하고 가장 마지막에 제거합니다.

```bash
cd ~/embweek13
sudo insmod prebuilt/drivers/fpga_interface_driver.ko
lsmod | grep fpga
```

## LED 실습

LED 드라이버를 적재하고 디바이스 노드를 만듭니다.

```bash
sudo insmod prebuilt/drivers/fpga_led_driver.ko
sudo mknod /dev/fpga_led c 260 0
sudo chmod 666 /dev/fpga_led
ls -l /dev/fpga_led
```

테스트 앱을 실행합니다.

```bash
./prebuilt/apps/fpga_test_led 255
./prebuilt/apps/fpga_test_led 0
./prebuilt/apps/fpga_test_led 170
```

정리합니다.

```bash
sudo rm -f /dev/fpga_led
sudo rmmod fpga_led_driver
```

## FND 실습

```bash
sudo insmod prebuilt/drivers/fpga_fnd_driver.ko
sudo mknod /dev/fpga_fnd c 261 0
sudo chmod 666 /dev/fpga_fnd
ls -l /dev/fpga_fnd
```

```bash
./prebuilt/apps/fpga_test_fnd 1234
./prebuilt/apps/fpga_test_fnd 5678
```

```bash
sudo rm -f /dev/fpga_fnd
sudo rmmod fpga_fnd_driver
```

## Dot Matrix 실습

```bash
sudo insmod prebuilt/drivers/fpga_dot_driver.ko
sudo mknod /dev/fpga_dot c 262 0
sudo chmod 666 /dev/fpga_dot
ls -l /dev/fpga_dot
```

```bash
./prebuilt/apps/fpga_test_dot 7
./prebuilt/apps/fpga_test_dot 3
```

```bash
sudo rm -f /dev/fpga_dot
sudo rmmod fpga_dot_driver
```

## Text LCD 실습

```bash
sudo insmod prebuilt/drivers/fpga_text_lcd_driver.ko
sudo mknod /dev/fpga_text_lcd c 263 0
sudo chmod 666 /dev/fpga_text_lcd
ls -l /dev/fpga_text_lcd
```

```bash
./prebuilt/apps/fpga_test_text_lcd hello world
./prebuilt/apps/fpga_test_text_lcd CAU EMBEDDED
```

```bash
sudo rm -f /dev/fpga_text_lcd
sudo rmmod fpga_text_lcd_driver
```

## Buzzer 실습

`fpga_test_buzzer`는 계속 실행되므로 관찰 후 `Ctrl+C`로 종료합니다.

```bash
sudo insmod prebuilt/drivers/fpga_buzzer_driver.ko
sudo mknod /dev/fpga_buzzer c 264 0
sudo chmod 666 /dev/fpga_buzzer
ls -l /dev/fpga_buzzer
```

```bash
./prebuilt/apps/fpga_test_buzzer
# Ctrl+C로 종료
```

```bash
sudo rm -f /dev/fpga_buzzer
sudo rmmod fpga_buzzer_driver
```

## Dip Switch 실습

`fpga_test_dip_switch`는 스위치 값을 반복해서 읽습니다. 관찰 후 `Ctrl+C`로 종료합니다.

```bash
sudo insmod prebuilt/drivers/fpga_dip_switch_driver.ko
sudo mknod /dev/fpga_dip_switch c 266 0
sudo chmod 666 /dev/fpga_dip_switch
ls -l /dev/fpga_dip_switch
```

```bash
./prebuilt/apps/fpga_test_dip_switch
# Ctrl+C로 종료
```

```bash
sudo rm -f /dev/fpga_dip_switch
sudo rmmod fpga_dip_switch_driver
```

## Push Switch 실습

`fpga_test_push_switch`는 버튼 값을 반복해서 읽습니다. 관찰 후 `Ctrl+C`로 종료합니다.

```bash
sudo insmod prebuilt/drivers/fpga_push_switch_driver.ko
sudo mknod /dev/fpga_push_switch c 265 0
sudo chmod 666 /dev/fpga_push_switch
ls -l /dev/fpga_push_switch
```

```bash
./prebuilt/apps/fpga_test_push_switch
# Ctrl+C로 종료
```

```bash
sudo rm -f /dev/fpga_push_switch
sudo rmmod fpga_push_switch_driver
```

## Step Motor 실습

모터는 반드시 정지 명령까지 실행한 뒤 정리합니다.

```bash
sudo insmod prebuilt/drivers/fpga_step_motor_driver.ko
sudo mknod /dev/fpga_step_motor c 267 0
sudo chmod 666 /dev/fpga_step_motor
ls -l /dev/fpga_step_motor
```

```bash
./prebuilt/apps/fpga_test_step_motor 1 0 10
./prebuilt/apps/fpga_test_step_motor 1 1 10
./prebuilt/apps/fpga_test_step_motor 0 0 10
```

```bash
sudo rm -f /dev/fpga_step_motor
sudo rmmod fpga_step_motor_driver
```

## 마지막 정리

모든 개별 디바이스 드라이버를 제거한 뒤 인터페이스 드라이버를 제거합니다.

```bash
lsmod | grep fpga
sudo rmmod fpga_interface_driver
lsmod | grep fpga
```

혹시 중간에 정리가 꼬였으면 아래 순서로 제거합니다. 실행 중인 테스트 앱이 있으면 먼저 `Ctrl+C`로 종료합니다.

```bash
sudo rm -f /dev/fpga_led /dev/fpga_fnd /dev/fpga_dot /dev/fpga_text_lcd
sudo rm -f /dev/fpga_buzzer /dev/fpga_push_switch /dev/fpga_dip_switch /dev/fpga_step_motor

sudo rmmod fpga_step_motor_driver 2>/dev/null
sudo rmmod fpga_push_switch_driver 2>/dev/null
sudo rmmod fpga_dip_switch_driver 2>/dev/null
sudo rmmod fpga_buzzer_driver 2>/dev/null
sudo rmmod fpga_text_lcd_driver 2>/dev/null
sudo rmmod fpga_dot_driver 2>/dev/null
sudo rmmod fpga_fnd_driver 2>/dev/null
sudo rmmod fpga_led_driver 2>/dev/null
sudo rmmod fpga_interface_driver 2>/dev/null
```

## 소스 코드에서 확인할 내용

각 디바이스 디렉토리의 드라이버 소스에서 다음 항목을 찾습니다.

1. Major number
2. Device name
3. `register_chrdev()`
4. `struct file_operations`
5. `open`, `read`, `write`, `release` 함수
6. `iom_fpga_itf_read()` 또는 `iom_fpga_itf_write()` 호출 위치

예를 들어 LED 드라이버에서는 다음 관계를 확인합니다.

```text
source/fpga_led/fpga_led_driver.c
  IOM_LED_MAJOR = 260
  IOM_LED_NAME = "fpga_led"
  register_chrdev(260, "fpga_led", ...)
        |
        v
sudo mknod /dev/fpga_led c 260 0
        |
        v
source/fpga_led/fpga_test_led.c
  open("/dev/fpga_led", O_RDWR)
  write()
  read()
  close()
```

## 과제

다음 항목을 결과보고서에 정리합니다.

1. `uname -r` 결과와 `.ko`의 `vermagic`이 왜 중요할지 설명합니다.
2. LED 또는 FND 중 하나를 선택해 major number, device node, driver file, test app의 관계를 설명합니다.
3. 선택한 드라이버의 `file_operations` 구조체를 찾고, 사용자 앱의 `open()`, `read()`, `write()`, `close()`와 어떻게 연결되는지 설명합니다.
4. `fpga_interface_driver.ko`를 먼저 적재해야 하는 이유를 `lsmod` 결과와 함께 설명합니다.
5. 출력 장치 1개와 입력 장치 1개 이상을 실행하고, 실행 명령과 관찰 결과를 정리합니다.
6. 실습 후 `/dev/fpga_*` 노드와 `lsmod | grep fpga` 결과를 확인하고, 남아 있는 항목이 있으면 직접 정리합니다.

## 문제 해결

### `insmod: Invalid module format`

실행 중인 커널과 `.ko`의 `vermagic`이 맞지 않을 때 발생합니다.

```bash
uname -r
strings prebuilt/drivers/fpga_led_driver.ko | grep '^vermagic='
```

### `insmod: Unknown symbol`

개별 디바이스 드라이버보다 `fpga_interface_driver.ko`를 먼저 적재했는지 확인합니다.

```bash
sudo insmod prebuilt/drivers/fpga_interface_driver.ko
```

### `File exists`

이미 같은 디바이스 노드가 남아 있을 수 있습니다.

```bash
ls -l /dev/fpga_led
sudo rm -f /dev/fpga_led
sudo mknod /dev/fpga_led c 260 0
```

### `Device open error`

디바이스 노드가 없거나 major number가 잘못되었을 가능성이 큽니다.

```bash
ls -l /dev/fpga_*
lsmod | grep fpga
```

### `Module is in use`

테스트 앱이 아직 실행 중이거나 디바이스가 사용 중일 수 있습니다. 실행 중인 테스트 앱을 `Ctrl+C`로 종료한 뒤 다시 제거합니다.

```bash
ps aux | grep fpga_test
sudo rmmod fpga_led_driver
```
