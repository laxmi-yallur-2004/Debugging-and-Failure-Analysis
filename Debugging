// ============================================================
// TOPIC: Debugging & Failure Analysis - ST-LINK ONLY
// Board: STM32F401CCU6 Black Pill
// Debug: ST-Link SWD Only (PA13 SWDIO, PA14 SWCLK)
// Serial: USB Serial - Sirf Reading ke liye (Baud 9600)
// 5 Real Bugs -> SWD se Detect -> SWD se Recover -> Serial me Report
// NO AUTO RECOVER - Recovery Sirf ST-Link SWD se
// UPDATED: BKPT only runs when debugger is attached, so PC/LR are correct
// ============================================================
#include <Arduino.h>

#define LED_PIN PA5

// ============ FAULT DATA - Reboot par bhi bachta hai ============
__attribute__((section(".noinit"))) volatile uint32_t saved_magic;
__attribute__((section(".noinit"))) volatile uint32_t saved_PC;
__attribute__((section(".noinit"))) volatile uint32_t saved_LR;
__attribute__((section(".noinit"))) volatile uint32_t saved_CFSR;
__attribute__((section(".noinit"))) volatile uint32_t saved_HFSR;
__attribute__((section(".noinit"))) volatile uint32_t saved_Code;
__attribute__((section(".noinit"))) volatile uint32_t saved_Count;

volatile uint32_t fault_Code = 0;
volatile uint32_t fault_Count = 0;
volatile uint32_t testCounter = 0;
volatile int32_t sensorValue = 0;

// ---- Common Fault Capture - SWD ONLY ----
extern "C" void Fault_Capture(uint32_t *stack) {
  saved_PC = stack[6];   // address of the faulting instruction
  saved_LR = stack[5];   // LR at the time of the fault
  saved_CFSR = *(volatile uint32_t*)0xE000ED28;
  saved_HFSR = *(volatile uint32_t*)0xE000ED2C;
  saved_Code = fault_Code;
  saved_Count = fault_Count + 1;
  saved_magic = 0xDEADBEEF;

  __DSB();

  // 1. DETECT THROUGH SWD ONLY
  // Halt only if ST-Link debugger is attached.
  // Without this check, BKPT causes a second HardFault that
  // overwrites PC/LR with wrong values (LR=0xFFFFFFF9).
  if (CoreDebug->DHCSR & CoreDebug_DHCSR_C_DEBUGEN_Msk) {
    __BKPT(0); // ST-Link Yahan Ruk Jayega
  }

  // 2. RECOVER THROUGH SWD ONLY - Khud Reset NAHI karega
  RCC->AHB1ENR |= (1 << 0);
  GPIOA->MODER &= ~(3 << 10);
  GPIOA->MODER |= (1 << 10);
  while(1) {
    GPIOA->ODR ^= (1 << 5);
    for (volatile uint32_t i = 0; i < 800000; i++) __NOP();
    __WFI(); // ST-Link se Reset ka Wait
  }
}

// ---- 4 Handlers ----
extern "C" void HardFault_Handler(void) __attribute__((naked));
extern "C" void HardFault_Handler(void) {
  __asm volatile("TST LR, #4 \n ITE EQ \n MRSEQ R0, MSP \n MRSNE R0, PSP \n B Fault_Capture \n");
}
extern "C" void MemManage_Handler(void) __attribute__((naked));
extern "C" void MemManage_Handler(void) {
  __asm volatile("TST LR, #4 \n ITE EQ \n MRSEQ R0, MSP \n MRSNE R0, PSP \n B Fault_Capture \n");
}
extern "C" void BusFault_Handler(void) __attribute__((naked));
extern "C" void BusFault_Handler(void) {
  __asm volatile("TST LR, #4 \n ITE EQ \n MRSEQ R0, MSP \n MRSNE R0, PSP \n B Fault_Capture \n");
}
extern "C" void UsageFault_Handler(void) __attribute__((naked));
extern "C" void UsageFault_Handler(void) {
  __asm volatile("TST LR, #4 \n ITE EQ \n MRSEQ R0, MSP \n MRSNE R0, PSP \n B Fault_Capture \n");
}

// ============ 5 REAL BUGS ============

void Bug1_LoopOverrun() {
  fault_Code = 1;
  volatile uint8_t sensorBuf[10];
  for (int i = 0; i < 200; i++) {
    sensorBuf[i] = 100;
    sensorValue = i;
  }
}

void Bug2_IntegerOverflow() {
  fault_Code = 2;
  volatile uint8_t battery = 200;
  volatile uint8_t charge = 56;
  volatile uint8_t total = battery + charge; // REAL: 200+56=256 -> wraps to 0
  sensorValue = total; // sensorValue will be 0, proof of overflow

  // Pure Overflow Fault: Overflow made total = 0, now divide by 0
  volatile int32_t distance = 100;
  volatile int32_t speed = distance / total; // 100 / 0 -> UsageFault
  sensorValue = speed;
}

