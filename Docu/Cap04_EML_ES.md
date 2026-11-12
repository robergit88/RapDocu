# Capítulo 4 — Entity Manipulation Language (EML)

> Resumen en español del capítulo 4 de *ABAP RESTful Application Programming Model*, 2ª ed. (Rheinwerk, 2025).
> Los diagramas están en **Mermaid**: se ven en GitHub, en VS Code (con extensión Mermaid) y en la mayoría de visores Markdown.
> Los fragmentos EML son **esqueletos mínimos** para reconocer la sintaxis, no programas completos.

---

## 🗺️ El capítulo en una imagen

```mermaid
mindmap
  root((EML))
    Qué es
      Parte del lenguaje ABAP
      Acceso tipado al BO
      Masivo: siempre tablas
      IN LOCAL MODE
    Tipos derivados
      %CID / %CID_REF
      %TKY / %KEY / %PID
      %CONTROL / %DATA
      %FAIL / %MSG / %ELEMENT
    Respuestas
      FAILED
      REPORTED
      MAPPED
    Sentencias
      READ ENTITIES
      MODIFY ENTITIES
      GET PERMISSIONS
      SET LOCKS
      COMMIT ENTITIES
      ROLLBACK ENTITIES
    Dónde se usa
      Behavior implementation
      Reports / clases
      ABAP Unit
      Sustituto de BAPIs
```

---

## 1. ¿Qué es EML?

**EML = el "SQL" de los business objects RAP**, pero integrado en ABAP. Con él lees y modificas instancias de un BO de forma **tipada**, usando solo las operaciones declaradas en la behavior definition.

```mermaid
flowchart LR
    subgraph Consumidores
        UI[Fiori / OData]
        REP[Report / Clase ABAP]
        TEST[ABAP Unit]
        BP[Tu propio<br/>behavior pool]
    end
    UI -- OData --> RT
    REP -- EML --> RT
    TEST -- EML --> RT
    BP -- EML<br/>IN LOCAL MODE --> RT
    RT[[RAP runtime]] --> BUF[(Buffer<br/>transaccional)]
    BUF -- COMMIT --> DB[(Base de datos)]
```

**Las 4 ideas clave**

| 💡 | Idea |
|---|---|
| 📋 | **Siempre masivo**: todas las operaciones trabajan con tablas de instancias (aunque sea una). |
| 🧠 | **Lee el buffer transaccional**, no la BD → ves datos aún no grabados. Ese es su gran valor frente a `SELECT`. |
| 🔒 | Solo puedes usar lo **declarado en la BDEF**. No hace falta service definition ni binding. |
| 🏠 | **`IN LOCAL MODE`**: solo dentro del behavior pool del propio BO. Salta autorizaciones, feature control y restricciones de campo (`readonly`…). |

**Doble vida de EML**

```mermaid
flowchart LR
    A["🔧 Vista interna (provider)<br/>implementar determinations,<br/>validations, actions…"] --- E((EML)) --- B["🛒 Vista externa (consumer)<br/>usar el BO desde fuera:<br/>reports, otras apps, tests"]
```

---

## 2. Tipos derivados — los campos `%`

El compilador genera **tipos automáticamente** a partir de la BDEF. Además de los campos de la entidad, llevan componentes técnicos que empiezan por **`%`** (para no chocar con tus campos).

```mermaid
flowchart TB
    subgraph CLAVES["🔑 Identificación"]
        CID["%CID<br/>clave temporal que pones tú al crear"]
        CIDREF["%CID_REF<br/>referencia a un %CID<br/>(como una 'FK temporal')"]
        KEY["%KEY<br/>claves de la entidad"]
        PID["%PID<br/>ID provisional<br/>(late numbering)"]
        TKY["%TKY = %KEY + %IS_DRAFT + %PID<br/>⭐ la clave 'transaccional' que debes usar"]
    end
    subgraph DATOS["📦 Contenido"]
        DATA["%DATA<br/>todos los campos"]
        CTRL["%CONTROL<br/>flags: qué campos envío / pido"]
        PARAM["%PARAM<br/>parámetro de action/function"]
    end
    subgraph RESP["📣 Respuesta"]
        FAIL["%FAIL<br/>motivo del fallo"]
        MSG["%MSG<br/>mensaje (IF_ABAP_BEHV_MESSAGE)"]
        ELEM["%ELEMENT<br/>campo afectado por el mensaje"]
    end
```

### ¿Cómo encajan %CID y %CID_REF?

Crear un padre y sus hijos **en una sola llamada** (todavía no existen claves reales):

```mermaid
flowchart LR
    P["Cabecera<br/>%CID = 'GASTO1'"]
    H1["Línea 1<br/>%CID_REF = 'GASTO1'<br/>%CID = 'LIN1'"]
    H2["Línea 2<br/>%CID_REF = 'GASTO1'<br/>%CID = 'LIN2'"]
    T["Texto de línea<br/>%CID_REF = 'LIN1'"]
    P --> H1
    P --> H2
    H1 --> T
```

