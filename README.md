# Proyecto: Actualización y Escalabilidad de Pipelines

## 📋 Descripción del Proyecto

Este proyecto implementa estrategias de escalado y versionado para pipelines de datos, como parte de un curso de ciencia de datos en la plataforma TalentOps, beca impulsada por Kranio. El ejercicio práctico aplica los conceptos teóricos de escalabilidad horizontal/vertical, deployments zero-downtime y gestión de versiones de datos.

### Objetivos de Aprendizaje
1. Entender estrategias de escalado horizontal y vertical
2. Aprender actualización de pipelines sin downtime
3. Comprender gestión de versiones de datos
4. Conocer optimizaciones de performance avanzadas

## 🏗️ Arquitectura del Proyecto

El proyecto está estructurado en los siguientes componentes:

### 1. Configuración de Escalado Horizontal (`docker-compose.scale.yml`)
- Configuración Docker Compose para Airflow con Celery Executor
- 5 workers para distribución de carga
- Redis como broker de mensajes

### 2. Gestor de Versionado de Datos (`data_version_manager.py`)
- Clase `DataVersionManager` para manejo de esquemas versionados
- Soporte para migraciones automáticas entre versiones
- Validación de esquemas y tipos de datos
- Generación de scripts de migración SQL

### 3. Script de Deployment Zero-Downtime (`deploy-zero-downtime.sh`)
- Estrategia Blue-Green deployment
- Health checks y smoke tests automatizados
- Cambio de tráfico sin interrupción
- Limpieza de versiones antiguas

## 📁 Estructura de Archivos

```
update_scale/
├── docker-compose.scale.yml     # Configuración de escalado horizontal
├── data_version_manager.py      # Gestor de versionado de datos
├── deploy-zero-downtime.sh      # Script de deployment zero-downtime
├── RESPUESTAS_VERIFICACION.md   # Respuestas a preguntas de verificación
├── THEORY.md                    # Contenido teórico del curso
├── PRACTICE.md                  # Ejercicio práctico
├── instructions.txt             # Instrucciones del proyecto
├── README.md                    # Este archivo
└── update_scale_venv/           # Entorno virtual Python
```

## 🛠️ Dependencias

### Requisitos del Sistema
- **Python 3.10+**
- **Docker 20.10+** y **Docker Compose 2.0+**
- **Git** para control de versiones

### Dependencias Python
El proyecto utiliza un entorno virtual con las siguientes dependencias implícitas:
- `typing` (incluido en Python 3.10+)
- `datetime` (incluido en Python estándar)
- `json` (incluido en Python estándar)

### Dependencias para Deployment (opcionales)
- **Kubernetes** o **Docker Swarm** para orquestación en producción
- **Redis** como broker de mensajes (incluido en docker-compose)

## 🚀 Cómo Ejecutar el Proyecto

### 1. Configuración Inicial

```bash
# Clonar el repositorio (si aplica)
git clone <url-del-repositorio>
cd update_scale

# Verificar que el entorno virtual esté activado
source update_scale_venv/bin/activate

# Verificar instalación de Python
python --version
```

### 2. Probar el Gestor de Versionado de Datos

```bash
# Ejecutar el gestor de versionado
python data_version_manager.py

# Ejemplo de salida esperada:
# Datos válidos: True
# Datos upgradeados a versión: 3
# Datos upgradeados válidos: True
```

### 3. Probar el Script de Deployment (simulado)

```bash
# Dar permisos de ejecución al script
chmod +x deploy-zero-downtime.sh

# Ejecutar deployment simulado
./deploy-zero-downtime.sh v1.0.0
```

### 4. Levantar Infraestructura con Docker Compose

```bash
# Iniciar Airflow con escalado horizontal
docker-compose -f docker-compose.scale.yml up -d

# Verificar que los servicios estén corriendo
docker-compose -f docker-compose.scale.yml ps

# Detener servicios
docker-compose -f docker-compose.scale.yml down
```

### 5. Ejecutar Pruebas de Validación

```bash
# Crear un script de prueba simple
cat > test_version_manager.py << 'EOF'
from data_version_manager import DataVersionManager

vm = DataVersionManager()

# Test 1: Validar datos legacy
legacy_data = {'id': '123', 'name': 'Test', 'created_at': '2024-01-01'}
result = vm.validate_schema(legacy_data)
print(f"Test 1 - Legacy data valid: {result['valid']}")

# Test 2: Upgrade data
upgraded = vm.upgrade_data(legacy_data.copy(), 3)
print(f"Test 2 - Upgraded to version: {upgraded['schema_version']}")
print(f"Test 2 - Has phone field: {'phone' in upgraded}")

# Test 3: Migration script
script = vm.create_migration_script(1, 2)
print(f"Test 3 - Migration script generated: {len(script) > 0}")
EOF

python test_version_manager.py
```

## 📊 Ejemplos de Uso

### Ejemplo 1: Migración de Datos

```python
from data_version_manager import DataVersionManager

# Inicializar gestor
vm = DataVersionManager()

# Datos en versión 1 (legacy)
data_v1 = {
    'id': 'user_001',
    'name': 'María González',
    'created_at': '2024-02-15T14:30:00'
}

# Validar esquema actual
validation = vm.validate_schema(data_v1)
print(f"Validación V1: {validation['valid']}")

# Migrar a versión 3
data_v3 = vm.upgrade_data(data_v1, 3)
print(f"Versión final: {data_v3['schema_version']}")
print(f"Campos agregados: email={data_v3['email']}, phone={data_v3['phone']}")
```

### Ejemplo 2: Generar Script de Migración SQL

```python
# Generar script para migrar base de datos
sql_script = vm.create_migration_script(1, 3)
print("Script de migración SQL:")
print(sql_script)
```

## 🤖 Autor

**Carlos Iglesias Vera**
- GitHub: [ciglesiasvera](https://github.com/ciglesiasvera)
- Email: ciglesiasvera@gmail.com
- Plataforma: TalentOps (Kranio Beca)

## 📝 Notas de Implementación

### Ajustes y Correcciones Realizadas

1. **Corrección en `data_version_manager.py`:**
   - Modificado el método `validate_schema` para permitir campos con valor `None` en la validación de tipos
   - Se agregó chequeo `data[field] is not None` antes de verificar el tipo
   - Esto permite que campos opcionales (como `email` y `phone` en versiones anteriores) puedan ser `None` sin causar errores de validación

2. **Mejora en el script de deployment:**
   - Se agregaron comandos simulados para entornos sin Kubernetes
   - Se mantuvo la lógica original para facilitar la transición a producción
   - Se incluyeron mensajes descriptivos para cada paso

3. **Estructura del proyecto:**
   - Se crearon todos los archivos especificados en PRACTICE.md
   - Se organizó el código en módulos separados para mejor mantenibilidad
   - Se incluyeron ejemplos de uso y pruebas

