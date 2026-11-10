# Capítulo 3 — Behavior Definition (resumen en español)

> Basado en *ABAP RESTful Application Programming Model*, 2ª ed. (Rheinwerk, 2025), capítulo 3.
> Resumen de estudio: conceptos explicados en español; los términos técnicos se mantienen en inglés porque así aparecen en ADT y en la documentación de SAP.
> Los fragmentos BDL son ilustrativos (mínimos) para reconocer la sintaxis; los nombres usan prefijo Z.

---

## Índice

1. [¿Qué es una behavior definition?](#1-qué-es-una-behavior-definition)
2. [Trabajar con la behavior definition en ADT](#2-trabajar-con-la-behavior-definition-en-adt)
3. [Tipos de implementación](#3-tipos-de-implementación)
4. [Strict mode](#4-strict-mode)
5. [Entity behavior definition](#5-entity-behavior-definition)
6. [Behavior pool](#6-behavior-pool)
7. [Numeración (numbering)](#7-numeración-numbering)
8. [Propiedades de campo (field properties)](#8-propiedades-de-campo-field-properties)
9. [Field mappings](#9-field-mappings)
10. [Operaciones estándar](#10-operaciones-estándar)
11. [Operaciones específicas: actions y functions](#11-operaciones-específicas-actions-y-functions)
12. [Concurrencia y bloqueos](#12-concurrencia-y-bloqueos)
13. [Lógica de negocio interna: determinations y validations](#13-lógica-de-negocio-interna-determinations-y-validations)
14. [Autorizaciones](#14-autorizaciones)
15. [Draft handling y side effects](#15-draft-handling-y-side-effects)
16. [Eventos](#16-eventos)
17. [Conceptos transversales](#17-conceptos-transversales)
18. [Glosario inglés → español](#18-glosario-inglés--español)
19. [Chuleta: qué método exige cada declaración](#19-chuleta-qué-método-exige-cada-declaración)
20. [Preguntas de autoevaluación](#20-preguntas-de-autoevaluación)

---

## 1. ¿Qué es una behavior definition?

La **behavior definition** (BDEF) es el objeto de desarrollo donde se **declara el comportamiento** de un business object usando el lenguaje **BDL** (*Behavior Definition Language*).

- Con CDS modelas la **estructura** (datos y relaciones). Eso basta para *leer*.
- Para **escribir** (crear, modificar, borrar) o añadir **lógica de negocio**, necesitas una behavior definition.
- **Business object RAP = modelo de datos CDS + behavior definition.**

### Reglas clave

- Solo se crea para la **entidad raíz** (`define root view entity`). Las entidades hijas no tienen BDEF propia; van *dentro* de la de la raíz.
- **Una sola** BDEF por entidad raíz, y **se llama igual** que ella (el nombre no se puede cambiar).
- Todo lo que declaras necesita implementación: o la da el **framework RAP** (p. ej. CRUD en managed) o la escribes tú en ABAP. Lo normal es una mezcla.
- Una aplicación puede tener **varios** business objects que se llaman entre sí.

### Las 4 zonas de una BDEF

| # | Zona | Qué contiene |
|---|------|--------------|
| 1 | **Cabecera de la BDEF** | Tipo de implementación (`managed`/`unmanaged`), `strict ( 2 )`, `with draft`, behavior pool global. Aplica a todo el árbol de composición. |
| 2 | **Zona de entity behavior definitions** | Un bloque `define behavior for …` por cada entidad del árbol (la raíz es obligatoria). |
| 3 | **Cabecera de cada entidad** | Propiedades transaccionales: `persistent table`, `draft table`, `lock master`, `etag master`, `authorization master`, numeración… |
| 4 | **Cuerpo `{ … }`** | Operaciones, campos, actions, determinations, validations, side effects, eventos… |

```
managed implementation in class zbp_r_expense unique;   // 1
strict ( 2 );
with draft;

define behavior for ZR_Expense alias Expense            // 2 + 3
persistent table zexpense_a
draft table zexpense_d
lock master
total etag LastChangedAt
authorization master ( instance )
{                                                        // 4
  create; update; delete;
  association _Item { create; }
  validation validateDates on save { create; field BeginDate, EndDate; }
}
```

### Reglas de sintaxis BDL

- **Palabras clave en minúsculas** (no se permite mezclar mayúsculas).
- Las sentencias terminan en `;` (salvo la cabecera de la entidad y sus propiedades).
- **Nombres sin distinción de mayúsculas**: `validateCustomer` y `ValidateCustomer` chocan. Se usa camelCase por legibilidad.
- Comentarios: `//` (línea) y `/* … */` (bloque).

### Mapa de "features" (lo que puedes declarar)

| Categoría | Ejemplos de palabras clave | ¿Quién implementa? |
|-----------|---------------------------|--------------------|
| **Transaccional** | `authorization master`, `lock master`, `etag master`, `total etag`, `early/late numbering`, `numbering : managed`, `persistent table`, `with draft`, `draft table` | Mezcla: locking/ETag/persistencia/draft los da RAP (en managed); autorización y numeración propia, tú |
| **Operaciones estándar** | `create`, `update`, `delete`, `association _X { create; }` | RAP (managed) / tú (unmanaged) |
| **Operaciones específicas** | `action`, `static action`, `factory action`, `function` | Tú |
| **Lógica interna** | `determination … on modify/on save`, `validation … on save`, `determine action` | Tú |
| **Campos** | `field ( readonly / mandatory … )`, `mapping for …` | RAP (control) |
| **Eventos** | `event`, `event … parameter`, `managed event` | Tú lanzas; los derivados los lanza RAP |
| **UI** | `default function GetDefaultsFor…`, `side effects { … }` | Interpretado por Fiori elements |
| **Transversales** | `features : instance/global`, `precheck`, `internal` | Tú (feature control y precheck) |

### Impacto de lo que declaras

Cada declaración en la BDEF afecta a:

1. **La interfaz del BO** → lo que ven los consumidores (EML, OData). *Ojo: cambiar algo aquí puede romper consumidores* (p. ej. pasar un campo a `readonly` en una API publicada).
2. **La implementación** → habilita métodos en el behavior pool y genera tipos derivados para EML.
3. **Los business services** → aparecen en el `$metadata` de OData.
4. **La UI Fiori elements** → botones de operaciones, flujo de draft, etc.

> 💡 **Consejo:** declara solo lo necesario. Todo lo que declaras es "contrato público". Lo que solo se use internamente, márcalo `internal`.

---

## 2. Trabajar con la behavior definition en ADT

- **Crear:** clic derecho sobre la entidad raíz → *New Behavior Definition* (o asistente `Alt+N` → *ABAP › Business Services › Behavior Definition*). Eliges el tipo de implementación y ADT genera un esqueleto con `create/update/delete`. Recomendación: pon un **alias** desde el principio.
- **Editar:** `Ctrl+Espacio` para autocompletar, `Ctrl+S` guardar, `Ctrl+F3` activar.
- **Buscar:** `Ctrl+Shift+A`, filtra con `type:BDEF`, admite comodines (`z*expense*`).
- **Documentar:** con **Knowledge Transfer Documents (KTD)** en Markdown, por elemento (p. ej. documentar la operación `create`).
- **Relation Explorer:** vista de relaciones del BO (*Show In › Relation Explorer*), también muestra la documentación KTD con `F2`.

---

## 3. Tipos de implementación

Es la **primera** decisión de la BDEF e indica **quién implementa** el BO. Aplica a **todo** el árbol (no puedes tener una entidad managed y otra unmanaged).

| Qué se implementa | `managed` | `managed with additional save` | `managed with unmanaged save` | `unmanaged` |
|---|---|---|---|---|
| Operaciones estándar + buffer transaccional | RAP | RAP | RAP | **Tú** |
| Save sequence completa | RAP | RAP | RAP excepto `SAVE` | **Tú** |
| Fase `SAVE` | – | Tú (lógica *adicional*) | **Tú** (sustituye la de RAP) | – |
| Actions, determinations, validations | Tú | Tú | Tú | Tú (determinations/validations solo con draft) |

### Managed

- RAP gestiona CRUD, buffer transaccional y grabación en BD.
- Necesitas `persistent table <tabla>` en cada entidad (y normalmente `mapping`).

### Opciones de grabación en managed

- **`with unmanaged save`** → tú implementas la grabación (`SAVE_MODIFIED`). **No** se permite `persistent table`. Útil para grabar vía BAPI/API legacy.
- **`with additional save`** → RAP graba y tú añades lógica extra (p. ej. documentos de cambio, lanzar eventos).
- Complementos: **`and cleanup`** (habilita el método `CLEANUP`) y **`with full data`** (RAP te pasa *todos* los campos, no solo los modificados; ideal cuando la API legacy exige el registro completo).
- Se puede declarar a nivel de cabecera (todo el BO) o **por entidad**.

### Unmanaged

- Tú implementas **todo**: interacción y save sequence. Requiere behavior pool en la cabecera (`unmanaged implementation in class … unique;`).
- Caso típico: reutilizar código legacy existente (cap. 12 del libro).

### Otros

- **`projection`** → para behavior definitions de proyección (cap. 6).
- **`abstract`** → sobre entidades abstractas, solo para tipar parámetros. Sin comportamiento transaccional.

---

## 4. Strict mode

- `strict;` (versión 1) o **`strict ( 2 );`** (versión 2, la que ADT pone por defecto).
- Activa **comprobaciones de sintaxis más estrictas** basadas en buenas prácticas, y hace el BO más robusto frente a cambios futuros del framework.
- Obliga a declarar **explícitamente** lo que antes era implícito. Ejemplos: en unmanaged exige definir locking y autorización para cada entidad; exige que las hijas sean `authorization dependent`.
- ✅ **Úsalo siempre.**

---

## 5. Entity behavior definition

- Se abre con `define behavior for <entidad CDS>`. Debe existir **como mínimo** la de la raíz; las hijas son opcionales.
- **`alias`**: nombre legible que se usa en los tipos derivados y en **EML** (`ENTITY Expense`).
  - ⚠️ El alias es **visible externamente**: cambiarlo rompe a los consumidores EML. Elígelo bien desde el principio.
- **`external '<nombre>'`**: nombre largo (más de 30 caracteres) que solo aparece en el `$metadata` OData, no en ABAP.

---

## 6. Behavior pool

El **behavior pool** es la **clase ABAP global** donde implementas el comportamiento (convención `ZBP_…`). Se declara con `implementation in class <clase> unique` (`unique` es obligatorio: evita implementaciones solapadas).

Se puede declarar en **3 niveles**:

| Nivel | Dónde | Uso |
|---|---|---|
| **BO completo** | Cabecera de la BDEF | Todo en una clase. **Obligatorio** si implementas save sequence (`unmanaged`, `additional save`, `unmanaged save`). |
| **Entidad** | Cabecera de `define behavior for` | Una clase por entidad → más claro y permite trabajo en paralelo. La save sequence sigue yendo en el pool del BO. |
| **Implementation group** | Dentro del cuerpo: `group <nombre> implementation in class … unique { … }` | Repartir el comportamiento de **una** entidad en varias clases. |

Reglas de los **implementation groups**:
- Si usas grupos en una entidad, esa entidad **no** puede tener pool propio (el del BO sí se permite) y **todo** lo que necesite implementación debe estar en algún grupo.
- El nombre del grupo no puede coincidir con el de una action, etc.
- Son **internos**: puedes renombrarlos/reorganizarlos sin romper a nadie.
- Úsalos solo si de verdad necesitas varias clases para la misma entidad.

---

## 7. Numeración (numbering)

Define **cómo se rellenan las claves** al crear instancias (`create` y create-by-association).

Conceptos:
- **Clave temporal**: identifica la instancia nueva solo dentro del buffer transaccional (para relacionar padre-hijo antes de grabar).
- **Clave permanente**: la definitiva con la que se graba.

Dos ejes:
- **Momento**: *early* (en la fase de interacción) o *late* (en la save sequence).
- **Origen**: *external* (la pone el consumidor) o *internal* (la pone el BO).

| Tipo | Cómo se declara | Cuándo usarlo | Implementación |
|---|---|---|---|
| **Early external** (por defecto) | Nada | Claves con significado que da el usuario/API (p. ej. nº de material) | Ninguna |
| **Early internal** | `early numbering` en cabecera | Tú generas el número al crear | Método `FOR NUMBERING` |
| **Early internal con UUID (managed numbering)** | `field ( numbering : managed ) KeyUuid;` | Claves técnicas UUID — lo más habitual | Ninguna (RAP) |
| **Late (siempre internal)** | `late numbering` en cabecera + clave `readonly` | Números sin huecos (rangos de números: facturas, pedidos) o claves que dependen de datos conocidos al grabar | Método `ADJUST_NUMBERS` del saver |

Notas:
- Si una hija tiene **clave compuesta** que incluye la del padre y el padre usa late numbering, la hija **también** debe usar late numbering.
- **Uniqueness check** (comprobar que la clave no existe):
  - Numeración interna → la garantiza tu lógica (con UUID no hace falta).
  - Externa → RAP la hace en managed sin draft (y en activas con draft). En el resto, la haces tú vía `precheck` o, en drafts, en la acción `Resume`.

---

## 8. Propiedades de campo (field properties)

Por defecto **todos** los campos se pueden escribir en `create`, create-by-association y `update`. Con `field ( … )` defines la "interfaz" de cada operación.

> ⚠️ Estas restricciones solo aplican a **consumidores externos**. Dentro de tu behavior implementation (EML `IN LOCAL MODE`) no se comprueban.

| Declaración | Efecto |
|---|---|
| `field ( mandatory ) f1, f2;` | Marca el campo como obligatorio (asterisco en la UI). **RAP no lo comprueba**: debes validarlo tú (validation). |
| `field ( mandatory : create ) f1;` | Obligatorio al crear. **RAP sí lo comprueba**: si falta → dump `BEHAVIOR_MANDATORY_FIELD`. Para campos imprescindibles (tipo de documento, clave externa). |
| `field ( readonly ) f1;` | Protegido contra escritura externa en create y update (si se intenta → error en ejecución). |
| `field ( readonly : update ) f1;` | Solo lectura en update; editable al crear. |
| `field ( mandatory : create, readonly : update ) f1;` | Obligatorio al crear e inmutable después. Típico de campos de control (tipo de documento) y claves externas. |

**Campos que suelen ser `readonly`:**
- Claves con numeración interna (UUID, late numbering).
- Claves foráneas de la composición (p. ej. `ParentUuid` en la hija).
- Campos administrativos (`CreatedBy`, `CreatedAt`, `LastChangedAt`…).
- Campos calculados por determinations (estado, importes totales).

Las propiedades también pueden decidirse **en tiempo de ejecución** → ver *dynamic feature control* (§17).

---

## 9. Field mappings

Relacionan los **campos de la entidad CDS** (nombres legibles, camelCase) con los de un **tipo estructurado del DDIC** (tabla o estructura con nombres técnicos).

Casos de uso:
1. Tabla de BD ↔ entidad (managed con `persistent table`) — **el más común**.
2. Estructura ↔ entidad (p. ej. interfaz de una BAPI o función legacy).
3. Estructura de control ("X-structure") ↔ entidad.
4. Estructura ↔ parámetro de una action.

```
mapping for zexpense_a corresponding
{
  ExpenseUuid = expense_uuid;
  BeginDate   = begin_date;
}
```

- **`corresponding`**: además mapea automáticamente los campos que se llaman igual.
- **`control <estructuraX>`**: mapea la estructura de *checkbox* de la BAPI con `%control` de RAP (a nivel global o por campo si los nombres difieren).
- En ABAP lo usas con `CORRESPONDING #( … MAPPING FROM ENTITY )` (entidad → estructura) y `MAPPING TO ENTITY` (estructura → entidad). Así defines el mapeo **una sola vez**, en la BDEF.

---

## 10. Operaciones estándar

### Create, read, update, delete

- `create;` `update;` `delete;` → declaras qué operaciones de escritura soporta la entidad. **`read` está siempre disponible implícitamente.**
- **`create` solo en la raíz.** Las hijas se crean con **create-by-association** desde su padre.
- `update` modifica campos no clave (la clave identifica la instancia y no cambia).
- `delete`: en la interacción se marca en el buffer; en la save sequence se borra de la BD.
- `update` y `delete` de las hijas se declaran en la propia hija.

### Operaciones por asociación

| Declaración | Significado |
|---|---|
| `association _Item { }` | **Read-by-association**: leer hijos desde el padre (o el padre desde el hijo con `association _Parent { }`). |
| `association _Item { create; }` | **Create-by-association**: crear hijos desde el padre (incluye la lectura). |

- Aunque están disponibles implícitamente para composiciones y *to parent*, **declarálas siempre explícitamente**.
- La lectura por asociación hacia el padre/raíz es **requisito** para `lock dependent by` y `authorization dependent by` (RAP la usa para leer la instancia maestra).

---

## 11. Operaciones específicas: actions y functions

### Actions (`action`)

Operación propia con **lógica de negocio que modifica datos** (escritura, con bloqueo). Siempre pertenece a una entidad concreta.

- **De instancia** (por defecto): se ejecuta sobre instancias concretas. Ej.: `action approve;`
- **Estática** (`static action`): no depende de instancias, actúa sobre la entidad. Ej.: marcar duplicados.

**Parámetro de entrada** (`parameter`, como máximo **uno**, estructurado):
- Estructura del DDIC, entidad CDS **abstracta**, o `$self` (solo en estáticas).
- ¿Por qué solo uno? Porque es estructurado: si añades campos, lo cambias en un sitio y no tocas firmas.
- ⚠️ En strict mode, `$self` como parámetro de entrada de una action de instancia es error de sintaxis.

**Parámetro de retorno** (`result [cardinalidad] tipo`, opcional en actions):

| Cardinalidad | Significado (por instancia de entrada) |
|---|---|
| `[0..1]` | Como mucho uno |
| `[1]` | Exactamente uno |
| `[0..*]` | Cualquier número |
| `[1..*]` | Al menos uno |

- Tipos posibles: `$self`, `entity <otra entidad con BDEF>`, entidad abstracta, estructura DDIC.
- En las estáticas, la cardinalidad indica cuántas instancias devuelve en total.
- `selective`: el consumidor puede pedir solo algunos campos del resultado (optimización).
- Si usas `entity …` como parámetro, esa entidad debe estar expuesta en la service definition (en OData sale como *EntityType*; las estructuras como *ComplexType*).

**Variantes de action:**

| Variante | Declaración | Para qué |
|---|---|---|
| **Factory** | `factory action copy [1];` | Crea **nuevas** instancias de la entidad (copiar). Retorno implícito `$self`, cardinalidad `[1]` obligatoria. Puede ser `static` (crear con valores por defecto). En hijas se usan normalmente de instancia. |
| **Repeatable** | `repeatable action setDiscount …` | Se puede llamar varias veces sobre la misma instancia en una misma llamada EML / change set. Añade `%cid` para distinguir llamadas. |
| **Save action** | `save(finalize) action …` / `save(adjustnumbers) action …` | Se ejecuta durante la save sequence (en *finalize* o en *adjust numbers*). |

### Functions (`function`)

- Operación de **solo lectura, sin efectos secundarios** → no bloquea. Para cálculos o selecciones complejas.
- Igual que actions: de instancia o `static`, con parámetro de entrada opcional.
- **Parámetro de retorno obligatorio.**
- Admite `selective`.
- No admiten control de autorización ni dynamic feature control (las protege la DCL al leer CDS).

### Functions de "defaulting" (valores por defecto en la UI)

- `default function GetDefaultsFor…` añadida **a la operación** cuyos valores quieres proponer (no se declara aparte).
- Válida para `create`, create-by-association, y actions/functions **con parámetro** (incluidas factory).
- El nombre debe empezar por **`GetDefaultsFor`** o **`GetDfltsFor`** (en `create` puede omitirse el nombre o ser `GetDefaultsForCreate`).
- **Solo la ejecuta Fiori elements automáticamente** (p. ej. rellenar el popup de una action). Vía EML no se dispara sola.
- Si quieres valores iniciales como **lógica de negocio** (independiente de la UI), usa una **determination on modify { create; }**.

---

## 12. Concurrencia y bloqueos

Dos mecanismos que se pueden combinar. **Si usas optimista, también debes declarar el pesimista** (el pesimista es siempre necesario).

### Bloqueo pesimista (`lock`)

- Antes de modificar se pone un **bloqueo de escritura** (enqueue server, tabla global de bloqueos). Otros intentos paralelos fallan.
- Se libera al terminar la sesión ABAP (cada *roundtrip* stateless) o con commit/rollback. Con draft, el bloqueo vive lo que viva el draft.
- **`lock master`** → solo en la **raíz**.
- **`lock dependent by _Root`** → en las hijas. Requiere `association _Root { }` declarada y **directa a la raíz** (aunque la hija esté en un nivel más profundo).
- Modificar un hijo bloquea **toda** la instancia raíz y, con ella, todos sus hijos.
- En managed lo implementa RAP. En unmanaged (o con `lock master unmanaged` en managed, p. ej. para reutilizar bloqueos de una app legacy) implementas el método **`FOR LOCK`**.

### Bloqueo optimista (`etag`)

- No hay bloqueo real: se detecta si la instancia **cambió desde que la leíste**, comparando un campo **ETag** (timestamp de última modificación, hash, contador de versión). Si no coincide → rechazo y el consumidor debe releer.
- RAP compara el ETag en `update`, `delete` y actions.
- **Solo relevante en OData**, no para consumidores EML locales.
- `etag master <campo>` → en la raíz o también en hijas (se puede tener varios).
- `etag dependent by _Parent` → la hija usa el ETag del padre (requiere read-by-association). El libro **recomienda** que cada hija sea su propio `etag master` (mejor experiencia en la UI).
- Convención de campos:
  - `LastChangedAt` (en la raíz) → último cambio de **todo** el BO.
  - `LocalLastChangedAt` → último cambio de **esa** entidad.
- Con draft se requiere además el **total etag** (ver §15).

---

## 13. Lógica de negocio interna: determinations y validations

Son lógica **interna** del BO (no las llama el consumidor directamente). Requieren implementación. Disponibles en managed (con o sin draft) y en unmanaged **solo con draft**.

### Disparadores (trigger conditions)

Comunes a ambas:
- Operaciones estándar: `create`, `update`, `delete` (solo las que estén declaradas).
- Campos: `field F1, F2`.
- Combinaciones: `{ create; field F1; }`.
- ⚠️ Los disparadores se combinan con **OR**: basta con que se cumpla uno.

### Determinations

Calculan o modifican datos automáticamente.

| Momento | Cuándo se ejecuta |
|---|---|
| `on modify` | Justo después de cada modificación en el buffer (fase de interacción). |
| `on save` | Justo antes de grabar, en la fase **FINALIZE** de la save sequence. |

Solo puede tener **uno** de los dos momentos.

Usos típicos: valores iniciales (estado "Nuevo"), sumar importes de hijos en el padre, recalcular al cambiar un campo, derivar estados, crear instancias.

Puntos importantes:
- **`delete` como disparador**: RAP no te dice qué instancias se borraron. Si mezclas `delete` con otros disparadores no podrás distinguirlas; usa una determination **solo** con `delete`.
- **Cascadas**: una determination que cambia un campo puede disparar otra (y otra…). Riesgo de bucles infinitos → usa disparadores **por campo** lo más específicos posible.
- En `on save`, `update` solo se permite junto con `create`. RAP **agrega** las operaciones de la transacción:

| Secuencia en la transacción | Disparador resultante |
|---|---|
| create + update | create |
| create + delete | delete |
| update + update | update |
| update + delete | delete |
| delete + create | create |

### Validations

Comprobaciones de consistencia. **Siempre `on save`**, se ejecutan en la fase **CHECK_BEFORE_SAVE**. Si fallan, RAP cancela la grabación de forma controlada (devuelves `failed` + `reported`).

- `update` solo junto con `create` (igual que determinations on save).
- **`delete` como disparador**: la instancia ya está borrada en el buffer y no puedes deshacerlo. Para decidir *si se puede borrar*, usa **`precheck`** (§17).
- **`mandatory` vs `mandatory:create`**:
  - `field ( mandatory )` + validation → el campo puede estar vacío durante la transacción y se comprueba al grabar (error al usuario).
  - `field ( mandatory : create )` → vacío = error de programación = dump. Solo para campos imprescindibles.

### Determine action

- `determine action check { validation validateX; determination Item~calcTotal; }`
- Agrupa determinations/validations **on save** y las hace **invocables desde fuera** durante la interacción (sin esperar al save).
- Se pueden referenciar las de entidades hijas con `Entidad~nombre`.
- `(always)` → se ejecuta aunque no se cumpla su disparador.
- Usos: validar en caliente desde la UI (combinado con side effects), o que un consumidor EML compruebe antes de grabar. Clave en draft (`Prepare`).

---

## 14. Autorizaciones

- Para **lectura**, la autorización la pone la **DCL** de CDS (access controls, cap. 2) — también en EML.
- Para **escritura y actions**, se declaran en la BDEF y se implementan en el behavior pool (normalmente con objetos de autorización `AUTHORITY-CHECK`).
- Las **functions** no se protegen aquí (solo vía DCL al leer).

### Global vs. instancia

| Tipo | Qué comprueba | Operaciones | Método |
|---|---|---|---|
| **Global** | ¿El usuario puede hacer esta operación en general? | `create`, `update`, `delete`, actions (incluidas static y factory) | `FOR GLOBAL AUTHORIZATION` |
| **Instancia** | ¿Puede hacerla sobre **esta** instancia según sus datos? | `update`, `delete`, create-by-association, actions de instancia | `FOR INSTANCE AUTHORIZATION` |

- `create` **no** tiene comprobación por instancia (aún no existe). Si depende de valores (p. ej. sociedad), usa **`precheck`**.
- Con `( global, instance )` primero se ejecuta la global y después la de instancia (puede pasar la global y fallar la de instancia; p. ej. autorizado a liberar pedidos, pero no por encima de cierto importe).

### Master y dependent

- **`authorization master ( global | instance | global, instance )`** → solo en la **raíz**.
- **`authorization dependent by _Root`** → en las hijas. Requiere asociación directa a la raíz declarada.
  - `update`, `delete` y create-by-association de la hija se **delegan** al master y se evalúan como un **update del master**.
  - Las **actions** de la hija se comprueban **en la propia hija** (no se delegan).
  - ✅ Declara todas las hijas como dependientes (strict mode lo exige).

### Delegar / excluir

- `delete ( authorization : update );` → el borrado se evalúa como update. También para create-by-association y actions. En hijas solo tiene sentido para actions.
- `authorization : none` → excluye la operación de la comprobación.

---

## 15. Draft handling y side effects

> 💡 **Consejo del libro:** usa draft incluso en casos sencillos; RAP lo implementa entero.

### Concepto

- El **draft** es una versión **temporal** (puede ser inconsistente) que se guarda en BD para seguir editando más tarde. La versión productiva es la **instancia activa**.
- Una activa → como máximo **un** draft.
- Los drafts se guardan en **tablas draft** separadas; no afectan a los datos productivos.
- El draft es un **buffer transaccional persistente**: permite que una "transacción lógica" dure varias peticiones REST stateless.
- Fiori elements guarda el draft automáticamente al editar y permite filtrar por drafts sin programar nada.
- Disponible en managed y unmanaged. En unmanaged, determinations/validations **solo** funcionan con draft (y solo sobre instancias draft).

### Activación: `with draft;`

- En la **cabecera** de la BDEF → aplica a todo el BO (no se puede por entidad).
- Se puede añadir más tarde aunque ya haya datos productivos.

### Lo que exige en cada entidad

1. **`draft table <tabla>`** en cada entidad del árbol.
   - Se crea con un *quick fix* en ADT.
   - Convención: `…_A_…` (activa) y `…_D_…` (draft).
   - Columnas = **nombres de campos de la entidad CDS** (no de la tabla activa).
   - Incluye los campos administrativos `SYCH_BDL_DRAFT_ADMIN_INC`.
   - Si cambias la tabla activa, debes **sincronizar** la draft a mano.
2. **`total etag <campo>`** en la **raíz**: ETag de todo el BO (no se activa parcialmente). Protege la transición draft → activo si la activa cambió tras expirar el bloqueo exclusivo (p. ej. un job).
3. **Asociaciones draftables**: automáticas con `with draft`; se puede poner explícito `association _Item { create; with draft; }`.

### Ciclo de vida y draft actions (en la raíz)

| Draft action | Qué hace |
|---|---|
| `create` | Crea un draft **nuevo** (no hay activa). |
| `draft action Edit;` | Copia una activa a un draft para editarla. |
| `draft action Activate;` | Convierte el draft en activa (ejecuta antes `Prepare`). |
| `draft action Discard;` | Descarta el draft. |
| `draft action Resume;` | Retoma un draft cuando el bloqueo exclusivo expiró: renueva el bloqueo y hace el uniqueness check. |
| `draft determine action Prepare { … }` | Lista de validations/determinations **on save** que se ejecutan antes de activar. Si falla una validation, **no se activa**. |

- Al crear/editar un draft, la activa queda con **bloqueo exclusivo** temporal para ese usuario. Al expirar, entra el ETag. El propio draft se protege con ETag (p. ej. el mismo usuario en dos navegadores).
- Los drafts abandonados los borra RAP automáticamente tras un tiempo.
- `with additional implementation` → añade tu código después de una draft action (p. ej. uniqueness check en `Resume` en unmanaged).

### Side effects (UI)

Indican a la UI **qué recargar** cuando algo cambia (porque una determination recalculó otro campo, etc.). Se convierten en anotaciones OData que Fiori elements interpreta; no hay que programar en el frontend.

```
side effects {
  field Quantity affects field TotalAmountItem, field _Expense.TotalAmount;
  action setDiscount affects field TotalAmountItem;
  determine action doValidateCustomer executed on field CustomerId affects messages;
}
```

- Se declara **una vez** `side effects { … }` por entidad; cada disparador solo una vez.
- **Disparadores**: `field f`, `action a`, `$self`, o `determine action … executed on field / $self / entity _Assoc`.
- **Objetivos** (`affects`): `field f`, `field _Assoc.f`, `messages` (solo OData V4), `permissions ( … )` (autorizaciones/feature control), `$self`, `entity _Assoc`.
- **Tres tipos:**
  1. **Por campo**: cambia la cantidad → recarga el importe.
  2. **Por action**: ejecuto `setDiscount` → recarga el importe.
  3. **Con determine action**: cambia `CustomerId` → ejecuta la validación y muestra los **mensajes** al momento.
- ⚠️ El disparador debe ser un campo que cambie el **usuario en la UI**, no uno calculado internamente.

---

## 16. Eventos

**Business events** para comunicación **asíncrona y desacoplada**. El BO no conoce a los receptores; estos se suscriben a la infraestructura de eventos (local, o **SAP Event Mesh** en BTP para eventos globales).

- Se lanzan desde la implementación con EML (típicamente al final de la save sequence; en managed, en el **additional save**).
- Siempre viajan las **claves**; opcionalmente un **payload** propio.
- Usos: instancia creada/modificada/borrada, cambio de estado (liberado, entregado…), superar un umbral.
- Forman parte de la **interfaz pública** → cambios compatibles, sobre todo si se consumen en remoto.

| Tipo | Declaración | Notas |
|---|---|---|
| Sin parámetros | `event created;` | Lo lanzas tú. |
| Con parámetros | `event cancelled parameter ZA_Cancelled;` | Payload tipado con entidad **abstracta**. |
| **Derivado** | `managed event cancelledWithCustomer on cancelled parameter ZR_CancelledCustomer;` | Lo lanza **RAP** cuando salta el evento base (no se puede lanzar por EML). El parámetro debe ser una entidad CDS **no abstracta y no custom** con las **mismas claves**; RAP lee de ella el payload enriquecido. |

---

## 17. Conceptos transversales

### 17.1 Dynamic feature control

Decidir **en tiempo de ejecución** las propiedades de campos, operaciones y actions.

| Feature | Qué se puede controlar |
|---|---|
| Campos | sin restricción / readonly / mandatory / ambas |
| Operaciones estándar | activa / inactiva |
| Actions | activa / inactiva |
| Functions | ❌ no soportado |

- `( features : instance )` → depende de los **datos de cada instancia**. Método `FOR FEATURES` (instance features).
- `( features : global )` → igual para todas las instancias (p. ej. un bloqueo de contabilización por periodo, un customizing). Método `FOR GLOBAL FEATURES`.
- No se pueden combinar ambos en el mismo elemento.

| | Campos | Operaciones | Actions |
|---|---|---|---|
| **Global** | – | create, create-by-assoc, update, delete | static action, static factory, action, factory |
| **Instancia** | `field` | update, delete, create-by-assoc | action, factory |

Ejemplos de uso:
- `field ( features : instance ) OrderedQuantity;` → editable solo si no se ha entregado.
- `delete ( features : instance );` → borrar solo si no está confirmado.
- `action ( features : instance ) approve;` → botón activo solo en estado "Pendiente".

Reglas:
- ⚠️ Un campo con `readonly` estático **no** puede tener además feature control dinámico.
- **Feature control ≠ autorización**: feature control = *¿tiene sentido de negocio?* (según el estado); autorización = *¿el usuario tiene permiso?* No mezcles comprobaciones de permisos en el feature control.
- Lo declarado estáticamente ya es feature control "estático y global"; lo dinámico ajusta eso en ejecución.

### 17.2 Precheck

- `create ( precheck );` (también update, delete, actions…).
- Comprobación **antes** de que la operación llegue al buffer transaccional → si falla, nada entra al buffer.
- Usos: uniqueness check con numeración externa, autorización dependiente de valores en `create` (p. ej. sociedad), impedir borrados por motivos de negocio.
- Método `FOR PRECHECK`.

### 17.3 `internal`

- Hace que una operación (estándar, action, determine action, function) sea **visible solo dentro del BO**. El acceso externo da error de sintaxis.
- Ventaja: puedes cambiarla sin preocuparte por los consumidores.
- Ejemplos:
  - `internal action reCalcTotalPrice;` → la llama una determination de la hija.
  - `association _StatusLog { internal create; }` → un log que solo rellena el BO; desde fuera es solo lectura.
  - `internal create;` + `static factory action createDefault… [1];` → desde fuera solo se puede crear mediante la factory action.

---

## 18. Glosario inglés → español

| Inglés | Español / significado |
|---|---|
| Behavior definition (BDEF) | Definición de comportamiento |
| BDL | Lenguaje de definición de comportamiento |
| Behavior pool | Clase ABAP que implementa el comportamiento |
| Composition tree | Árbol de composición (raíz + hijas) |
| Root / child entity | Entidad raíz / hija |
| Transactional buffer | Buffer transaccional (datos en memoria durante la transacción) |
| Interaction phase | Fase de interacción (modificaciones antes de grabar) |
| Save sequence | Secuencia de grabación (finalize, check_before_save, adjust_numbers, save, cleanup) |
| Managed / unmanaged | Gestionado por RAP / implementado por ti |
| Numbering (early/late, internal/external) | Numeración (temprana/tardía, interna/externa) |
| Determination | Determinación: cálculo automático |
| Validation | Validación: comprobación de consistencia |
| Trigger condition | Condición de disparo |
| Action / function | Acción (escribe) / función (solo lee) |
| Factory action | Acción que crea instancias nuevas |
| Lock master / dependent | Maestro / dependiente del bloqueo |
| ETag | Marca de versión para bloqueo optimista |
| Authorization master / dependent | Maestro / dependiente de autorización |
| Draft / active instance | Borrador / instancia activa |
| Side effects | Efectos secundarios de UI (qué recargar) |
| Feature control | Control dinámico de habilitación de campos y operaciones |
| Precheck | Comprobación previa (antes del buffer) |
| Payload | Datos adicionales de un evento |

---

## 19. Chuleta: qué método exige cada declaración

| Declaración en BDEF | Método en el behavior pool |
|---|---|
| `authorization master ( global )` | `FOR GLOBAL AUTHORIZATION` |
| `authorization master ( instance )` | `FOR INSTANCE AUTHORIZATION` |
| `( features : instance )` | `FOR INSTANCE FEATURES` |
| `( features : global )` | `FOR GLOBAL FEATURES` |
| `lock master` en unmanaged o `lock master unmanaged` | `FOR LOCK` |
| `early numbering` | `FOR NUMBERING` |
| `late numbering` | `ADJUST_NUMBERS` (saver) |
| `( precheck )` | `FOR PRECHECK` |
| `action …` / `function …` | `FOR MODIFY … ACTION` / `FOR READ … FUNCTION` |
| `determination …` | `FOR DETERMINE ON MODIFY / ON SAVE` |
| `validation …` | `FOR VALIDATE ON SAVE` |
| `default function GetDefaultsFor…` | Método de la función de defaulting |
| `with unmanaged save` | `SAVE_MODIFIED` (saver) |
| `with additional save` | `SAVE_MODIFIED` (saver, lógica adicional) |
| `… and cleanup` | `CLEANUP` (saver) |
| `unmanaged` | Todo: create/update/delete/read/lock + saver completo |

> Los detalles de cada método están en el capítulo 5 (Behavior Implementation).

---

## 20. Preguntas de autoevaluación

1. ¿Por qué una entidad hija no puede tener su propia behavior definition?
2. ¿Qué diferencia hay entre `managed with additional save` y `managed with unmanaged save`? ¿Cuál impide usar `persistent table`?
3. ¿Qué tipo de numeración usarías para un número de factura sin huecos? ¿Y qué método tendrías que implementar?
4. `field ( mandatory )` vs. `field ( mandatory : create )`: ¿cuál comprueba RAP y qué ocurre si falla?
5. ¿Por qué no debes usar una validation con disparador `delete` para impedir un borrado? ¿Qué usarías?
6. ¿Una determination puede ejecutarse `on modify` **y** `on save` a la vez?
7. ¿Qué ocurre con los bloqueos si modificas la posición 20 de un pedido y otro usuario intenta modificar la posición 10?
8. ¿Por qué el bloqueo optimista no afecta a los consumidores EML locales?
9. ¿Qué tres cosas exige `with draft` en las entidades del BO?
10. ¿Qué hace `Prepare` y qué pasa si una de sus validations falla?
11. ¿Cuándo usarías dynamic feature control y cuándo una comprobación de autorización?
12. ¿Cómo conseguirías que un BO solo se pueda crear desde una factory action?
13. ¿Qué diferencia hay entre una *default function* y una determination `on modify { create; }` para poner valores iniciales?
14. Un side effect con disparador `field TotalAmount` (campo calculado): ¿por qué no funciona?
15. ¿Qué requisitos tiene la entidad CDS que se usa como parámetro de un `managed event`?

<details>
<summary>Respuestas breves</summary>

1. Porque el comportamiento se define para el BO completo; la BDEF se crea solo para la raíz y las hijas se declaran dentro.
2. *Additional*: RAP graba y tú añades lógica. *Unmanaged save*: tú grabas en lugar de RAP. `unmanaged save` impide `persistent table`.
3. `late numbering` (con rango de números) → `ADJUST_NUMBERS`.
4. `mandatory` solo marca el campo (lo validas tú). `mandatory:create` lo comprueba RAP → dump `BEHAVIOR_MANDATORY_FIELD`.
5. Porque la instancia ya está borrada en el buffer y no se puede deshacer → `delete ( precheck )` o feature control de instancia.
6. No, solo uno de los dos.
7. Se bloquea la raíz (lock master) y con ella todas las posiciones → el segundo usuario es rechazado.
8. Porque el ETag es un mecanismo del protocolo OData (cabeceras If-Match).
9. `draft table` en cada entidad, `total etag` en la raíz, asociaciones draftables (automáticas).
10. Ejecuta las validations/determinations on save listadas antes de activar; si una validation falla, no se activa.
11. Feature control: motivo de negocio según datos/estado. Autorización: permisos del usuario.
12. `internal create;` + `static factory action … [1];`.
13. La default function solo la usa la UI Fiori elements (propuesta); la determination es lógica de negocio que se aplica siempre.
14. Porque los disparadores deben ser cambios hechos por el usuario en la UI; un campo calculado no lo cambia el usuario.
15. No abstracta, no custom, con las mismas claves que la entidad que declara el evento.

</details>
