# Diagramación del sistema SimulaFin

**Proyecto analizado:** `SimulaFin_MVC_18OCT`  
**Fecha y hora:** 2026-10-18 14:15:58  
**Alcance:** navegación, actividades, capas, clases del dominio y persistencia existentes. Documento descriptivo; no modifica el código.

> En el workspace no se encontraron `flujoAPPsimulaFin.md` ni `detalleFlujoAppSimulaFIN.txt`. Los diagramas se basan en el código Kotlin y `AndroidManifest.xml` de esta versión.

## 1. Visión general

SimulaFin es una aplicación Android con persistencia local SQLite (`simulafin.db`). El usuario entra mediante `LoginActivity`; la sesión activa se conserva en memoria por `SesionUsuario`. El menú principal lleva a cuentas, ítems, edición de usuario, respaldos y simulaciones.

Hay dos modalidades de simulación:

- **Paso a paso:** Activities enlazadas por Intents, con `BaseDatosHelper` y estado temporal en `SesionSimulacion`.
- **Síntesis interactiva:** `SimulacionActivity` coordina `SimulacionInputFragment` y `SimulacionOutputFragment`; utiliza `SimulacionController`, el contrato `SimulacionRepository`, `SQLiteSimulacionRepository` y el dominio de cálculo.

La base relacional contiene `Usuarios`, `Cuentas`, `Items`, `Simulaciones` y `DetallesSimulaciones`. La app permite generar PDF, exportar consultas a JSON/CSV y operar respaldos locales.

## 2. Navegación global

```mermaid
flowchart TD
    Start([Inicio]) --> Login[LoginActivity]
    Login --> DB[Inicializa SQLite y prepara tablas]
    DB --> Complete{¿Campos completos?}
    Complete -->|No| Login
    Complete -->|Sí| HasUsers{¿Hay usuarios registrados?}
    HasUsers -->|No| Register[registroUsuarioNuevo]
    HasUsers -->|Sí| Auth{¿Credenciales válidas?}
    Auth -->|No| Login
    Auth -->|Sí| Session[Establece SesionUsuario.usuarioActivo]
    Register --> ValidRegister{¿Nombre disponible y claves iguales?}
    ValidRegister -->|No| Register
    ValidRegister -->|Sí| SaveUser[Guarda usuario con hash y salt]
    SaveUser --> Session
    Register -->|Cancelar o atrás| Login
    Session --> Main[MainActivity]

    Main -->|Simulaciones| SimMenu[MenuSimActivity]
    Main -->|Cuentas| Accounts[CuentasActivity]
    Main -->|Ítems| Items[ItemsActivity]
    Main -->|Registro| Profile[RegistroActivity]
    Main -->|Respaldo| Backup[RespaldoActivity]
    Accounts -->|Home| Main
    Items -->|Home| Main
    Profile -->|Guardar o Home| Main
    Backup -->|Home| Main

    SimMenu -->|Paso a paso| StepAccount[InputSimCuentaActivity]
    SimMenu -->|Síntesis MVC| MVC[SimulacionActivity]
    SimMenu -->|Consulta| Query[ConsultaSimulacionesActivity]
    SimMenu -->|Home| Main
    StepAccount -->|Selección válida| StepItems[InputSimItemsActivity]
    StepAccount -->|Home| Main
    StepItems -->|Lista no vacía| Summary[OutSimResumenActivity]
    StepItems -->|Home| Main
    Summary -->|Editar ítems| StepItems
    Summary -->|Nueva simulación| StepAccount
    Summary -->|Guardar o imprimir| Summary
    Summary -->|Menú| SimMenu
    Summary -->|Home| Main
    MVC -->|Home| Main
    Query -->|Home| Main
```

## 3. Actividad de simulación

El siguiente flujo reúne las dos modalidades y deja visibles las validaciones, las operaciones de cálculo, edición y guardado.

