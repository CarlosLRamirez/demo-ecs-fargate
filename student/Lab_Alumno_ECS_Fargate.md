---
tags: [aws-academy, lab, ecs, fargate]
---

# Lab: Correr una aplicación en Amazon ECS con Fargate detrás de un Load Balancer

## Qué vas a construir

Vas a desplegar una aplicación web de muestra dentro de contenedores, corriendo en un clúster de Amazon ECS con AWS Fargate (sin administrar ningún servidor), accesible desde internet a través de un Application Load Balancer que reparte el tráfico entre varias copias de la aplicación corriendo al mismo tiempo.

**Tiempo estimado:** 45-55 minutos.

**Requisitos previos:** una cuenta de AWS con acceso a la consola (CloudFormation, VPC, ECS, EC2).

---

## Antes de empezar: cómo está organizado este lab

Vas a trabajar en dos partes:

1. **Parte 1 — Red base:** despliegas una plantilla de CloudFormation que crea automáticamente la VPC, las subredes, y los security groups que la aplicación va a necesitar. Esto lo haces con un solo clic, no tienes que configurar nada manualmente.
2. **Parte 2 — Contenedores:** creas el clúster ECS, la definición de la aplicación (task definition), el servicio que la mantiene corriendo, y el balanceador de carga — paso a paso, en la consola.

---

## Parte 1: Desplegar la red base con CloudFormation

### 1.1 — Descargar la plantilla

Descarga el archivo `vpc-base.yaml` de la carpeta `cloudformation/` de este repositorio a tu computadora.

### 1.2 — Crear el stack

1. En la consola de AWS, busca **CloudFormation** en la barra de búsqueda superior y entra al servicio.
2. Haz clic en **Create stack** → **With new resources (standard)**.
3. En "Prepare template", deja seleccionado **Template is ready**.
4. En "Template source", selecciona **Upload a template file**.
5. Haz clic en **Choose file** y selecciona el `vpc-base.yaml` que descargaste.
6. Haz clic en **Next**.
7. Stack name: escribe `ecs-lab-red-base`.
8. En "Parameters", deja el valor de `EnvironmentName` como `ecs-demo` (o cámbialo si quieres, pero recuerda qué pusiste — lo vas a ver reflejado en los nombres de los recursos).
9. Haz clic en **Next**.
10. En la siguiente pantalla (Configure stack options), no cambies nada — haz clic en **Next**.
11. Revisa el resumen y haz clic en **Submit**.

### 1.3 — Esperar a que termine

El stack tarda 2-3 minutos en crearse. Puedes ver el progreso en la pestaña **Events** — cuando el estado en la parte superior diga `CREATE_COMPLETE`, está listo.

✅ **Checkpoint:** el status del stack `ecs-lab-red-base` debe decir `CREATE_COMPLETE` antes de seguir.

### 1.4 — Anotar los valores de salida (Outputs)

1. Con el stack seleccionado, ve a la pestaña **Outputs**.
2. Vas a ver 5 valores. Anótalos — los vas a necesitar en la Parte 2:

| Output | Anota aquí |
|---|---|
| `VpcId` | vpc-07a3c9c053cce0da5 |
| `PublicSubnet1Id` | subnet-070a258dd520358d6 |
| `PublicSubnet2Id` | subnet-0abf583c2791167d8 |
| `ALBSecurityGroupId` | sg-0d4d73ab61b7849dd |
| `FargateTaskSecurityGroupId` | sg-08875e88a85d6d6f9 |

> [!tip] ¿Qué acabas de crear?
> Una VPC con dos subredes públicas en dos zonas de disponibilidad distintas (para que la aplicación siga funcionando aunque una zona tenga problemas), y dos "security groups" — uno que va a proteger el balanceador de carga (permitiendo tráfico web desde internet) y otro que va a proteger tu aplicación (permitiendo tráfico únicamente desde el balanceador, no directamente desde internet).

---

## Parte 2: Crear el clúster ECS

1. En la barra de búsqueda superior, busca **Elastic Container Service** y entra al servicio.
2. En el menú izquierdo, selecciona **Clusters**.
3. Haz clic en **Create cluster**.
4. Cluster name: escribe `demo-cluster`.
5. En la sección "Infrastructure", asegúrate de que **AWS Fargate (serverless)** esté marcado, y que **Amazon EC2 instances** NO esté marcado.
6. Deja el resto de las opciones por defecto.
7. Haz clic en **Create**.

