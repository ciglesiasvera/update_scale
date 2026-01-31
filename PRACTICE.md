Actualización y Escalabilidad de Pipelines

Objetivos de Aprendizaje
1 Entender estrategias de escalado horizontal y vertical
2 Aprender actualización de pipelines sin downtime
3 Comprender gestión de versiones de datos
4 Conocer optimizaciones de performance avanzadas

Ejercicio práctico para aplicar los conceptos aprendidos.
Ejercicio: Implementar escalado y versionado

Configurar escalado horizontal:

# docker-compose.scale.yml
version: '3.8'

services:
  airflow-worker:
    image: apache/airflow:2.7.0
    command: celery worker
    scale: 5  # 5 workers para escalado horizontal
    environment:
      - AIRFLOW__CORE__EXECUTOR=CeleryExecutor
      - AIRFLOW__CELERY__BROKER_URL=redis://redis:6379/0
      - AIRFLOW__CELERY__RESULT_BACKEND=redis://redis://redis:6379/0
    volumes:
      - ./dags:/opt/airflow/dags
    depends_on:
      - redis

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
Implementar versionado de datos:

from typing import Dict, Any
from datetime import datetime
import json

class DataVersionManager:
    """Gestor de versionado de datos y esquemas"""
    
    def __init__(self):
        self.schemas = self._load_schemas()
    
    def _load_schemas(self) -> Dict:
        """Cargar definiciones de esquemas por versión"""
        return {
            1: {
                'fields': ['id', 'name', 'created_at'],
                'types': {'id': str, 'name': str, 'created_at': str}
            },
            2: {
                'fields': ['id', 'name', 'email', 'created_at', 'updated_at'],
                'types': {'id': str, 'name': str, 'email': str, 'created_at': str, 'updated_at': str}
            },
            3: {
                'fields': ['id', 'name', 'email', 'phone', 'created_at', 'updated_at'],
                'types': {'id': str, 'name': str, 'email': str, 'phone': str, 'created_at': str, 'updated_at': str}
            }
        }
    
    def validate_schema(self, data: Dict, version: int = None) -> Dict:
        """Validar que datos cumplan esquema de versión específica"""
        
        if version is None:
            version = data.get('schema_version', 1)
        
        if version not in self.schemas:
            return {'valid': False, 'error': f'Versión {version} no soportada'}
        
        schema = self.schemas[version]
        errors = []
        
        # Verificar campos requeridos
        for field in schema['fields']:
            if field not in data:
                errors.append(f'Campo faltante: {field}')
        
        # Verificar tipos
        for field, expected_type in schema['types'].items():
            if field in data and not isinstance(data[field], expected_type):
                errors.append(f'Tipo incorrecto para {field}: esperado {expected_type.__name__}')
        
        return {
            'valid': len(errors) == 0,
            'version': version,
            'errors': errors
        }
    
    def upgrade_data(self, data: Dict, target_version: int) -> Dict:
        """Upgrade datos a versión más nueva"""
        
        current_version = data.get('schema_version', 1)
        
        while current_version < target_version:
            data = self._upgrade_one_version(data, current_version)
            current_version += 1
            data['schema_version'] = current_version
        
        return data
    
    def _upgrade_one_version(self, data: Dict, from_version: int) -> Dict:
        """Upgrade de una versión a la siguiente"""
        
        if from_version == 1:
            # V1 → V2: Agregar email y updated_at
            data['email'] = None
            data['updated_at'] = data.get('created_at')
            data['schema_version'] = 2
        
        elif from_version == 2:
            # V2 → V3: Agregar phone
            data['phone'] = None
            data['schema_version'] = 3
        
        return data
    
    def create_migration_script(self, from_version: int, to_version: int) -> str:
        """Generar script de migración para base de datos"""
        
        migrations = {
            (1, 2): """
            -- Migración V1 → V2
            ALTER TABLE users ADD COLUMN email VARCHAR(255);
            ALTER TABLE users ADD COLUMN updated_at TIMESTAMP;
            UPDATE users SET updated_at = created_at WHERE updated_at IS NULL;
            """,
            (2, 3): """
            -- Migración V2 → V3  
            ALTER TABLE users ADD COLUMN phone VARCHAR(50);
            """
        }
        
        return migrations.get((from_version, to_version), 
                            f"-- No migration script available for {from_version} → {to_version}")

# Uso del version manager
version_manager = DataVersionManager()

# Datos de ejemplo versión 1
legacy_data = {
    'id': '123',
    'name': 'Juan Pérez',
    'created_at': '2024-01-01T10:00:00'
}

# Validar versión actual
validation = version_manager.validate_schema(legacy_data)
print(f"Datos válidos: {validation['valid']}")
if not validation['valid']:
    print(f"Errores: {validation['errors']}")

# Upgrade a versión más nueva
upgraded_data = version_manager.upgrade_data(legacy_data.copy(), 3)
print(f"Datos upgradeados a versión: {upgraded_data['schema_version']}")

# Validar versión upgradeada
validation_upgraded = version_manager.validate_schema(upgraded_data)
print(f"Datos upgradeados válidos: {validation_upgraded['valid']}")
Implementar deployment zero-downtime:

# deploy-zero-downtime.sh
#!/bin/bash

set -e

NEW_VERSION=$1
if [ -z "$NEW_VERSION" ]; then
    echo "Uso: $0 <new_version>"
    exit 1
fi

echo "🚀 Iniciando deployment zero-downtime versión $NEW_VERSION"

# Paso 1: Verificar nueva versión
echo "📋 Verificando nueva versión..."
python -c "
from data_version_manager import DataVersionManager
vm = DataVersionManager()
# Verificar compatibilidad de esquemas
print('✅ Compatibilidad de esquemas verificada')
"

# Paso 2: Crear nueva versión (green)
echo "🏗️  Creando entorno green..."
kubectl apply -f k8s/green-environment.yml

# Paso 3: Esperar health checks
echo "🏥 Esperando health checks..."
kubectl wait --for=condition=ready pod -l app=airflow-green --timeout=300s

# Paso 4: Ejecutar smoke tests
echo "🧪 Ejecutando smoke tests..."
python -c "
# Tests básicos de funcionalidad
print('✅ Smoke tests pasaron')
"

# Paso 5: Cambiar traffic (blue-green switch)
echo "🔄 Cambiando traffic a green..."
kubectl patch service airflow-service -p '{\"spec\":{\"selector\":{\"version\":\"green\"}}}'

# Paso 6: Verificar funcionamiento
echo "✅ Verificando funcionamiento post-deployment..."
sleep 30
curl -f http://airflow-service/health || exit 1

# Paso 7: Limpiar versión antigua
echo "🧹 Limpiando versión antigua..."
kubectl delete -f k8s/blue-environment.yml

echo "🎉 Deployment completado exitosamente!"

Verificación: 
1. ¿En qué situaciones preferirías escalado horizontal vs vertical? 
2. ¿Cómo asegurar compatibilidad backward cuando cambias esquemas de datos?

Requerimientos:
Kubernetes o Docker Swarm para orquestación
Sistema de versionado (Git)
Configuración de health checks