# Lab-Solid
Repositorio para el Laboratorio de SOLID de la materia Ingeniería de Software II G3 del Departamento de Ingeniería de Sistemas e Industrial - Facultad de Ingeniería de la Universidad Nacional de Colombia.

**Lenguaje elegido:** Python (traducción 1:1 del código base en Java, conservando los defectos de diseño).

## Miembros del Equipo de trabajo

* Pablo Andres Niño Barreto (pninob@unal.edu.co)
* Sergio Tovar Vasquez (setovarv@unal.edu.co)


## Bloque 0 - Commit Inicial - Traducción de Java a Python 
Se subió el bloque 0 del laboratorio, que incluye la traducción de los códigos del laboratorio, originalmente en Java y traducidos a Python. La salida del programa principal quedó congelada en `salida_original.txt` como prueba de caracterización.

---

## Bloque 1 — Diagnóstico

Objetivo: **encontrar los problemas de diseño y medir el "antes"**, sin corregir nada todavía. Todas las evidencias, salidas y métricas de esta sección se obtuvieron ejecutando el código real de este repositorio.

### 1.1 Tabla de hallazgos

Hay al menos un problema por cada letra de SOLID; algunas clases acumulan varios. La columna de **consecuencia** está escrita en términos del negocio (qué le pasa al banco o al cliente).

| Clase / método | Letra | Evidencia en el código | Consecuencia para el banco o el cliente |
|---|:---:|---|---|
| `TransaccionService.transferir` (`transaccion_service.py`, líneas 11–47) | **S** | Un mismo método valida montos, calcula la comisión, mueve el dinero, persiste en Oracle, imprime el comprobante, envía el SMS y registra la auditoría (7 bloques comentados `# 1.` a `# 7.`). | Si el área legal pide cambiar el **texto del comprobante**, hay que tocar el mismo método que **mueve el dinero**. Un error al editar el comprobante puede terminar cobrando mal una transferencia o duplicando un cargo en producción. |
| `TransaccionService.transferir` (`transaccion_service.py`, líneas 19–26) | **O** | La comisión se decide con un `if/elif/else` sobre `tipo`: `MISMO_BANCO`, `OTRO_BANCO`, `INTERNACIONAL`. | Cada vez que el negocio lanza un **tipo nuevo de transferencia** (por llave, por ejemplo) hay que **abrir y modificar la clase central** que mueve el dinero. Cada cambio arriesga romper los tipos que ya funcionaban. |
| `CDT` hereda de `Cuenta` y sobreescribe `retirar` (`cdt.py`, líneas 9–11) | **L** | `CDT` extiende `Cuenta` pero **sobreescribe `retirar()` para lanzar `RuntimeError`** si aún no ha vencido. No cumple el contrato de su clase padre. | Cualquier proceso que trate un `CDT` como una `Cuenta` y llame `retirar()` **se cae**. El cobro masivo de cuota de manejo revienta al llegar al primer CDT (ver Experimento 1, con traceback real). |
| `CobroCuotaManejo.cobrar_mensual` (`cobro_cuota_manejo.py`, líneas 7–10) | **L** | Recorre una lista de `Cuenta` y llama `retirar()` asumiendo que **todas** permiten retirar, lo cual es falso para `CDT`. | En el batch nocturno de un millón de cuentas, basta **un** CDT en la lista para detener todo el proceso a mitad de camino: unas cuentas quedan cobradas y otras no. |
| `ProductoBancario` (`producto_bancario.py`, líneas 3–21) | **I** | Interfaz "gorda" (`ABC`) con 5 métodos abstractos obligatorios: `depositar`, `retirar`, `calcular_intereses`, `pagar_cuota`, `generar_extracto`. | Los implementadores quedan obligados a definir métodos que no les sirven. Peor: un `CreditoVivienda.depositar()` **vacío** acepta la llamada y **no hace nada** — el dinero "entra" sin error y desaparece silenciosamente. |
| `TransaccionService.__init__` (`transaccion_service.py`, líneas 7–9) | **D** | Crea sus dependencias directamente: `self._repositorio = OracleRepositorio()` y `self._sms = SmsGateway()`. | Es **imposible probar** una transferencia sin conectarse a la base de producción ni enviar un SMS real al cliente (ver Experimento 2). Migrar de Oracle a otro motor obliga a editar esta clase. |

**Hallazgos adicionales (clases con evidencia de apoyo):**

| Clase / método | Letra | Evidencia en el código | Consecuencia para el banco o el cliente |
|---|:---:|---|---|
| `Cuenta.retirar` (`cuenta.py`, líneas 21–24) | **L** | La clase base promete `retirar()` incondicional para **toda** cuenta; de ahí nace la violación del CDT. | El contrato mal diseñado en la raíz es lo que permite que un subtipo (CDT) no pueda cumplirlo. |
| `TarjetaCredito.depositar` (`tarjeta_credito.py`, línea 9) | **I** | Método vacío: `pass  # No aplica`. | Una tarjeta de crédito "acepta" un depósito que no hace nada: comportamiento engañoso para quien consume la interfaz. |
| `CreditoVivienda.depositar` y `CreditoVivienda.retirar` (`credito_vivienda.py`, líneas 7–11) | **I** | Dos métodos vacíos: `pass  # No aplica`. | Mismo riesgo de operación silenciosa que no falla pero tampoco ejecuta nada. |

### 1.2 Dos experimentos

#### Experimento 1 — El CDT

**Qué hicimos:** agregamos el CDT de Ana a la lista de `CobroCuotaManejo.cobrar_mensual`. El script está en `experimentos/experimento1_cdt.py`.

