# STM32 3.3V ADC 인터페이스 규격에 맞춘 전압 분배 및 측정 회로 만들기



![](배터리모니터링5.jpg)
![](배터리모니터링1.jpg)
### 1. ADC 입력 전압 범위

| 배터리 상태 | VBAT | Vadc (분배 후) |
| :--- | :--- | :--- |
| 완충 | 8.4V (4.2V/셀) | 2.837V |
| 공칭 전압 | 7.4V (3.7V/셀) | 2.499V |
| **충전 권장 시점** | **6.6V (3.3V/셀)** | **2.229V** |
| 방전 하한(위험) | 6.0V (3.0V/셀) | 2.027V |

### 2. ADC Raw 값 (12비트, Vref=3.3V 기준)

| 상태 | Vadc | ADC Raw (12bit) |
| :--- | :--- | :--- |
| 완충 (8.4V) | 2.837V | 약 3519 |
| 공칭 (7.4V) | 2.499V | 약 3100 |
| **충전 권장 (6.6V)** | **2.229V** | **약 2765** |
| 방전 하한 (6.0V) | 2.027V | 약 2515 |

*공칭 : 배터리의 표준/평균 동작 전압*

### 3. 충전 필요 판단 기준 

리튬이온 셀 특성상 3.0V 이하는 급격히 손상 위험이 커지므로,
**여유를 두고 3.3V/셀에서 부저**을 울리는 것으로 결정

| 판단 | 셀당 전압 | 총 VBAT | 동작 |
| :--- | :--- | :--- | :--- |
| 정상 | > 3.4V | > 6.8V | 알람 OFF |
| **충전 알람 ON** | **≤ 3.3V** | **≤ 6.6V** | **알람 시작** |

---

### 3.회로도 


**배터리 제원:**
* 3.7V * 2 = 7.4V @ 2600mAh

**분배 회로 구성 요소:**
* R1  (10K)
* R2  (가변저항 10K)--> 뉴클레오 103RB 회로보호용 부착
* R3  (5.1K)---> 10K로 2개로 제작
* C1  100nF
* A0 핀--->PC3 연결
* 부저 --->PB7 연결
---
### 4.실측
* 가변저항 10K --> A0 : 2.79V
* 가변저항 0K  --> A0 : 1.68V

# NUCLEO-F103RB 2S 리튬 배터리 전원 측정 및 방전 부저 경고 가이드

이 문서는 NUCLEO-F103RB(STM32F103RB) 보드를 이용하여 2S 리튬 배터리 전압을 ADC로 측정하고, 방전 시 부저(TMB12A05)로 경고를 울리는 시스템의 하드웨어 구성, 측정 데이터, STM32CubeMX 설정 및 C 코드 구현 내용을 정리한 가이드입니다.

---

## 1. 시스템 사양 및 하드웨어 구성

### 1.1 대상 기기 및 전원
- **마이크로컨트롤러**: NUCLEO-F103RB (STM32F103RBT6)
- **ADC 기준 전압 ($V_{REF}$)**: $3.3\text{V}$ (12-bit 해상도: $0 \sim 4095$)
- **측정 대상 전원**: 2S 리튬 배터리 파크 (3.7V x 2 = 공칭 7.4V)
  - 만충 전압: $8.4\text{V}$
  - 방전 컷오프 전압: $6.4\text{V}$ (셀당 3.2V 기준)

### 1.2 저항 분배 회로 (Voltage Divider)
STM32 ADC 핀의 최대 입력 허용 전압은 $3.3\text{V}$이므로 전압을 낮추기 위한 저항 분배 회로를 구성합니다.
- **전원 측 저항 ($R_1$)**: $10\text{ k}\Omega$
- **GND 측 저항 ($R_2$)**: $5\text{ k}\Omega$
- **분배 비율 ($K$)**: 
  $$K = \frac{R_1 + R_2}{R_2} = \frac{10\text{k}\Omega + 5\text{k}\Omega}{5\text{k}\Omega} = 3$$

### 1.3 경고 부저
- **부저 모델**: **TMB12A05** (능동 부저 / Active Buzzer)
- **제어 핀**: **PB7** (아두이노 헤더 **D4** 핀)

---

## 2. 실제 측정 데이터 및 상태 테이블

실제 측정된 전압값과 계산된 배터리 상태 테이블입니다.

### 2.1 실제 측정 예시 데이터
- **측정된 ADC Raw 값**: `2090`
- **ADC 입력 핀 전압 ($V_{ADC}$)**: `1.684 V`
- **계산된 실제 배터리 전압 ($V_{IN}$)**:
  $$V_{IN} = 1.684\text{ V} \times 3 = \mathbf{5.052\text{ V}}$$

### 2.2 2S 배터리 전압 대조표

| 배터리 상태 | 실제 전원 전압 ($V_{IN}$) | ADC 입력 전압 ($V_{ADC}$) | ADC Raw 값 | 부저 동작 상태 |
| :--- | :--- | :--- | :--- | :--- |
| **100% 만충** | **`8.40 V`** | **`2.80 V`** (2.79V 측정) | **약 3475** | 정상 (OFF) |
| **75% 충전** | **`7.80 V`** | **`2.60 V`** | **약 3227** | 정상 (OFF) |
| **50% (공칭 전압)** | **`7.40 V`** | **`2.47 V`** | **약 3061** | 정상 (OFF) |
| **실제 측정 데이터** | **`5.05 V`** | **`1.684 V`** | **`2090`** | ⚠️ 방전 경고 (ON) |
| **0% (방전 컷오프)** | **`6.40 V` 이하** | **`2.13 V` 이하** | **약 2647 이하** | ⚠️ 방전 경고 (ON) |

