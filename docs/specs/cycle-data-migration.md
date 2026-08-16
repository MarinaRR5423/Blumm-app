# Spec: Migración e importación de datos de ciclo

**Estado:** Borrador para discusión
**Autor:** Propuesta preparada con Claude Code, a partir de la petición de Marina
**Fecha:** 2026-08-15
**Repo:** Este documento vive en la web de marketing (`Blumm-app`) porque es donde se gestionó la conversación, pero la implementación real corresponde al repositorio de la app móvil.

---

## 1. Resumen

Permitir que una usuaria que llega a Blumm con historial de ciclo ya registrado en otra app (Flo, Clue, Natural Cycles, Apple Salud/Health, etc.) no tenga que volver a introducirlo a mano. Dos vías, priorizadas en este orden:

1. **Apple Health (HealthKit)** — lectura de los datos de ciclo que el sistema operativo ya centraliza.
2. **Importación manual desde otra app** — el usuario exporta un archivo (CSV/JSON) desde su app anterior y Blumm lo interpreta.

## 2. Problema

El registro de ciclo es el dato de mayor fricción para arrancar el uso de la app: si la usuaria lleva 1-2 años de historial en otra app, empezar "en blanco" en Blumm produce predicciones peores en las primeras semanas (duración de ciclo, ventana fértil, etc. se calculan mejor con histórico) y es una razón de abandono en el onboarding.

## 3. Objetivos

- Reducir la fricción de onboarding para usuarias que migran desde otra app de ciclo.
- Mejorar la precisión de las predicciones desde el día 1, aprovechando histórico ya existente.
- Ofrecer una vía de importación que no dependa de que cada competidor coopere (Apple Health como capa neutral).

**No-objetivos (fuera de alcance de esta spec):**
- Sincronización bidireccional continua con Apple Health (Blumm *escribiendo* en Health) — se deja como posible fase futura, ver §11.
- Integración con wearables de terceros (Oura, Whoop) más allá de lo que ya centraliza HealthKit.
- Migración de datos de nutrición/entrenamiento (fuera del alcance de "ciclo").

## 4. Usuarios y casos de uso

- **Caso A:** Usuaria nueva en onboarding, tiene Flo/Clue instalada y ha estado registrando ahí su ciclo.
- **Caso B:** Usuaria que ya registra su ciclo directamente en la app nativa de Salud de iPhone (algunas lo hacen sin usar ninguna app de terceros).
- **Caso C:** Usuaria Android — sin HealthKit; equivalente sería Health Connect, si Blumm tiene o planea versión Android.

## 5. Fuentes de datos

### 5.1 Apple HealthKit (iOS) — vía principal

HealthKit expone un dominio "Salud reproductiva" con tipos de dato estandarizados. Blumm pediría **permiso de solo lectura** sobre:

| Tipo HealthKit | Qué representa |
|---|---|
| `HKCategoryTypeIdentifierMenstrualFlow` | Día de sangrado + intensidad (`none`, `light`, `medium`, `heavy`, `unspecified`) |
| `HKCategoryTypeIdentifierIntermenstrualBleeding` | Sangrado intermenstrual (spotting) |
| `HKCategoryTypeIdentifierCervicalMucusQuality` | Calidad del moco cervical |
| `HKCategoryTypeIdentifierOvulationTestResult` | Resultado de test de ovulación |
| `HKCategoryTypeIdentifierSexualActivity` | Actividad sexual (opt-in explícito y granular, ver §9) |
| `HKCategoryTypeIdentifierContraceptive` | Método anticonceptivo declarado |
| `HKQuantityTypeIdentifierBasalBodyTemperature` | Temperatura basal |
| `HKCategoryTypeIdentifierPregnancyTestResult` / `ProgesteroneTestResult` | Tests relacionados (iOS 16+) |

Cualquier app (incluida la propia Salud, Flo, Clue, Natural Cycles...) que la usuaria haya autorizado a escribir en Health alimenta estos mismos tipos. Por eso HealthKit es la vía de mayor cobertura con menor esfuerzo de integración: no dependemos de que cada competidor tenga export, dependemos solo de que la usuaria haya usado *alguna* app conectada a Salud o la propia app de Salud.

**Importante:** en Android no existe HealthKit. El equivalente es **Health Connect** (`androidx.health.connect`, tipos `MenstruationFlowRecord`, `MenstruationPeriodRecord`, `CervicalMucusRecord`, `OvulationTestRecord`, `BasalBodyTemperatureRecord`, `SexualActivityRecord`). Si Blumm tiene o planea versión Android, esta spec aplica igual con Health Connect como contraparte — se detalla como fase separada en §11.