void Bug3_DivByZero() {
  fault_Code = 3;
  volatile int32_t distance = 100;
  volatile int32_t time = 0;
  volatile int32_t speed = distance / time;
  sensorValue = speed;
}

void Bug4_NullPointer() {
  fault_Code = 4;
  volatile uint32_t *sensorPtr = (volatile uint32_t*)0;
  *sensorPtr = 1234;
}

void Bug5_UnalignedAccess() {
  fault_Code = 5;
  sensorValue = 1234;
  volatile uint32_t *tedhaPtr = (volatile uint32_t*)0x20000001;
  *tedhaPtr = 0x12345678; // UsageFault - UNALIGNED
}

void InjectBug(uint8_t num, const char* name, void (*func)()) {
  Serial.print("[INFO] Injecting Bug ");
  Serial.print(num);
  Serial.print(": ");
  Serial.println(name);
  Serial.flush();
  delay(200);
  func();
}

void setup() {
  pinMode(LED_PIN, OUTPUT);
  digitalWrite(LED_PIN, LOW);
  Serial.begin(9600);

  SCB->CCR |= SCB_CCR_DIV_0_TRP_Msk | SCB_CCR_UNALIGN_TRP_Msk;
  SCB->SHCSR |= SCB_SHCSR_MEMFAULTENA_Msk | SCB_SHCSR_BUSFAULTENA_Msk | SCB_SHCSR_USGFAULTENA_Msk;

  if (saved_magic == 0xDEADBEEF) {
    fault_Count = saved_Count;
    Serial.println("\n=== RECOVERED FROM FAULT ===");
    Serial.print("[FAULT] BUG CODE="); Serial.println(saved_Code);
    if (saved_Code == 1) Serial.println("[ROOT CAUSE] Loop Overrun - Array Index Out of Bounds");
    if (saved_Code == 2) Serial.println("[ROOT CAUSE] Integer Overflow - uint8_t 200+56=256->0, Div By 0");
    if (saved_Code == 3) Serial.println("[ROOT CAUSE] Division By Zero - time=0");
    if (saved_Code == 4) Serial.println("[ROOT CAUSE] Null Pointer - Sensor Not Initialized");
    if (saved_Code == 5) Serial.println("[ROOT CAUSE] Unaligned Access - 4-byte data on 0x20000001");
    Serial.print("[FAULT] PC=0x"); Serial.println(saved_PC, HEX);
    Serial.print("[FAULT] LR=0x"); Serial.println(saved_LR, HEX);
    Serial.print("[FAULT] CFSR=0x"); Serial.println(saved_CFSR, HEX);
    Serial.print("[FAULT] HFSR=0x"); Serial.println(saved_HFSR, HEX);
    Serial.print("[INFO] Total Faults="); Serial.println(fault_Count);
    Serial.println("[INFO] RECOVERY: Recovered via SWD Reset");
    Serial.flush();
    saved_magic = 0;
    delay(2000);
  }

  Serial.println("=== Debugging & Failure Analysis ===");
  Serial.println("[INFO] Board: STM32F401CCU6 - ST-Link SWD Debug");
  Serial.println("[INFO] 1=Loop Overrun  2=Integer Overflow  3=Div By Zero");
  Serial.println("[INFO] 4=Null Pointer  5=Unaligned Access");
  Serial.println("[INFO] SWD: Fault Detect + Recover via SWD Only");
  Serial.flush();
}

void loop() {
  testCounter++;
  digitalWrite(LED_PIN, HIGH);
  delay(500);
  digitalWrite(LED_PIN, LOW);
  delay(500);

  if (Serial.available()) {
    char c = Serial.read();
    if (c == '\r' || c == '\n') return;
    Serial.print("Got:"); Serial.println(c);
    Serial.flush();
    if (c == '1') InjectBug(1, "Loop Overrun", Bug1_LoopOverrun);
    if (c == '2') InjectBug(2, "Integer Overflow", Bug2_IntegerOverflow);
    if (c == '3') InjectBug(3, "Division By Zero", Bug3_DivByZero);
    if (c == '4') InjectBug(4, "Null Pointer", Bug4_NullPointer);
    if (c == '5') InjectBug(5, "Unaligned Access", Bug5_UnalignedAccess);
  }

  static uint32_t lastPrint = 0;
  if (millis() - lastPrint > 5000) {
    lastPrint = millis();
    Serial.print("[INFO] Alive Counter="); Serial.print(testCounter);
    Serial.print(" Sensor="); Serial.print(sensorValue);
    Serial.print(" Faults="); Serial.println(fault_Count);
    Serial.flush();
  }
}