```mermaid
flowchart TD
    Start([Iniciar simulación]) --> Mode{Elegir modalidad}
    Mode -->|Paso a paso| Select[Seleccionar o crear simulación]
    Mode -->|Síntesis MVC| Interactive[Seleccionar escenario en fragment de entrada]

    Select --> Account[Seleccionar o crear cuenta y definir saldo]
    Account --> ValidStep{¿Simulación, cuenta y saldo válidos?}
    ValidStep -->|No| Select
    ValidStep -->|Sí| HasDetails{¿Hay detalles guardados?}
    HasDetails -->|Sí| Overwrite{¿Confirma sobrescritura?}
    Overwrite -->|No| Select
    Overwrite -->|Sí| Restore[Precarga detalles actuales]
    HasDetails -->|No| Catalog
    Restore --> Catalog[Cargar catálogo de ítems]
    Catalog --> Add[Seleccionar ítem e importe y agregar]
    Add --> Sum[Recalcular total mensual]
    Sum --> More{¿Agregar o quitar otro ítem?}
    More -->|Sí| Add
    More -->|No| NonEmpty{¿Hay al menos un ítem?}
    NonEmpty -->|No| Catalog
    NonEmpty -->|Sí| Summary[Presentar resumen paso a paso]
    Summary --> Calculate[Calcular meses y excedente]
    Calculate --> SaveStep{¿Guardar simulación?}
    SaveStep -->|Sí| PersistStep[Guardar cabecera y reemplazar detalles]
    SaveStep -->|No| StepOutput[Imprimir o navegar a ítems, nueva simulación, menú o Home]
    PersistStep --> EndStep([Fin del guardado])

    Interactive --> Edit[Editar cuenta e ítems en fragments]
    Edit --> Notify[Notificar cambios a SimulacionActivity]
    Notify --> Live[Actualizar resultados en fragment de salida]
    Live --> Action{Acción}
    Action -->|Editar| Edit
    Action -->|Guardar| Validate[Validar usuario, selección, importes y detalles]
    Validate --> Persistable{¿Datos válidos y persistibles?}
    Persistable -->|No| Reject[Mostrar rechazo o error]
    Persistable -->|Sí| Existing{¿Existen detalles guardados?}
    Existing -->|Sí| Confirm{¿Confirma sobrescritura?}
    Confirm -->|No| New[Ofrecer crear otra simulación]
    Confirm -->|Sí| Transaction[Guardar cabecera y detalles en transacción]
    Existing -->|No| Transaction
    Transaction --> Saved[Informar guardado]
    Action -->|Eliminar simulación| Delete[Confirmar y eliminar simulación]
    Action -->|Limpiar| Clear[Limpiar datos de entrada y salida]
    Action -->|Imprimir| PDF[Crear PDF y solicitar visor externo]
    Reject --> Action
    New --> Edit
    Saved --> EndMVC([Fin del guardado])
```

**Cálculo:** el total de egresos suma los valores de los ítems. Si es mayor que cero, `meses = saldo / egresos` usando división entera y `excedente = saldo - meses * egresos`. Con egresos cero, el dominio entrega cero meses y el saldo como excedente; el resumen paso a paso etiqueta la duración como indefinida.

## 4. Arquitectura de componentes

```mermaid
flowchart LR
    subgraph UI[Presentación Android]
        Login[LoginActivity]
        Main[MainActivity]
        Menu[MenuSimActivity]
        CRUD[CuentasActivity e ItemsActivity]
        Step[InputSimCuentaActivity -> InputSimItemsActivity -> OutSimResumenActivity]
        Activity[SimulacionActivity]
        Input[SimulacionInputFragment]
        Output[SimulacionOutputFragment]
        Query[ConsultaSimulacionesActivity]
        Backup[RespaldoActivity]
    end
    subgraph Contracts[Contratos y estado]
        Listener[OnSimulacionDatosListener]
        Provider[SimulacionControllerProvider]
        UserSession[SesionUsuario]
        SimSession[SesionSimulacion]
    end
    subgraph Logic[Lógica de simulación]
        Controller[SimulacionController]
        Calculator[SimulacionCalculo]
        Command[GuardarSimulacionCommand y resultado]
    end
    subgraph Data[Acceso y persistencia]
        Repository[SimulacionRepository]
        SqlRepository[SQLiteSimulacionRepository]
        Helper[BaseDatosHelper]
        DB[(SQLite simulafin.db)]
    end
    Login --> Helper
    Login --> UserSession
    Main --> Menu
    Main --> CRUD
    Main --> Step
    Main --> Activity
    Main --> Query
    Main --> Backup
    Activity --- Input
    Activity --- Output
    Activity -. implementa .-> Listener
    Activity -. implementa .-> Provider
    Input -. notifica .-> Listener
    Output -. delega acciones .-> Listener
    Input --> Controller
    Activity --> Controller
    Controller --> Calculator
    Controller --> Command
    Controller --> Repository
    SqlRepository -. implementa .-> Repository
    SqlRepository --> Helper
    Helper --> DB
    CRUD --> Helper
    Step --> Helper
    Query --> Helper
    Backup --> Helper
    Step --> SimSession
```

