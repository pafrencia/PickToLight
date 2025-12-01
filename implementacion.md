

# MODIFICACIONES EN BASE DE DATOS (SQL Server)

# Documentación: Tabla `CarroDisplayQueue`

La tabla **`CarroDisplayQueue`** funciona como una **cola de tareas (queue)** para enviar instrucciones al **carro inteligente / Pick-To-Light**.  
Cada registro representa una orden pendiente para actualizar un display LED de un carro.

---

## 📌 ¿Para qué sirve esta tabla?

Esta tabla almacena **eventos que deben ser enviados al controlador del carro**, como:

- Encender un LED de un slot determinado.  
- Cambiar el color del LED (rojo, verde, azul, etc.).  
- Mostrar valores numéricos o cantidades.  
- Marcar que un display ya fue procesado.  
- Indicar qué tipo de acción es (“DISPLAY”, "RESET", o futuros tipos).

El sistema que gestiona el carro (ServicioPickToLightWorker) va leyendo esta tabla para saber qué debe hacer y va procesando cada registro.
lo hace atravez de los siguientes store procedures:
sp_EnqueuePickToLight_Producto_ByScanUM (para actualizar cantidades en los slot del carro)
sp_EnqueuePickToLight_ResetCarro (para resetear todos los slot del carro)

---

## 🧠 Funcionamiento general

1. **La aplicación inserta un registro** en esta tabla cada vez que necesita que un carro muestre algo.
2. IMPORTANTE el registro solo se insertara SI el parametro
Picking_UsaCarroInteligente es igual a 1
y el carro es 1, 2 o 3.
3. El registro queda con:
   - `Procesado = 0`
   - `FechaCreado = GETDATE()`
   - `Tipo = 'DISPLAY'`
4. Un **servicio de backend (ServicioPickToLightWorker)** monitorea esta tabla y toma las filas no procesadas.
5. Al enviar la instrucción al carro físico (por Modbus), el servicio:
   - Actualiza `Procesado = 1`
   - Completa `FechaProcesado`
6. La fila queda como histórico, permitiendo auditoría del uso del carro.

---

## 🧩 Descripción de campos

| Campo | Tipo | Descripción |
|-------|------|-------------|
| **Id** | int IDENTITY | Identificador único de la instrucción. |
| **Carro** | int | Número del carro físico al que se envía la orden. |
| **Display** | int | Número de display/slot dentro del carro. |
| **EstadoLed** | int | Estado a enviar (encender, apagar, parpadear, etc.). |
| **ColorLed** | int | Código del color del LED. |
| **Valor** | int | Valor numérico a mostrar (si aplica). |
| **Procesado** | bit | 0 = pendiente, 1 = ya enviado al carro. |
| **FechaCreado** | datetime | Fecha/hora en que se generó la instrucción. |
| **FechaProcesado** | datetime | Fecha/hora en que el carro la ejecutó. |
| **Tipo** | varchar(20) | Tipo de acción (por defecto “DISPLAY”). |

---

## 📚 Ejemplo de uso

Cuando un operador debe colocar 4 unidades para el Cliente 3 en el slot 3:

1. El sistema agrega un registro:
   - Carro = 1  
   - Display = 3  
   - EstadoLed = 1  
   - ColorLed = 2  
   - Valor = 4  
   - Procesado = 0  
2. El servicio lo lee y envía la instrucción al carro.  
3. Actualiza `Procesado = 1` y la fecha correspondiente.

---

## ✔️ Resumen

La tabla **CarroDisplayQueue** es esencial para coordinar el funcionamiento del carro inteligente. Permite:

- Controlar displays LED.
- Registrar cada instrucción enviada.
- Procesar acciones de forma asincrónica y ordenada.
- Mantener un historial de lo que sucedió en cada carro.


## INSERT PARAMETRO Picking_UsaCarroInteligente