### 5.2 Importación manual desde otra app (CSV/JSON)

No todas las usuarias tendrán datos en Health (algunas apps no escriben ahí, o la usuaria nunca dio permiso). Como red de seguridad, se ofrece importar un archivo exportado manualmente:

- **Flo:** exporta PDF de resumen por defecto; exportación de datos crudos vía "solicitud de mis datos" (más lenta, formato variable).
- **Clue:** permite exportar CSV desde ajustes.
- **Natural Cycles:** exporta CSV de temperatura basal.
- **Apple Salud (app):** permite exportar *todo* el histórico como XML (`export.xml`), que incluye los mismos tipos que HealthKit.

Dado que cada app tiene su propio formato (y lo cambia sin avisar), la recomendación es **no** construir parsers a medida por competidor como primer paso — es frágil y de mantenimiento caro. En su lugar:

- MVP: **importador CSV genérico** con mapeo de columnas asistido por el usuario ("¿cuál de estas columnas es la fecha de inicio del sangrado?").
- Soporte explícito y probado para el export XML de Apple Salud (formato estable, documentado, cubre el 80% de los casos ya que es la fuente común).
- Añadir parsers dedicados por competidor solo si el volumen de usuarias lo justifica (medir cuántas intentan importar y de qué app vienen, ver §12).

## 6. Modelo de datos objetivo (mapeo)

| Campo en Blumm | Origen HealthKit | Origen CSV genérico |
|---|---|---|
| Fecha de sangrado + intensidad | `MenstrualFlow` | columna fecha + columna intensidad (mapeo manual) |
| Spotting | `IntermenstrualBleeding` | columna opcional |
| Duración de ciclo (derivada) | calculada a partir de fechas de sangrado, no se importa directamente | igual |
| Síntomas | no cubierto por HealthKit de forma estándar (Blumm ya los captura manualmente) | fuera de alcance del import |
| Temperatura basal | `BasalBodyTemperature` | columna opcional |
| Método anticonceptivo | `Contraceptive` | — |

Todo lo importado se marca internamente con `source: healthkit | csv_import | manual` y `imported_at`, para poder auditar y para la lógica de deduplicación de §8.2.

## 7. Flujo de usuario (UX)

1. En onboarding, tras el registro, pantalla: **"¿Ya llevas tu ciclo en otra app?"** → dos botones: *Conectar con Salud* / *Importar archivo* / *Empezar desde cero*.
2. **Conectar con Salud:** prompt nativo de permisos de iOS (`HKHealthStore.requestAuthorization`), mostrando explícitamente qué tipos se piden y por qué (copy claro, no el texto genérico del sistema únicamente).
3. Import corre en background con una barra de progreso ligera; al terminar, pantalla-resumen: *"Importamos 14 meses de historial · 6 ciclos detectados"* con opción de revisar antes de confirmar (evita importar datos erróneos sin que la usuaria los vea).
4. **Importar archivo:** selector de archivo (CSV o el `export.xml` de Salud) → si es CSV sin reconocer, pantalla de mapeo de columnas → mismo resumen de confirmación.
5. Si ya existen registros manuales previos en Blumm que solapan con lo importado, se avisa y se pide elegir qué prevalece (ver §8.2).

## 8. Diseño técnico

### 8.1 Lectura de HealthKit
- Permisos de solo lectura (`toShare: []`, `toRead: [...]`) — no se solicita escritura en esta fase.
- Import inicial: consulta histórica completa (`HKSampleQuery` sin límite de fecha, o límite razonable p. ej. 5 años).
- Nada de `HKObserverQuery`/background delivery en el MVP: es import puntual, no sincronización continua (eso es explícitamente un no-objetivo, §3).

### 8.2 Deduplicación y conflictos
- Si una fecha ya tiene un registro manual en Blumm y también aparece en el import, el registro manual gana por defecto (la usuaria lo introdujo con intención), pero se le pregunta si quiere sobrescribir con el dato importado.
- Los registros importados nunca se auto-fusionan silenciosamente con distinta intensidad/valor en la misma fecha — se marca como conflicto para revisión, no se decide algorítmicamente.

