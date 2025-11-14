# Flujo Operativo - Nuevo Sistema de Picking con Carro de Slots

## 1. Descripción General

El cliente plantea una evolución del proceso de **picking y desconsolidado**, utilizando un **nuevo carro con slots numerados (1..12)** y un **slot superior (13)** exclusivo para **bultos (BU)**.

El objetivo del nuevo flujo es que el **pikeador** realice la **separación de productos por cliente directamente durante el picking**, de modo que el carro llegue al área de **Picking Isla** ya “desconsolidado”, optimizando el tiempo de armado de pedidos y reduciendo pasos intermedios.

---

## 2. Flujo de Trabajo

### 2.1 Asignación

- Se asigna un **consolidado** a un pikeador.
- En el dispositivo del pikeador se muestran las **cantidades totales a pikear por producto**.
- El pikeador toma un **carro con slots**:
  - **Slots 1–12:** asignados a clientes/pedidos.
  - **Slot 13 (superior):** reservado para **bultos (BU)** compartido entre todos los clientes.

---

### 2.2 Picking “Desconsolidado” en Origen

- Por cada producto del consolidado:
  - **Si la unidad pedida es DI o UN:**  
    El pikeador reparte las cantidades directamente en el **slot correspondiente al cliente**.
  - **Si algún cliente pide BU (bultos completos o equivalentes):**
  
    OPCION 1:
    - Todos los Bu que completen cada pedido se suman y se coloca ese numero en el display de arriba 
    - Los **BU** se colocan en la **parte superior del carro (slot 13)**, compartido.  
    - **Los sobrantes** (lo que no completa un BU) se colocan en el **slot del cliente**.

#### Ejemplo 1
Pedido del cliente X: 20 DI  
Conversión: **1 BU = 15 DI**  
→ El sistema indica pikear 20 DI en el mobil.
→ El el carro muestra **1 BU arriba (slot 13)**
**5 DI** en el **slot del cliente X**.
#### Ejemplo 2
Pedido del cliente X: 20 DI 
Pedido del cliente Y: 30 DI

Conversión: **1 BU = 15 DI**

→ El sistema indica pikear 50 DI en el mobil.
→ El el carro muestra **3 BU arriba (slot 13)** y **5 DI** en el **slot del cliente X**.


  OPCION 2:


---

### 2.3 Capacidad y Distribución

- ¿Que pasa si el **slot del cliente se llena** y todavia faltan prodcutos por pikear?
- ¿Que pasa si el **slot 13** (zona BU) se llena?
- ¿Si no hay stock suficiente de un articulo para cubrir el total que criterio se usara para repartir los productos existentes en los slots que corresponda?

---

### 2.4 Salida de Picking

- Al finalizar, **no pasa por el control de consolidado tradicional**, ya que el carro quedó desconsolidado.
- El carro se dirige **directamente al área de Picking Isla**.

---

### 2.5 Picking Isla (Empaque y Cierre)

- Los operarios reciben el carro con:
  - **Slots por cliente** con sus DI/UN.
  - **Zona superior (slot 13)** con los BU compartidos.

#### En isla:
- Para cada pedido:
  - Se embala lo del **slot del cliente**.
  - Se toman los **BU necesarios** desde la zona superior.
  - Se **etiquetan las cajas** y se envían a despacho.

---

## 3. Reglas Operativas Clave

| Tipo de Unidad | Destino en el Carro | Observaciones |
|----------------|---------------------|----------------|
| **BU (Bultos)** | Slot 13 (superior) | Compartido entre clientes |
| **DI / UN** | Slot del cliente | Se separan por pedido |
| **Sobrantes** | Slot del cliente | Si no completa un BU |
| **Overflow** | ??? | a definir |
| **Sin stock** | N/A | Se marca como pendiente |

---

## 4. Ejemplo de Distribución

| Cliente | Pedido | Producto | Cantidad Pedida | Equivalencia | Slot |
|----------|---------|-----------|-----------------|---------------|------|
| A | 1001 | PROD-01 | 20 DI | 1 BU = 15 DI | 1 BU arriba (slot 13) + 5 DI en slot A |
| B | 1002 | PROD-01 | 30 DI | 1 BU = 15 DI | 2 BU arriba (slot 13) |
| C | 1003 | PROD-02 | 12 UN | - | Slot C |

---

## 5. Puntos de Control

1. **Salida de picking:**  
   - Verificar que cada slot tenga tarjeta Cliente+Pedido.
   - ¿como se hara esto?
   - ¿como sabra en picking isla que slot corresponde a cada pedido/cliente?

2. **Recepción en isla:**  
   - Validar coincidencia entre listado y contenido de slots.

3. **Cierre de pedido:**  
   - Confirmar conteo final de BU + DI/UN.  
   - Etiqueta de cierre aplicada.
---

## 6. Glosario

| Término | Descripción |
|----------|--------------|
| **Consolidado** | Grupo de pedidos de distintos clientes asignados a un mismo recorrido. |
| **Desconsolidado** | Separación por cliente durante el picking. |
| **Slot** | Compartimento del carro asignado a un cliente/pedido. |
| **BU / DI / UN** | Unidades de medida: Bulto, Display, Unidad. |
| **Slot 13** | Espacio superior del carro para BU compartidos. |

---

© 2025 – Documento funcional de flujo operativo del nuevo carro de picking.