> 📌 El `%CID` debe ser **único** dentro de la llamada. Con **`AUTO FILL CID`** RAP lo genera por ti.

### %TKY — usa siempre este

```mermaid
flowchart LR
    TKY["%TKY"] --> K["%KEY<br/>(campos clave)"]
    TKY --> D["%IS_DRAFT<br/>(solo con draft)"]
    TKY --> PI["%PID<br/>(solo late numbering)"]
```

> ✅ Si usas `%TKY` tu código funciona igual con o sin draft y con o sin late numbering.

---

## 3. Parámetros de respuesta: FAILED · REPORTED · MAPPED

Son tablas anidadas con **una tabla por entidad** de la BDEF.

```mermaid
flowchart LR
    OP[Operación EML] --> F["❌ FAILED<br/>¿qué instancias fallaron?<br/>%TKY / %CID + %FAIL"]
    OP --> R["💬 REPORTED<br/>mensajes por instancia<br/>%TKY / %CID + %MSG + %ELEMENT"]
    OP --> M["🔁 MAPPED<br/>%CID → clave real asignada"]
```

**¿Qué rellena cada sentencia?**

| Sentencia | FAILED | REPORTED | MAPPED |
|---|:-:|:-:|:-:|
| `READ ENTITIES` | ✅ | ✅ | – |
| `MODIFY ENTITIES` | ✅ | ✅ | ✅ |
| `GET PERMISSIONS` | ✅ | ✅ | – |
| `SET LOCKS` | ✅ | ✅ | – |
| `COMMIT ENTITIES` | ✅ | ✅ | – |
| `ROLLBACK ENTITIES` | ✅ | ✅ | – |

> 📌 **MAPPED solo aparece al modificar**: es cuando se crean instancias y hay que decirte qué clave le tocó a cada `%CID`.

---

## 4. Las 6 sentencias EML

```mermaid
flowchart TB
    subgraph LEER["📖 Leer"]
        READ[READ ENTITIES]
    end
    subgraph ESCRIBIR["✏️ Escribir"]
        MOD[MODIFY ENTITIES<br/>CREATE · CREATE BY · UPDATE · DELETE · EXECUTE]
    end
    subgraph CONTROL["🛡️ Controlar"]
        GP[GET PERMISSIONS]
        SL[SET LOCKS]
    end
    subgraph TRANS["💾 Transacción<br/>(solo fuera del behavior pool)"]
        CO[COMMIT ENTITIES]
        RB[ROLLBACK ENTITIES]
    end
```

### 4.1 READ ENTITIES 📖

Lee del **buffer transaccional** (si no está ahí, RAP va a la BD).

```
READ ENTITIES OF <BO raíz> [IN LOCAL MODE]
  ENTITY <alias>
    FIELDS ( campo1 campo2 ) | ALL FIELDS
    WITH <tabla de claves>
  RESULT DATA(lt_result)
  [BY \_asociación …]          " leer hijos/padre
  FAILED … REPORTED … .
```

| Variante | Para qué |
|---|---|
| `FIELDS ( … )` / `ALL FIELDS` | Leer campos de instancias concretas |
| `BY \_Asociacion` | **Read-by-association**: p. ej. leer todas las líneas de un gasto para sumar importes |
| `EXECUTE <function>` | Ejecutar una function (es lectura) |
| `LINK` | Obtener la relación origen ↔ destino de la asociación |

> ⚠️ **Managed vs. unmanaged**: en managed RAP lee el buffer por ti. En unmanaged tienes que implementarlo tú (método `FOR READ`).

### 4.2 MODIFY ENTITIES ✏️

```mermaid
flowchart LR
    M[MODIFY ENTITIES] --> C["CREATE<br/>(solo raíz)"]
    M --> CBA["CREATE BY \_assoc<br/>(hijos)"]
    M --> U[UPDATE]
    M --> D[DELETE]
    M --> E["EXECUTE action"]
```

```
MODIFY ENTITIES OF <BO raíz> [IN LOCAL MODE]
  ENTITY <alias>
    UPDATE FIELDS ( Estado )
    WITH VALUE #( FOR key IN keys ( %tky = key-%tky  Estado = 'A' ) )
  FAILED … MAPPED … REPORTED … .
```

**Reglas prácticas**

