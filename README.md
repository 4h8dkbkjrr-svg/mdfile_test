# 1일차 과제 2 : 스위치 폴링 및 외부 인터럽트 LED 제어

> **광운대학교 로봇학부**  
> **작성자:** 이동엽
> **제출일:** 2026년 7월 30일

---

## 1. 개요 (Overview)
본 과제는 ATmega128의 GPIO 입출력과 외부 인터럽트를 이용하여 8비트 LED를 제어하는 것을 목표로 함. 스위치 입력은 폴링과 인터럽트 두 방식으로 나누어 구현하고, 두 방식의 차이를 확인함.

### 핵심 목표
* DDRx / PORTx / PINx 레지스터를 이용한 입출력 방향 설정 및 Active Low 회로 제어
* 외부 풀업 스위치의 폴링 처리 (SW1, SW2)
* EICRA / EIMSK 설정을 통한 falling edge 외부 인터럽트 처리 및 ISR 작성
* 인터럽트 플래그(EIFR)를 이용한 중복 요청 예외처리

---

## 2. 개발 환경 (Environment)

| 항목 | 내용 |
| :--- | :--- |
| **MCU** | ATmega128A (16MHz External Crystal) |
| **IDE / Compiler** | Visual Studio Code / AVR-GCC 14.3.0 (macOS) |
| **Flasher Tool** | NEWTC USB_ISP (STK500v2) / avrdude |
| **언어** | C Language |
| **주요 부품** | ROBIT 실습보드, 8-Bit LED, 택트 스위치 4개 |

---

## 3. 하드웨어 구성 및 핀 맵 (Hardware Structure)

### Pin Configuration

```text
[ATmega128]                 [Target Component]
 PORTA (PA0 ~ PA7)   ----->   8-Bit LED (Active Low)
 PC0                 <-----   SW1 (Polling)
 PC1                 <-----   SW2 (Polling)
 PD3 (INT3)          <-----   SW4 (External Interrupt, Falling Edge)
 PD2 (INT2)          <-----   SW3 (External Interrupt, Falling Edge)
```

### 주요 회로 특징
* **LED:** 애노드가 5V, 캐소드가 PA 핀에 연결된 Active Low 구조. 0을 출력해야 점등됨
* **스위치:** 5V - 10kohm - 핀 - 스위치 - GND 의 외부 풀업 구조. 눌림 = LOW
* **디바운스:** 각 스위치에 104(0.1uF) 커패시터가 병렬 연결됨. RC 시정수 약 1ms
* **주의사항:** 07/29 수정 회로도에서 LCD가 I2C로 변경되며 SW1, SW2가 PD0/PD1에서 PC0/PC1으로 이동함. PC 포트에는 외부 인터럽트 핀이 없어 폴링만 가능함
* **INT4 대체 사유:** 과제 요구사항은 우측 이동을 INT4로 지정하나, INT4는 PE4에 고정되어 있고 본 보드는 해당 핀에 스위치가 실장되어 있지 않음. 점퍼선 미확보로 동일하게 EICRA로 설정 가능한 INT2(PD2, SW3)로 대체하여 구현함

---

## 4. 프로젝트 구조 (Directory Structure)
> 구현부(.c), 선언부(.h)만 구조에 표기함.
```text
├── Day 1/hw2/
│   ├── hw2.c      # 폴링 + 외부 인터럽트 LED 제어 전체
│   └── README.md
```
외부 라이브러리 없이 단일 소스로 구성함. Makefile 및 프로젝트 파일 없이 아래 명령만으로 빌드됨.

```text
avr-gcc -mmcu=atmega128 -Os -Wall -std=gnu99 -o firmware.elf hw2.c
avr-objcopy -O ihex -R .eeprom firmware.elf firmware.hex
```

최적화 옵션 -Os 를 생략하면 _delay_ms() 가 설계된 지연 시간대로 동작하지 않으므로 반드시 지정해야 함.

---

## 5. 핵심 코드 및 레지스터 설정 (Key Implementation)

### 외부 인터럽트 초기화 (hw2.c)
```c
DDRD &= ~((1 << PD3) | (1 << PD2));  // INT3, INT2 입력 설정
PORTD |= (1 << PD3) | (1 << PD2);    // 내부 풀업 (외부 10k 보조)

// 스위치는 눌리면 HIGH -> LOW 이므로 falling edge (ISCn1 = 1, ISCn0 = 0)
EICRA = (1 << ISC31) | (1 << ISC21);
EIMSK = (1 << INT3) | (1 << INT2);

sei();
```
ISCn0 비트는 리셋 초기값이 0이므로 별도로 건드리지 않음. 비트 위치를 숫자로 직접 쓰면 어느 INT를 설정하는지 혼동되기 쉬워 ISC31 과 같은 이름 상수를 사용함.

