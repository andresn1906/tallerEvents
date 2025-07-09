# ✏️ Actividad:
Haciendo uso de las siguientes tablas para la base de datos de pizza realice los siguientes ejercicios de Events centrados en el uso de *ON COMPLETION PRESERVE* y *ON COMPLETION NOT PRESERVE*:
```sql
CREATE DATABASE actividadevents;
USE actividadevents;

CREATE TABLE IF NOT EXISTS resumen_ventas (
  fecha DATE PRIMARY KEY,
  total_pedidos INT,
  total_ingresos DECIMAL(12,2),
  creado_en DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS ingrediente (
  id INT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  nombre VARCHAR(100) NOT NULL,
  stock INT NOT NULL
);

CREATE TABLE IF NOT EXISTS pedido (
  id INT PRIMARY KEY AUTO_INCREMENT,
  fecha_pedido DATETIME NOT NULL,
  total DECIMAL(12, 2) NOT NULL
);

CREATE TABLE IF NOT EXISTS alerta_stock (
  id INT AUTO_INCREMENT PRIMARY KEY,
  ingrediente_id INT UNSIGNED NOT NULL,
  stock_actual INT NOT NULL,
  fecha_alerta DATETIME NOT NULL,
  creado_en DATETIME DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (ingrediente_id) REFERENCES ingrediente(id)
);
```
1- Resumen Diario Único: crear un evento que genere un resumen de ventas una sola vez al finalizar el día de ayer y luego se elimine automáticamente llamado *ev_resumen_diario_unico*.

```sql
INSERT INTO pedido (fecha_pedido, total) VALUES
(NOW() - INTERVAL 1 DAY, 150.00),
(NOW() - INTERVAL 1 DAY, 230.00),
(NOW() - INTERVAL 1 DAY, 180.00);

DELIMITER //

DROP EVENT IF EXISTS ev_resumen_diario_unico;
CREATE EVENT ev_resumen_diario_unico
ON SCHEDULE AT CURRENT_TIMESTAMP
ON COMPLETION NOT PRESERVE
COMMENT 'Resumen de ventas diarias'
DO
BEGIN
  DECLARE v_total_pedidos INT DEFAULT 0;
  DECLARE v_total_ingresos DECIMAL(12,2) DEFAULT 0.00;

  SELECT COUNT(*), SUM(total)
    INTO v_total_pedidos, v_total_ingresos
  FROM pedido
  WHERE DATE(fecha_pedido) = CURDATE() - INTERVAL 1 DAY;

  INSERT INTO resumen_ventas (fecha, total_pedidos, total_ingresos) VALUES
  (CURDATE() - INTERVAL 1 DAY, 
  IFNULL(v_total_pedidos, 0),
  IFNULL(v_total_ingresos, 0.00))
ON DUPLICATE KEY UPDATE
  total_pedidos = VALUES(total_pedidos),
  total_ingresos = VALUES(total_ingresos);

END //

DELIMITER ;
```
#### Pruebas:
```sql
SHOW EVENTS LIKE 'ev_resumen_diario_unico';

SELECT * FROM resumen_ventas;
```