> **Nota de fidelidad:** en `main.py` el CDT se crea con `vencimiento=date(2026, 9, 30)`, una fecha **ya vencida** hoy; eso haría que el CDT se comporte como "ya liberado" y el experimento **no** fallaría. El Java original usa `LocalDate.now().plusMonths(6)` (siempre a futuro). Para que el experimento sea fiel al enunciado, el script usa un CDT **no vencido** (`date.today() + 180 días`). *Recomendación:* corregir esa fecha en `main.py` para que la traducción sea fiel (ver nota al final).

**Qué pasa (traceback real):**

```
>>> Lista a cobrar: [ana, cdt_ana, luis]
Cuota de manejo cobrada a 001-1
Traceback (most recent call last):
  File "experimentos/experimento1_cdt.py", line 12, in <module>
    CobroCuotaManejo().cobrar_mensual([ana, cdt_ana, luis])
  File "cobro_cuota_manejo.py", line 9, in cobrar_mensual
    cuenta.retirar(self.CUOTA)
  File "cdt.py", line 11, in retirar
    raise RuntimeError("Un CDT no permite retiros antes del vencimiento")
RuntimeError: Un CDT no permite retiros antes del vencimiento
```

Se cobra la cuota a `001-1` (Ana), el proceso llega al CDT, invoca `retirar()`, el CDT lanza `RuntimeError` y el programa **se detiene**: **`001-2` (Luis) nunca recibe el cobro**. El recorrido muere en el elemento problemático y no continúa.

**Qué pasaría en producción:** si el batch corre de noche sobre **un millón de cuentas** y la cuenta número **500 000 es un CDT**, el proceso cobra correctamente las primeras 499 999, explota en la 500 000 y **deja sin cobrar las 500 000 restantes**. Resultado: cierre contable inconsistente (medio banco cobrado, medio no) y una falla que probablemente nadie note hasta la conciliación. Un único dato "raro" tumba todo el proceso.



#### Experimento 2 — La prueba imposible

**Qué intentamos:** escribir una prueba que verifique que una transferencia a otro banco cobra $7 500 de comisión, **sin conectarse a Oracle ni enviar SMS**. El script está en `experimentos/test_prueba_imposible.py`.

**Qué pasó al ejecutarla (salida real de `pytest -s`):**

```
[ORACLE] Conectando a jdbc:oracle:thin:@prod-db:1521/BANCO...
[ORACLE] INSERT INTO transacciones VALUES ('001-1', '001-2', 100000, 7500.0)
===== BANCO ANDINO - COMPROBANTE =====
Origen: 001-1
Destino: 001-2
Monto: $100000
Comisión: $7500.0
=====================================
[SMS] Conectando al proveedor de mensajeria...
[SMS] Para Ana: Transferiste $100000 a la cuenta 001-2
[AUDITORIA] 2026-10-01 ... OTRO_BANCO 001-1->001-2 $100000
.
1 passed in 0.01s
```

**¿Lo logramos?** **No.** El `assert` sobre la comisión sí pasa, pero la prueba **no pudo evitar** que se ejecutaran el repositorio y el SMS: en la salida aparecen `[ORACLE] Conectando a ... prod-db ...` y `[SMS] Conectando al proveedor ...`.

**¿Qué lo impide?** `TransaccionService.__init__` **crea sus dependencias adentro** (`OracleRepositorio()` y `SmsGateway()`). No hay ningún punto (parámetro ni setter) por donde inyectar una versión falsa. Por eso, cada corrida de la prueba:

- Golpea la "base de producción" (`[ORACLE] Conectando a jdbc:oracle:thin:@prod-db...`).
- Envía un "SMS real" al cliente (`[SMS] Para Ana...`).

Capturar la consola no desacopla nada: seguimos conectando a Oracle y mandando SMS en cada corrida. Este es el síntoma exacto del problema **D (DIP)**, que se resuelve en el Punto de control D del Bloque 2.

### 1.3 Medición "antes"

| Métrica | Antes |
|---|:---:|
| Líneas del método `transferir` | **23 líneas de código** (cuerpo de 36 líneas: 23 de código + 7 comentarios + 6 en blanco) |
| Número de razones distintas por las que `TransaccionService` podría cambiar | **7** (validación · comisión · movimiento de dinero · persistencia · comprobante · notificación · auditoría) |
| Clases concretas que `TransaccionService` instancia directamente (`new`) | **2** (`OracleRepositorio`, `SmsGateway` — líneas 8–9) |
| Métodos vacíos o que lanzan excepción por "no aplica" | **3 vacíos** (`TarjetaCredito.depositar`, `CreditoVivienda.depositar`, `CreditoVivienda.retirar`) **+ 1 que lanza excepción** (`CDT.retirar`) |
| ¿Se puede probar `transferir` sin Oracle ni SMS? | **No** (demostrado en el Experimento 2) |


## Bloque 2 — Refactorización

En este bloque se refactoriza el sistema aplicando los principios SOLID de forma progresiva, manteniendo el comportamiento original del programa.

Después de cada punto de control se compara la salida del programa con `salida_original.txt` para verificar que la refactorización no haya alterado su comportamiento.

### Punto de Control S — Single Responsibility Principle

En el código original, el método `transferir()` de `TransaccionService` concentraba diferentes responsabilidades dentro de una misma clase:

1. Validación del monto.
2. Cálculo de la comisión.
3. Movimiento del dinero entre las cuentas.
4. Persistencia de la transacción.
5. Generación del comprobante.
6. Notificación mediante SMS.
7. Registro de auditoría.

Esta concentración hacía que `TransaccionService` tuviera múltiples razones para cambiar. Por ejemplo, un cambio en el formato del comprobante o en la forma de realizar la auditoría obligaría a modificar la misma clase encargada de coordinar la transferencia.

Para aplicar el principio de Responsabilidad Única (SRP), se separaron varias de estas responsabilidades en clases independientes:

- `ValidadorTransferencia`: se encarga de validar el monto de la transferencia.
- `CalculadorComision`: se encarga de calcular la comisión según el tipo de transferencia.
- `GeneradorComprobante`: se encarga de generar e imprimir el comprobante.
- `AuditorTransferencia`: se encarga de registrar la auditoría de la operación.

De esta forma, `TransaccionService` pasó a encargarse principalmente de coordinar el flujo de una transferencia utilizando los componentes especializados.

#### Estructura después de aplicar SRP

```text
TransaccionService
├── ValidadorTransferencia
├── CalculadorComision
├── GeneradorComprobante
├── AuditorTransferencia
├── OracleRepositorio
└── SmsGateway
```

### Punto de Control O — Open/Closed Principle

En el código original, el cálculo de la comisión dependía de una
estructura condicional que verificaba el tipo de transferencia.
Para agregar un nuevo tipo era necesario modificar el código
existente de `CalculadorComision`.

Para aplicar el principio Abierto/Cerrado (OCP), se creó la
abstracción `TipoTransferencia`, que define el comportamiento que
debe tener cada tipo de transferencia.

Se implementaron las siguientes clases:

- `TransferenciaMismoBanco`
- `TransferenciaOtroBanco`
- `TransferenciaInternacional`

Cada una implementa su propia forma de calcular la comisión.

`CalculadorComision` ahora depende de la abstracción
`TipoTransferencia` y simplemente delega en ella el cálculo:

```python
def calcular(self, monto: float, tipo: TipoTransferencia) -> float:
    return tipo.calcular_comision(monto)
```

### Punto de Control L — Sustitución de la jerarquía de cuentas

#### Problema encontrado

La clase `Cuenta` original definía la operación `retirar()`, por lo que todas sus subclases debían cumplir con este comportamiento.

Esto generaba un problema con `CDT`, ya que un CDT no permite retiros antes de su fecha de vencimiento. Por lo tanto, `CDT` no puede cumplir correctamente el contrato de una cuenta que permite retirar dinero en cualquier momento.

El problema corresponde al principio de **Liskov Substitution Principle (LSP)**: una subclase debe poder utilizarse donde se espera su clase base sin romper las expectativas del programa.

#### Refactorización realizada

Se separó la capacidad de retirar dinero de la clase general `Cuenta`.

Se creó:

```text
Cuenta
   ├── CuentaRetirable
   │      └── CuentaAhorros
   │
   └── CDT
```
### Punto de Control I — Interface Segregation Principle (ISP)

#### Problema encontrado

La interfaz `ProductoBancario` original obligaba a todos los productos a implementar los siguientes métodos:

- `depositar()`
- `retirar()`
- `calcular_intereses()`
- `pagar_cuota()`
- `generar_extracto()`

Esto generaba métodos que no aplicaban a determinados productos. Por ejemplo, `TarjetaCredito` tenía que implementar `depositar()` aunque esta operación no correspondía a una tarjeta de crédito. De igual manera, `CreditoVivienda` tenía que implementar `depositar()` y `retirar()` aunque estas operaciones no aplicaban a este producto.

#### Refactorización realizada

Se redujo la interfaz `ProductoBancario` para que únicamente defina la operación común a todos los productos:

```python
class ProductoBancario(ABC):
    @abstractmethod
    def generar_extracto(self) -> str:
        pass
```
### Punto de Control D — Dependency Inversion Principle (DIP)

#### Problema encontrado

Inicialmente, `TransaccionService` creaba directamente sus dependencias de infraestructura:

- `OracleRepositorio`
- `SmsGateway`

Esto generaba un acoplamiento entre la lógica de negocio y las implementaciones concretas de persistencia y mensajería.

#### Refactorización realizada:

Se crearon las abstracciones:

- `RepositorioTransacciones`
- `Notificador`

`OracleRepositorio` ahora implementa `RepositorioTransacciones`, mientras que `SmsGateway` implementa `Notificador`.

Posteriormente, `TransaccionService` fue modificado para recibir estas dependencias mediante su constructor.

De esta forma, el servicio ya no decide qué implementación concreta utilizar.

#### Inyección de dependencias

El armado de las dependencias se trasladó a `main.py`.

El programa principal es ahora responsable de crear:

- `OracleRepositorio`
- `SmsGateway`
- `ValidadorTransferencia`
- `CalculadorComision`
- `GeneradorComprobante`
- `AuditorTransferencia`

y posteriormente inyectarlos en `TransaccionService`.

Esto permite cambiar las implementaciones sin modificar la lógica del servicio.

#### Prueba con dobles

Para comprobar que la inversión de dependencias funciona, se creó:

`experimentos/experimento2_dip.py`

En este experimento se utilizaron:

- `FakeRepositorio`
- `FakeNotificador`

en lugar de `OracleRepositorio` y `SmsGateway`.

El resultado fue:

- Saldo de Ana: `$1.850.000`
- Saldo de Luis: `$650.000`
- Transacciones guardadas: `1`
- Notificaciones enviadas: `1`

Además, durante esta prueba no se realizaron conexiones al repositorio Oracle ni al proveedor de SMS.

Esto demuestra que `TransaccionService` puede probarse utilizando dobles de prueba sin depender de las implementaciones concretas de infraestructura.

## Bloque 3 — Pruebas unitarias

### Pruebas implementadas

Se utilizó `pytest` junto con dobles de prueba (`FakeRepositorio` y `FakeNotificador`) para probar `TransaccionService` sin conectarse a Oracle ni enviar SMS.

Se implementaron las cinco pruebas solicitadas:

1. **Transferencia al mismo banco:** verifica que no se cobre comisión y que el monto transferido se descuente y deposite correctamente.
2. **Transferencia a otro banco:** verifica una comisión de `$7.500` y que se descuente del origen el monto más la comisión.
3. **Saldo insuficiente:** verifica que la operación sea rechazada, que los saldos no cambien y que no se guarde ni notifique la transferencia.
4. **Transferencia exitosa:** verifica que la transacción se guarde exactamente una vez y que se genere exactamente una notificación.
5. **Tipo de transferencia desconocido:** verifica que la operación sea rechazada y que el saldo de origen permanezca sin cambios.