---

## 3. STM32CubeMX (.ioc) 설정 방법

### 3.1 ADC1 설정
1. **Pinout**: `PA0` 핀을 **`ADC1_IN0`** (아두이노 A0)으로 지정
2. **Parameter Settings**:
   - **Mode**: Independent mode
   - **Data Alignment**: Right alignment (12-bit)
   - **Scan / Continuous Mode**: Disabled
   - **Sampling Time**: `239.5 Cycles` (안정적인 측정)
3. **Clock Configuration 경고 해결**:
   - STM32F103 ADC 최대 허용 클럭은 **14MHz**입니다.
   - PCLK2가 64MHz일 경우, **`ADC Prescaler`**를 **`/ 6`** (10.67MHz) 또는 **`/ 8`** (8MHz)로 설정하여 붉은색 경고 박스를 해제합니다.

### 3.2 부저 GPIO 설정
- **Pinout**: **`PB7`** 핀을 **`GPIO_Output`** 으로 설정

---

## 4. 소스 코드 구현 (`main.c`)

프로젝트의 `main.c` 파일 내 사용자 코드 구역(`/* USER CODE BEGIN ... */`)에 적용된 핵심 코드입니다.

### 4.1 변수 및 선언 (`USER CODE BEGIN PV`, `PFP`)
```c
/* USER CODE BEGIN PV */
extern ADC_HandleTypeDef hadc1;  // ADC 핸들

uint32_t adc_raw = 0;
float v_adc = 0.0f;
float v_input = 0.0f;

#define R1_OHM  10000.0f  // 10k 저항
#define R2_OHM   5000.0f  // 5k 저항
#define VREF     3.3f
static uint32_t last_battery_tick = 0;
/* USER CODE END PV */

/* USER CODE BEGIN PFP */
uint32_t Read_ADC_Average(uint8_t count);
/* USER CODE END PFP */
```

### 4.2 ADC 샘플링 함수 (`USER CODE BEGIN 0`)
```c
/* USER CODE BEGIN 0 */
// ADC 노이즈 제거를 위한 N회 평균 측정 함수
uint32_t Read_ADC_Average(uint8_t count)
{
    uint32_t sum = 0;
    for (uint8_t i = 0; i < count; i++)
    {
        HAL_ADC_Start(&hadc1);
        if (HAL_ADC_PollForConversion(&hadc1, 10) == HAL_OK)
        {
            sum += HAL_ADC_GetValue(&hadc1);
        }
        HAL_ADC_Stop(&hadc1);
    }
    return (count > 0) ? (sum / count) : 0;
}
/* USER CODE END 0 */
```

### 4.3 초기화 (`USER CODE BEGIN 2` & `MX_GPIO_Init_2`)
```c
/* USER CODE BEGIN 2 */
HAL_ADCEx_Calibration_Start(&hadc1);  // ADC 보정 수행
HAL_GPIO_WritePin(GPIOB, GPIO_PIN_7, GPIO_PIN_RESET); // 부저 초기 OFF
/* USER CODE END 2 */

/* USER CODE BEGIN MX_GPIO_Init_2 */
GPIO_InitTypeDef GPIO_InitStruct_Buzzer = {0};
GPIO_InitStruct_Buzzer.Pin = GPIO_PIN_7;
GPIO_InitStruct_Buzzer.Mode = GPIO_MODE_OUTPUT_PP;
GPIO_InitStruct_Buzzer.Pull = GPIO_NOPULL;
GPIO_InitStruct_Buzzer.Speed = GPIO_SPEED_FREQ_LOW;
HAL_GPIO_Init(GPIOB, &GPIO_InitStruct_Buzzer);
HAL_GPIO_WritePin(GPIOB, GPIO_PIN_7, GPIO_PIN_RESET);
/* USER CODE END MX_GPIO_Init_2 */
```

### 4.4 메인 루프 모니터링 (`USER CODE BEGIN WHILE`)
```c
/* USER CODE BEGIN WHILE */
// 1초에 한 번 배터리 전압 체크 및 방전 시 PB7 부저 울림 (Non-blocking)
if (now - last_battery_tick >= 1000) {
    adc_raw = Read_ADC_Average(10);
    v_adc = ((float)adc_raw / 4095.0f) * VREF;
    v_input = v_adc * ((R1_OHM + R2_OHM) / R2_OHM);

    if (v_input <= 6.4f) { // 2S 배터리 6.4V 이하 방전 감지
        printf("⚠️ [경고] 2S 배터리 방전! (전압: %.2fV)\r\n", v_input);
        // PB7 부저 0.15초 켜졌다가 끔
        HAL_GPIO_WritePin(GPIOB, GPIO_PIN_7, GPIO_PIN_SET);
        HAL_Delay(150);
        HAL_GPIO_WritePin(GPIOB, GPIO_PIN_7, GPIO_PIN_RESET);
    } else {
        HAL_GPIO_WritePin(GPIOB, GPIO_PIN_7, GPIO_PIN_RESET);
    }
    last_battery_tick = now;
}
/* USER CODE END WHILE */
```

---

## 5. 결론 및 동작 특징

1. **안정성**: 1초마다 0.15ms 폴링 방식으로 측정하므로 메인 루프(자이로 적분, 모터 제어, UART 통신)에 지장을 주지 않습니다.
2. **정밀도**: STM32F103 ADC 오차 보정(`HAL_ADCEx_Calibration_Start`)과 10회 평균화를 적용하여 노이즈를 최소화했습니다.
3. **직관적인 경고**: 6.4V 이하 방전 시 TMB12A05 능동 부저(PB7)를 통해 1초 간격으로 경고음이 울려 배터리 과방전을 방지합니다.





