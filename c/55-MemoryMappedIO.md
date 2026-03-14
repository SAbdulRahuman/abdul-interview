# Memory-Mapped I/O

---

## Hardware Register Access

```c
// Access hardware register at fixed address
#define GPIO_PORT (*(volatile unsigned int *)0x40021000)

GPIO_PORT |= (1 << 5);   // set bit 5
GPIO_PORT &= ~(1 << 5);  // clear bit 5

// volatile prevents compiler from caching the value in a register
```

## Common Patterns

```c
// Read-only register
#define STATUS_REG (*(volatile const unsigned int *)0x40021004)

// Write-only register
#define COMMAND_REG (*(volatile unsigned int *)0x40021008)

// Register struct overlay
typedef struct {
    volatile uint32_t CR;     // control register
    volatile uint32_t SR;     // status register
    volatile uint32_t DR;     // data register
} UART_TypeDef;

#define UART1 ((UART_TypeDef *)0x40011000)
UART1->CR |= (1 << 0);  // enable UART
```

---

## Interview Q&A

**Q: Why is `volatile` essential for memory-mapped I/O?**

```
Without volatile, the compiler may:
  - Cache the register value in a CPU register and never re-read
  - Eliminate "redundant" reads/writes
  - Reorder accesses
volatile forces every read/write to go to the actual memory address.
```

---
