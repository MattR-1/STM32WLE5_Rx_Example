# STM32WLE5_Rx_Example

Umsetzung der Sequence in RM0461 Kapitel 4.9.1 mit HAR_Reference ... UM2642 und dem Beispiel aus dem STM32CubeWL firmware package (UM2643)



## 4.9.1 Basic sequence for LoRa, (G)MSK and (G)FSK transmit operation

The sub-GHz radio can be set in LoRa, (G)MSK or (G)FSK transmit operation mode with
the following steps:

1. Define the location of the transmit payload data in the data buffer, with
   Set_BufferBaseAddress().
2. Write the payload data to the transmit data buffer with Write_Buffer().
3. Select the packet type (generic or LoRa) with Set_PacketType().
4. Define the frame format with Set_PacketParams().
5. Define synchronization word in the associated packet type SUBGHZ_xSYNCR(n) with
   Write_Register( ).
6. Define the RF frequency with Set_RfFrequency().
7. Define the PA configuration with Set_PaConfig().
8. Define the PA output power and ramping with Set_TxParams().
9. Define the modulation parameters with Set_ModulationParams().
10. Enable TxDone and timeout interrupts by configuring IRQ with Cfg_DioIrq().
11. Start the transmission by setting the sub-GHz radio in TX mode with Set_Tx(). After
    the transmission is finished, the sub-GHz radio enters automatically the Standby mode.
12. Wait for sub-GHz radio IRQ interrupt and read interrupt status with
    Get_IrqStatus():
    a) On a TxDone interrupt, the packet is successfully sent
    b) On a timeout interrupt, the transmission is timeout.
13. Clear interrupt with Clr_IrqStatus().
14. Optionally, send a Set_Sleep() command to force the sub-GHz radio in Sleep mode.



### 1. Buffer Adresse setzen

#### code

RadioParam[0] = 0x80U;

RadioParam[1] = 0x00U;

if (HAL_SUBGHZ_ExecSetCmd(&hsubghz, RADIO_SET_BUFFERBASEADDRESS, &RadioParam, 2) != *HAL_OK*)

{

​	Error_Handler();

}



#### Erklärung

Optcode: 0x8fU



**Set_BufferBaseAddress() command**
Set_BufferBaseAddress(TxBaseAddr, RxBaseAddr) sets the data buffer base
address for the packet handling in TX and RX.

byte 0 (w) bits 7:0 Opcode: 0x8F
byte 1 (w) bits 7:0 TxBaseAddr[7:0]: Tx base address offset relative to the sub-GHz RAM base address
byte 2 (w) bits 7:0 RxBaseAddr[7:0]: Rx base address offset relative to the sub-GHz RAM base address



Aus RM0461 - Kapitel 4.6:

In order to retrieve data after Sleep mode
retention, the default values must be used (TxBaseAddr = 0x80 and RxBaseAddr = 0x00),
or RxPayloadLength and RxBufferPointer must be stored in the CPU memory.



Dementsprechend werden folgende Bytes gesendet:

|      | byte 1      | byte 2      | byte 3      |
| ---- | ----------- | ----------- | ----------- |
| hex  | 0x8F        | 0x80        | 0x00        |
| bin  | 0b1000 1111 | 0b1000 0000 | 0b0000 0000 |



### 2. Payload in den Buffer schreibe

#### code

uint8_t payload[64] = "Hello World!";

if (HAL_SUBGHZ_WriteBuffer(&hsubghz, 0, &payload, sizeof(payload)) != *HAL_OK*)

{

​	Error_Handler();

}

#### Erklärung

die 0 ist der Offset im Buffer. Der rest ist selbsterklärend



### 3. Packet Type setzen

#### code

RadioParam[0] = 0x01U;

if (HAL_SUBGHZ_ExecSetCmd(&hsubghz, *RADIO_SET_PACKETTYPE*, &RadioParam, 1) != HAL_OK)

{

​	Error_Handler();

}