| ✅ / ⚠️ | Regla |
|---|---|
| ✅ | Indica los campos con **`FIELDS ( … ) WITH`** (o rellena `%CONTROL`). El resto queda inicial / sin tocar. |
| ⚠️ | **Evita `SET FIELDS WITH`**: no permite la comprobación estática de `readonly` y no puede poner campos a inicial. |
| ✅ | En `CREATE BY \_assoc` usa `%CID_REF` (padre nuevo) o la **clave del padre** (padre existente) + `%target` con los hijos. |
| ✅ | `AUTO FILL CID` → RAP rellena el `%CID`. |
| ✅ | Varias entidades en una sola sentencia: `ENTITY Cabecera … ENTITY Linea …`. |
| 🔧 | `MODIFY AUGMENTING ENTITIES` → solo en **proyecciones**, para completar/modificar la petición antes de pasarla al BO base. |

### 4.3 GET PERMISSIONS 🛡️

Pregunta **"¿puedo?"** antes de hacerlo: autorizaciones (global/instancia) y feature control de create, create-by-association, update, delete, actions y campos.

```mermaid
flowchart LR
    Q["REQUEST<br/>%create = on"] --> GP[GET PERMISSIONS<br/>ONLY GLOBAL AUTHORIZATION …] --> R["RESULT<br/>permitido / no permitido"]
```

> Con `ONLY …` eliges qué se comprueba (global, instancia, features…).

### 4.4 SET LOCKS 🔒

Bloquea **explícitamente** instancias de cualquier BO.

```mermaid
flowchart LR
    A["Instancias de MI BO<br/>en el método FOR MODIFY"] -->|RAP las bloquea solo| OK[🔒 automático]
    B["Instancias de OTRO BO<br/>que necesito proteger"] -->|SET LOCKS| OK2[🔒 manual]
```

> ⚠️ Úsalo solo para lo que **no** bloquea ya RAP automáticamente.

### 4.5 COMMIT ENTITIES 💾

Dispara la **save sequence**: el buffer se graba en BD.

```mermaid
flowchart LR
    subgraph OData["🌐 Petición OData / Fiori"]
        F1[RAP es el dueño] --> F2[RAP hace el COMMIT]
    end
    subgraph Report["📄 Report / clase / test"]
        R1[TÚ eres el dueño] --> R2[TÚ haces COMMIT ENTITIES]
    end
```

| Complemento | Para qué |
|---|---|
| `RESPONSE OF <BO>` | Recoger FAILED / REPORTED por BO |
| `IN SIMULATION MODE` | Ejecuta la save sequence (finalize, check_before_save, cleanup) **sin grabar**. Útil para validar desde apps no-RAP. |
| `CONVERT KEY` | Obtener la **clave final** (late numbering) de instancias de **otro** BO. Solo en la fase tardía del commit. |

> ⚠️ **Nunca** dentro de tu behavior implementation: allí manda RAP.

### 4.6 ROLLBACK ENTITIES ↩️

`ROLLBACK ENTITIES.` → vacía el buffer y vuelve al estado del último commit. También solo **fuera** del behavior pool.

---

## 5. Dónde se usa EML

```mermaid
flowchart TB
    EML((EML))
    EML --> BI["🔧 Behavior implementation<br/>IN LOCAL MODE<br/>sin COMMIT"]
    EML --> RP["📄 Report / clase ABAP<br/>con COMMIT ENTITIES"]
    EML --> UT["🧪 ABAP Unit<br/>datos de prueba con<br/>if_cds_test_environment"]
    EML --> BAPI["🔄 Sustituto de BAPIs<br/>BO interfaces de SAP<br/>(ABAP Cloud)"]
```

### Flujo típico en un report / clase

```mermaid
sequenceDiagram
    participant P as Report / Clase
    participant RAP as RAP runtime
    participant BUF as Buffer
    participant DB as BD
    P->>RAP: READ ENTITIES
    RAP->>BUF: ¿está en buffer?
    BUF-->>DB: si no, lee BD
    RAP-->>P: RESULT
    P->>RAP: MODIFY ENTITIES (UPDATE)
    RAP->>BUF: cambia en buffer
    RAP-->>P: FAILED / REPORTED
    alt FAILED vacío
        P->>RAP: COMMIT ENTITIES
        RAP->>DB: save sequence → grabar
    else Hay errores
        P->>RAP: ROLLBACK ENTITIES
    end
```

### EML como sustituto de BAPIs (ABAP Cloud) 🚀

```mermaid
flowchart LR
    OLD["❌ Antes<br/>BAPI_SALESORDER_CREATE…<br/>BAPI_ACC_DOCUMENT_POST"] ==> NEW["✅ Ahora<br/>MODIFY ENTITIES OF I_SalesOrderTP<br/>EXECUTE Post en I_JournalEntryTP"]
```

- SAP publica **business object interfaces** (`…TP`) como sucesores de las BAPIs.
- Una sola sentencia crea **cabecera + posiciones + textos** enlazados con `%CID` / `%CID_REF`.
- ⚠️ Algunas interfaces **no** ofrecen create/update/delete, solo **actions/functions** (p. ej. `I_JournalEntryTP`: `Post`, `Reverse`, `Change`, `Validate`).