### Resultado

Las cinco pruebas fueron ejecutadas mediante `pytest`:

![Resultado de las 5 pruebas](tests/SS_5_tests.jpg)
## Documentación de ejecución

### Requisitos

El proyecto utiliza **Python 3** y `pytest` para las pruebas unitarias. No se requiere una base de datos PostgreSQL real ni un proveedor real de SMS/PUSH: las integraciones de persistencia y notificación están simuladas mediante mensajes en consola.

### Ejecutar el programa principal

Desde la raíz del proyecto se ejecuta:

```powershell
python main.py
```

La ejecución demuestra los escenarios principales del Bloque 4:

- **R1:** transferencia por llave de `$50.000` con comisión `$0`.
- **R2:** transferencia desde una cuenta infantil y rechazo de un segundo retiro que supera el límite diario de `$200.000`.
- **R3:** una transferencia exitosa genera mensajes `[SMS]` y `[PUSH]`.
- **R4:** una transferencia exitosa genera `[AUDITORIA]` y `[ANTIFRAUDE]`.
- **R5:** la persistencia se simula mediante mensajes `[POSTGRES]`.

En la ejecución realizada, R1 descontó exactamente `$50.000`; la cuenta infantil realizó un primer retiro de `$120.000` y rechazó un segundo retiro de `$100.000`, manteniendo el saldo en `$880.000`. Las transferencias exitosas mostraron `[SMS]`, `[PUSH]`, `[AUDITORIA]`, `[ANTIFRAUDE]` y `[POSTGRES]`.

### Ejecutar las pruebas unitarias

Las pruebas oficiales del Bloque 3 se ejecutan con:

```powershell
python -m pytest tests -v
```

El resultado esperado y obtenido es **5 pruebas aprobadas**. Las pruebas utilizan dobles de prueba para el repositorio y el notificador, por lo que no necesitan Oracle ni un servicio de SMS real.

> **Nota sobre `experimentos/test_prueba_imposible.py`:** si se ejecuta `python -m pytest` sin indicar la carpeta `tests`, pytest también descubre ese archivo experimental. Su objetivo es documentar la prueba imposible del Bloque 1 y utiliza la antigua construcción de `TransaccionService()` sin inyección de dependencias; por ello no forma parte de las cinco pruebas oficiales del Bloque 3.

### Evidencias de ejecución

La evidencia de las cinco pruebas unitarias se encuentra en `tests/SS_5_tests.jpg`. La salida de `main.py` permite comprobar visualmente los criterios de aceptación de R1 a R5.

## Respuestas a las preguntas del laboratorio

### Control S — Single Responsibility Principle

**Pregunta:** Después del cambio, describan en una frase qué hace `TransaccionService`. ¿Aparece la palabra “y”? Si el área legal pide cambiar el formato del comprobante, ¿qué archivo tocan?

**Respuesta:** `TransaccionService` coordina el flujo de una transferencia: valida, calcula la comisión, mueve el dinero y delega la persistencia, el comprobante, las notificaciones y la auditoría en componentes especializados. La palabra “y” puede aparecer al describir el flujo, pero ya no representa múltiples responsabilidades implementadas directamente en el mismo método. Si el área legal cambia el formato del comprobante, se modifica `generador_comprobante.py`, sin tocar la lógica central de la transferencia.

### Control O — Open/Closed Principle

**Pregunta:** Si mañana llega un tipo de transferencia nuevo, ¿qué archivos existentes tendrían que modificar? Enumérenlos. Lo ideal es que solo aparezca el punto donde se arma el sistema.

**Respuesta:** Se crea una nueva clase que implemente `TipoTransferencia`, siguiendo el mismo esquema de `TransferenciaMismoBanco`, `TransferenciaOtroBanco`, `TransferenciaInternacional` y `TransferenciaLlave`. La lógica de `CalculadorComision` y `TransaccionService` no necesita modificarse. Para utilizar el nuevo tipo en una demostración concreta, se modifica el punto de composición (`main.py`) donde se arma el sistema.

### Control L — Sustitución de Liskov

**Pregunta:** ¿Su solución detecta el error al compilar (o con el verificador de tipos) o al ejecutar? ¿Por qué es mejor lo primero? Si alguien propone “envolver el retiro en un `try/catch` e ignorar los CDT”, ¿por qué eso no resuelve el problema de diseño?

**Respuesta:** En Python, la incompatibilidad del contrato se manifiesta al ejecutar: el problema original aparece cuando un `CDT` recibe una llamada a `retirar()` y lanza una excepción. La solución refactorizada separa las cuentas retirables mediante `CuentaRetirable`, por lo que `CobroCuotaManejo` trabaja con una abstracción que garantiza la capacidad necesaria. Detectar un problema antes de ejecutar es preferible porque evita que una operación inválida llegue a producción. Envolver el retiro en `try/catch` (o `try/except`) y simplemente ignorar los CDT no corrige el contrato: solo oculta el error y deja un proceso parcialmente ejecutado.

### Control I — Interface Segregation Principle

**Pregunta:** ¿Pudieron lograr que un mismo generador de extractos funcione para cuentas, tarjetas y créditos a la vez? ¿Qué interfaz necesitó para eso, y por qué no necesitó conocer los demás métodos de cada producto?

**Respuesta:** Sí. Se separaron las capacidades de `ProductoBancario` en interfaces pequeñas, entre ellas `ProductoDepositable`, `ProductoRetirable`, `ProductoConIntereses` y `ProductoConCuota`. `GeneradorExtractos` utiliza únicamente la capacidad que necesita para generar el extracto. De esta manera no necesita conocer ni depender de métodos de otras capacidades que no correspondan al producto.

