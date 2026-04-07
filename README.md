# Laboratorio: Ensamblado y Ejecución Paso a Paso
**Arquitectura de Computadores — Unidad 3: Manejo del DEBUG**
Post-Contenido 2 | Ingeniería de Sistemas | 2026

---

## Descripción del Laboratorio

El presente laboratorio tiene como objetivo ensamblar programas directamente en memoria usando el comando `A` del DEBUG, ejecutar instrucciones paso a paso con el comando `T`, registrar el estado de los registros y banderas en tablas de traza, y analizar el comportamiento de un programa con bucle utilizando el registro CX y la instrucción LOOP. Los resultados se documentan mediante tablas de traza y capturas de pantalla.

---

## Preparación del Entorno

```
Z:\> MOUNT C C:\DOSWork
Z:\> C:
C:\> MD LAB3POST2
C:\> CD LAB3POST2
C:\LAB3POST2> DEBUG
-
```

---

## Parte A — Programa de Suma con Traza Completa

### Programa ensamblado

```
-A 100
1357:0100 MOV AX, 000A   ; AX = 10
1357:0103 MOV BX, 0005   ; BX = 5
1357:0106 MOV CX, 0003   ; CX = 3
1357:0109 ADD AX, BX     ; AX = AX + BX = 15
1357:010B ADD AX, CX     ; AX = AX + CX = 18
1357:010D INT 20
```

### Verificación con U

```
-U 100 10E
1357:0100 B80A00   MOV AX,000A
1357:0103 BB0500   MOV BX,0005
1357:0106 B90300   MOV CX,0003
1357:0109 03C3     ADD AX,BX
1357:010B 03C1     ADD AX,CX
1357:010D CD20     INT 20
```

### Checkpoint 1 — Tabla de traza: Programa de suma

| # T | Instrucción ejecutada | AX     | BX     | CX     | IP sig. | ZF | CF | SF |
|-----|-----------------------|--------|--------|--------|---------|----|----|----|
| 1   | MOV AX, 000A          | 000A   | 0000   | 0000   | 0103    | NZ | NC | PL |
| 2   | MOV BX, 0005          | 000A   | 0005   | 0000   | 0106    | NZ | NC | PL |
| 3   | MOV CX, 0003          | 000A   | 0005   | 0003   | 0109    | NZ | NC | PL |
| 4   | ADD AX, BX            | 000F   | 0005   | 0003   | 010B    | NZ | NC | PL |
| 5   | ADD AX, CX            | 0012   | 0005   | 0003   | 010D    | NZ | NC | PL |

**Observaciones:**
- Las instrucciones MOV no alteran las banderas; ZF, CF y SF conservan el valor que tenían antes de ejecutarse.
- La primera operación aritmética `ADD AX, BX` produce 0x000A + 0x0005 = 0x000F (15 decimal). El resultado no es cero (ZF = NZ), no genera acarreo (CF = NC) y es positivo (SF = PL).
- La segunda operación `ADD AX, CX` produce 0x000F + 0x0003 = 0x0012 (18 decimal). Las banderas mantienen el mismo comportamiento: resultado no nulo, sin acarreo, signo positivo.
- El valor final en AX es 0x0012 = 18 decimal, que corresponde a la suma 10 + 5 + 3 = 18, verificando la correcta ejecución del programa.

**Captura:** `capturas/CP1_traza_suma.png`

---

## Parte B — Programa con Bucle usando LOOP y CX

### Programa ensamblado

```
-A 100
1357:0100 MOV AX, 0000   ; Acumulador = 0
1357:0103 MOV CX, 0004   ; Contador de iteraciones = 4
1357:0106 ADD AX, 0002   ; Suma 2 al acumulador (inicio del bucle)
1357:010A LOOP 0106      ; Decrementa CX, salta a 0106 si CX != 0
1357:010C INT 20         ; Terminar (AX debe ser 0x0008 = 8)
```

### Verificación con U

```
-U 100 10D
1357:0100 B80000   MOV AX,0000
1357:0103 B90400   MOV CX,0004
1357:0106 050200   ADD AX,+02
1357:0109 E2FB     LOOP 0106
1357:010B CD20     INT 20
```

**Nota sobre la codificación de LOOP:** El opcode E2 corresponde a la instrucción LOOP; el byte FB es el desplazamiento relativo firmado (-5 en decimal), calculado como 0x0106 − 0x010B = −5 = 0xFB. Los saltos cortos del 8086 usan desplazamientos relativos de 8 bits con signo, válidos en un rango de ±127 bytes respecto a la instrucción siguiente.

### Checkpoint 2 — Tabla de traza: Programa con bucle LOOP

