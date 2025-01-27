# Integración Cross-Account de EventBridge

Esta guía te ayudará a configurar el envío de eventos entre dos cuentas AWS usando EventBridge.

## Diagrama de Arquitectura

```
Cuenta A (Productor) -> EventBridge -> Event Bus de Cuenta B -> Targets en Cuenta B (SQS, Lambda, etc.)
```

## Pasos Detallados:

### En la Cuenta B (Cuenta Receptora):

1. Primero, crea un Event Bus personalizado:

```bash
# Usando AWS CLI desde la Cuenta B
aws events create-event-bus --name "incoming-events-from-account-a"
```

2. Agrega permisos para permitir que la Cuenta A envíe eventos:

```bash
# En Cuenta B
aws events put-permission \
    --event-bus-name "incoming-events-from-account-a" \
    --action events:PutEvents \
    --principal "{ID-Cuenta-A}" \
    --statement-id "AcceptEventsFromAccountA"
```

O mediante un documento de política:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowAccountAToInvokeThisEventBus",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::ID-CUENTA-A:root"
      },
      "Action": "events:PutEvents",
      "Resource": "arn:aws:events:region:ID-CUENTA-B:event-bus/incoming-events-from-account-a"
    }
  ]
}
```

3. Crea reglas en la Cuenta B para procesar los eventos:

```bash
aws events put-rule \
    --event-bus-name "incoming-events-from-account-a" \
    --name "process-account-a-events" \
    --event-pattern '{
        "source": ["com.myapp.orders"],
        "detail-type": ["OrderCreated"]
    }'
```

### En la Cuenta A (Cuenta Emisora):

1. Crea una regla para reenviar eventos:

```bash
aws events put-rule \
    --name "forward-to-account-b" \
    --event-pattern '{
        "source": ["com.myapp.orders"],
        "detail-type": ["OrderCreated"]
    }'
```

2. Agrega el event bus de la Cuenta B como target:

```bash
aws events put-targets \
    --rule "forward-to-account-b" \
    --targets "Id"="1","Arn"="arn:aws:events:region:ID-CUENTA-B:event-bus/incoming-events-from-account-a"
```

3. Importante: Agrega un Rol IAM en la Cuenta A
   El rol necesita permisos para enviar eventos al event bus de la Cuenta B:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "events:PutEvents",
      "Resource": "arn:aws:events:region:ID-CUENTA-B:event-bus/incoming-events-from-account-a"
    }
  ]
}
```

## Configuración usando la Consola AWS:

### En la Cuenta B:

1. Ve a EventBridge
2. Crea un event bus personalizado
3. Agrega permisos para la Cuenta A
4. Crea reglas para procesar los eventos entrantes

### En la Cuenta A:

1. Ve a EventBridge
2. Crea una regla
3. Configura el patrón de eventos
4. Agrega target -> selecciona "Event bus in another AWS account"
5. Ingresa el ARN del event bus de la Cuenta B
6. Crea/selecciona un rol IAM con los permisos adecuados

## Pruebas:

Desde la Cuenta A, envía un evento de prueba:

```bash
aws events put-events --entries '[
    {
        "Source": "com.myapp.orders",
        "DetailType": "OrderCreated",
        "Detail": "{\"orderId\": \"123\"}",
        "EventBusName": "default"
    }
]'
```

## Mejores Prácticas:

1. Seguridad:

   - Usa el principio de mínimo privilegio para los permisos
   - Considera usar AWS Organizations para un control más granular
   - Audita regularmente el acceso entre cuentas

2. Monitoreo:

   - Configura métricas de CloudWatch en ambas cuentas
   - Monitorea las entregas fallidas
   - Configura alarmas para eventos críticos

3. Manejo de Errores:

   - Usa DLQs en la Cuenta B para procesamientos fallidos
   - Implementa mecanismos de reintento
   - Registra eventos fallidos para solución de problemas

4. Gestión de Costos:
   - Monitorea el uso de eventos en ambas cuentas
   - Configura alertas de facturación
   - Considera usar filtrado de eventos para reducir eventos innecesarios

## Usos Comunes:

- Logging centralizado
- Arquitecturas multi-cuenta
- Separación de ambientes de desarrollo y producción
- Gestión de eventos organizacionales
- Monitoreo y alertas entre cuentas

## Importante:

Los eventos se cobran en ambas cuentas:

- La Cuenta A paga por enviar eventos
- La Cuenta B paga por recibir y procesar eventos

Recuerda reemplazar los siguientes valores en los ejemplos:

- `ID-CUENTA-A`: El ID de 12 dígitos de la cuenta emisora
- `ID-CUENTA-B`: El ID de 12 dígitos de la cuenta receptora
- `region`: La región de AWS donde estás trabajando (ej: us-east-1)
