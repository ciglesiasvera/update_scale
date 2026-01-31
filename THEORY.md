Actualización y Escalabilidad de Pipelines

Objetivos de Aprendizaje
1 Entender estrategias de escalado horizontal y vertical
2 Aprender actualización de pipelines sin downtime
3 Comprender gestión de versiones de datos
4 Conocer optimizaciones de performance avanzadas

Actividades y Aprendizajes
Task 1: Estrategias de Escalado (10 minutos)
Escalado vertical vs horizontal:

Escalado Vertical:

Más CPU, memoria, disco en misma máquina
Más simple de implementar
Límite físico de hardware
Downtime durante upgrade
Escalado Horizontal:

Más máquinas trabajando en paralelo
Mayor complejidad de coordinación
Teóricamente ilimitado
Zero-downtime upgrades
Implementación en Airflow:

# Escalado horizontal con Celery
from airflow.executors import CeleryExecutor

# Configuración para múltiples workers
executor_config = {
    'KubernetesPodOperator': {
        'image': 'my-custom-image',
        'namespace': 'airflow',
        'worker_container_repository': 'my-worker',
        'worker_container_tag': 'latest'
    }
}
Task 2: Zero-Downtime Deployments (10 minutos)
Técnicas para deployments sin interrupción:

1. Blue-Green Deployment:

# Script de deployment blue-green
def deploy_blue_green(new_version):
    """Deploy con estrategia blue-green"""
    
    # Crear nueva versión (green)
    create_green_environment(new_version)
    
    # Verificar health de green
    if verify_green_health():
        # Cambiar traffic a green
        switch_traffic_to_green()
        
        # Green se convierte en blue
        promote_green_to_blue()
        
        # Limpiar antigua versión
        cleanup_old_blue()
    
    return True
2. Rolling Deployment:

def rolling_deployment(new_dag_version, batch_size=2):
    """Deploy gradual por batches"""
    
    workers = get_all_workers()
    
    for i in range(0, len(workers), batch_size):
        batch = workers[i:i+batch_size]
        
        # Deploy to batch
        deploy_to_workers(batch, new_dag_version)
        
        # Wait for health checks
        wait_for_batch_health(batch)
        
        # Continue with next batch
Task 3: Versionado de Datos y Esquemas (10 minutos)
Gestión de cambios en esquemas:

1. Versionado forward-compatible:

# Esquema con versionado
data_schema_v1 = {
    'version': 1,
    'fields': {
        'id': 'string',
        'name': 'string',
        'created_at': 'datetime'
    }
}

data_schema_v2 = {
    'version': 2,
    'fields': {
        'id': 'string', 
        'name': 'string',
        'email': 'string',  # Campo nuevo
        'created_at': 'datetime'
    }
}

def process_data_versioned(data, target_version=2):
    """Procesar datos con versionado de esquema"""
    
    current_version = data.get('schema_version', 1)
    
    if current_version < target_version:
        # Upgrade schema
        data = upgrade_schema(data, current_version, target_version)
    
    return data
2. Backward compatibility:

def handle_legacy_data(data):
    """Manejar datos de versiones anteriores"""
    
    if 'schema_version' not in data:
        # Legacy data (version 1)
        data['schema_version'] = 1
        data['email'] = None  # Campo faltante
    
    return data