✅ **Checkpoint:** deberías ver `demo-cluster` en la lista de clústeres, con status `ACTIVE`.

> [!tip] ¿Por qué no elegiste ningún tipo de servidor?
> Con Fargate no necesitas decidir cuántas instancias EC2 usar ni de qué tamaño — AWS administra esa capa por ti. Solo vas a decirle a Fargate cuánta CPU y memoria necesita tu aplicación, y él se encarga del resto.

---

## Parte 3: Crear la task definition (la "receta" de tu aplicación)

Una task definition es como un plano: le dice a ECS qué imagen de contenedor usar, cuántos recursos darle, y en qué puerto escucha.

1. En el menú izquierdo de ECS, selecciona **Task definitions**.
2. Haz clic en **Create new task definition** → **Create new task definition**.
3. Task definition family: escribe `demo-task`.
4. En "Infrastructure requirements":
   - Launch type: **AWS Fargate**.
   - Operating system/Architecture: deja **Linux/X86_64**.
   - CPU: **0.25 vCPU**.
   - Memory: **0.5 GB**.
5. En "Task roles":
   - Task role: deja **None**.
   - Task execution role: deja **Create new role** (la consola va a crear automáticamente los permisos mínimos que Fargate necesita).
6. Baja hasta "Container - 1" y completa:
   - Name: `demo-container`
   - Image URI: `public.ecr.aws/docker/library/httpd:2.4`
   - Container port: `80`
   - Protocol: `TCP`
   - App protocol: `HTTP`
7. Deja el resto de las secciones (Environment variables, Logging, etc.) en sus valores por defecto.
8. Haz clic en **Create**.

✅ **Checkpoint:** deberías ver `demo-task` en la lista de Task definitions, con al menos una revisión (`1`) en estado `ACTIVE`.

> [!tip] ¿Qué es esa imagen que usaste?
> `public.ecr.aws/docker/library/httpd:2.4` es el servidor web Apache HTTP Server oficial, disponible en el registro público de ECR de AWS. Es la imagen que la propia documentación de AWS usa en sus ejemplos de ECS con Fargate. No necesitas construir ni subir ninguna imagen tú mismo para este lab.

---

## Parte 4: Crear el service y el Load Balancer

Este es el paso más largo del lab — aquí conectas el clúster, la task definition, y el balanceador de carga, todo en una sola pantalla.

1. Ve a **Clusters** → haz clic en `demo-cluster`.
2. Ve a la pestaña **Services** → haz clic en **Create**.
3. En "Environment", confirma que "Existing cluster" muestre `demo-cluster`.
4. En "Deployment configuration":
   - Application type: **Service**.
   - Family: selecciona `demo-task`.
   - Revision: deja la más reciente (LATEST).
   - Service name: escribe `demo-service`.
   - Desired tasks: escribe `2`.
5. En "Networking":
   - VPC: selecciona la VPC con el `VpcId` que anotaste en la Parte 1.
   - Subnets: marca **ambas** subredes (`PublicSubnet1Id` y `PublicSubnet2Id` de tu tabla).
   - Security group: selecciona **Use an existing security group** → elige la que corresponde a `FargateTaskSecurityGroupId`.
   - Public IP: cambia el switch a **Turn on**.
6. En "Load balancing":
   - Load balancer type: selecciona **Application Load Balancer**.
   - Marca **Create a new load balancer**.
   - Load balancer name: escribe `demo-alb`.
   - Container to load balance: debería aparecer `demo-container:80` automáticamente.
   - Listener: deja **Create new listener**, puerto `80`, protocolo `HTTP`.
   - Target group: deja **Create new target group**, nombre `demo-tg`, health check path `/`.
7. Baja hasta el final y haz clic en **Create**.
8. Espera 2-3 minutos mientras se aprovisionan las tasks y el balanceador.

> [!warning] Sobre el security group del Load Balancer
> Si en el paso de "Load balancing" la consola te permite elegir un security group existente para el ALB, selecciona el que corresponde a `ALBSecurityGroupId` (el que anotaste en la Parte 1). Si no aparece esa opción en tu pantalla y la consola crea uno nuevo automáticamente, no hay problema — el lab funciona igual.

✅ **Checkpoint:** en la pestaña Services de `demo-cluster`, deberías ver `demo-service` con status `ACTIVE`.

---

## Parte 5: Verificar que tu aplicación está funcionando

