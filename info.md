# 🚀 Guía de Instalación de LibreChat en Minikube

## 📋 Requisitos Previos

- Minikube instalado y corriendo
- kubectl instalado
- Helm 3 instalado
- Al menos 4GB de RAM disponibles para Minikube

## 🔧 Paso 1: Preparar Minikube

```bash
# Iniciar Minikube con suficientes recursos
minikube start --memory=4096 --cpus=2

# Verificar que está corriendo
minikube status

# Habilitar addons necesarios
minikube addons enable storage-provisioner
minikube addons enable default-storageclass
```

## 📝 Paso 2: Preparar los Archivos de Configuración

### 2.1 Editar `secrets.yaml`

**⚠️ IMPORTANTE:** Antes de aplicar, edita el archivo y reemplaza:

- `OPENAI_API_KEY: "sk-tu-api-key-aqui"` con tu API key real de OpenAI

Si usas otro proveedor (Azure, Anthropic, etc.), puedes agregar más keys:

```yaml
stringData:
  # ... otras claves ...
  ANTHROPIC_API_KEY: "tu-anthropic-key"
  AZURE_API_KEY: "tu-azure-key"
```

### 2.2 Aplicar el Secret

```bash
# Aplica el secret con las credenciales
kubectl apply -f secrets.yaml

# Verifica que se creó correctamente
kubectl get secret librechat-credentials-env
kubectl describe secret librechat-credentials-env
```

## 🎯 Paso 3: Instalar LibreChat con Helm

```bash
# Desinstala cualquier instalación anterior (si existe)
helm uninstall librechat 2>/dev/null || true

# Espera a que se limpien los recursos
sleep 10

# Instala LibreChat con la configuración optimizada
helm install librechat \
  oci://ghcr.io/danny-avila/librechat-chart/librechat \
  --values values.yaml \
  --timeout 10m

# Alternativa: si tienes el chart localmente
# helm install librechat ./helm/librechat --values values.yaml
```

## 👀 Paso 4: Monitorear la Instalación

```bash
# Ver el progreso de los pods en tiempo real
kubectl get pods -w

# En otra terminal, ver los logs de LibreChat
kubectl logs -f -l app.kubernetes.io/name=librechat

# Ver logs de MongoDB
kubectl logs -f -l app.kubernetes.io/name=mongodb

# Ver logs de Meilisearch
kubectl logs -f -l app.kubernetes.io/name=meilisearch
```

### Estados esperados:

1. **Primero:** MongoDB y Meilisearch deben estar `Running` y `Ready (1/1)`
2. **Después:** LibreChat debería pasar a `Running` y luego a `Ready (1/1)`

```bash
# Ejemplo de salida esperada después de ~2-3 minutos:
NAME                                   READY   STATUS    RESTARTS   AGE
librechat-librechat-xxxxx-xxxxx        1/1     Running   0          3m
librechat-mongodb-xxxxx-xxxxx          1/1     Running   0          3m
librechat-meilisearch-xxxxx-xxxxx      1/1     Running   0          3m
```

## 🌐 Paso 5: Acceder a LibreChat

### Opción A: Usando NodePort (Recomendado para Minikube)

```bash
# Obtener la URL de acceso
minikube service librechat-librechat --url

# Esto devolverá algo como: http://192.168.49.2:30080
# Abre esa URL en tu navegador
```

### Opción B: Usando Port Forwarding

```bash
# Reenviar el puerto 3080 a tu localhost
kubectl port-forward service/librechat-librechat 3080:3080

# Accede en tu navegador a: http://localhost:3080
```

### Opción C: Usando Minikube Tunnel (requiere sudo)

```bash
# En una terminal separada, mantén corriendo:
minikube tunnel

# Luego obtén la IP externa:
kubectl get service librechat-librechat
```

## ✅ Paso 6: Verificar que Todo Funciona

1. **Abre LibreChat en tu navegador**
2. **Registra un nuevo usuario** (primera vez)
3. **Inicia sesión**
4. **Prueba enviando un mensaje**

## 🔍 Comandos Útiles para Diagnóstico

### Ver el estado de todos los recursos

