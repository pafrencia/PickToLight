
# 🟦 Servicio PickToLight – Documentación de Pruebas  
Comunicación Modbus TCP/IP entre el Servicio Windows, el Master y los Carros Pick-To-Light

---

## 📌 1. Descripción General

Este documento resume las pruebas realizadas sobre el **Servicio PickToLight** desarrollado en .NET 8 como **Windows Service**, encargado de:

- Leer registros desde la base de datos.
- Enviar comandos Modbus al *Master*.
- Confirmar recepción.
- Actualizar la BD marcando los registros como procesados.
- Operar en **modo Test** para entornos sin base de datos enviando datos simulados directamente a los carros.

Las pruebas se realizaron tanto en **entorno de desarrollo local** (con emulador) como en **servidor real de AFG** conectado a hardware real.

---

## 🧪 2. Alcance de las Pruebas

### Este documento cubre:

- Pruebas locales con **Modbus Simulator (ModRSsim2)**
- Pruebas en servidor AFG con **Master y Carros reales**
- Verificación del flujo completo:
  1. Leer BD  
  2. Enviar al Master  
  3. Confirmar recepción  
  4. Actualizar registro (`Procesado = 1`)
- Pruebas del nuevo **modo Test**:
  - Sin base de datos  
  - Carro fijo configurable  
  - Envío completo de 13 slots  

---

## 🖥️ 3. Ambientes de Prueba

### 🔹 3.1 Entorno Local (Desarrollo)
- Windows 10/11
- Servicio PickToLight en .NET 8
- Emulador Modbus: **ModRSsim2**
- Base SQL Server local (`OrderManager_p3`)
- Pruebas con registro real vía BD

### 🔹 3.2 Servidor AFG (Producción / Pre-Producción)
- Windows Server
- Servicio desplegado como Windows Service
- Conectado al **Master real**
- Carros físicos conectados por red dedicada
- Sin acceso a BD → uso de **modo Test**

---

## ⚙️ 4. Funcionamiento del Servicio

### 🔸 4.1 Modo Normal (Producción)
1. Leer registros de `CarroDisplayQueue` donde `Procesado = 0`.
2. Enviar a Master:
   - Estado LED  
   - Color LED  
   - Valor del display  
3. Calcular dirección Modbus según Carro/Display.
4. Si el Master recibe correctamente → BD:
   ```sql
   UPDATE CarroDisplayQueue 
   SET Procesado = 1, FechaProcesado = GETDATE()
   ```

### 🔸 4.2 Modo Test (Hardware Sin BD)
- Carro fijo configurado en `appsettings.json`:  
  ```json
  "TestCarroId": 1
  ```
- Envía **13 slots siempre**, uno por cada display.
- Estado y color fijos (`1`).
- Valor incremental `1..13`.
- No se accede a la BD.

---

## 🧪 5. Pruebas Realizadas

### ✔ 5.1 Prueba Local con Emulador Modbus
**Objetivo:** Validar el ciclo completo BD → Servicio → Modbus.

**Resultados:**
- El servicio leyó correctamente los registros pendientes.
- El emulador (ModRSsim2) mostró correctamente los valores recibidos.
- Los registros se actualizaron en BD (`Procesado = 1`).
- Funcionamiento stable.

**Estado:** 🟢 **APROBADO**

---

### ✔ 5.2 Prueba en Servidor AFG con Master y Carros reales
**Objetivo:** Validar comunicación con el hardware real.

**Resultados:**
- El Master recibió los comandos Modbus enviados por el servicio.
- Los carros respondieron correctamente:
  - LED encendido según estado enviado.
  - Color cambiado según valor enviado.
  - Display 7 segmentos mostró los valores esperados.
- No se detectaron fallas de red ni pérdida de paquetes.

**Estado:** 🟢 **APROBADO**

---

### ✔ 5.3 Validación de Correspondencia Carro/Display
Se verificó:

| Display | Valor recibido | Resultado |
|---------|----------------|-----------|
| 1       | 1              | OK |
| 2       | 2              | OK |
| ...     | …              | OK |
| 13      | 13             | OK |

**Estado:** 🟢 **APROBADO**

---

## 📝 6. Conclusiones

- El servicio funciona correctamente tanto en entorno simulado como real.
- La comunicación Modbus está confirmada funcionando con:
  - Master real
  - Carros físicos  
- Modo Test permite validar hardware sin necesidad de BD.
- Modo Normal procesa BD y actualiza correctamente los registros.

El sistema es estable y apto para comenzar **pruebas integradas completas**.

---

## 🔜 7. Próximas Pruebas Recomendadas

1. **Prueba completa con BD real en el servidor AFG**
2. **Pruebas de estrés**
3. **Pruebas de reconexión**
4. **Persistencia de logs**
5. **Pruebas multi-carro simultáneo**

---