### 스위치 폴링 (Active Low)
```c
if ((!(PINC & (1 << PINC0))) && (!(PINC & (1 << PINC1))))
    PORTA = 0x00;        // 둘 다 눌림 -> 전체 점등
else if (!(PINC & (1 << PINC0)))
    PORTA = 0x0F;        // SW1 -> LED 4~7 점등
```
입력값은 반드시 PINC 레지스터로 읽어야 함. 내부 풀업을 켠 상태에서 PORTC 로 읽으면 출력 래치값인 1이 항상 반환되어 스위치 상태를 알 수 없음.

### 이동 애니메이션 및 예외처리
```c
ISR(INT3_vect)                       // 좌측 이동
{
    uint8_t temp;
    for (temp = 0b10000000; temp != 0b00000001; temp = temp >> 1) {
        PORTA = ~temp;
        _delay_ms(200);
    }
    PORTA = ~temp;
    _delay_ms(200);

    EIFR = (1 << INTF3) | (1 << INTF2);   // 중복 요청 폐기
}
```
보드의 물리적 배치가 PA0을 왼쪽 끝, PA7을 오른쪽 끝에 두고 있어 비트 시프트 방향과 화면상 방향이 반대임. 따라서 좌측 이동에 오른쪽 시프트를, 우측 이동에 왼쪽 시프트를 사용함. EIFR 레지스터는 0이 아닌 1을 써야 플래그가 지워짐.

---

## 6. 동작 설명 및 결과 (Results)

### 동작 시나리오
1. 전원 인가 시 PORTA를 출력으로 설정하고 전체 소등 상태로 초기화함
2. 스위치 미입력 시 0.5초 주기로 LED 8개가 점멸함
3. SW1 입력 시 LED 4~7만, SW2 입력 시 LED 0~3만 점등함
4. SW1과 SW2 동시 입력 시 LED 8개가 모두 점등함
5. SW4(INT3) 입력 시 LED 1개가 우측 끝에서 좌측 끝으로 이동함
6. SW3(INT2) 입력 시 LED 1개가 좌측 끝에서 우측 끝으로 이동함
7. 이동 중 추가 입력은 EIFR 클리어로 폐기되어 애니메이션이 중복 실행되지 않음

### 검증 과정에서 확인한 사항
초기 구현에서 이동 방향이 요구사항과 반대로 동작함. 코드상 왼쪽 시프트 연산이 실제 보드에서는 우측 이동으로 나타났으며, 원인은 LED의 물리적 배치 순서임. 회로도만으로는 판단할 수 없어 실제 보드 동작을 확인한 뒤 방향을 수정함.

또한 INT3이 SW3이 아닌 SW4에 대응한다는 점을 확인하지 못해 초기에 미동작으로 오인함. 스위치 번호와 인터럽트 번호가 한 칸씩 어긋나 있음.

### 동작 사진 / 영상

| 정면 동작 모습 | 인터럽트 이동 동작 |
| :---: | :---: |
| ![Hardware Setup](개인_구글드라이브_링크_첨부) | ![Interrupt Demo](개인_구글드라이브_링크_첨부) |

---

## 7. AI 툴 활용 명시 (AI Tools Declaration)
본 과제 작성 및 구현 과정에서 활용한 AI 도구(Generative AI)의 사용 현황 및 목적은 다음과 같음.

| 도구명 (Tool) | 활용 영역 | 세부 사용 목적 및 내용 |
| :--- | :--- | :--- |
| **Claude** | 코드 작성 & 디버깅 | - 외부 인터럽트 레지스터 설정 검토 및 오류 원인 분석<br>- 이동 방향 불일치 원인 진단<br>- 주석 작성 및 코드 가독성 개선 |
| **Claude** | 회로도 해석 & 문서화 | - 07/29 수정 회로도와 원본 회로도의 핀 배치 차이 정리<br>- 보고서 구조 및 서술 정리 |

### AI 활용 및 검증 원칙
1. **코드 검증:** AI가 제시한 레지스터 설정은 ATmega128 데이터시트의 External Interrupts 항목(p.89~93)과 대조하여 확인하였으며, 실제 보드에 업로드하여 동작을 직접 검증하였습니다.
2. **하드웨어 판단:** INT4 대체, 스위치 핀 변경 등 회로와 관련된 결정은 실습보드와 수정 회로도를 직접 확인한 뒤 판단하였습니다.
3. **학습 주도성:** 동작 요구사항의 해석과 구현 방향은 직접 결정하였으며, AI는 문법 오류 및 하드웨어 문제의 원인 분석을 돕는 보조 도구로 활용하였습니다.
