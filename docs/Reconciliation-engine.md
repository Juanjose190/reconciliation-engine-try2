# Evaluación: Motor de Reconciliación Formance

## Resumen

Diseña e implementa un motor de reconciliación multi-tenant que sincronice estados internos del ledger con una red blockchain simulada usando Formance Ledger. Este proyecto evalúa tu capacidad para construir sistemas financieros complejos y mantenibles usando NestJS, TypeORM y NextJS.

---

## Justificación

Esta evaluación está diseñada para evaluar:

- **Diseño de Sistemas:** Cómo estructuras un sistema de contabilidad de doble entrada multi-tenant.
- **Diseño de APIs:** Cómo diseñas interfaces RESTful limpias para datos complejos.
- **Resolución de Problemas:** Cómo manejas la consistencia asíncrona, discrepancias de reconciliación y flujos de error.
- **Comunicación:** Cómo articulas tus decisiones de diseño y compromisos en un documento escrito.

> Priorizamos código funcional y mantenible sobre optimización prematura. Queremos ver cómo escribes código que sea fácil de leer, probar y depurar.

---

## Pasos del Proyecto

### 1. Documento de Diseño

Antes de escribir código, entrega un **Documento de Diseño de 2-3 páginas** como Pull Request.

- **Contenido:** Enfoque de diseño, diagrama de arquitectura, esquema de BD, definición de API, máquina de estados de reconciliación.
- **Formato:** Markdown en la carpeta `docs/`.
- **Objetivo:** Obtener aprobación de tu arquitectura antes de la implementación.

### 2. Implementación

Divide tu trabajo en **3-5 Pull Requests**.

- No entregues un PR masivo al final.
- Fusiona los PRs tras recibir la aprobación del mentor (o auto-fusionar si trabajas solo/simulando revisión).
- **Stack tecnológico:**
  - **Backend:** NestJS, TypeORM, PostgreSQL.
  - **Frontend:** NextJS.
  - **Ledger:** Formance Ledger v2 (vía Docker o Cloud).

### 3. Revisión

Tu entrega será evaluada en:

- **Correctitud:** ¿El ledger está balanceado? ¿Se detectan las discrepancias?
- **Calidad de Código:** ¿Es el código DRY, legible y bien estructurado?
- **UX:** ¿Es el dashboard intuitivo para un operador que corrige una discrepancia?

---

## Requisitos del Sistema

El sistema debe construirse como una aplicación unificada de nivel producción que satisfaga todos los siguientes requisitos funcionales.

### 1. Ledger Principal y Contabilidad

- **Infraestructura:** Implementa un backend robusto usando NestJS, TypeORM y PostgreSQL, integrado con Formance Ledger v2.
- **Principios de Doble Entrada:** Aplica contabilidad de doble entrada estricta (Créditos = Débitos) en todas las transacciones.
- **Gestión de Saldos:**
  - Trata Formance Ledger como la única fuente de verdad para todos los saldos de cuentas.
  - Expón APIs para obtener saldos en tiempo real e historial de transacciones (`GET /balances/:account`).
- **Registro de Transacciones:**
  - Implementa `POST /transactions` para registrar transferencias internas (ej. Usuario A → Usuario B).
  - Asegura que todas las transacciones sean inmutables una vez confirmadas.
- **Plan de Cuentas:** Implementa un **Account Builder** estructurado para estandarizar las rutas de cuentas.
  - **Categorías:** Usa categorías de nivel superior: `asset`, `liability`, `revenue`, `expense`, `equity`.
  - **Estructura:** Define plantillas para los tipos de cuenta clave:
    - `asset:blockchain:{chain}:{token}` — Activos externos/blockchain
    - `liability:user:{userId}:{currency}` — Saldos de billeteras de usuarios
    - `revenue:fees:{feeType}:{currency}` — Ingresos de la plataforma
    - `expense:fees:{feeType}:{currency}` — Gastos de la plataforma
  - **Validación:** Asegura que todas las operaciones del ledger usen rutas generadas estrictamente por el builder para evitar "magic strings".
- **Multi-Tenancy:** Implementa multi-tenancy creando un nuevo ledger por cada empresa. Cada empresa tendrá múltiples usuarios. El sistema estará denominado en USD.

### 2. Simulación e Integración Blockchain

- **Servicio de Simulación:** Desarrolla un servicio de "Fake Blockchain" para simular movimientos externos de dinero de forma asíncrona:
  - **Depósitos:** Simula transferencias de billetera externa → billetera de usuario.
  - **Retiros:** Simula transferencias de billetera de usuario → billetera de comercio/externa.
  - **Latencia de Liquidación:** Simula demoras del mundo real (ej. ventana de liquidación de 4 horas).
