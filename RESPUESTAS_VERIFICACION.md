# Respuestas a las preguntas de verificación

## 1. ¿En qué situaciones preferirías escalado horizontal vs vertical?

### Escalado Vertical (Scale Up):
- **Situaciones preferibles:**
  1. **Cargas de trabajo con procesos únicos intensivos:** Cuando una aplicación tiene un proceso monolítico que no se puede paralelizar fácilmente y requiere más recursos de CPU/memoria.
  2. **Bases de datos relacionales tradicionales:** Sistemas como PostgreSQL, MySQL que tienen limitaciones en la distribución horizontal.
  3. **Entornos con limitaciones de licencia:** Algunas aplicaciones comerciales se licencian por socket o núcleo, haciendo más económico añadir recursos a una máquina existente.
  4. **Simplificación operativa:** En equipos pequeños con poca experiencia en sistemas distribuidos, donde la complejidad del escalado horizontal supera los beneficios.
  5. **Aplicaciones con estado compartido:** Sistemas donde el estado es difícil de distribuir entre múltiples instancias.

- **Limitaciones:**
  - Límite físico del hardware máximo disponible
  - Downtime durante upgrades de hardware
  - Punto único de fallo

### Escalado Horizontal (Scale Out):
- **Situaciones preferibles:**
  1. **Servicios web y APIs:** Aplicaciones stateless que pueden distribuir carga entre múltiples instancias.
  2. **Procesamiento por lotes (batch processing):** Cargas de trabajo que se pueden paralelizar fácilmente (MapReduce, ETL).
  3. **Alta disponibilidad requerida:** Sistemas que no pueden tolerar downtime (zero-downtime deployments).
  4. **Crecimiento elástico:** Entornos cloud donde se necesita escalar dinámicamente basado en demanda.
  5. **Microservicios:** Arquitecturas distribuidas donde cada servicio puede escalar independientemente.

- **Beneficios:**
  - Teóricamente ilimitado (añadir más máquinas)
  - Mayor resiliencia (sin punto único de fallo)
  - Zero-downtime upgrades
  - Mejor relación costo-rendimiento en cloud

**Decisión práctica:** En pipelines de datos modernos (Airflow, Spark), se prefiere escalado horizontal porque:
- Los workers pueden distribuir tareas independientes
- Permite zero-downtime deployments (blue-green, rolling)
- Mejora la tolerancia a fallos
- Se adapta mejor a cargas variables

## 2. ¿Cómo asegurar compatibilidad backward cuando cambias esquemas de datos?

### Estrategias para mantener backward compatibility:

#### 1. **Versionado de Esquemas:**
   - Incluir campo `schema_version` en todos los registros
   - Mantener definiciones de todas las versiones soportadas
   - Implementar migraciones automáticas (upgrade/downgrade)

#### 2. **Reglas de Evolución de Esquemas:**
   - **Solo añadir campos:** Nunca eliminar campos existentes
   - **Campos opcionales:** Nuevos campos deben ser nullable o tener valores por defecto
   - **No cambiar tipos:** Si es necesario, crear nuevo campo con sufijo de versión
   - **Mantener nombres:** No renombrar campos existentes

#### 3. **Diseño de Migraciones:**
   ```sql
   -- En lugar de:
   ALTER TABLE users DROP COLUMN old_field;
   
   -- Preferir:
   ALTER TABLE users ADD COLUMN new_field VARCHAR(255);
   -- Mantener old_field disponible durante periodo de transición
   ```

#### 4. **Validación en Dos Fases:**
   - **Fase 1:** Nueva versión escribe ambos formatos (viejo y nuevo)
   - **Fase 2:** Actualizar lectores para usar nuevo formato
   - **Fase 3:** Remover soporte para formato viejo (después de confirmar que todos los consumidores están actualizados)

#### 5. **Contratos de Datos:**
   - Definir interfaces explícitas entre productores y consumidores
   - Usar herramientas como Avro, Protobuf o JSON Schema con soporte de versionado
   - Implementar registros de esquema (Schema Registry)

#### 6. **Testing de Compatibilidad:**
   - Tests que verifican que datos viejos pueden ser leídos por nueva versión
   - Simulaciones de rollback para asegurar que versiones anteriores pueden procesar datos nuevos
   - Validación de que migraciones son reversibles

#### 7. **Estrategias en el Código:**
   ```python
   class DataVersionManager:
       def handle_legacy_data(self, data):
           """Manejar datos de versiones anteriores"""
           if 'schema_version' not in data:
               # Legacy data (version 1)
               data['schema_version'] = 1
               data['email'] = None  # Campo faltante agregado con valor por defecto
           return data
       
       def upgrade_data(self, data, target_version):
           """Migración progresiva entre versiones"""
           current_version = data.get('schema_version', 1)
           while current_version < target_version:
               data = self._upgrade_one_version(data, current_version)
               current_version += 1
           return data
   ```

#### 8. **Documentación y Comunicación:**
   - Documentar cambios de esquema y periodos de soporte
   - Comunicar fechas de deprecación a todos los consumidores
   - Mantener changelog detallado de versiones

### Ejemplo práctico del proyecto:
En nuestro `DataVersionManager` implementamos:
- Esquemas versionados (v1, v2, v3)
- Migraciones automáticas que solo añaden campos (email, phone, updated_at)
- Campos nuevos son opcionales (None por defecto)
- Validación que permite campos faltantes en versiones anteriores
- Scripts de migración SQL que solo añaden columnas (no eliminan)

Esta aproximación asegura que:
1. **Datos antiguos** siguen siendo válidos con nuevas versiones del código
2. **Nuevo código** puede leer datos antiguos (usando valores por defecto para campos faltantes)
3. **Rollbacks** son posibles sin pérdida de datos
4. **Transiciones** entre versiones son suaves y sin downtime