#### Erklärung 

Set_PacketType(PktType) allows the selection of packet frame format. This command
must be the first command of a sub-GHz radio configuration sequence.
Changing from one sub-GHz radio configuration to another is done using
Set_PacketType(). The parameters from the previous sub-GHz radio configuration are
lost. The switch from one configuration mode to another is only accepted in Standby mode.

byte 0 (w)  bits 7:0 Opcode: 0x8A.
byte 1 (w)  bits 7:2 Reserved, must be kept at reset value.
		   bits 1:0 PktType[1:0]: Packet type definition
0: FSK generic packet type
1: LoRa packet type
2: BPSK packet type
3: MSK generic packet type
Other: reserved

byte 1: 0b00000001 = 0x01



### 4. Frame Format festlegen

#### code

RadioParam[0] = 0x00U;

RadioParam[1] = 0x0CU;

RadioParam[2] = 0x00U;

RadioParam[3] = 0x40U;

RadioParam[4] = 0x01U;

RadioParam[5] = 0x00U;

if (HAL_SUBGHZ_ExecSetCmd(&hsubghz, RADIO_SET_PACKETPARAMS, &RadioParam, 6) != *HAL_OK*)

{

​	Error_Handler();

}

#### Erklärung

**LoRa Set_PacketParams() command**
Set_PacketParams(PbLength, HeaderType, PayloadLength, CrcType,
InvertIQ) is used to configure the packet handling for the sub-GHz radio. Depending on
the selected packet type in Set_PacketType() sent prior to this function, the parameters
are interpreted as below.

byte 0           bits 7:0       Opcode: 0x8C.
bytes 2:1      bits 15:0     PbLength[15:0]: Preamble length in number of symbols
                                             0x0000: reserved
                                             0x0001 - 0xFFFF: 1 to 65535 symbols
byte 3           bits 7:1       Reserved, must be kept at reset value.
                      bit 0            HeaderType: Header type definition
                                             0: explicit header for variable length payload
                                             1: implicit header for fixed length payload
byte 4           bits 7:0       PayloadLength[7:0]: Payload length in number of bytes
                                             0x00- 0xFF: 0 to 255 bytes
byte 5           bits 7:1       Reserved, must be kept at reset value.
                      bit 0            CrcType CRC enable
                                             0: CRC disabled
                                             1: CRC enabled
byte 6           bits 7:1       Reserved, must be kept at reset value.
                      bit 0            InvertIQ: IQ setup
                                             0: standard IQ setup
                                             1: inverted IQ setup



|        | hex  | bin        |
| ------ | ---- | ---------- |
| byte 1 | 0x00 | 0b00000000 |
| byte 2 | 0x0C | 0b00001100 |
| byte 3 | 0x00 | 0b00000000 |
| byte 4 | 0x40 | 0b01000000 |
| byte 5 | 0x01 | 0b00000001 |
| byte 6 | 0x00 | 0b00000000 |

Erklärung in RM0461 - 4.5.2

byte 1 und byte 2: By default, the packet is configured with a 12-symbol-long preamble sequence. 

byte 3: packets mit Header ausgewählt - explicit header mode

byte 4: da explicit header mode gewählt sollte dieser Wert egal sein. es wird aber 64 gewählt, da das die hälfte der RxBuffers ist und so zwei Packets gebuffert werden könnten.

byte 5: CRC (Cyclic redundancy check) wird enabled. Ka was das genau macht aber eine Redundanz kann mal nicht schaden

byte 6: das ist irgendwas super black magick artiges. Da machen wir einfach Standard und gut is.



### 5. Synchonisations word definieren

#### code

RadioParam[0] = 0x14U;

RadioParam[1] = 0x24U;

if (HAL_SUBGHZ_WriteRegisters(&hsubghz, (uint16_t) 0x740, &RadioParam, 2) != *HAL_OK*)

{

​	Error_Handler();

}

#### Erklärung