2- Resumen Semanal Recurrente: cada lunes a las 01:00 AM, generar el total de pedidos e ingresos de la semana pasada, manteniendo el evento para que siga ejecutándose cada semana llamado *ev_resumen_semanal*.
```sql
TRUNCATE TABLE pedido;
TRUNCATE TABLE resumen_ventas;
-- Se usa el TRUNCATE para eliminar los datos de las tablas sin eliminar las "tablas" y tener que crearlas de nuevo.

INSERT INTO pedido (fecha_pedido, total) VALUES
(CURDATE() - INTERVAL 1 WEEK - INTERVAL WEEKDAY(CURDATE()) DAY, 1500.00),
(CURDATE() - INTERVAL 1 WEEK - INTERVAL WEEKDAY(CURDATE()) DAY + INTERVAL 3 DAY, 2500.00), 
(CURDATE() - INTERVAL 1 WEEK - INTERVAL WEEKDAY(CURDATE()) DAY + INTERVAL 6 DAY, 3000.00); 

DELIMITER //

DROP EVENT IF EXISTS ev_resumen_semanal;
CREATE EVENT ev_resumen_semanal
ON SCHEDULE 
  EVERY 1 WEEK
  STARTS (CURRENT_DATE + INTERVAL ((9 - DAYOFWEEK(CURRENT_DATE)) % 7) DAY + INTERVAL 1 HOUR)
ON COMPLETION PRESERVE
COMMENT 'Resumen semanal de ventas'
DO
BEGIN
  DECLARE v_total_pedidos INT DEFAULT 0;
  DECLARE v_total_ingresos DECIMAL(12,2) DEFAULT 0.00;

  SELECT COUNT(*), SUM(total)
    INTO v_total_pedidos, v_total_ingresos
  FROM pedido
  WHERE fecha_pedido >= CURDATE() - INTERVAL 7 DAY AND fecha_pedido < CURDATE();

  INSERT INTO resumen_ventas (fecha, total_pedidos, total_ingresos)
  VALUES (CURDATE() - INTERVAL 7 DAY, 
    IFNULL(v_total_pedidos,0),
    IFNULL(v_total_ingresos,0.00))
  ON DUPLICATE KEY UPDATE
    total_pedidos = VALUES(total_pedidos),
    total_ingresos = VALUES(total_ingresos);
END //

DELIMITER ;
```
### Pruebas:
```sql
SHOW EVENTS LIKE 'ev_resumen_semanal';

SELECT * FROM resumen_ventas;
```

3- Alerta de Stock Bajo Única: en un futuro arranque del sistema (requerimiento del sistema), generar una única pasada de alertas (alerta_stock) de ingredientes con stock < 5, y luego autodestruir el evento.
```sql
INSERT INTO ingrediente (nombre, stock) VALUES 
('Harina', 10),    
('Azúcar', 4),
('Huevos', 2),
('Leche', 5),
('Mantequilla', 3),
('Sal', 8),
('Pimienta', 1);

DELIMITER //

DROP EVENT IF EXISTS ev_alerta_stock_unica;
CREATE EVENT ev_alerta_stock_unica
ON SCHEDULE AT NOW() + INTERVAL 2 MINUTE
ON COMPLETION NOT PRESERVE
COMMENT 'Alerta de ingredintes con un stock menor a 5'
DO
BEGIN
  INSERT INTO alerta_stock (ingrediente_id, stock_actual, fecha_alerta)
  SELECT id, stock, NOW()
  FROM ingrediente
  WHERE stock < 5;
END //

DELIMITER ;
```
### Pruebas:
```sql
SHOW EVENTS LIKE 'ev_alerta_stock_unica';

SELECT * FROM alerta_stock;
```

4- Monitoreo Continuo de Stock: cada 30 minutos, revisar ingredientes con stock < 10 e insertar alertas en alerta_stock, dejando el evento activo para siempre llamado *ev_monitor_stock_bajo*.
```sql
DELIMITER //

DROP EVENT IF EXISTS ev_monitor_stock_bajo;
CREATE EVENT ev_monitor_stock_bajo
ON SCHEDULE EVERY 30 MINUTE
ON COMPLETION PRESERVE
COMMENT 'Monitoreo de stock bajo'
DO
BEGIN
  INSERT INTO alerta_stock (ingrediente_id, stock_actual, fecha_alerta)
  SELECT id, stock, NOW()
  FROM ingrediente
  WHERE stock < 10;
END //

DELIMITER ;
```
### Pruebas:
```sql
SHOW EVENTS LIKE 'ev_monitor_stock_bajo';

SELECT * FROM alerta_stock;
```

5- Limpieza de Resúmenes Antiguos: una sola vez, eliminar de resumen_ventas los registros con fecha anterior a hace 365 días y luego borrar el evento llamado *ev_purgar_resumen_antiguo*.
```sql
DELIMITER //

DROP EVENT IF EXISTS ev_purgar_resumen_antiguo;
CREATE EVENT ev_purgar_resumen_antiguo
ON SCHEDULE AT CURRENT_TIMESTAMP
ON COMPLETION NOT PRESERVE
COMMENT 'Eliminar resumen de ventas'
DO
BEGIN
  DELETE FROM resumen_ventas
  WHERE fecha < CURDATE() - INTERVAL 365 DAY;
END //

DELIMITER ;
```
### Pruebas:
```sql
SHOW EVENTS LIKE 'ev_purgar_resumen_antiguo';

SELECT * FROM resumen_ventas;
```