---

## 6. EML vs. SQL — cuándo usar cada uno

| | `SELECT` (SQL/CDS) | `READ ENTITIES` (EML) |
|---|---|---|
| Lee de… | Base de datos | **Buffer transaccional** (y si no, BD) |
| ¿Ve cambios no grabados? | ❌ | ✅ |
| Filtros libres | ✅ | ❌ (por claves) |
| Respeta lógica del BO | ❌ | ✅ |
| Escritura | ❌ (prohibido en RAP) | ✅ `MODIFY ENTITIES` |
| Ideal para… | Informes, búsquedas masivas | Lógica de negocio dentro de la transacción |

---

## 7. ⚠️ Errores típicos

| ❌ Error | ✅ Correcto |
|---|---|
| Usar `SELECT` en una validation para leer los datos que se están cambiando | `READ ENTITIES … IN LOCAL MODE` (lee el buffer) |
| Usar `%KEY` en un BO con draft | Usar `%TKY` |
| `COMMIT ENTITIES` dentro del behavior pool | Dejar que RAP haga el commit |
| Olvidar `COMMIT ENTITIES` en un report | Sin commit, nada se graba |
| No comprobar `FAILED` antes del commit | `IF failed IS INITIAL. COMMIT ENTITIES. ENDIF.` |
| Repetir el mismo `%CID` en una llamada | Un `%CID` único por instancia (o `AUTO FILL CID`) |
| `SET FIELDS WITH` | `FIELDS ( … ) WITH` |
| `IN LOCAL MODE` desde fuera del BO | Solo dentro de su propio behavior pool |

---

## 8. 🧩 Chuleta de una página

```mermaid
flowchart LR
    subgraph DENTRO["🔧 Dentro del behavior pool"]
        direction TB
        a1["READ ENTITIES … IN LOCAL MODE"]
        a2["MODIFY ENTITIES … IN LOCAL MODE"]
        a3["Devolver failed / reported / mapped"]
        a4["❌ sin COMMIT / ROLLBACK"]
    end
    subgraph FUERA["📄 Fuera (report, clase, test)"]
        direction TB
        b1["READ ENTITIES"]
        b2["MODIFY ENTITIES"]
        b3["GET PERMISSIONS (opcional)"]
        b4["✅ COMMIT ENTITIES / ROLLBACK ENTITIES"]
    end
```

---

## 9. ✍️ Autoevaluación

1. ¿Por qué `READ ENTITIES` es mejor que `SELECT` dentro de una validation?
2. ¿Qué hace `IN LOCAL MODE` y dónde se puede usar?
3. ¿Para qué sirven `%CID` y `%CID_REF`? Pon un ejemplo con cabecera y líneas.
4. ¿Qué contiene `%TKY` y por qué se recomienda frente a `%KEY`?
5. ¿Qué parámetro de respuesta solo aparece en `MODIFY ENTITIES` y por qué?
6. ¿Quién hace el `COMMIT` en una app Fiori? ¿Y en un report?
7. ¿Cuándo tiene sentido `SET LOCKS`?
8. ¿Qué hace `COMMIT ENTITIES IN SIMULATION MODE`?
9. ¿Por qué no se recomienda `SET FIELDS WITH`?
10. Para añadir una línea a un gasto **ya existente**, ¿usas `%CID_REF` o la clave del padre?

<details>
<summary>Respuestas</summary>

1. Porque lee el buffer transaccional: ve los cambios aún no grabados.
2. Accede al BO saltándose autorizaciones, feature control y restricciones de campo. Solo dentro del behavior pool del propio BO.
3. `%CID` = clave temporal de una instancia nueva; `%CID_REF` = referencia a ese `%CID` desde otra instancia (p. ej. la línea apunta a la cabecera nueva).
4. `%KEY` + `%IS_DRAFT` + `%PID`. El código funciona con o sin draft y con o sin late numbering.
5. `MAPPED`: es cuando se crean instancias y hay que devolver la relación `%CID` → clave real.
6. Fiori/OData: RAP. Report: tú, con `COMMIT ENTITIES`.
7. Para bloquear instancias de otros BO que RAP no bloquea automáticamente.
8. Ejecuta la save sequence (finalize, check_before_save, cleanup) sin grabar.
9. No permite la comprobación estática de `readonly` y no permite poner campos a valor inicial.
10. La clave del padre (ya existe); `%CID_REF` es para padres creados en la misma llamada.

</details>

---

### 👉 Siguiente paso

El **capítulo 5 (Behavior Implementation)** usa todo esto dentro de los métodos `FOR MODIFY`, `FOR READ`, `FOR DETERMINE`, `FOR VALIDATE`… y del saver.