Hier wird definiert, ob es sich um ein privates oder öffentliches Netzwerk handelt. Also keine Ahnung, was das mit dem Netzwerk soll ... aber der default ist privat, also nehmen wir das.

4.10.33 Sub-GHz radio LoRa synchronization word MSB register
(SUBGHZ_LSYNCRH)
Address offset: 0x740
Reset value: 0x14

Bits 7:0 SYNCWORD[15:8]: LoRa synchronization word MSB bits [15:8]
0x14: LoRa private network
0x34: LoRa public network
Others: reserved

4.10.34 Sub-GHz radio LoRa synchronization word LSB register
(SUBGHZ_LSYNCRL)
Address offset: 0x741
Reset value: 0x24

Bits 7:0 SYNCWORD[7:0]: LoRa synchronization word LSB bits [7:0]
0x24: LoRa private network
0x44: LoRa public network
Others: reserved



### 6. Rf Frequenz definieren

#### code

RadioParam[0] = 0x33U;

RadioParam[1] = 0xBCU;

RadioParam[2] = 0xA1U;

RadioParam[3] = 0x00U;

if (HAL_SUBGHZ_ExecSetCmd(&hsubghz, RADIO_SET_RFFREQUENCY, &RadioParam, 4) != *HAL_OK*)

{

​	Error_Handler();

}

#### Erklärung

**Set_RfFrequency() command**
Set_RfFrequency(RfFreq) is used to lock the RF-PLL frequency to the transmit and receive frequency.

byte 0          bits 7:0       Opcode: 0x86
bytes 4:1     bits 31:0     RfFreq[31:0]: RF frequency
                                            RF-PLL frequency = 32e^6 x RFfreq / 2^25



Die Rf-Frequenz wird auf 868000000 Hz gesetzt (868MHz) das ist 33 BC A1 00



### 7. PA Konfiguration festlegen

#### code

RadioParam[0] = 0x04U;

RadioParam[1] = 0x00U;

RadioParam[2] = 0x01U;

RadioParam[3] = 0x01U;

if (HAL_SUBGHZ_ExecSetCmd(&hsubghz, RADIO_SET_PACONFIG, &RadioParam, 4) != *HAL_OK*)

{

​	Error_Handler();

}

#### Erklärung

Set_PaConfig() command
Set_PaConfig(PaDutyCycle, HpMax, PaSel, 0x01) is used to customize the
maximum output power and PA efficiency.

byte 0 bits 7:0 Opcode: 0x95
byte 1 bits 7:3 Reserved, must be kept at reset value.
bits 2:0 PaDutyCycle[2:0]: PA duty cycle (conduit angle) control
Duty cycle = 0.2 + 0.04 x PaDutyCycle[2:0] (see Table 27 for settings)
Caution: The following restrictions must be observed to avoid over-stress on
the PA:

- LP PA mode with synthesis frequency >400 MHz, PaDutyCycle
  must be < 0x7.
- LP PA mode with synthesis frequency < 400 MHz, PaDutyCycle
  must be < 0x4.
- HP PA mode, PaDutyCycle must be < 0x4.
  byte 2 bits 2:0 HpMax[2:0]: HP PA output power (see Table 27 for settings)
  bits 7:3 Reserved, must be kept at reset value.
  byte 3 bits 7:1 Reserved, must be kept at reset value.
  bit 0 PaSel: PA selection.
  0: HP PA selected
  1: LP PA selected (default)
  byte 4 bits 7:0 0x01



Da es sich bei den Dev Board um die Le - (Hardwarekonfiguration LP) handelt, wird einfach die mittlere Stufe mit 14dbm output eingestellt, da die Leistungsbeschränkungen des Chips berücksichtigt werden müssen. Das ist die mittlere Zeile. Dementsprechend gilt:

|        | hex  | bin  |
| ------ | ---- | ---- |
| byte 1 | 0x04 |      |
| byte 2 | 0x00 |      |
| byte 3 | 0x01 |      |
| byte 4 | 0x01 |      |