### Control D — Dependency Inversion Principle

**Pregunta:** ¿Cuántas clases concretas conoce ahora `TransaccionService`? ¿Quién decide si se usa Oracle o si se notifica por SMS? Vuelvan al experimento 2 del bloque 1: ¿ya es posible esa prueba?

**Respuesta:** `TransaccionService` depende de abstracciones como `RepositorioTransacciones` y `Notificador`, además de los componentes especializados que recibe por inyección. Ya no crea directamente `OracleRepositorio` ni `SmsGateway`. La decisión de usar PostgreSQL, Oracle, SMS o PUSH se realiza en `main.py`, que es el punto de composición. Sí es posible realizar la prueba sin Oracle ni SMS: las cinco pruebas del Bloque 3 utilizan `FakeRepositorio` y `FakeNotificador` y pasan correctamente.

### Bloque 3 — Pruebas unitarias

**Pregunta:** ¿Cuánto tardan en ejecutarse todas sus pruebas? ¿Cuántas líneas de `TransaccionService` tuvieron que cambiar para poder probarla? ¿Qué habría pasado si intentaran estas mismas pruebas en el bloque 1?

**Respuesta:** Las cinco pruebas oficiales se ejecutan en milisegundos; en la ejecución realizada se obtuvieron **5 pruebas aprobadas**. El cambio fundamental para hacer testeable `TransaccionService` fue reemplazar la creación interna de dependencias concretas por dependencias inyectadas mediante abstracciones. En el Bloque 1 las pruebas no podían aislarse correctamente: `TransaccionService` creaba `OracleRepositorio` y `SmsGateway` internamente, por lo que una prueba terminaba ejecutando esas integraciones en lugar de trabajar con dobles.

## Bloque 4 — Nuevos requerimientos de negocio

En este bloque se aplicaron cinco requerimientos de negocio sobre el código ya refactorizado. El objetivo era comprobar que, gracias al diseño SOLID, cada cambio se resuelve agregando clases nuevas y afectando la menor cantidad posible de código existente, sin volver a tocar la lógica central de `TransaccionService`.

A diferencia del Bloque 2, aquí la salida del programa **sí cambia respecto a `salida_original.txt`**, porque los requerimientos agregan comportamiento nuevo (notificación push, reporte antifraude, persistencia en PostgreSQL). Lo que no cambia son las pruebas unitarias del Bloque 3, que siguen pasando sin modificación alguna (criterio de aceptación de R5).

### R1 — Transferencias por llave

Se agregó la clase `TransferenciaLlave`, que implementa la abstracción `TipoTransferencia` con comisión de `$0`. No se modificó ninguna clase existente: se aprovechó el patrón Estrategia introducido en el Punto de control O del Bloque 2. La búsqueda de la cuenta a partir de la llave no se implementa, según lo indicado en el enunciado.

- Clases nuevas: `transferencia_llave.py`
- Clases modificadas: ninguna (solo el armado de la demostración en `main.py`)

Criterio de aceptación verificado: una transferencia de tipo `LLAVE` por `$50.000` descuenta exactamente `$50.000` de la cuenta de origen (comisión `$0`).

### R2 — Cuenta infantil

Se agregó la clase `CuentaInfantil`, que hereda de `CuentaRetirable`. Recibe depósitos sin límite y limita los retiros a `$200.000` en un mismo día, llevando un acumulado diario que se reinicia al cambiar la fecha. Al heredar de `CuentaRetirable` puede usarse como origen de transferencias y se le cobra la cuota de manejo como a cualquier cuenta retirable.

- Clases nuevas: `cuenta_infantil.py`
- Clases modificadas: ninguna

Criterio de aceptación verificado: si la cuenta ya retiró `$150.000` hoy, un retiro de `$60.000` se rechaza (`$150.000 + $60.000 > $200.000`) y el saldo no cambia.

### R3 — Notificaciones push

Se agregó `PushNotifier`, que implementa la misma abstracción `Notificador` que `SmsGateway`, y `NotificadorCompuesto`, que aplica el patrón Composite para agrupar varios notificadores y tratarlos como uno solo. En `main.py` se inyecta `NotificadorCompuesto([SmsGateway(), PushNotifier()])`. Gracias a esto, **`TransaccionService` no cambia**: sigue recibiendo un único `Notificador`.

- Clases nuevas: `push_notifier.py`, `notificador_compuesto.py`
- Clases modificadas: ninguna (solo el armado en `main.py`)

Criterio de aceptación verificado: por cada transferencia exitosa aparecen en consola un mensaje `[SMS]` y un mensaje `[PUSH]`.

### R4 — Sistema antifraude

Se agregó la abstracción `ObservadorTransaccion` y la clase `AntifraudeService`, que la implementa y reporta cada transacción exitosa (mensaje `[ANTIFRAUDE]`). `TransaccionService` recibe una lista opcional de observadores y los invoca al final del flujo, después de la auditoría, solo cuando la transferencia fue exitosa. El parámetro es opcional y por defecto una lista vacía, de modo que la auditoría actual se mantiene y las pruebas del Bloque 3 no cambian.

- Clases nuevas: `observador_transaccion.py`, `antifraude_service.py`
- Clases modificadas: `transaccion_service.py` (se añadió un punto de extensión por observadores, sin alterar el cálculo central)

Criterio de aceptación verificado: por cada transferencia exitosa aparecen un mensaje `[AUDITORIA]` y uno `[ANTIFRAUDE]`; una transferencia rechazada no genera ninguno de los dos, porque el servicio lanza la excepción antes de llegar a ese punto.

### R5 — Migración a PostgreSQL

Se agregó `PostgresRepositorio`, que implementa la misma abstracción `RepositorioTransacciones` que `OracleRepositorio`. La migración consistió únicamente en inyectar la nueva clase en lugar de la de Oracle desde `main.py`. La clase `OracleRepositorio` se conserva intacta por si hay que devolverse durante la migración.

