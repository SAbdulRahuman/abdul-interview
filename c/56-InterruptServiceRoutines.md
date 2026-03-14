# Interrupt Service Routines

---

## ISR Basics

```c
void __attribute__((interrupt)) timer_isr(void) {
    // Keep ISR short – set flag, clear interrupt, return
    flag = 1;
    TIMER_REG &= ~INTERRUPT_FLAG;
}
```

## Rules for ISRs

```
- Keep ISRs as short as possible
- Set flags for main loop to process
- Use volatile for shared variables
- Only call async-signal-safe / reentrant functions
- Avoid: malloc, printf, floating point, complex computations
- Clear the interrupt flag before returning
- Nested interrupts: be aware of priority levels
```

## ISR-Safe Communication

```c
volatile int data_ready = 0;
volatile int buffer[BUFFER_SIZE];

// ISR:
void data_isr(void) {
    buffer[index++] = DATA_REG;
    data_ready = 1;
    CLEAR_IRQ();
}

// Main loop:
while (1) {
    if (data_ready) {
        process(buffer);
        data_ready = 0;
    }
}
```

---

## Interview Q&A

**Q: What is an ISR?**

```
Interrupt Service Routine – a function that runs in response to a hardware
or software interrupt. Interrupts the normal program flow, executes quickly,
then returns to the interrupted code.
```

**Q: Why avoid `printf` in ISRs?**

```
printf is not reentrant. It uses internal buffers and locks.
If the main program was inside printf when the interrupt fires,
calling printf again from the ISR can corrupt state or deadlock.
```

---