### 8. PA output power definieren

#### code

RadioParam[0] = 0x0EU;

RadioParam[1] = 0x04U;

if (HAL_SUBGHZ_ExecSetCmd(&hsubghz, RADIO_SET_TXPARAMS, &RadioParam, 2) != *HAL_OK*)

{

​	Error_Handler();

}

#### Erklärung

Set_TxParams() command
Set_TxParams(Power, RampTime) is used to set the transmit output power and the PA
ramp-up time.

byte 0 bits 7:0 Opcode: 0x8E
byte 1 bits 7:0 Power[7:0]: Output power setting
LP PA selected in Set_PaConfig()
0x0E:+ 14 dB
...
0x00: 0 dB
...
0xEF: - 17 dB
Others: reserved
HP PA selected in Set_PaConfig()
0x16:+ 22 dB
...
0x00: 0 dB
...
0xF7: - 9 dB
Others: reserved
byte 2 bits 7:0 RampTime[7:0]: PA ramp time for FSK, MSK and LoRa modulation
0x00: 10 μs
0x01: 20 μs
0x02; 40 μs
0x03: 80 μs
0x04: 200 μs
0x05: 800 μs
0x06: 1700 μs
0x07: 3400 μs
Others: reserved
Note: In BPSK mode, the ramping time is specific and RampTime[7:0] is not
used.



byte 1: Definiert die Power, welche in Tabelle 27 und Abschnitt 8 in diesem Document bereits festgelegt

byte 2: In diesem Fall wurden moderate 200 us gewählt. Einfach so. Weil ich habs ja nicht eilig.
butter bei die Fische: ich vermute mal, dass ne kurze PA ramp Zeit Störungen verursacht. Aber wissen tu ich hier nix.



|        | hex  | bin  |
| ------ | ---- | ---- |
| byte 1 | 0x0E |      |
| byte 2 | 0x04 |      |

### 9. Modulationsparameter festlegen

#### code

RadioParam[0] = 0x07U;
RadioParam[1] = 0x09U;
RadioParam[2] = 0x01U;
RadioParam[3] = 0x00U;

if (HAL_SUBGHZ_ExecSetCmd(&hsubghz, RADIO_SET_MODULATIONPARAMS, &RadioParam, 4) != HAL_OK)
{
	Error_Handler();
}

#### Erklärung

LoRa Set_ModulationParams() command
Set_ModulationParams(Sf, Bw, Cr, Ldro) is used to configure the LoRa
modulation parameters for the sub-GHz radio. Depending on the selected packet type in
Set_PacketType() sent prior to this function, the parameters for LoRa are interpreted as
below.

byte 0          bits 7:0           Opcode: 0x8B
byte 1          bits 7:4           Reserved, must be kept at reset value.
                     bits 3:0           Sf[3:0]: Spreading factor
                                                0x5: Spreading factor 5
                                                0x6: Spreading factor 6
                                                0x7: Spreading factor 7
                                                0x8: Spreading factor 8
                                                0x9: Spreading factor 9
                                                0xA: Spreading factor 10
                                                0xB: Spreading factor 11
                                                0xC: Spreading factor 12
                                                Others: reserved
byte 2          bits 7:0           Bw[7:0]: Bandwidth
                                                0x00: bandwidth 7 (7.81 kHz)
                                                0x08: bandwidth 10 (10.42 kHz)
                                                0x01: bandwidth 15 (15.63 kHz)
                                                0x09: bandwidth 20 (20.83 kHz)
                                                0x02: bandwidth 31 (31.25 kHz)
                                                0x0A: bandwidth 41 (41.67 kHz)
                                                0x03: bandwidth 62 (62.50 kHz)
                                                0x04: bandwidth 125 (125 kHz)
                                                0x05: bandwidth 250 (250 kHz)
                                                0x06: bandwidth 500 (500 kHz)
                                                Others: reserved
