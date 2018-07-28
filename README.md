;totally new to this whole Github thing, so if you're reading this, chances are I did it wrong here.
;anyways there's no license on this because its free

;this is a .asm file, used and assembled in MPLABX, and programmed via the PICKIT-3 to the PIC12F509 chip
;semi-colons are for writing comments.

LIST       P=12F509          ; tells the linker we're using the 12F509 processor
#INCLUDE  <P12F509.INC>      ; common pic opcodes, etc. 

      ;reset off, code protect off, watchdog timer off, int RC (4MHz) Oscil. on
__CONFIG  _MCLRE_OFF  &  _CP_OFF  & _WDT_OFF  & _IntRC_OSC

          UDATA_SHR ;Unitilized-shared data, not specifying where in memory, however shared means they are mapped in every bank
                    ; so no bank select with these
sGPIO     RES 1     ; reserve 1 byte of data memory for sGPIO: sGPIO to be shadow register of GPIO
dly_cnt   RES 1     ; reserve 1 byte of data memory for dly_cnt (delay-count)
db_cnt    RES 1     ; reserve 1 byte of data memory for db_cnt  (debounce count)
dc1       RES 1     ; reserve 1 byte of data memory for dc1 (debounce counter-1)

RCCAL     CODE    0X3FF   ;Processor reset vector, contains a 'MOVLW K' instruction, which loads the working register with the
          RES     1       ; internal factory recallibration vector: important in baseline PICs to ensure they run at their rated
                          ; frequency. Don't play with the circuit while you're programming it: can cause this area in memory to 
                          ; become over-written
                          
RESET     CODE    0X000   ; This is where your code starts when you first power it on, or reset the PIC
          MOVWF   OSCCAL  ; Load calibration vector into w
          
START
      ;Configure Ports:
          CLRF    GPIO
          CLRF    sGPIO
          MOVLW   B'111100'
          TRIS    GPIO
      ;Configure Timer: 
          MOVLW   B'11110110'
                  ;'--1-----' counter mode enabled (TOCS bit = 1)
                  ;'----0---' prescaler assigned to TIMER0 (PSA bit = 0)
                  ;'-----110' prescale = 128 (PS bits = 110)
                  ;should not that our circuit is using an external oscillating circuit connected to GP2
                  ;it's a 32.768 kHz watch crystal, the prescaler increments the counter by 32.768kHz/128= 256Hz
                  ; 256 Hz!! Yes, that's write, our TIMER0 will overflow its register ONCE a SECOND!
                  ; that means TIMER0<7> will change its state ever 500ms
          OPTION  ; LOADING OPTION WITH WORKING REGISTER.
          
       ;Main Loop
MAIN_LOOP
DB_DN     MOVLW   .13
          MOVWF   db_cnt
          MOVLW   .96
          MOVWF   dc1
DN_DLY
          BCF     sGPIO,0   ;Good place to monitor TIMER0-bit7, as this is where the majority of the program resides during
          BTFSC   TMR0,7    ;program execution 
          BSF     sGPIO,0
          MOVF    sGPIO,W
          MOVWF   GPIO
          
          DECFSZ  dc1,F   ;decrement file register and skip next line if file register becomes zero, save to file register
          GOTO    DN_DLY    ;(8us * 96) - 1 + 4 = 771us
          
          BTFSC   GPIO,3 ;bit test file register GPIO, GP3, skip next line if the input reads that the button has been pushed
          GOTO    DB_DN  ;--otherwise reset count and continue monitoring for a button switch
          
          DECFSC  db_cnt,F  ;decrement file register db_cnt,, yadada
          GOTO    DN_DLY  ; { (771us + 5us) * 13 } - 1 = 10,087us ==> about 10ms to debounce
          
          MOVF    sGPIO,W       ;move current value of sGPIO into working register
          XORLW   B'000010'     ; exclusive-or W with bitmask to flip value of GP1 in sGPIO
          MOVWF   sGPIO         ; update sGPIO
          MOVWF   GPIO          ; update the actual port on GPIO
          
          
DB_UP     MOVLW   .13           ; *******************same thing for these lines, they do the same as above except...*****
          MOVWF   db_cnt
          MOVLW   .96
          MOVWF   dc1
UP_DLY
          BCF     sGPIO,0       ; second place that the code will most likely reside while the PB-is being held down
          BTFSC   TMR0,7
          BSF     sGPIO,0
          MOVF    sGPIO,W
          MOVWF   GPIO
          
          DECFSZ  dc1,F
          GOTO    UP_DLY      ;8us*96 - 1 + 4 = 771us
          
          BTFSS   GPIO,3    ;**********......Except here: tests to see if the switch is being held continuously high*****
          GOTO    DB_UP     ; otherwise it resets
          
          DECFSZ  db_cnt,F
          GOTO    UP_DLY     ;(771us + 5us) * 13 - 1 = 10,087us ==> about 10ms
          
          GOTO    MAIN_LOOP   ; start again
          
          END
          
          
          
    ; 4 ports being used in this circuit.
    ; GP3 is connect to a switch, and the circuit is held high, once the switch is pressed,
    ; it is debounced
    
    ; GP2 is an input, it contains the 32.768 kHz watch crystal and inverter circuit
    
    ; GP1 is active high, connects to an LED and ground
    ; GP0 is active high, connects to an LED and ground 
       
          
                  
                  
                  
                  
                  
                  
                  
                  
