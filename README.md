# MySQL EVENT Actividad

Haciendo uso de las siguientes tablas para la base de datos de `pizza` realice los siguientes ejercicios de `Events` centrados en el uso de **ON COMPLETION PRESERVE** y **ON COMPLETION NOT PRESERVE** :

**Tablas:**

```sql
CREATE TABLE IF NOT EXISTS resumen_ventas (
fecha       DATE      PRIMARY KEY,
total_pedidos INT,
total_ingresos DECIMAL(12,2),
creado_en DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS ingredientes(
	id INT UNSIGNED NOT NULL AUTO_INCREMENT,
    nombre VARCHAR(40),
    stock INT
);

CREATE TABLE IF NOT EXISTS alerta_stock (
  id              INT AUTO_INCREMENT PRIMARY KEY,
  ingrediente_id  INT UNSIGNED NOT NULL,
  stock_actual    INT NOT NULL,
  fecha_alerta    DATETIME NOT NULL,
  creado_en       DATETIME DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (ingrediente_id) REFERENCES ingredientes(id)
);

CREATE TABLE IF NOT EXISTS pedidos(
    id INT AUTO_INCREMENT PRIMARY KEY,
    monto_pedido DECIMAL(10,2)
);
```

1. Resumen Diario Único : crear un evento que genere un resumen de ventas **una sola vez** al finalizar el día de ayer y luego se elimine automáticamente llamado `ev_resumen_diario_unico`.

   ```sql
   DELIMITER //
   
   CREATE PROCEDURE sp_generar_resumen_diario()
   BEGIN
     INSERT INTO resumen_ventas (fecha, total_pedidos, total_ingresos)
     SELECT
       CURDATE() - INTERVAL 1 DAY,
       COUNT(*),
       SUM(monto_pedido)
     FROM pedidos
     WHERE DATE(NOW()) = CURDATE(); -- Simulación
   END;
   //
   
   DELIMITER ;
   
   DROP EVENT IF EXISTS ev_resumen_diario_unico;
   DELIMITER //
   
   CREATE EVENT ev_resumen_diario_unico
     ON SCHEDULE AT NOW() + INTERVAL 1 MINUTE
     ON COMPLETION NOT PRESERVE
   DO
   BEGIN
     CALL sp_generar_resumen_diario();
   END;
   //
   
   DELIMITER ;
   ```

   

2. Resumen Semanal Recurrente: cada lunes a las 01:00 AM, generar el total de pedidos e ingresos de la semana pasada, **manteniendo** el evento para que siga ejecutándose cada semana llamado `ev_resumen_semanal`.

   ```sql
   DROP EVENT IF EXISTS ev_resumen_semanal;
   DELIMITER //
   
   CREATE EVENT ev_resumen_semanal
     ON SCHEDULE
       EVERY 1 WEEK
       STARTS (CURRENT_DATE + INTERVAL (8 - DAYOFWEEK(CURRENT_DATE)) DAY + INTERVAL 1 HOUR)
     ON COMPLETION PRESERVE
   DO
   BEGIN
     INSERT INTO resumen_ventas (fecha, total_pedidos, total_ingresos)
     SELECT
       CURDATE(),
       COUNT(*),
       SUM(monto_pedido)
     FROM pedidos
     WHERE fecha BETWEEN CURDATE() - INTERVAL 7 DAY AND CURDATE();
   END;
   //
   
   DELIMITER ;
   ```

   

3. Alerta de Stock Bajo Única: en un futuro arranque del sistema (requerimiento del sistema), generar una única pasada de alertas (`alerta_stock`) de ingredientes con stock < 5, y luego autodestruir el evento.

   ```sql
   DROP EVENT IF EXISTS ev_alerta_stock_unica;
   DELIMITER //
   
   CREATE EVENT ev_alerta_stock_unica
     ON SCHEDULE AT NOW() + INTERVAL 1 MINUTE
     ON COMPLETION NOT PRESERVE
   DO
   BEGIN
     INSERT INTO alerta_stock (ingrediente_id, stock_actual, fecha_alerta)
     SELECT id, stock, NOW()
     FROM ingredientes
     WHERE stock < 5;
   END;
   //
   
   DELIMITER ;
   ```

   

4. Monitoreo Continuo de Stock: cada 30 minutos, revisar ingredientes con stock < 10 e insertar alertas en `alerta_stock`, **dejando** el evento activo para siempre llamado `ev_monitor_stock_bajo`.

   ```sql
   DROP EVENT IF EXISTS ev_monitor_stock_bajo;
   DELIMITER //
   
   CREATE EVENT ev_monitor_stock_bajo
     ON SCHEDULE EVERY 30 MINUTE
     STARTS NOW()
     ON COMPLETION PRESERVE
   DO
   BEGIN
     INSERT INTO alerta_stock (ingrediente_id, stock_actual, fecha_alerta)
     SELECT id, stock, NOW()
     FROM ingredientes
     WHERE stock < 10;
   END;
   //
   
   DELIMITER ;
   ```

   

5. Limpieza de Resúmenes Antiguos: una sola vez, eliminar de `resumen_ventas` los registros con fecha anterior a hace 365 días y luego borrar el evento llamado `ev_purgar_resumen_antiguo`.

   ```sql
   DROP EVENT IF EXISTS ev_purgar_resumen_antiguo;
   DELIMITER //
   
   CREATE EVENT ev_purgar_resumen_antiguo
     ON SCHEDULE AT NOW() + INTERVAL 1 MINUTE
     ON COMPLETION NOT PRESERVE
   DO
   BEGIN
     DELETE FROM resumen_ventas
     WHERE fecha < CURDATE() - INTERVAL 365 DAY;
   END;
   //
   
   DELIMITER ;
   ```

   