byte 3          bits 7:3           Reserved, must be kept at reset value.
                     bits 2:0           Cr[2:0]: Forward error correction coding rate
                                                0x0: no forward error correction coding rate 4/4
                                                0x1: forward error correction coding rate 4/5
                                                0x2: forward error correction coding rate 4/6
                                                0x3:forward error correction coding rate 4/7
                                                0x4: forward error correction coding rate 4/8
                                                Others: reserved
byte 4          bits 7:1           Reserved, must be kept at reset value.
                     bit 0                Ldro: low data rate optimization enable
                                                0: low data rate optimization disabled
                                                1: low data rate optimization enabled

byte 1: SF (Spreading factor) Default ist 7, also nehmen wir das. aber eigentlich wäre höher besser.

byte 2: BW (Bandwidth) Da gibts kein Default. Aber Je höher, desto mehr linkbudget aber weniger range. Also wird das untere drittel gewählt: 20.83kHz -> 0x09

byte 3: CR (Forward error correction coding rate) zwar ist der default 4/6 aber RM0461 sagt, dass 4/5 der beste trade off ist. Also nehmen wir 4/5

byte 4: LDRO (Low data rate optimization) Das hilft bei hoher BW oder niedrigem SF. Wir haben hier das gegenteil, also können wir uns das sparen



|        | hex  | bin  |
| ------ | ---- | ---- |
| byte 1 | 0x07 |      |
| byte 2 | 0x09 |      |
| byte 3 | 0x01 |      |
| byte 4 | 0x00 |      |



### 10. Interrupts configurieren

#### code

RadioParam[0] = 0x02U;
RadioParam[1] = 0xC1U;
RadioParam[2] = 0x00U;
RadioParam[3] = 0x01U;
RadioParam[4] = 0x02U;
RadioParam[5] = 0x00U;
RadioParam[6] = 0x00U;
RadioParam[7] = 0x00U;

if (HAL_SUBGHZ_ExecSetCmd(&hsubghz, RADIO_CFG_DIOIRQ, &RadioParam, 8) != HAL_OK)
{
	Error_Handler();
}

#### Erklärung

Cfg_DioIrq() command
Cfg_DioIrq(IrqMask, Irq1Mask, Irq2Mask, Irq3Mask) allows interrupts to be
masked and mapped on the IRQ lines.

byte 0 bits 7:0 Opcode: 0x08
bytes 2:1 bits 15:0 IrqMask[15:0]: Global interrupt enable
See Table 29 for interrupt bit map definition. For each bit:
0: IRQ disabled
1: IRQ enabled
bytes 4:3 bits 15:0 Irq1Mask[15:0]: IRQ1 line Interrupt enable
0: interrupt on IRQ1 line disable
1: interrupt on IRQ1 line enabled
bytes 6:5 bits 15:0 Irq2Mask[15:0]: IRQ2 line Interrupt enable
0: interrupt on IRQ2 line disable
1: interrupt on IRQ2 line enabled
bytes 8:7 bits 15:0 Irq3Mask[15:0]: IRQ3 line Interrupt enable
0: interrupt on IRQ3 line disable
1: interrupt on IRQ3 line enabled



Es soll festgestellt werden, ob das Paket gesendet wurde oder ob es einen Timeout gab. Dementsprechend wird nach Tabelle 29 in RM0461 IRQ-Source 0 (Tx Done) und IRQ-Source 9 (Timeout) aktiviert. Dabei sollen IRQ1 Tx-Done signalisieren und IRQ2 Timeout signalisieren. IRQ3 soll deaktiviert bleiben.

|        | hex  | bin        |
| ------ | ---- | ---------- |
| byte 1 | 0x02 | 0b00000010 |
| byte 2 | 0x01 | 0b00000001 |
| byte 3 | 0x00 | 0b00000000 |
| byte 4 | 0x01 | 0b00000001 |
| byte 5 | 0x02 | 0b00000010 |
| byte 6 | 0x00 | 0b00000000 |
| byte 7 | 0x00 | 0b00000000 |
| byte 8 | 0x00 | 0b00000000 |