Las Activities de mantenimiento, consulta, respaldo y paso a paso acceden a `BaseDatosHelper` directamente. El patrón Repository y la capa Controller están aplicados principalmente al flujo de síntesis MVC.

## 5. Diagrama de clases UML

```mermaid
classDiagram
    direction LR
    class AppCompatActivity
    class Fragment
    class LoginActivity
    class MainActivity
    class MenuSimActivity
    class registroUsuarioNuevo
    class RegistroActivity
    class CuentasActivity
    class ItemsActivity
    class InputSimCuentaActivity
    class InputSimItemsActivity
    class OutSimResumenActivity
    class ConsultaSimulacionesActivity
    class RespaldoActivity
    class SimulacionActivity
    class SimulacionInputFragment
    class SimulacionOutputFragment

    AppCompatActivity <|-- LoginActivity
    AppCompatActivity <|-- MainActivity
    AppCompatActivity <|-- MenuSimActivity
    AppCompatActivity <|-- registroUsuarioNuevo
    AppCompatActivity <|-- RegistroActivity
    AppCompatActivity <|-- CuentasActivity
    AppCompatActivity <|-- ItemsActivity
    AppCompatActivity <|-- InputSimCuentaActivity
    AppCompatActivity <|-- InputSimItemsActivity
    AppCompatActivity <|-- OutSimResumenActivity
    AppCompatActivity <|-- ConsultaSimulacionesActivity
    AppCompatActivity <|-- RespaldoActivity
    AppCompatActivity <|-- SimulacionActivity
    Fragment <|-- SimulacionInputFragment
    Fragment <|-- SimulacionOutputFragment

    MainActivity --> MenuSimActivity : navegación
    MainActivity --> CuentasActivity : navegación
    MainActivity --> ItemsActivity : navegación
    MainActivity --> RegistroActivity : navegación
    MainActivity --> RespaldoActivity : navegación
    MenuSimActivity --> InputSimCuentaActivity : paso a paso
    MenuSimActivity --> SimulacionActivity : síntesis MVC
    MenuSimActivity --> ConsultaSimulacionesActivity : consulta
    InputSimCuentaActivity --> InputSimItemsActivity : Intent y extras
    InputSimItemsActivity --> OutSimResumenActivity : Intent y resultados

    class OnSimulacionDatosListener {
        <<interface>>
        +onDatosSimulacionIngresados(totalGastos, totalCuenta)
        +onSolicitarGuardarSimulacion()
        +onSolicitarEliminarSimulacion()
        +onSolicitarLimpiarSimulacion()
        +onSolicitarImprimirSimulacion()
    }
    class SimulacionControllerProvider {
        <<interface>>
        +getSimulacionController()
    }
    SimulacionActivity ..|> OnSimulacionDatosListener
    SimulacionActivity ..|> SimulacionControllerProvider
    SimulacionActivity o-- SimulacionInputFragment
    SimulacionActivity o-- SimulacionOutputFragment
    SimulacionInputFragment ..> OnSimulacionDatosListener : notifica
    SimulacionInputFragment ..> SimulacionControllerProvider : obtiene controller
    SimulacionOutputFragment ..> OnSimulacionDatosListener : delega

    class SimulacionController {
        -repository: SimulacionRepository
        +guardar(command, sobrescribirConfirmado)
        +obtenerSimulaciones(idUsuario)
        +obtenerCuentas(idUsuario)
        +obtenerItems(idUsuario)
        +eliminarSimulacion(idSimulacion)
    }
    class SimulacionRepository {
        <<interface>>
        +guardarCompleta(...)
        +obtenerSimulaciones(idUsuario)
        +obtenerDetallesSimulacion(idSimulacion)
    }
    class SQLiteSimulacionRepository {
        -dbHelper: BaseDatosHelper
        +guardarCompleta(...)
    }
    SimulacionController --> SimulacionRepository
    SQLiteSimulacionRepository ..|> SimulacionRepository
    SQLiteSimulacionRepository --> BaseDatosHelper
    SimulacionController ..> SimulacionCalculo

    class SimulacionCalculo {
        <<object>>
        +calcular(totalDisponible, detalles)
        +calcularTotales(totalDisponible, totalEgresos)
    }
    class GuardarSimulacionCommand
    class GuardarSimulacionResultado
    SimulacionController ..> GuardarSimulacionCommand
    SimulacionController ..> GuardarSimulacionResultado

    class BaseDatosHelper {
        +generaBaseDatos()
        +CreacionBaseDatosSimula()
        +insertarSimulacion(...)
        +reemplazarDetallesSimulacion(...)
        +obtenerSimulaciones(idUsuario)
        +obtenerDetallesSimulacion(idSimulacion)
    }
    class SesionUsuario {
        <<object>>
        +usuarioActivo: String
    }
    class SesionSimulacion {
        <<object>>
        +simulacion: String
        +cuenta: String
        +valorCuenta: String
        +listaItemsAgregados: MutableList
    }
    class Usuario {
        +idUsuario: Int
        +nombreUsuario: String
        +claveUsuario: String
        +fechaHoraUsuario: String
        +esAdmin: Boolean
        +preferenciaTema: String
    }
    class Cuenta {
        +idCuenta: Int
        +idUsuario: Int
        +nombreCuenta: String
        +totalCuenta: Int
        +fechaHoraCuenta: String
    }
    class Item {
        +idItem: Int
        +idUsuario: Int
        +nombreItem: String
        +valorItem: Int
    }
    class Simulacion {
        +idSim: Int
        +idCuenta: Int
        +idUsuario: Int
        +nombreSim: String
        +fechaHoraSim: String
        +totalDisponible: Int
        +totalEgresos: Int
        +meses: Int
        +excedente: Int
    }
    class DetalleSimulacion {
        +idDetalle: Int
        +idSim: Int
        +idItem: Int nullable
        +nombreItem: String
        +valorItem: Int
    }
    class DetalleSimulacionEntrada
    class ResultadoCalculoSimulacion
    BaseDatosHelper ..> Usuario
    BaseDatosHelper ..> Cuenta
    BaseDatosHelper ..> Item
    BaseDatosHelper ..> Simulacion
    BaseDatosHelper ..> DetalleSimulacion
    SimulacionRepository ..> Cuenta
    SimulacionRepository ..> Item
    SimulacionRepository ..> Simulacion
    SimulacionRepository ..> DetalleSimulacion
    SimulacionCalculo ..> DetalleSimulacionEntrada
    SimulacionCalculo ..> ResultadoCalculoSimulacion
    Usuario "1" --> "0..*" Cuenta : posee
    Usuario "1" --> "0..*" Item : posee
    Usuario "1" --> "0..*" Simulacion : crea
    Cuenta "1" --> "0..*" Simulacion : financia
    Simulacion "1" *-- "0..*" DetalleSimulacion : contiene
    Item "0..1" --> "0..*" DetalleSimulacion : referencia
```