- Clases nuevas: `postgres_repositorio.py`
- Clases modificadas: ninguna (`OracleRepositorio` no se toca; solo el armado en `main.py`)

Criterio de aceptación verificado: el programa guarda en PostgreSQL (mensaje `[POSTGRES]`) y las cinco pruebas unitarias del Bloque 3 siguen pasando sin cambios.

### Tabla: estimación vs. realidad de archivos modificados

La columna de estimación corresponde a cuántos archivos habría que tocar en el **código original** (monolítico) del Bloque 0, donde casi todo pasaba por la clase `TransaccionService`. La columna real corresponde a lo que efectivamente se hizo sobre el **código refactorizado**.

| Req | Estimado en el código original | Real en el código refactorizado | Clase núcleo modificada |
|---|:---:|---|:---:|
| R1 | 1 (modificar el `switch` de comisiones en `TransaccionService`) | 1 clase nueva, 0 clases modificadas | No |
| R2 | 1–2 (nueva clase + ajustes en la jerarquía de cuentas) | 1 clase nueva, 0 clases modificadas | No |
| R3 | 2 (modificar `TransaccionService` + nueva clase push) | 2 clases nuevas, 0 clases modificadas | No |
| R4 | 2 (modificar `TransaccionService` + nueva clase antifraude) | 2 clases nuevas, 1 clase modificada (punto de extensión) | Sí (aditivo) |
| R5 | 2 (modificar `TransaccionService`/`main` + nueva clase repositorio) | 1 clase nueva, 0 clases modificadas | No |

En el código original, cuatro de los cinco requerimientos (R1, R3, R4, R5) habrían obligado a modificar la misma clase central `TransaccionService`, con el riesgo de romper lo que ya funcionaba. En el código refactorizado, cuatro de los cinco se resolvieron **solo agregando clases nuevas**, sin tocar ninguna clase núcleo. El único cambio sobre `TransaccionService` (R4) fue aditivo: un parámetro opcional que no altera el cálculo central ni rompe las pruebas existentes. Los cambios restantes se concentraron en `main.py`, que es el punto de armado (composición) del sistema y donde es legítimo decidir qué implementaciones concretas se inyectan.

### Commits del bloque 4

`req-1`, `req-2`, `req-3`, `req-4`, `req-5`.

### Lo que más nos costo Entender o Extender

Lo que más nos costó entender fue seguir el flujo completo de una transferencia, porque la responsabilidad está distribuida entre varias abstracciones y servicios. También fue necesario revisar cómo se ensamblan las dependencias en `main.py` para extender el sistema sin modificar la lógica central. Aunque al principio requirió más lectura, esta separación facilitó agregar nuevos tipos de transferencia y repositorios.


## Bloque 6 - Cierre 

### Diagrama código inicial
```mermaid
classDiagram
    direction TB

    class Cuenta {
        <<class>>
        -numero
        -titular
        -saldo
        +depositar(monto)
        +retirar(monto)
    }

    class CuentaAhorros {
        <<class>>
    }

    class CDT {
        <<class>>
        -vencimiento
        +retirar(monto)
    }

    class ProductoBancario {
        <<interface>>
        +depositar(monto)
        +retirar(monto)
        +calcular_intereses()
        +pagar_cuota(monto)
        +generar_extracto()
    }

    class TarjetaCredito {
        <<class>>
        -cupo
        -deuda
        +depositar(monto)
        +retirar(monto)
        +calcular_intereses()
        +pagar_cuota(monto)
        +generar_extracto()
    }

    class CreditoVivienda {
        <<class>>
        -saldo_pendiente
        +depositar(monto)
        +retirar(monto)
        +calcular_intereses()
        +pagar_cuota(monto)
        +generar_extracto()
    }

    class TransaccionService {
        <<class>>
        -repositorio
        -sms
        +transferir(origen,destino,monto,tipo)
    }

    class OracleRepositorio {
        <<class>>
        +guardar_transaccion(origen,destino,monto,comision)
    }

    class SmsGateway {
        <<class>>
        +enviar(destinatario,mensaje)
    }

    class CobroCuotaManejo {
        <<class>>
        +cobrar_mensual(cuentas)
    }

    class Main {
        <<class>>
        +main()
    }

    %% Herencia
    Cuenta <|-- CuentaAhorros
    Cuenta <|-- CDT

    ProductoBancario <|.. TarjetaCredito
    ProductoBancario <|.. CreditoVivienda

    %% Dependencias problemáticas
    TransaccionService ..> OracleRepositorio : crea con new
    TransaccionService ..> SmsGateway : crea con new

    TransaccionService ..> Cuenta : utiliza
    CobroCuotaManejo ..> Cuenta : llama retirar()

    Main ..> TransaccionService : crea
    Main ..> CuentaAhorros : crea
    Main ..> CDT : crea
    Main ..> TarjetaCredito : crea
    Main ..> CreditoVivienda : crea
    Main ..> CobroCuotaManejo : crea
```


### Diagrama UML Código Final