### 11. Übertragung starten

#### code

RadioParam[0] = 0x09U;
RadioParam[1] = 0xC4U;
RadioParam[2] = 0x00U;

if (HAL_SUBGHZ_ExecSetCmd(&hsubghz, RADIO_SET_TX, &RadioParam, 3) != HAL_OK)
{
	Error_Handler();
}

#### Erklärung

Set_Tx() command
Set_Tx(Timeout) is used to set the sub-GHz radio in TX mode.

byte 0         bits 7:0       Opcode: 0x83
bytes 3:1    bits 23:0     Timeout[23:0]: Transmit packet timeout
                                            0x000000: Timeout disabled
                                            0x000001 - 0xFFFFFF: Timeout enabled, resolution 15.625 μs

Time-out duration is computed as follows:
Time-out duration = Timeout x 15.625 μs (maximum time-out duration = 262.14 s)
When Set_Tx(Timeout) is sent in Standby or Receive mode, the sub-GHz radio passes
through the FS mode (no need to send Set_Fs()). In this case, the RF-PLL frequency
must be set by Set_RfFrequency() prior sending Set_Tx(Timeout).

Es soll mal ein Timeout von 10s eingestellt werden, um zu verhindern, dass der Chip sich zu sehr aufheitzt.

Timeout = 10s/15.625us = 640000 = 0x9C400



|        | hex  | bin  |
| ------ | ---- | ---- |
| byte 1 | 0x09 |      |
| byte 2 | 0xC4 |      |
| byte 3 | 0x00 |      |















## Rx Bsp

4.9.2 Basic sequence for LoRa and (G)FSK receive operation
The sub-GHz radio can be set in LoRa or (G)FSK receive operation mode with the following
steps:

1. Define the location where the received payload data must be stored in the data buffer,
   with Set_BufferBaseAddress().
2. Select the packet type (generic or LoRa) with Set_PacketType().
3. Define the frame format with Set_PacketParams().
4. Define synchronization word in the associated packet type SUBGHZ_xSYNCR(n) with
   Write_Register().
5. Define the RF frequency with Set_RfFrequency().
6. Define the modulation parameters with Set_ModulationParams().
7. Enable RxDone and timeout interrupts by configuring IRQ with Cfg_DioIrq().
8. Start the receiver by setting the sub-GHz radio in RX mode with Set_Rx():
   – When in continuous receiver mode, the sub-GHz radio remains in RX mode to
   look for packets until stopped with Set_Standby().
   – In single mode (with or without timeout), when the reception is finished, the
   sub-GHz radio enters automatically the Standby mode.
   – In listening mode, the sub-GHz radio repeatedly switches between RX single with
   timeout mode and Sleep mode.
9. Wait for sub-GHz radio IRQ interrupt and read the interrupt status with
   Get_IrqStatus():
   a) On a RxDone interrupt, a packet is received:
   – Check received packet error status (header error, crc error) with
   Get_IrqStatus().
   – When a valid packet is received, read the receive start buffer pointer and received
   payload length with Get_RxBufferStatus().
   – Read the received payload data from the receive data buffer with
   Read_Buffer().
   b) On a timeout interrupt, the reception is timed out.
10. Clear interrupts with Clr_IrqStatus().
11. Optionally, send a Set_Sleep() command to force the sub-GHz radio in Sleep mode.



Es können vom Tx Bsp folgende Punkte übernommen werden: 1, 2, 3, 4, 5, 6. Diese entsprechen nich der Reihenfolge aus der Tx Bsp:

|                           | Tx Bsp | Rx Bsp |
| ------------------------- | ------ | ------ |
| Set Buffer Address        | 1      | 1      |
| Set Packet Type           | 3      | 2      |
| Set Packet Params         | 4      | 3      |
| Set Synchronisation Word  | 5      | 4      |
| Set Rf Frequency          | 6      | 5      |
| Set Modulation Parameters | 9      | 6      |