[AgregarParametro_Picking_UsaCarroInteligente.sql](https://github.com/user-attachments/files/23859065/AgregarParametro_Picking_UsaCarroInteligente.sql)

### Este parametro servira de flag para encolar en la tabla los registros solo si el parametro esta en 1

```sql
INSERT INTO PARAMETROS (ID_Parametro, Valor, Descripcion)
VALUES ('Picking_UsaCarroInteligente', '1', 'Habilita o deshabilita el uso del carro PickToLight');
```

---


<img width="695" height="124" alt="image" src="https://github.com/user-attachments/assets/0dc1d4e7-063a-4e2a-a12f-d2610ed27f70" />

## CREATE TABLE CarroDisplayQueue

### Esta tabla es la que leera el worker y actualizara

```sql
USE [OrderManager];
GO

/****** Object:  Table [dbo].[CarroDisplayQueue]    Script Date: 1/12/2025 09:22:41 ******/
SET ANSI_NULLS ON;
GO

SET QUOTED_IDENTIFIER ON;
GO

CREATE TABLE [dbo].[CarroDisplayQueue](
    [Id] [int] IDENTITY(1,1) NOT NULL,
    [Carro] [int] NOT NULL,
    [Display] [int] NOT NULL,
    [EstadoLed] [int] NOT NULL,
    [ColorLed] [int] NOT NULL,
    [Valor] [int] NOT NULL,
    [Procesado] [bit] NOT NULL,
    [FechaCreado] [datetime] NOT NULL,
    [FechaProcesado] [datetime] NULL,
    [Tipo] [varchar](20) NULL,
PRIMARY KEY CLUSTERED 
(
    [Id] ASC
)WITH (
    PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF,
    ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON,
    OPTIMIZE_FOR_SEQUENTIAL_KEY = OFF
) ON [PRIMARY]
) ON [PRIMARY];
GO

ALTER TABLE [dbo].[CarroDisplayQueue]
    ADD DEFAULT ((0)) FOR [Procesado];
GO

ALTER TABLE [dbo].[CarroDisplayQueue]
    ADD DEFAULT (GETDATE()) FOR [FechaCreado];
GO

ALTER TABLE [dbo].[CarroDisplayQueue]
    ADD CONSTRAINT [DF_CarroDisplayQueue_Tipo]
        DEFAULT ('DISPLAY') FOR [Tipo];
GO
```

---

## CREATE StoreProcedure sp_EnqueuePickToLight_Producto_ByScanUM

[sp_EnqueuePickToLight_Producto_ByScanUM_Logs.sql](https://github.com/user-attachments/files/23859398/sp_EnqueuePickToLight_Producto_ByScanUM_Logs.sql)

### Este sp sirve para agregar a la tabla las filas que correspondan segun las cantidades por producto por slot al escanear el prodcuto


---

## CREATE StoreProcedure sp_EnqueuePickToLight_ResetCarro

[sp_EnqueuePickToLight_ResetCarro_Logs.sql](https://github.com/user-attachments/files/23859428/sp_EnqueuePickToLight_ResetCarro_Logs.sql)

### Este sp sirve para agregar a la tabla una fila con tipo = RESET lo que al ser leida por el worker mandara a resetear todos los slot del carro de dicha fila

---

## StoreProcedure mob_sp_save_picking

[mob_sp_save_picking.sql](https://github.com/user-attachments/files/23853810/mob_sp_save_picking.sql)

### contiene una modificacion del Sp agregandole el uso del SP de reset de carro, para que al pasar al siguiente producto el carro resetee todos los slots

---

## StoreProcedure mob_sp_prod_picking

[mob_sp_prod_picking.sql](https://github.com/user-attachments/files/23853889/mob_sp_prod_picking.sql)

### contiene una modificacion del Sp agregandole el uso del sp_EnqueuePickToLight_Producto_ByScanUM, para que al pasar al escanear un producto se agreguen a la tabla CarroDisplayQueue las filas correspondientes

---

## StoreProcedure mob_sp_picking_asignar_contenedor_pedidos

[mob_sp_picking_asignar_contenedor_pedidos.sql](https://github.com/user-attachments/files/23854111/mob_sp_picking_asignar_contenedor_pedidos.sql)

### contiene una modificacion del Sp modificando el orden en el que asignara los slots a los clientes para que correspondan por numero de cliente empezando por el mas chico en el slot 1