## 6. Secuencia de guardado MVC

```mermaid
sequenceDiagram
    autonumber
    actor Usuario
    participant Output as SimulacionOutputFragment
    participant Activity as SimulacionActivity
    participant Input as SimulacionInputFragment
    participant Controller as SimulacionController
    participant Calc as SimulacionCalculo
    participant Repo as SQLiteSimulacionRepository
    participant DB as SQLite

    Usuario->>Output: Pulsa Guardar
    Output->>Activity: onSolicitarGuardarSimulacion()
    Activity->>Input: Lee simulación, cuenta, saldo y detalles
    Activity->>Activity: Construye GuardarSimulacionCommand
    Activity->>Controller: guardar(command, false)
    Controller->>Controller: Valida usuario, nombres, importes y detalles
    alt Datos inválidos
        Controller-->>Activity: Rechazada(mensaje)
        Activity-->>Usuario: Muestra validación
    else Datos válidos
        Controller->>Calc: calcular(saldo, detalles)
        Calc-->>Controller: egresos, meses y excedente
        Controller->>Repo: Busca cuenta y simulación existentes
        Repo->>DB: Consulta identificadores y detalles
        alt Hay detalles previos y falta confirmar
            Controller-->>Activity: RequiereConfirmacionSobrescritura
            Activity-->>Usuario: Solicita confirmación
            alt Usuario cancela
                Activity-->>Usuario: Conserva datos en pantalla
            else Usuario confirma
                Activity->>Controller: guardar(command, true)
                Controller->>Repo: guardarCompleta(...)
            end
        else Simulación nueva o sin detalles
            Controller->>Repo: guardarCompleta(...)
        end
        opt Se autorizó el guardado y se invocó guardarCompleta
            Repo->>DB: BEGIN TRANSACTION
            Repo->>DB: Inserta o actualiza Simulaciones
            Repo->>DB: Elimina detalles previos de la simulación
            Repo->>DB: Inserta los detalles actuales
            alt Escrituras completas
                Repo->>DB: COMMIT
                Repo-->>Controller: id de simulación
                Controller-->>Activity: Guardada(id)
                Activity-->>Usuario: Confirma guardado
            else Error SQLite
                Repo->>DB: ROLLBACK al cerrar la transacción
                Controller-->>Activity: Error sin cambios parciales
                Activity-->>Usuario: Informa el error
            end
        end
    end
```