| Paso | Instrucción   | AX antes | AX después | CX antes | CX después | IP sig. | LOOP salta |
|------|---------------|----------|------------|----------|------------|---------|------------|
| 1    | MOV AX, 0000  | —        | 0000       | —        | —          | 0103    | —          |
| 2    | MOV CX, 0004  | 0000     | 0000       | —        | 0004       | 0106    | —          |
| 3    | ADD AX, +02   | 0000     | 0002       | 0004     | 0004       | 0109    | —          |
| 4    | LOOP 0106     | 0002     | 0002       | 0004     | 0003       | 0106    | Sí         |
| 5    | ADD AX, +02   | 0002     | 0004       | 0003     | 0003       | 0109    | —          |
| 6    | LOOP 0106     | 0004     | 0004       | 0003     | 0002       | 0106    | Sí         |
| 7    | ADD AX, +02   | 0004     | 0006       | 0002     | 0002       | 0109    | —          |
| 8    | LOOP 0106     | 0006     | 0006       | 0002     | 0001       | 0106    | Sí         |
| 9    | ADD AX, +02   | 0006     | 0008       | 0001     | 0001       | 0109    | —          |
| 10   | LOOP 0106     | 0008     | 0008       | 0001     | 0000       | 010B    | No         |

**Observaciones:**
- La instrucción LOOP decrementa CX implícitamente en cada ejecución antes de evaluar si CX es cero; este comportamiento es atómico y no altera las banderas ZF ni CF.
- En las primeras tres ejecuciones de LOOP (pasos 4, 6 y 8), CX pasa de 4 a 3, de 3 a 2 y de 2 a 1 respectivamente; en todos los casos CX ≠ 0, por lo que el salto se realiza y el flujo regresa a 0x0106.
- En el paso 10, CX pasa de 1 a 0; la condición CX = 0 impide el salto y el flujo continúa secuencialmente hacia INT 20.
- El valor final en AX es 0x0008 = 8 decimal, equivalente a 4 iteraciones × 2 = 8, lo que verifica la correcta ejecución del bucle.

**Captura:** `capturas/CP2_traza_loop.png`

---

## Parte C — Análisis del Código Máquina con D

### Paso 8 — Volcado del programa de bucle

```
-D CS:100 L0C
1357:0100  B8 00 00 B9 04 00 05 02-00 E2 FB CD 20
```

**Análisis byte a byte:**

| Bytes       | Instrucción     | Tamaño | Análisis de codificación |
|-------------|-----------------|--------|--------------------------|
| B8 00 00    | MOV AX, 0000    | 3 bytes | Opcode B8 = MOV AX, imm16. El inmediato 0x0000 se almacena en little-endian: byte bajo 00, byte alto 00. |
| B9 04 00    | MOV CX, 0004    | 3 bytes | Opcode B9 = MOV CX, imm16. El inmediato 0x0004 se almacena como 04 00 en little-endian. |
| 05 02 00    | ADD AX, +02     | 3 bytes | Opcode 05 = ADD AX, imm16. El inmediato 0x0002 se almacena como 02 00 en little-endian. |
| E2 FB       | LOOP 0106       | 2 bytes | Opcode E2 = LOOP con desplazamiento relativo de 8 bits. FB = −5 en complemento a dos: la CPU suma −5 a IP=010B y obtiene 0106. |
| CD 20       | INT 20          | 2 bytes | Opcode CD = INT con número de interrupción inmediato de 8 bits. El valor 20h invoca la interrupción de terminación de programas DOS. |

El programa completo ocupa 13 bytes en memoria para realizar 4 sumas iterativas, lo que ilustra la eficiencia del código máquina del 8086 en comparación con secuencias lineales equivalentes. La instrucción LOOP en particular comprime en 2 bytes la lógica equivalente a un decremento de CX, una comparación con cero y un salto condicional.

---

## Conclusiones

La ejecución paso a paso con el comando `T` permitió observar directamente cómo el procesador actualiza los registros y las banderas instrucción a instrucción, transformando el concepto abstracto del ciclo fetch-decode-execute en una evidencia concreta y verificable. La tabla de traza del programa de suma confirmó que las instrucciones MOV no afectan las banderas, mientras que ADD sí lo hace en función del resultado. El análisis del bucle LOOP evidenció cómo CX actúa como contador de iteraciones implícito y cómo la instrucción codifica el salto relativo en un único byte de desplazamiento. El volcado del código máquina en el Paso 8 consolidó la comprensión de la codificación little-endian, el uso de opcodes de 1 byte y la naturaleza de los desplazamientos relativos firmados en los saltos cortos del 8086.

---

## Referencias

- Irvine, K. R. (2019). *Assembly Language for x86 Processors* (8th ed.). Pearson.
- Patterson, D. A., & Hennessy, J. L. (2020). *Computer Organization and Design: The Hardware/Software Interface* (RISC-V ed.). Morgan Kaufmann.