```bash
# Pods
kubectl get pods

# Servicios
kubectl get svc

# PersistentVolumeClaims
kubectl get pvc

# Secrets
kubectl get secrets

# Eventos recientes
kubectl get events --sort-by='.lastTimestamp' | tail -20
```

### Ver detalles de un pod específico

```bash
# Reemplaza POD_NAME con el nombre real
kubectl describe pod <POD_NAME>
kubectl logs <POD_NAME>
kubectl logs <POD_NAME> --previous  # logs del contenedor anterior si crasheó
```

### Ejecutar comandos dentro de MongoDB

```bash
# Obtén el nombre del pod de MongoDB
MONGO_POD=$(kubectl get pods -l app.kubernetes.io/name=mongodb -o jsonpath='{.items[0].metadata.name}')

# Conecta a MongoDB
kubectl exec -it $MONGO_POD -- mongosh

# Dentro de mongosh, prueba:
# > show dbs
# > use LibreChat
# > show collections
```

## 🛠️ Solución de Problemas Comunes

### Problema 1: Pods en estado "Pending"

```bash
# Ver por qué está pendiente
kubectl describe pod <POD_NAME>

# Solución común: Liberar recursos
kubectl delete pod <POD_NAME>
```

### Problema 2: LibreChat crashea con "connection refused"

```bash
# Verificar que MongoDB esté corriendo
kubectl get pods -l app.kubernetes.io/name=mongodb

# Ver logs de MongoDB
kubectl logs -l app.kubernetes.io/name=mongodb

# Si MongoDB no está listo, LibreChat seguirá crasheando
# Espera a que MongoDB esté Ready (1/1)
```

### Problema 3: "ImagePullBackOff"

```bash
# Ver el error específico
kubectl describe pod <POD_NAME>

# Puede que necesites cambiar la versión de la imagen en values.yaml
```

### Problema 4: Problemas con Secrets

```bash
# Verificar que el secret existe y tiene las claves correctas
kubectl get secret librechat-credentials-env -o yaml

# Si necesitas actualizar el secret:
kubectl delete secret librechat-credentials-env
kubectl apply -f secrets.yaml

# Reiniciar los pods para que tomen los nuevos valores
kubectl rollout restart deployment librechat-librechat
```

## 🗑️ Desinstalar LibreChat

```bash
# Desinstalar el chart de Helm
helm uninstall librechat

# Eliminar el secret (opcional)
kubectl delete secret librechat-credentials-env

# Eliminar PVCs (opcional, si tienes persistencia habilitada)
kubectl delete pvc --all

# Verificar que todo se eliminó
kubectl get all
```

## 🔄 Actualizar la Configuración

Si necesitas cambiar algo en `values.yaml`:

```bash
# Actualiza el deployment con los nuevos valores
helm upgrade librechat \
  oci://ghcr.io/danny-avila/librechat-chart/librechat \
  --values values.yaml

# O fuerza una reinstalación completa
helm uninstall librechat
helm install librechat \
  oci://ghcr.io/danny-avila/librechat-chart/librechat \
  --values values.yaml
```

## 📚 Recursos Adicionales

- **Documentación oficial de LibreChat:** https://www.librechat.ai/docs
- **Repositorio de GitHub:** https://github.com/danny-avila/LibreChat
- **Generador de credenciales:** https://www.librechat.ai/toolkit/creds_generator

## ⚠️ Notas Importantes

1. **Persistencia deshabilitada:** Los datos se perderán si reinicias los pods. Para producción, habilita `persistence: true` en MongoDB.

2. **Sin autenticación en MongoDB:** Solo para desarrollo. En producción, habilita `auth: enabled: true`.

3. **Recursos limitados:** La configuración está optimizada para Minikube. Para producción, aumenta los recursos.

4. **Security:** Las credenciales están en el archivo `secrets.yaml`. Nunca lo subas a Git. Usa `.gitignore`:

```bash
echo "secrets.yaml" >> .gitignore
```

## 🎉 ¡Listo!

Si seguiste todos los pasos, deberías tener LibreChat corriendo exitosamente en tu Minikube.

**Accede a:** `http://localhost:3080` o usa `minikube service librechat-librechat --url`