## 7. Esquema relacional

```mermaid
erDiagram
    Usuarios ||--o{ Cuentas : posee
    Usuarios ||--o{ Items : posee
    Usuarios ||--o{ Simulaciones : crea
    Cuentas ||--o{ Simulaciones : financia
    Simulaciones ||--o{ DetallesSimulaciones : contiene
    Items o|--o{ DetallesSimulaciones : referencia

    Usuarios {
        INTEGER idUsuario PK
        TEXT nombreUsuario UK
        TEXT claveUsuario
        TEXT saltUsuario
        TEXT fechaHoraUsuario
        INTEGER esAdmin
        TEXT preferenciaTema
    }
    Cuentas {
        INTEGER idCuenta PK
        INTEGER idUsuario FK
        TEXT nombreCuenta
        INTEGER totalCuenta
        TEXT fechaHoraCuenta
    }
    Items {
        INTEGER idItem PK
        INTEGER idUsuario FK
        TEXT nombreItem
        INTEGER valorItem
    }
    Simulaciones {
        INTEGER idSim PK
        INTEGER idCuenta FK
        INTEGER idUsuario FK
        TEXT nombreSim
        TEXT fechaHoraSim
        INTEGER totalDisponible
        INTEGER totalEgresos
        INTEGER meses
        INTEGER excedente
    }
    DetallesSimulaciones {
        INTEGER idDetalle PK
        INTEGER idSim FK
        INTEGER idItem FK "nullable"
        TEXT nombreItem
        INTEGER valorItem
    }
```

`idItem` es nullable en `DetallesSimulaciones`; al borrar un ítem relacionado, la clave puede quedar nula y el detalle mantiene nombre y monto. Los nombres de cuenta e ítem son únicos por usuario, y el nombre de simulación también es único por usuario.

## 8. Otros procesos

### Consulta de simulaciones

1. Carga las simulaciones del usuario activo.
2. Al seleccionar una, presenta cuenta, fecha, saldos, egresos, meses, excedente y detalles.
3. JSON y CSV solicitan una ruta antes de escribir el archivo.
4. PDF genera un documento y solicita abrirlo con una aplicación externa.
5. La acción **Enviar** solo muestra un aviso; no transmite información a un servicio externo.

### Respaldo y restauración

1. El respaldo normal copia `simulafin.db` a `simulafin_backup.db` en Descargas o a una ubicación elegida mediante SAF.
2. Restaurar copia un archivo seleccionado sobre la base activa; también se pueden cargar datos de prueba.
3. Vaciar la base pide autorización admin y la recrea preservando al usuario activo como administrador.
4. **Respaldo limpio** vacía primero la base activa y luego copia esa base vacía; no conserva los datos que había antes.

## 9. Componentes declarados en Android

`LoginActivity` es la Activity launcher. El manifiesto declara las demás Activities como no exportadas y un `FileProvider` para compartir/abrir PDF con permiso de lectura. No se incluyen servicios ni backend remoto porque no aparecen en el manifiesto analizado.