- **Bucle de Reconciliación:**
  - Implementa un proceso en segundo plano que sondee periódicamente la "Fake Blockchain".
  - Refleja las transacciones confirmadas en cadena dentro del Formance Ledger.
  - Mantiene una "World Account" para contabilizar correctamente los activos que entran o salen del sistema.
- **Gestión de Estados:** Rastrea y persiste el ciclo de vida de las transacciones externas (`PENDING` → `CONFIRMED` / `FAILED`).

### 3. Reconciliación y Manejo de Errores

- **Detección de Discrepancias:** El sistema debe identificar automáticamente inconsistencias entre el Ledger interno y la "Fake Blockchain", incluyendo:
  - **Entradas Huérfanas:** Transacciones registradas internamente pero fallidas en cadena.
  - **Diferencias de Monto:** Diferencias entre los montos solicitados y los montos liquidados finales (ej. por comisiones de gas o fluctuaciones del tipo de cambio).
- **Simulación de Fallos:** Introduce caos controlado para probar la resiliencia:
  - Falla aleatoriamente ~10% de las transacciones externas tras la iniciación interna.
  - Simula montos de liquidación variables.
- **Flujo de Corrección:**
  - Provee una UI/API para "Registrar una Corrección" (crear una transacción compensatoria en Formance).
  - Mantiene un log de auditoría explicando la causa y resolución de cada discrepancia.

### 4. Reportes y Operaciones

- **Dashboard Operativo:**
  - Construye un frontend profesional (NextJS) usando una librería de componentes moderna (ej. Shadcn/UI, MUI).
  - Muestra "Alertas de Reconciliación" para discrepancias activas.
  - Permite a los operadores ver listas de cuentas, saldos en tiempo real y detalles de transacciones.
- **Reportes:** Genera "Reportes de Reconciliación" completos (JSON/CSV) que muestren saldos de apertura, movimientos y saldos de cierre para un período dado.
- **Atomicidad y Consistencia:** Asegura consistencia distribuida entre los metadatos de PostgreSQL y Formance Ledger, implementando patrones para manejar fallos parciales de forma elegante.
- **Documentación:** Provee instrucciones claras en el README para ejecutar la simulación, disparar errores y operar el dashboard.

---

## Guía Técnica

### La Blockchain "Falsa"

Dado que es una simulación, no necesitas una blockchain real. Crea un simple servicio en tabla o en memoria que:

1. Acepte una solicitud.
2. Retorne un `tx_id`.
3. Actualice el estado a `CONFIRMED` o `FAILED` después de un retraso configurable.
4. Exponga una API para consultar el estado de la transacción por `tx_id`.

### Integración con Formance

- Usa la API de Formance para todas las operaciones del ledger.
- **Versión:** Apunta a los endpoints de la API Ledger v2 (ej. `POST /v2/ledgers/{ledger}/transactions`).
- **Transacciones:** Puedes usar postings JSON estándar o Numscript para lógica compleja.
- No mantengas saldos en tu base de datos SQL; Formance es la fuente de verdad para saldos.
- Usa tu base de datos SQL (PostgreSQL) para almacenar metadatos: Usuarios, Tenants, Solicitudes de Transacciones y Estado de Reconciliación.

### Compromisos y "Atajos"

- **Seguridad:** Puedes omitir OAuth/JWT complejo. API keys simples o autenticación simulada están bien.
- **Pruebas:** Las pruebas automatizadas son opcionales. Si las omites, proporciona pasos de verificación manual en tus PRs.

---

## Plan de Implementación y Hitos

| Hito | Descripción |
|------|-------------|
| **Documento de Diseño** | Redactar y refinar el diseño del sistema; obtener aprobación antes de codificar. |
| **Hito 1** | Andamiar el proyecto NestJS, esquema de BD, migraciones TypeORM, módulo fake blockchain, sembrar tenants/empresas. |
| **Hito 2** | Implementar endpoints de ingesta de transacciones, servicio de ledger, cliente REST de Formance y lógica de posting de doble entrada. |
| **Hito 3** | Construir el servicio de reconciliación (scheduler, cálculos de deriva, persistencia, exportación de reportes, endpoint de envío). |
| **Hito 4** | Entregar la UI en NextJS (dashboards, páginas de detalle, correcciones manuales) e integrar con las APIs del backend. |
| **Hito 5** | Pulir, documentar pruebas manuales y ensamblar el resumen final. |