```mermaid
classDiagram
    direction TB

    %% =========================
    %% CUENTAS
    %% =========================

    class ProductoBancario {
        <<abstract>>
        +generar_extracto()
    }

    class Cuenta {
        <<class>>
        -numero
        -titular
        -saldo
        +get_numero()
        +get_titular()
        +get_saldo()
        +depositar(monto)
        +generar_extracto()
    }

    class CuentaRetirable {
        <<class>>
        +retirar(monto)
    }

    class CuentaAhorros {
        <<class>>
    }

    class CuentaInfantil {
        <<class>>
        -LIMITE_DIARIO
        -retirado_hoy
        -fecha_retiro
        +retirar(monto)
    }

    class CDT {
        <<class>>
        -vencimiento
    }

    ProductoBancario <|-- Cuenta
    Cuenta <|-- CuentaRetirable
    CuentaRetirable <|-- CuentaAhorros
    CuentaRetirable <|-- CuentaInfantil
    Cuenta <|-- CDT

    %% =========================
    %% PRODUCTOS / ISP
    %% =========================

    class ProductoDepositable {
        <<interface>>
        +depositar(monto)
    }

    class ProductoRetirable {
        <<interface>>
        +retirar(monto)
    }

    class ProductoConIntereses {
        <<interface>>
        +calcular_intereses()
    }

    class ProductoConCuota {
        <<interface>>
        +pagar_cuota(monto)
    }

    class TarjetaCredito {
        <<class>>
        -cupo
        -deuda
        +retirar(monto)
        +calcular_intereses()
        +pagar_cuota(monto)
        +generar_extracto()
    }

    class CreditoVivienda {
        <<class>>
        -saldo_pendiente
        +calcular_intereses()
        +pagar_cuota(monto)
        +generar_extracto()
    }

    ProductoBancario <|-- TarjetaCredito
    ProductoRetirable <|.. TarjetaCredito
    ProductoConIntereses <|.. TarjetaCredito
    ProductoConCuota <|.. TarjetaCredito

    ProductoBancario <|-- CreditoVivienda
    ProductoConIntereses <|.. CreditoVivienda
    ProductoConCuota <|.. CreditoVivienda

    %% =========================
    %% TRANSFERENCIAS
    %% =========================

    class TipoTransferencia {
        <<interface>>
        +calcular_comision(monto)
        +nombre()
    }

    class TransferenciaMismoBanco {
        +calcular_comision(monto)
        +nombre()
    }

    class TransferenciaOtroBanco {
        +calcular_comision(monto)
        +nombre()
    }

    class TransferenciaInternacional {
        +calcular_comision(monto)
        +nombre()
    }

    class TransferenciaLlave {
        +calcular_comision(monto)
        +nombre()
    }

    TipoTransferencia <|.. TransferenciaMismoBanco
    TipoTransferencia <|.. TransferenciaOtroBanco
    TipoTransferencia <|.. TransferenciaInternacional
    TipoTransferencia <|.. TransferenciaLlave

    %% =========================
    %% PERSISTENCIA - DIP
    %% =========================

    class RepositorioTransacciones {
        <<interface>>
        +guardar_transaccion(origen,destino,monto,comision)
    }

    class OracleRepositorio {
        +guardar_transaccion(origen,destino,monto,comision)
    }

    class PostgresRepositorio {
        +guardar_transaccion(origen,destino,monto,comision)
    }

    RepositorioTransacciones <|.. OracleRepositorio
    RepositorioTransacciones <|.. PostgresRepositorio

    %% =========================
    %% NOTIFICACIONES
    %% =========================

    class Notificador {
        <<interface>>
        +enviar(destinatario,mensaje)
    }

    class SmsGateway {
        +enviar(destinatario,mensaje)
    }

    class PushNotifier {
        +enviar(destinatario,mensaje)
    }

    class NotificadorCompuesto {
        -notificadores
        +enviar(destinatario,mensaje)
    }

    Notificador <|.. SmsGateway
    Notificador <|.. PushNotifier
    Notificador <|.. NotificadorCompuesto

    NotificadorCompuesto o-- Notificador : contiene

    %% =========================
    %% ANTIFRAUDE / OBSERVADOR
    %% =========================

    class ObservadorTransaccion {
        <<interface>>
        +registrar_transaccion(origen,destino,monto,comision,tipo)
    }

    class AntifraudeService {
        +registrar_transaccion(origen,destino,monto,comision,tipo)
    }

    ObservadorTransaccion <|.. AntifraudeService

    %% =========================
    %% SERVICIOS
    %% =========================

    class ValidadorTransferencia {
        -TOPE_DIARIO
        +validar(monto)
    }

    class CalculadorComision {
        +calcular(monto,tipo)
    }

    class GeneradorComprobante {
        +generar(origen,destino,monto,comision)
    }

    class AuditorTransferencia {
        +registrar(tipo,origen,destino,monto)
    }

    class GeneradorExtractos {
        +generar(producto)
    }

    class CobroCuotaManejo {
        -CUOTA
        +cobrar_mensual(cuentas)
    }

    CalculadorComision ..> TipoTransferencia : utiliza
    GeneradorExtractos ..> ProductoBancario : utiliza
    CobroCuotaManejo ..> CuentaRetirable : utiliza

    %% =========================
    %% TRANSACCION SERVICE
    %% =========================

    class TransaccionService {
        -repositorio
        -notificador
        -validador
        -calculador_comision
        -comprobante
        -auditor
        -observadores
        +transferir(origen,destino,monto,tipo)
    }

    TransaccionService ..> RepositorioTransacciones : depende de
    TransaccionService ..> Notificador : depende de
    TransaccionService ..> ValidadorTransferencia : utiliza
    TransaccionService ..> CalculadorComision : utiliza
    TransaccionService ..> GeneradorComprobante : utiliza
    TransaccionService ..> AuditorTransferencia : utiliza
    TransaccionService ..> ObservadorTransaccion : utiliza
    TransaccionService ..> CuentaRetirable : origen
    TransaccionService ..> Cuenta : destino
    TransaccionService ..> TipoTransferencia : recibe

    %% =========================
    %% MAIN - COMPOSICIÓN
    %% =========================

    class Main {
        <<class>>
        +main()
    }

    Main ..> TransaccionService : ensambla
    Main ..> PostgresRepositorio : selecciona
    Main ..> NotificadorCompuesto : configura
    Main ..> SmsGateway : configura
    Main ..> PushNotifier : configura
    Main ..> AntifraudeService : configura

```
### Tabla comparativa

