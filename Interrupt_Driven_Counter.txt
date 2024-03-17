.text
.global _start
display:
	@ r0 for input of Seven Segment
	 
	cmp r0, #0
	moveq r0, #63 @Decimal to Binary converter-0b00111111
	cmp r0, #1
	moveq r0, #6 @Decimal to Binary converter-0b00000110
	bxeq lr
	cmp r0, #2
	moveq r0, #91 @Decimal to Binary converter-0b01011011
	cmp r0, #3
	moveq r0, #79 @Decimal to Binary converter-0b01001111
	cmp r0, #4
	moveq r0, #102 @Decimal to Binary converter-0b01100110
	cmp r0, #5
	moveq r0, #109 @Decimal to Binary converter-0b01010101
	cmp r0, #6
	moveq r0, #125 @Decimal to Binary converter-0b01110110
	cmp r0, #7
	moveq r0, #7 @Decimal to Binary converter-0b00000111
	cmp r0, #8
	moveq r0, #127 @Decimal to Binary converter-0b01111111
	cmp r0, #9
	moveq r0, #111 @Decimal to Binary converter-0b01101111
	bx lr
timer:
	push {r6, r7, r8}
	b timer_loop
timer_loop:
	@ get the current count
	str r6, [r4, #16]
	@ now get the low count
    ldr r7, [r4, #16]
	@ get the high count
    ldr r8, [r4, #20]
	add r7, r8, lsl #16
	cmp r7, #0
    beq timer_end
    b timer_loop
timer_end:
	pop {r8, r7, r6}
	bx lr

.data
timer_T: .word 100000000

.text
_start:
	mov r1, #0b11010010         
	msr cpsr_c, r1  @This instruction writes the value of register r1 to the 
					@Current Processor Status Register (CPSR), which is a special 
					@register that controls various aspects of the processor's behavior, 
					@such as the operating mode and the interrupt settings.            
	ldr sp, =0xffffffff - 3  @This instruction loads the value of the stack pointer (sp)
							 @from memory. The value is calculated by subtracting 3 from
							 @the maximum 32-bit value (0xffffffff) and then loading that
							 @value into sp. This sets the stack pointer to the highest
							 @possible address, which is where the stack typically starts
							 @in ARM systems.
	mov r1, #0b11010011 @This instruction moves the value 0b11010011 into register r1, 
						@which is another value that modifies the CPSR.        
	msr cpsr, r1        @This instruction writes the value of register r1 to the CPSR, 
						@which modifies the processor mode to a different operating mode.     
	ldr sp, =0x3fffffff - 3     @This instruction loads the value of the stack pointer (sp) from memory.
								@The value is calculated by subtracting 3 from the maximum 30-bit value 
								@(0x3fffffff) and then loading that value into sp. This sets the stack pointer 
								@to a lower address than before.
	bl  CONFIG_GIC              
	ldr r0, =0xff200050 @This instruction loads the value 0xff200050 into register r0.
						@This is likely a memory-mapped I/O address that corresponds to a hardware
						@device on the system.
	mov r1, #0xf                
	str r1, [r0, #0x8]       
	mov r0, #0b01010011 @This instruction moves the value 0b01010011 into register r0,
						@which modifies the CPSR once again. The exact meaning of this value also
						@depends on the context and the system architecture.           
	msr cpsr_c, r0	@This instruction writes the value of register r0 to the CPSR,
					@which sets the processor mode to a different operating mode.
	
	@Registers Initialization
	ldr r5, =0xff200020 @ seven-segment displays
	ldr r4, =0xff202000 @ timer
	ldr r6, adr_T
	@ initialize the 1 s count
	ldr r7, [r6]
	str r7, [r4, #8]
	mov r1, r7, lsr #16
	str r1, [r4, #12]
	mov r1, #6
	str r1, [r4, #4]
	@ use address 0x3000 to store the flag bit
	@ if data in 0x3000 = 1, then go to add_loop
	@ if data in 0x3000 = 2, then go to sub_loop
	mov  r2, #1 @ go to add_loop at first
	mov  r0, #0x3000 @ initialize flag bit
	str  r2, [r0]
	mov r1, #0
	mov r2, #0
	mov r3, #0
	mov r8, #0
	mov r7, #1
	
main_loop: 
	ldr r0, =0x3000 @ read the data in 0x3000
	ldr r7, [r0] @ store it in r7
	mov r0, r1
	bl display @ for display, r0 as the input
	add r8, r0
	mov r0, r2
	bl display @ for display
	add r8, r0, lsl  #8 
	mov r0, r3
	bl display @ for display
	add r8, r0, lsl #16
	str r8, [r5] @ write data to seven-segment displays
	mov r8, #0
	bl timer @ for delay
	@ determine whether to add——loop or sub_loop 
	cmp r7, #1
	beq add_loop @ counter up
	bne subtract_loop @ counter down
add_loop:
	add r1, #1 @ increase counter
	@ if r1>10, then r1=0, r2++
	cmp r1, #10
	moveq r1, #0
	addeq r2, #1
	@ if r2>10, then r2=0, r3++
	cmp r2, #10
	moveq r2, #0
	addeq r3, #1
	cmp r3, #10
	moveq r3, #0
	b main_loop
subtract_loop:
	@ if r1=0, then r1=10, r2--
	cmp r1, #0
	bne subtract_end
first:
	moveq r1, #10
	cmp r2,#0
	subne r2, #1
	bne subtract_end
second:	
	@ if r2=0, then r2=9, r3--
	moveq r2, #9
	cmp r3, #0
	subne r3, #1
	moveq r3, #9
	b subtract_end
subtract_end:
	sub r1, #1 @ decrease counter ones
	b main_loop

/*Define the exception service routines*/
/*--- Undefined instructions --------------------------------------------------*/
SERVICE_UND:
	B SERVICE_UND
/*--- Software interrupts -----------------------------------------------------*/
SERVICE_SVC:
	B SERVICE_SVC
/*--- Aborted data reads ------------------------------------------------------*/
SERVICE_ABT_DATA:
	B SERVICE_ABT_DATA
/*--- Aborted instruction fetch -----------------------------------------------*/
SERVICE_ABT_INST:
	B SERVICE_ABT_INST
/*--- IRQ ---------------------------------------------------------------------*/
SERVICE_IRQ:
	push {r0-r7, lr}
	/*Read the ICCIAR from the CPU Interface*/
	ldr r4, =0xfffec100
	ldr r5, [r4, #0x0c]  @ read from ICCIAR
FPGA_IRQ1_HANDLER:
	cmp r5, #73
UNEXPECTED:
	bne UNEXPECTED @ if not recognized, stop here
	bl KEY_ISR
EXIT_IRQ:
	/*Write to the End of Interrupt Register (ICCEOIR)*/
	str r5, [r4, #0x10] @ write to ICCEOIR
	pop {r0-r7, lr}
	subs pc, lr, #4

/*--- FIQ ---------------------------------------------------------------------*/
SERVICE_FIQ:
	B	SERVICE_FIQ

/**Configure the Generic Interrupt Controller (GIC)*/
.global CONFIG_GIC
CONFIG_GIC:
	push   {lr}
	/*To configure the FPGA keys interrupt (ID 73):
	 *1. set the target to cpu0 in the ICDIPTRn register
	 *2. enable the interrupt in the ICDISERn register*/
	/*CONFIG_INTERRUPT (int_ID (r0), CPU_target (r1));*/
	mov	r0, #73         // KEY port (Interrupt ID = 73)
	mov    r1, #1          // this field is a bit-mask; bit 0 targets cpu0
	bl	CONFIG_INTERRUPT

	/*configure the GIC CPU Interface*/
	ldr	r0, =0xfffec100  // base address of CPU Interface
	/*Set Interrupt Priority Mask Register (ICCPMR)*/
	ldr    r1, =0xffff      // enable interrupts of all priorities levels
	str    r1, [r0, #0x04]
	/*Set the enable bit in the CPU Interface Control Register (ICCICR).
	 *This allows interrupts to be forwarded to the CPU(s)*/
	mov    r1, #1
	str    r1, [r0]

	/*Set the enable bit in the Distributor Control Register (ICDDCR).
	 *This enables forwarding of interrupts to the CPU Interface(s)*/
	ldr    r0, =0xfffed000
	str    r1, [r0]
	pop    {pc}

/*
 *Configure registers in the GIC for an individual Interrupt ID
 *We configure only the Interrupt Set Enable Registers (ICDISERn) and
 *Interrupt Processor Target Registers (ICDIPTRn). The default (reset)
 *values are used for other registers in the GIC
 *Arguments: r0 = Interrupt ID, N
 *	     r1 = CPU target
*/
CONFIG_INTERRUPT:
	PUSH   {R4-R5, LR}

	/*Configure Interrupt Set-Enable Registers (ICDISERn).
	 *reg_offset = (integer_div(N / 32)*4
	 *value = 1 << (N mod 32)*/
	LSR    R4, R0, #3         // calculate reg_offset
	BIC    R4, R4, #3         // R4 = reg_offset
	LDR    R2, =0xFFFED100
	ADD    R4, R2, R4         // R4 = address of ICDISER

	AND    R2, R0, #0x1F      // N mod 32
	MOV    R5, #1             // enable
	LSL    R2, R5, R2         // R2 = value

	/*Using the register address in R4 and the value in R2 set the
	 *correct bit in the GIC register*/
	LDR    R3, [R4]           // read current register value
	ORR    R3, R3, R2         // set the enable bit
	STR    R3, [R4]           // store the new register value

	/*Configure Interrupt Processor Targets Register (ICDIPTRn)
	 *reg_offset = integer_div(N / 4)*4
	 *index = N mod 4*/
	BIC    R4, R0, #3         // R4 = reg_offset
	LDR    R2, =0xFFFED800
	ADD    R4, R2, R4         // R4 = word address of ICDIPTR
	AND    R2, R0, #0x3       // N mod 4
	ADD    R4, R2, R4         // R4 = byte address in ICDIPTR

	/*Using register address in R4 and the value in R2 write to
	 *(only) the appropriate byte*/
	STRB   R1, [R4]

	POP {R4-R5, PC}


/*************************************************************************
 *Pushbutton - Interrupt Service Routine
 *
 *This routine checks which KEY has been pressed. It writes to HEX0
 ************************************************************************/
.global KEY_ISR
KEY_ISR:
	@ if any key is pressed, read the data in 0x2000
	MOV     R0, #0x3000
	LDR     R1, [R0]
	@ if data in 0x3000 = 1, then wirte 2 to 0x3000
	@ if data in 0x3000 = 2, then wirte 1 to 0x3000
	cmp     R1,#1
	moveq   r2, #2
	movne   r2, #1
	STR     R2, [R0]
	LDR     R0, =0xFF200050  // base address of pushbutton KEY port
	LDR     R1, [R0, #0xC]   // read edge capture register
	MOV     R2, #0xF
	STR     R2, [R0, #0xC]   // clear the interrupt

END_KEY_ISR:
	BX LR

adr_T: .word timer_T
.section .vectors, "ax"
	B _start             // reset vector
	B SERVICE_UND        // undefined instruction vector
	B SERVICE_SVC        // software interrrupt vector
	B SERVICE_ABT_INST   // aborted prefetch vector
	B SERVICE_ABT_DATA   // aborted data vector
.word 0                  // unused vector
	B SERVICE_IRQ        // IRQ interrupt vector
	B SERVICE_FIQ        // FIQ interrupt vector