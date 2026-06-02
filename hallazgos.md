Reporte de Auditoría de Arquitectura y Código - Staff Engineer

  A continuación se presenta el diagnóstico técnico del sistema de
  calificaciones. Este reporte se enfoca en fallas estructurales, riesgos de
  escalabilidad y deuda técnica crítica.

  1. Diagnóstico de Arquitectura y Puntos Críticos

  Fallas de Diseño y Arquitectura
   * Ausencia de Front Controller (Riesgo de Seguridad y Mantenibilidad): El
     sistema utiliza un patrón de "Page Controller" disperso. Archivos como
     api/getMateriasByPnf.php o controladores/authUsuario/login.php son
     accesibles directamente. Esto obliga a repetir lógica de seguridad (CSRF,
     Auth, Sesiones) en cada archivo, aumentando la superficie de ataque y
     dificultando actualizaciones globales.
   * Inexistencia de Capa de Abstracción de Datos (DB Leak): La función
     conectar() en config/conexion.php es una falla de diseño crítica. Crea una
     nueva instancia PDO cada vez que se invoca. En un entorno de producción con
     tráfico concurrente, esto provocará el error Too many connections al agotar
     el pool de hilos del servidor MySQL.
   * Acoplamiento de Reglas de Negocio (Hardcoding): La lógica de aprobación
     (Proyecto ≥ 16 vs Regular ≥ 12) está incrustada en CalificacionService.php
     mediante comparaciones de strings (strpos). Esto hace que el sistema sea
     extremadamente frágil ante cambios en el currículo académico.

  Código Duplicado y Cuellos de Botella
   * Lógica de Sesión y CSRF: Cada controlador implementa manualmente el bloque
     de validación de tokens y verificación de tiempo de actividad. Esta lógica
     debería ser un Middleware o parte del ciclo de vida del Front Controller.
   * Manejo de Errores Inconsistente: Algunos archivos devuelven JSON con
     códigos HTTP 500, otros redirigen con header() y parámetros en la URL. No
     hay una estrategia unificada de manejo de excepciones.

  Riesgos de Escalabilidad
   * Persistencia de Sesión: El uso de session_start() basado en archivos
     (default de PHP) impedirá escalar el sistema horizontalmente (balanceo de
     carga entre múltiples servidores) sin usar un almacén externo como Redis.
   * Transacciones Atómicas: Aunque se usan transacciones en servicios, la
     lógica de validación previa se hace fuera de estas, lo que permite
     condiciones de carrera (Race Conditions) en estados de inscripción.

  ---

  2. Estrategia de Refactorización

   1. Capa de Persistencia (Singleton/DI): Migrar la conexión a una clase
      Singleton para asegurar una única conexión por Request.
   2. Unificación de Entrada: Implementar un Enrutador básico para eliminar el
      acceso directo a archivos en controladores/ y api/.
   3. Desacoplamiento de Reglas de Dominio: Extraer los umbrales de aprobación a
      una configuración o tabla de parámetros para evitar el uso de strpos en
      los servicios.
   4. Middleware de Seguridad: Centralizar la validación CSRF y de sesión.

  ---

  3. Código Optimizado (Producción Ready)

  A. Persistencia Escalable (config/Database.php)
  Reemplaza el modelo procedimental por un Singleton con manejo de errores
  centralizado.

    1 <?php
    2 namespace App\Config;
    3
    4 use PDO;
    5 use PDOException;
    6 use Exception;
    7
    8 class Database {
    9     private static ?PDO $instance = null;
   10
   11     private function __construct() {}
   12
   13     public static function getConnection(): PDO {
   14         if (self::$instance === null) {
   15             $host = $_ENV['DB_HOST'] ?? getenv('DB_HOST');
   16             $db   = $_ENV['DB_NAME'] ?? getenv('DB_NAME');
   17             $user = $_ENV['DB_USER'] ?? getenv('DB_USER');
   18             $pass = $_ENV['DB_PASS'] ?? getenv('DB_PASS');
   19
   20             $dsn = "mysql:host=$host;dbname=$db;charset=utf8mb4";
   21             $options = [
   22                 PDO::ATTR_ERRMODE            => PDO::ERRMODE_EXCEPTION,
   23                 PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC,
   24                 PDO::ATTR_EMULATE_PREPARES   => false,
   25                 PDO::ATTR_PERSISTENT         => true // Mejora rendimiento
      en reconexiones
   26             ];
   27
   28             try {
   29                 self::$instance = new PDO($dsn, $user, $pass, $options);
   30             } catch (PDOException $e) {
   31                 error_log("DB Error: " . $e->getMessage());
   32                 throw new Exception("Error interno de persistencia.");
   33             }
   34         }
   35         return self::$instance;
   36     }
   37 }

  B. Servicio de Calificaciones Refactorizado
  (servicios/CalificacionService.php)
  Eliminación de lógica hardcoded y mejora de integridad transaccional.

    1 <?php
    2 namespace App\Services;
    3
    4 use App\Config\Database;
    5 use Exception;
    6 use PDO;
    7
    8 class CalificacionService {
    9     private PDO $db;
   10     
   11     // Constantes para evitar Hardcoding
   12     private const MATERIA_PROYECTO = 'proyecto socio tecnológico';
   13     private const MIN_NOTA_PROYECTO = 16;
   14     private const MIN_NOTA_REGULAR = 12;
   15
   16     public function __construct() {
   17         $this->db = Database::getConnection();
   18     }
   19
   20     private function determinarEstatus(string $nombreMateria, float
      $nota): string {
   21         $esProyecto = str_contains(strtolower($nombreMateria),
      self::MATERIA_PROYECTO);
   22         $minimo = $esProyecto ? self::MIN_NOTA_PROYECTO :
      self::MIN_NOTA_REGULAR;
   23         return ($nota >= $minimo) ? 'Aprobada' : 'Reprobada';
   24     }
   25
   26     public function registrarNota(int $inscripcionId, float $nota, string
      $periodo, int $usuarioId): array {
   27         try {
   28             $this->db->beginTransaction();
   29
   30             // Lock para lectura para evitar inconsistencias en
      concurrencia
   31             $stmt = $this->db->prepare("
   32                 SELECT m.nombre FROM inscripciones i 
   33                 JOIN secciones s ON i.seccion_id = s.id 
   34                 JOIN materias m ON s.materia_id = m.id 
   35                 WHERE i.id = ? FOR UPDATE
   36             ");
   37             $stmt->execute([$inscripcionId]);
   38             $materia = $stmt->fetch();
   39
   40             if (!$materia) throw new Exception("Inscripción no válida.");
   41
   42             $estatus = $this->determinarEstatus($materia['nombre'],
      $nota);
   43
   44             // Upsert atómico
   45             $sql = "INSERT INTO calificaciones (inscripcion_id,
      nota_numerica, periodo_academico, usuario_id, estado_materia) 
   46                     VALUES (?, ?, ?, ?, 'Finalizada')
   47                     ON DUPLICATE KEY UPDATE nota_numerica =
      VALUES(nota_numerica)";
   48             $this->db->prepare($sql)->execute([$inscripcionId, $nota,
      $periodo, $usuarioId]);
   49
   50             // Actualización de estado de inscripción
   51             $this->db->prepare("UPDATE inscripciones SET estatus = ? WHERE
      id = ?")
   52                      ->execute([$estatus, $inscripcionId]);
   53
   54             $this->db->commit();
   55             return ['status' => 'success', 'estatus_final' => $estatus];
   56
   57         } catch (Exception $e) {
   58             if ($this->db->inTransaction()) $this->db->rollBack();
   59             throw $e;
   60         }
   61     }
   62 }

  C. Controlador de Autenticación (Limpieza de Lógica)
  Separación de la lógica de respuesta HTTP de la validación de seguridad.

    1 <?php
    2 // Simplificación drástica para evitar lógica procedural pesada
    3 require_once '../../vendor/autoload.php';
    4
    5 try {
    6     // 1. Validaciones de infraestructura (Centralizar en Middleware en el
      futuro)
    7     SecurityMiddleware::validateCSRF($_POST['csrf_token'] ?? '');
    8     
    9     // 2. Ejecución de lógica vía Servicio
   10     $authService = new AuthService(Database::getConnection());
   11     $user = $authService->authenticate($_POST['cedula'], $_POST['clave']);
   12
   13     // 3. Respuesta limpia
   14     session_regenerate_id(true);
   15     $_SESSION['user'] = $user;
   16     header("Location: ../../vistas/home.php");
   17
   18 } catch (AuthException $e) {
   19     $_SESSION['error'] = $e->getMessage();
   20     header("Location: ../../index.php?error=1");
   21 } catch (Exception $e) {
   22     error_log($e->getMessage());
   23     header("Location: ../../index.php?error=internal");
   24 }