La siguiente tabla compara el estado del sistema antes de la refactorización SOLID
(Bloque 0/1) con el estado final después de los Bloques 2, 3 y 4.

| Métrica | Antes | Después |
|---|:---:|:---:|
| Líneas del método `transferir` | **23 líneas de código** | **35 líneas de código** |
| Razones distintas por las que `TransaccionService` podría cambiar | **7** | **1** |
| Clases concretas que `TransaccionService` crea con `new` | **2** (`OracleRepositorio`, `SmsGateway`) | **0** |
| Métodos vacíos o que lanzan "no aplica" | **4** (3 vacíos + 1 excepción) | **0** |
| ¿Se puede probar `transferir` sin Oracle ni SMS? | **No** | **Sí** |
| Número total de archivos | **11** | **34** |
| Archivos existentes modificados en total en el Bloque 4 | **—** | **2** |

#### Interpretación de los resultados

La comparación muestra que, aunque el código final contiene más archivos,
las responsabilidades están mejor distribuidas. `TransaccionService` dejó de
crear directamente sus dependencias concretas y ahora trabaja mediante
abstracciones inyectadas desde `main.py`.

El número de razones de cambio de `TransaccionService` se redujo de siete,
correspondientes a las diferentes responsabilidades que concentraba
originalmente, a una responsabilidad principal: **coordinar el flujo de una
transferencia**.

También desaparecieron los métodos vacíos o con comportamientos de
"no aplica", ya que las capacidades fueron separadas mediante abstracciones
más pequeñas. Esto permite que cada producto bancario implemente únicamente
las operaciones que realmente necesita.

Finalmente, el código final puede probar `transferir` utilizando dobles de
prueba (`FakeRepositorio` y `FakeNotificador`), sin depender de Oracle ni de
SMS. En el Bloque 4 se modificaron únicamente **dos archivos existentes**:
`main.py` y `transaccion_service.py`; el resto de los cambios se resolvieron
mediante nuevas clases.

### Respuestas a Preguntas de Reflexión

#### a) ¿El código final tiene muchos más archivos que el original? ¿Es eso un problema? ¿En qué situación sí lo sería?

**Respuesta:** No necesariamente. El código final tiene más archivos porque las responsabilidades y capacidades que antes estaban concentradas en pocas clases fueron separadas en componentes especializados. Esto mejora la mantenibilidad y permite extender el sistema con cambios más localizados. Sí sería un problema si la cantidad de clases aumentara sin representar responsabilidades reales, si las abstracciones no aportaran valor o si el sistema se volviera más difícil de entender y mantener.

#### b) ¿En qué requerimiento del Bloque 4 se notó más la diferencia entre el código original y el refactorizado? ¿Por qué?

**Respuesta:** La diferencia se nota especialmente en **R3 y R5**, porque ambos requerimientos permiten aprovechar directamente las abstracciones creadas durante el refactor. En R3 se agregó `PushNotifier` y `NotificadorCompuesto` sin modificar la lógica de transferencia. En R5 se agregó `PostgresRepositorio` como otra implementación de `RepositorioTransacciones` y se cambió únicamente la composición del sistema. En el diseño original, estos cambios habrían obligado a modificar la clase central `TransaccionService`, que conocía directamente las implementaciones concretas.

#### c) ¿Hubo algún requerimiento que su diseño no aguantó bien? ¿Qué cambiarían?

**Respuesta:** El requerimiento que más exigió una modificación de una clase existente fue **R4**, porque fue necesario agregar a `TransaccionService` el punto de extensión para los observadores de transacciones. Sin embargo, el cambio fue aditivo, opcional y no rompió las cinco pruebas existentes. Para reducir todavía más la responsabilidad del servicio, una posible mejora sería encapsular la publicación de eventos de transferencia exitosa en un componente dedicado, de manera que `TransaccionService` solo coordine el flujo y delegue completamente la notificación de eventos.

#### d) ¿Qué les dijo la otra pareja en la revisión cruzada? ¿Están de acuerdo?

**Respuesta:** La otra pareja nos dio una opinión bastante positiva sobre el resultado del
proyecto. Destacaron que la refactorización no consistió solamente en separar
archivos, sino que realmente se aplicaron los principios SOLID. En especial,
mencionaron la separación de responsabilidades de `TransaccionService`, el
uso de `TipoTransferencia`, la solución del problema del CDT mediante
`CuentaRetirable` y la facilidad para agregar los requerimientos del bloque 4
sin modificar demasiado la lógica existente.

También nos recomendaron fortalecer las pruebas automatizadas para algunos de
los nuevos requerimientos, como `CuentaInfantil`, `NotificadorCompuesto`,
`AntifraudeService` y `PostgresRepositorio`. Además, nos señalaron algunas
inconsistencias que deberíamos revisar en el README, principalmente en las
métricas y en la descripción de los archivos modificados durante el bloque 4.

Estamos de acuerdo con sus observaciones. Consideramos que la revisión fue
útil porque nos permitió ver que, aunque el diseño cumple con los objetivos del
laboratorio, todavía hay aspectos que podemos mejorar, especialmente en la
cobertura de pruebas y en mantener la documentación completamente alineada con
el estado final del código.

#### e) Si tuvieran que convencer a su jefe de invertir dos semanas en refactorizar el backend real del banco, ¿qué argumento usarían, basándose en los datos de hoy?

**Respuesta:** El argumento principal sería que la refactorización reduce el costo y el riesgo de los cambios futuros. En el Bloque 4 se implementaron cinco requerimientos y la mayoría se resolvieron agregando clases nuevas, sin modificar la lógica central de `TransaccionService`. Además, las cinco pruebas unitarias continuaron pasando y pudieron ejecutarse sin Oracle ni SMS. Esto permite cambiar infraestructura y agregar capacidades con menor impacto sobre el código que ya funciona y con una red de pruebas que detecta regresiones rápidamente.
