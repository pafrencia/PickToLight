

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
2. El registro queda con:
   - `Procesado = 0`
   - `FechaCreado = GETDATE()`
   - `Tipo = 'DISPLAY'`
3. Un **servicio de backend (ServicioPickToLightWorker)** monitorea esta tabla y toma las filas no procesadas.
4. Al enviar la instrucción al carro físico (por Modbus), el servicio:
   - Actualiza `Procesado = 1`
   - Completa `FechaProcesado`
5. La fila queda como histórico, permitiendo auditoría del uso del carro.

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

### Esta sp sirve para agregar a la tabla las filas que correspondan segun las cantidades por producto por slot al escanear el prodcuto

```sql

USE [OrderManager]
GO

/****** Object:  StoredProcedure [dbo].[sp_EnqueuePickToLight_Producto_ByScanUM]    Script Date: 1/12/2025 09:31:01 ******/
SET ANSI_NULLS ON
GO

SET QUOTED_IDENTIFIER ON
GO


CREATE PROCEDURE [dbo].[sp_EnqueuePickToLight_Producto_ByScanUM]
(
    @numero_asignacion NVARCHAR(50),
    @codigo_producto   NVARCHAR(50),
    @um_scan           NVARCHAR(5)  -- 'BU' | 'DI' | 'UN'
)
AS
BEGIN
    SET NOCOUNT ON;

    ----------------------------------------------------------------------
    -- 1) Conversiones a UN (para el producto)
    ----------------------------------------------------------------------
    DECLARE @UNxDI INT;
    DECLARE @UNxBU INT;

    SELECT @UNxDI = cp.Conversion
    FROM dbo.CONVERSION_PRODUCTOS cp
    WHERE cp.Codigo_Producto = @codigo_producto
      AND cp.Unidad_1 = 'DI'
      AND cp.Unidad_2 = 'UN';

    SELECT @UNxBU = cp.Conversion
    FROM dbo.CONVERSION_PRODUCTOS cp
    WHERE cp.Codigo_Producto = @codigo_producto
      AND cp.Unidad_1 = 'BU'
      AND cp.Unidad_2 = 'UN';

    IF @UNxDI IS NULL OR @UNxDI <= 0 SET @UNxDI = 1;             -- fallback para evitar /0
    IF @UNxBU IS NULL OR @UNxBU <= 0 SET @UNxBU = @UNxDI * 10;    -- heurística razonable

    ----------------------------------------------------------------------
    -- 2) CTE: Totalizar UN por slot y particionar en capas BU, DI, UN.
    --    IMPORTANTE: la PRIMERA sentencia que sigue a la CTE debe usarla.
    ----------------------------------------------------------------------
    IF OBJECT_ID('tempdb..#Cant') IS NOT NULL DROP TABLE #Cant;

    ;WITH Base AS
    (
        SELECT 
            dcpp.id_contenedor AS Carro,
            CASE 
                WHEN dcpp.ubicacion_1  = '1' THEN 1
                WHEN dcpp.ubicacion_2  = '1' THEN 2
                WHEN dcpp.ubicacion_3  = '1' THEN 3
                WHEN dcpp.ubicacion_4  = '1' THEN 4
                WHEN dcpp.ubicacion_5  = '1' THEN 5
                WHEN dcpp.ubicacion_6  = '1' THEN 6
                WHEN dcpp.ubicacion_7  = '1' THEN 7
                WHEN dcpp.ubicacion_8  = '1' THEN 8
                WHEN dcpp.ubicacion_9  = '1' THEN 9
                WHEN dcpp.ubicacion_10 = '1' THEN 10
                WHEN dcpp.ubicacion_11 = '1' THEN 11
                WHEN dcpp.ubicacion_12 = '1' THEN 12
                /* Si usás 13 para clientes, habilitalo:
                   WHEN dcpp.ubicacion_13 = '1' THEN 13 */
                WHEN dcpp.ubicacion_14 = '1' THEN 14
                WHEN dcpp.ubicacion_15 = '1' THEN 15
            END AS SlotCarro,
            dcpp.um,
            dcpp.cantidad_pedida
        FROM dbo.DETALLE_CONTENEDOR_PEDIDOS_PICKING dcpp
        WHERE dcpp.numero_asignacion = @numero_asignacion
          AND dcpp.codigo_producto   = @codigo_producto
    ),
    TotUN AS
    (
        SELECT
            b.Carro,
            b.SlotCarro,
            SUM(CASE b.um
                    WHEN 'UN' THEN CAST(b.cantidad_pedida AS INT)
                    WHEN 'DI' THEN CAST(b.cantidad_pedida AS INT) * @UNxDI
                    WHEN 'BU' THEN CAST(b.cantidad_pedida AS INT) * @UNxBU
                    ELSE CAST(b.cantidad_pedida AS INT)
                END) AS TotalUN
        FROM Base b
        WHERE b.SlotCarro IS NOT NULL
        GROUP BY b.Carro, b.SlotCarro
    ),
    Particion AS
    (
        SELECT
            t.Carro,
            t.SlotCarro,
            t.TotalUN,
            (t.TotalUN / NULLIF(@UNxBU,0))                    AS BU_full,
            (t.TotalUN % NULLIF(@UNxBU,0))                    AS RestoDespuesBU,
            ((t.TotalUN % NULLIF(@UNxBU,0)) / NULLIF(@UNxDI,0)) AS DI_full,
            ((t.TotalUN % NULLIF(@UNxBU,0)) % NULLIF(@UNxDI,0)) AS UN_rest
        FROM TotUN t
    ),
    CantidadSegunScan AS
    (
        SELECT
            p.Carro,
            p.SlotCarro,
            CASE 
                WHEN @um_scan = 'BU' THEN p.BU_full
                WHEN @um_scan = 'DI' THEN p.DI_full
                ELSE p.UN_rest                 -- 'UN' ⇒ remanente final
            END AS CantMostrar
        FROM Particion p
    )
    SELECT 
        c.Carro,
        c.SlotCarro,
        c.CantMostrar
    INTO #Cant
    FROM CantidadSegunScan c;
    -- <<< IMPORTANTE: Acá la CTE queda "consumida" por este SELECT INTO

    ----------------------------------------------------------------------
    -- 3) Inserciones condicionadas (por-slot y slot 13 resumen)
    ----------------------------------------------------------------------
    DECLARE @Color INT;
    DECLARE @Carro INT;
    DECLARE @Total INT;

    SET @Color = CASE @um_scan WHEN 'BU' THEN 4 WHEN 'DI' THEN 3 ELSE 2 END;

    SELECT TOP 1 @Carro = Carro FROM #Cant;

    SELECT @Total = ISNULL(SUM(CantMostrar), 0) FROM #Cant;

    IF @Total > 0
    BEGIN
        -- por-slot (>0)
        INSERT INTO dbo.CarroDisplayQueue (Carro, Display, EstadoLed, ColorLed, Valor, Tipo, Procesado)
        SELECT 
            Carro,
            SlotCarro AS Display,
            1         AS EstadoLed,
            1    AS ColorLed,
            CantMostrar AS Valor,
            'DISPLAY' AS Tipo,
            0         AS Procesado
        FROM #Cant
        WHERE CantMostrar > 0
          AND SlotCarro IS NOT NULL;

        -- slot 13 con TOTAL
        IF @Carro IS NOT NULL
        BEGIN
            INSERT INTO dbo.CarroDisplayQueue (Carro, Display, EstadoLed, ColorLed, Valor, Tipo, Procesado)
            VALUES (@Carro, 13, 1, @Color, @Total, 'DISPLAY', 0);
        END
    END
    ELSE
    BEGIN
        -- No hay nada para esta capa → slot 13 en 0
        IF @Carro IS NULL
        BEGIN
            SELECT TOP 1 @Carro = dcpp.id_contenedor
            FROM dbo.DETALLE_CONTENEDOR_PEDIDOS_PICKING dcpp
            WHERE dcpp.numero_asignacion = @numero_asignacion
              AND dcpp.codigo_producto   = @codigo_producto;
        END

        IF @Carro IS NOT NULL
        BEGIN
            INSERT INTO dbo.CarroDisplayQueue (Carro, Display, EstadoLed, ColorLed, Valor, Tipo, Procesado)
            VALUES (@Carro, 13, 1, @Color, 0, 'DISPLAY', 0);
        END
    END
END
GO

```

---


## CREATE StoreProcedure sp_EnqueuePickToLight_ResetCarro

### Esta sp sirve para agregar a la tabla una fila con tipo = RESET lo que al ser leida por el worker mandara a resetear todos los slot del carro de dicha fila

```sql
CREATE PROCEDURE sp_EnqueuePickToLight_ResetCarro
(
    @numero_asignacion NVARCHAR(50)
)
AS
BEGIN
    SET NOCOUNT ON;

    DECLARE @Carro INT;

    SELECT TOP 1 @Carro = id_contenedor
    FROM DETALLE_CONTENEDOR_PEDIDOS_PICKING
    WHERE numero_asignacion = @numero_asignacion;

    IF @Carro IS NULL
        RETURN;

    INSERT INTO CarroDisplayQueue (Carro, Display, EstadoLed, ColorLed, Valor, Tipo, Procesado)
    VALUES (@Carro, 0, 0, 0, 0, 'RESET', 0);
END;
```

---