### 8.3 Importador CSV
- Parser tolerante a formato (`,` o `;`, fechas en varios formatos comunes DD/MM/YYYY, MM/DD/YYYY, ISO).
- UI de mapeo de columnas reutilizable, no específica de una app — la misma pantalla sirve para cualquier CSV.
- Parser dedicado para `export.xml` de Apple Salud como caso especial de alta cobertura (formato fijo, no requiere mapeo).

## 9. Privacidad, seguridad y cumplimiento

- Los datos de ciclo son **categoría especial de datos de salud** (GDPR art. 9) — ya se trata así en la política de privacidad actual (`privacy.html`). El import no cambia esa clasificación, solo la fuente.
- **Apple exige explícitamente** (App Store Review Guideline 2.5.10 / 5.1.3) que los datos leídos de HealthKit no se usen para publicidad, no se vendan a brokers de datos ni se compartan con terceros para fines no relacionados con salud/fitness. Esto debe quedar reflejado en política de privacidad y respetarse en analítica (PostHog no debe recibir estos campos, igual que ya se declara para el resto de datos de salud).
- Info.plist necesita `NSHealthShareUsageDescription` con copy claro (no boilerplate) explicando por qué se piden estos datos.
- El dato de **actividad sexual** (`SexualActivity`) es especialmente sensible: si se llega a pedir, debe ser opt-in aparte y no un permiso agrupado con el resto — o simplemente excluirlo del alcance si no aporta valor al producto.
- Import de archivo: el archivo se procesa y se descarta (no se sube tal cual a analítica ni a logs); solo se guarda el resultado estructurado.
- Derecho de portabilidad (ya mencionado en `privacy.html`, "Exportar tus datos") debería, por coherencia, ofrecer el camino inverso: exportar desde Blumm en un formato igual de abierto.

## 10. Requisitos de plataforma

- **iOS:** entitlement HealthKit habilitado en el proyecto, capacidad "HealthKit" activada en el Apple Developer Portal, `NSHealthShareUsageDescription` en Info.plist. Sin uso de datos clínicos/investigación no aplican requisitos adicionales de "Clinical Health Records".
- **Android (si aplica):** declarar permisos de Health Connect (`android.permission.health.READ_MENSTRUATION` y equivalentes), y Health Connect debe estar instalado en el dispositivo (Android 14+ lo trae de serie; en versiones previas es una app aparte de Google).

## 11. Fases de entrega propuestas

1. **Fase 1 (MVP):** Lectura HealthKit (solo iOS) + parser dedicado para `export.xml` de Apple Salud. Cubre la mayoría de casos con el menor esfuerzo.
2. **Fase 2:** Importador CSV genérico con mapeo de columnas, para quien no use Salud.
3. **Fase 3:** Medir de qué apps viene la demanda real de import (telemetría de la Fase 2) y evaluar parsers dedicados solo para las 1-2 apps más solicitadas.
4. **Fase 4 (opcional, requiere validación de producto):** Sincronización continua con HealthKit (no solo import puntual) y/o escritura de Blumm hacia Health. Paridad Android vía Health Connect si existe versión Android.

## 12. Métricas de éxito

- % de usuarias en onboarding que usan "Conectar con Salud" o "Importar archivo" vs. "Empezar desde cero".
- Retención D7/D30 comparando quienes importaron histórico vs. quienes no.
- Nº de ciclos importados de media por usuaria que usa la función.
- Tasa de error/abandono durante el flujo de import (para detectar si el mapeo de CSV es confuso).

## 13. Riesgos

| Riesgo | Mitigación |
|---|---|
| Formatos de export de competidores cambian sin aviso | No depender de ellos como vía principal; HealthKit como capa neutral |
| Datos importados con mala calidad (huecos, duplicados) generan predicciones peores que empezar de cero | Pantalla de revisión antes de confirmar import (§7.3) |
| Percepción de la usuaria de que "otra app" puede leer sus datos de Blumm | Dejar claro que el permiso es de solo lectura y unidireccional (Health → Blumm) |
| Rechazo de review de Apple por copy de permisos poco claro | Redactar `NSHealthShareUsageDescription` específico y revisar contra guideline 2.5.10 antes de submit |

## 14. Preguntas abiertas

- ¿Blumm tiene o planea versión Android? Determina si Health Connect entra en el roadmap ahora o se pospone.
- ¿Vale la pena invertir en parsers específicos de Flo/Clue en el MVP, o se prioriza HealthKit + CSV genérico y se decide después con datos reales de demanda (Fase 3)?
- ¿Se quiere ofrecer también exportación *desde* Blumm (portabilidad en la otra dirección) en el mismo proyecto, o se trata como iniciativa separada?