### 7. Set interrupts

Wird auch übersprungen



### 8. Set Rx

#### code

RadioParam[0] = 0x0EU;
RadioParam[1] = 0xA6U;
RadioParam[2] = 0x00U;

if (HAL_SUBGHZ_ExecSetCmd(&hsubghz, RADIO_SET_RX, &RadioParam, 3) != *HAL_OK*)
{
	Error_Handler();
}

#### Erklärung

Set_Rx() command
Set_Rx(Timeout) is used to set the sub-GHz radio in Receive mode.

byte 0 bits 7:0 Opcode: 0x82
bytes 3:1 bits 23:0 Timeout[23:0]: Transmit packet timeout
0x000000: timeout disabled
0x000001 - 0xFFFFFE: timeout enabled, single packet receive mode,
resolution 15.625 μs
0xFFFFFF: timeout disabled, continuous receive mode

Time-out duration is computed by the following formula:
Time-out duration = Timeout x 15.625 μs (maximum Time-out duration = 262.14 s)
When Set_Rx(Timeout) is sent in Standby mode or Transmit mode, the sub-GHz radio
passes through the FS mode (no need to send Set_Fs()). In this case, the RF-PLL
frequency must be set by Set_RfFrequency() prior sending Set_Rx(Timeout).

Es soll mal ein Timeout von 15s eingestellt werden, um dem Sender genug Zeit zu geben.

Timeout = 15s/15.625us = 960000 = 0xEA600



### 9. Read Buffer

#### code

uint8_t payload[64] = 0;

if (HAL_SUBGHZ_ReadBuffer(&hsubghz, 0, &payload, 64) != *HAL_OK*)

{

​	Error_Handler();

}

#### Erklärung

Read_Buffer() command
Read_Buffer(Offset, Status, Data0, Data1, to Datan) allows receive packet payload data to be read from a contiguous data memory area, starting from the specified offset. The offset is auto incremented after each byte. When the offset exceeds the value 255, it is wrapped around to 0 (providing a 256 byte circular buffer).

byte 0       bits 7:0    Opcode:         0x1E
byte 1       bits 7:0    Offset[7:0]:    first read address offset
byte 2       bits 7:0    Status[7:0]:    see Get_Status() command
byte 3       bits 7:0    Data0[7:0]:    data read from offset address
... ...
byte n+3  bits 7:0     Datan[7:0]:   data read from offset address + n (n = number of bytes to read)



Dabei wird die Maximale größe ausgelsesen, also 64 Bytes. Ebenfalls wird der Buffer von Anfang an gelesen - also ist der offset 0
















## Mit HAL Funktionen:

• **HAL_SUBGHZ_WriteRegisters()**
**• HAL_SUBGHZ_ReadRegisters()**
**• HAL_SUBGHZ_WriteRegister()**
**• HAL_SUBGHZ_ReadRegister()**
**• HAL_SUBGHZ_ExecSetCmd()**
**• HAL_SUBGHZ_ExecGetCmd()**
**• HAL_SUBGHZ_WriteBuffer()**
**• HAL_SUBGHZ_ReadBuffer()**
• HAL_SUBGHZ_IRQHandler()
• HAL_SUBGHZ_TxCpltCallback()
• HAL_SUBGHZ_RxCpltCallback()
• HAL_SUBGHZ_PreambleDetectedCallback()
• HAL_SUBGHZ_SyncWordValidCallback()
• HAL_SUBGHZ_HeaderValidCallback()
• HAL_SUBGHZ_HeaderErrorCallback()
• HAL_SUBGHZ_CRCErrorCallback()
• HAL_SUBGHZ_CADStatusCallback()
• HAL_SUBGHZ_RxTxTimeoutCallback()
• HAL_SUBGHZ_LrFhssHopCallback()