

# Tabla: CarroDisplayQueue (SQL Server)
<img width="695" height="124" alt="image" src="https://github.com/user-attachments/assets/0dc1d4e7-063a-4e2a-a12f-d2610ed27f70" />

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
