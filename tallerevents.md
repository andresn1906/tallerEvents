# ✏️ Actividad:
Haciendo uso de las siguientes tablas para la base de datos de pizza realice los siguientes ejercicios de Events centrados en el uso de *ON COMPLETION PRESERVE* y *ON COMPLETION NOT PRESERVE* :
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
DELIMITER //

DROP EVENT IF EXISTS ev_resumen_diario_unico;
CREATE EVENT ev_resumen_diario_unico
ON SCHEDULE AT NOW() + INTERVAL 1 MINUTE
ON COMPLETION NOT PRESERVE
COMMENT 'Resumen de ventas diarias'
DO
BEGIN
  DECLARE v_total_pedidos INT DEFAULT 0;
  DECLARE v_total_ingresos DECIMAL(12,2) DEFAULT 0.00;

  SELECT COUNT(*), SUM(total)
    INTO v_total_pedidos, v_total_ingresos
  FROM pedido
  WHERE DATE(fecha) = CURDATE() - INTERVAL 1 DAY;

  INSERT INTO resumen_ventas (fecha, total_pedidos, total_ingresos) VALUES
  (CURDATE() - INTERVAL 1 DAY, 
  IFNULL(v_total_pedidos, 0),
  IFNULL(v_total_ingresos, 0.00))
ON DUPLICATE KEY UPDATE
  total_pedidos = VALUES(total_pedidos),
  total_ingresos = VALUES(total_ingresos);

END //

DELIMITER ;


INSERT INTO pedido (fecha_pedido, total) VALUES
(NOW() - INTERVAL 1 DAY, 130.00),
(NOW() - INTERVAL 1 DAY, 510.00),
(NOW(), 230.00);
```

2- Resumen Semanal Recurrente: cada lunes a las 01:00 AM, generar el total de pedidos e ingresos de la semana pasada, manteniendo el evento para que siga ejecutándose cada semana llamado *ev_resumen_semanal*.
3- Alerta de Stock Bajo Única: en un futuro arranque del sistema (requerimiento del sistema), generar una única pasada de alertas (alerta_stock) de ingredientes con stock < 5, y luego autodestruir el evento.
4- Monitoreo Continuo de Stock: cada 30 minutos, revisar ingredientes con stock < 10 e insertar alertas en alerta_stock, dejando el evento activo para siempre llamado *ev_monitor_stock_bajo*.
5- Limpieza de Resúmenes Antiguos: una sola vez, eliminar de resumen_ventas los registros con fecha anterior a hace 365 días y luego borrar el evento llamado *ev_purgar_resumen_antiguo*.