1. Ve a `demo-cluster` → pestaña **Services** → `demo-service`.
2. Confirma que "Running tasks" muestre `2 / 2`. Si todavía dice `0 / 2` o `1 / 2`, espera un minuto más y refresca.
3. Ahora ve a buscar la URL de tu aplicación:
   - En la barra de búsqueda superior, busca **EC2** y entra.
   - En el menú izquierdo, busca **Load Balancers** (dentro de la sección "Load Balancing").
   - Selecciona `demo-alb`.
   - Copia el valor de **DNS name** (se ve como `demo-alb-123456789.us-east-1.elb.amazonaws.com`).
4. Pega esa dirección en una pestaña nueva de tu navegador (con `http://` adelante si tu navegador no lo agrega solo).
5. Deberías ver cargar la página de la aplicación de muestra de ECS.

✅ **Checkpoint final:** si la página carga en el navegador, tu aplicación está corriendo correctamente en ECS con Fargate, balanceada entre dos tasks.

---

## Qué acaba de pasar (resumen conceptual)

1. Creaste un **clúster** — el espacio lógico donde van a vivir tus contenedores.
2. Creaste una **task definition** — el plano que describe qué imagen correr y cuántos recursos darle.
3. Creaste un **service** — que se encargó de lanzar 2 copias (**tasks**) de tu aplicación a partir de ese plano, y de mantenerlas corriendo.
4. El **Application Load Balancer** distribuye el tráfico entrante entre esas 2 tasks automáticamente.
5. En ningún momento elegiste un tipo de servidor ni configuraste manualmente cuántas instancias EC2 usar — **Fargate se encargó de correr los contenedores por ti**.

---

## Para ir más allá (opcional)

Si terminaste y quieres experimentar más:

1. **Ver el escalado en vivo:** ve a `demo-service` → **Update service** → cambia "Desired tasks" de `2` a `4` → **Update**. Observa en la pestaña Tasks cómo aparecen 2 tasks nuevas en cuestión de segundos.
2. **Refresca la página del navegador varias veces seguidas** y piensa: ¿cómo sabe el Load Balancer a cuál de las tasks mandar cada solicitud? (Pista: revisa el concepto de "target group" y "health check" que configuraste en la Parte 4.)

---

## Limpieza — importante hacer esto al terminar

El Load Balancer y las tasks de Fargate generan costo mientras están activos. Sigue este orden exacto:

### 1. Eliminar el service (esto también elimina el ALB y el target group)

1. Ve a `demo-cluster` → pestaña **Services** → selecciona `demo-service`.
2. Haz clic en **Delete**.
3. Escribe `delete demo-service` para confirmar (o el texto que te pida la consola) y confirma.
4. Espera a que las tasks terminen de detenerse (1-2 minutos).

### 2. Eliminar el clúster

1. Ve a **Clusters** → selecciona `demo-cluster`.
2. Haz clic en **Delete cluster** y confirma.

### 3. Eliminar el stack de CloudFormation

1. Ve a **CloudFormation** → selecciona el stack `ecs-lab-red-base`.
2. Haz clic en **Delete** y confirma.

> [!warning] Respeta este orden
> Si intentas eliminar el stack de CloudFormation (paso 3) **antes** de eliminar el service de ECS (paso 1), la eliminación va a fallar — la VPC todavía tendría el Load Balancer y las tasks usándola. Siempre limpia ECS primero.

---

## Solución de problemas

| Lo que ves | Causa probable | Qué revisar |
|---|---|---|
| El stack de CloudFormation termina en `CREATE_FAILED` o `ROLLBACK_COMPLETE` | Puede que ya exista un stack con el mismo nombre, o recursos con nombres en conflicto | Revisa la pestaña "Events" del stack para ver el mensaje de error específico |
| Las tasks quedan en `PROVISIONING` mucho tiempo o pasan a `STOPPED` | Falta la IP pública, o el security group no coincide | Revisa que en la Parte 4, paso 5, "Public IP" haya quedado en **On** |
| El navegador no carga nada en la URL del ALB | El balanceador todavía está inicializando | Espera 1-2 minutos más y refresca — los health checks tardan un poco en marcar las tasks como saludables |
| El target group muestra las tasks como "unhealthy" | El health check path o el security group no están bien configurados | Confirma que el health check path sea `/` y que elegiste el security group correcto (`FargateTaskSecurityGroupId`) para las tasks |
| No encuentras el `VpcId` o las subredes al crear el service | El stack de CloudFormation no terminó de crearse | Vuelve a la Parte 1.3 y confirma que el status sea `CREATE_COMPLETE` |
