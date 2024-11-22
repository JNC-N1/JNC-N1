## Structure
Client:
~~~
com.example
├── network # Chứa các lớp giao tiếp với server
│   ├── TCPClient.java # Lớp cơ bản kết nối server
│   ├── UserClient.java # Kế thừa từ lớp TCPClient tạo request
│   └── ...
├── models # Chứa các lớp model
│   ├── User.java # Lớp chứa thuộc tính của bảng User
│   └── ...
├── views # Chứa giao diện sử dụng các lớp Client để giao tiếp mới server
└── App.java # main

~~~
Server
~~~
server
├── config
│   ├── ConnectionPool.java # Tạo kết nối csdl
│   ├── TCPServer.java  # Lớp chính khởi chạy server
│   ├── ClientHandler.java # Dùng để sử lý client(Chạy trong Thread)
│   └── ...
├── controllers # Điểm khiển xử lý (Chủ yếu là gọi lại services)
│   ├── BaseController.java # Interface định nghĩa các hàm của controller
│   ├── UserController.java # Implement BaseController
│   └── ...
|   # Có thể dùng annotaion và không cần kế thừa Interface kia nhưng vẫn có nó
├── repositories # Xử lý giao tiếp với csdl, viết sql
│   ├── BasicRepository.java # Lớp cơ bản để sử các hàm tương tác với csdl
│   ├── UserRepository.java # Kế thừa BasicRepository.java
│   └── ...
├── services # Sử lý logic nghiệp vụ (Gọi tới repositoties)
│   ├── UserService.java
│   └── ...
├── models # Chứa các lớp model
│   ├── User.java # Lớp chứa thuộc tính của bảng User
│   └── ...
├── views # Có thể có (Chắc hiển thị danh sách client kết nối tới)
└── App.java # main
~~~
### Ghi chú
- models ở client với server nên giống nhau (cả package name, copy luôn từ server ra client luôn cũng đc)
- ConnectionPool giới hạn số lượng connection, các connection được để trong stack, khi lấy connection nếu stack không có connection thì tạo connection mới, dùng xong thì phải trả lại stack.
- Response nên có key `status` với dữ liệu là `success` với `error` hoặc gì cũng được miễn là dễ dàng sử lý lỗi.
- Request nên có một ket `action` để các định hàm nào của controller được sử dụng.

## Class
- Chưa test nên đây là ví dụ thôi
### TCPServer
```java
public class TCPServer {
    private static final int PORT = 12345;
    private final Map<String, BaseController.Handler> handlers = new HashMap<>();

    public TCPServer() {
        registerHandlers();
    }

    private void registerHandlers() {
        // Đăng ký các handler từ các controller
        UserController userController = new UserController();
        EventController eventController = new EventController();

        handlers.put("getUser", userController::getUser);
        handlers.put("updateUser", userController::updateUser);
        handlers.put("logEvent", eventController::logEvent);
    }

    public void start() {
        try (ServerSocket serverSocket = new ServerSocket(PORT)) {
            System.out.println("Server is running on port " + PORT);

            while (true) {
                Socket clientSocket = serverSocket.accept();
                new Thread(new ClientHandler(clientSocket, handlers)).start();
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```
### ClientHandler
```java
public class ClientHandler implements Runnable {
    private final Socket socket;
    private final Map<String, BaseController.Handler> handlers;

    public ClientHandler(Socket socket, Map<String, BaseController.Handler> handlers) {
        this.socket = socket;
        this.handlers = handlers;
    }

    @Override
    public void run() {
        try (
            ObjectInputStream input = new ObjectInputStream(socket.getInputStream());
            ObjectOutputStream output = new ObjectOutputStream(socket.getOutputStream())
        ) {
            
            Map<String, Object> request = (Map<String, Object>) input.readObject();

            String action = (String) request.get("action");
            if (action == null || !handlers.containsKey(action)) {
                output.writeObject(Map.of("status", "error", "message", "Unknown action"));
                return;
            }

            // Gọi handler tương ứng
            Map<String, Object> response = handlers.get(action).handle(request);
            output.writeObject(response);

        } catch (Exception e) { // Cái Exception còn lại hình như là cái ClassNotFound gì đó
            e.printStackTrace();
        }
    }
}
```
### BaseController
```java
public interface BaseController {
    @FunctionalInterface
    interface Handler {
        Map<String, Object> handle(Map<String, Object> request);
    }
}
```
### Nếu dùng annotation cho controller
### HandlerMapping
```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface HandlerMapping {
    String action();
}
```
- Controller thì cần thêm `@HandlerMapping(action = "<action-key>")` trước hàm là đc. Còn `<action-key>` thay thế thành cái gì có nghĩa là đc (Miễm là dể hiểu, dể nhớ - Client nó còn dùng tới).
- Ví dụ controller
```java
public class EventController {
    @HandlerMapping(action = "logEvent")
    public Map<String, Object> logEvent(Map<String, Object> request) {
        String event = (String) request.get("event");
        System.out.println("Event logged: " + event);

        return Map.of("status", "success", "message", "Event logged successfully");
    }
}
```
### Hàm đăng ký handler trong TCPServer
```java
private void registerHandlers() {
    // Danh sách các controller cần quét
    Object[] controllers = {
        new UserController(),
        new EventController()
    };

    for (Object controller : controllers) {
        for (Method method : controller.getClass().getDeclaredMethods()) {
            if (method.isAnnotationPresent(HandlerMapping.class)) {
                HandlerMapping annotation = method.getAnnotation(HandlerMapping.class);
                String action = annotation.action();

                // Đăng ký handler
                handlers.put(action, request -> {
                    try {
                        return (Map<String, Object>) method.invoke(controller, request);
                    } catch (Exception e) {
                        e.printStackTrace();
                        return Map.of("status", "error", "message", "Server error");
                    }
                });

                System.out.println("Registered action: " + action + " -> " + method.getName());
            }
        }
    }
}
```
### ConnectionPool
#### Nếu dùng  HikariCP (Tự động quản lý kết nối)
```java
public class ConnectionPool {
    private static HikariDataSource dataSource;

    static {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:mysql://localhost:3306/yourdb");  // Cấu hình URL kết nối
        config.setUsername("yourusername");
        config.setPassword("yourpassword");
        config.setMaximumPoolSize(10);  // Giới hạn số lượng kết nối trong pool

        dataSource = new HikariDataSource(config);
    }

    public static Connection getConnection() throws SQLException {
        if (dataSource == null) {
            throw new SQLException("Data source is not initialized.");
        }
        return dataSource.getConnection();
    }

    public static void close() {
        if (dataSource != null) {
            dataSource.close();
        }
    }
}
```
#### Nếu không dùng Hikari
```java
public class ConnectionPool {

    private static final int MAX_POOL_SIZE = 10;
    private static final Stack<Connection> pool = new Stack<>();
    private static final String DB_URL = "jdbc:mysql://localhost:3306/your_database";
    private static final String DB_USERNAME = "your_username";
    private static final String DB_PASSWORD = "your_password";

    // Tạo một kết nối mới
    private static Connection createConnection() throws SQLException {
        return DriverManager.getConnection(DB_URL, DB_USERNAME, DB_PASSWORD);
    }

    // Lấy một kết nối từ pool
    public static synchronized Connection getConnection() throws SQLException {
        synchronized (pool) {
            if (pool.isEmpty()) {
                return createConnection();
            } else {
                return pool.pop();
            }
        }
    }

    // Trả kết nối về pool
    public static synchronized void releaseConnection(Connection conn) {
        if (conn != null) {
            synchronized (pool) {
                if (pool.size() < MAX_POOL_SIZE) {
                    pool.push(conn);
                } else {
                    try {
                        conn.close();  // Nếu pool đầy, đóng kết nối
                    } catch (SQLException e) {
                        e.printStackTrace();
                    }
                }
            }
        }
    }

    // Đóng tất cả kết nối trong pool
    public static void close() {
        synchronized (connectionPool) {
            for (Connection conn : connectionPool) {
                try {
                    conn.close();
                } catch (SQLException e) {
                    e.printStackTrace();
                }
            }
        }
    }
}
```
- close gọi khi kết thúc chương trình
### BasicController
```java
public abstract class BasicRepository {
    protected Connection conn;

    public BasicRepository() throws SQLException {
        this.conn = ConnectionPool.getConnection();  // Lấy kết nối từ pool
        this.conn.setAutoCommit(false);
    }

    // Phương thức thực thi câu lệnh thằng này cần thêm Trasation với commit
    public boolean exe(PreparedStatement pre) {
        try {
            int affectedRows = pre.executeUpdate();
            this.conn.commit();
            return affectedRows > 0;
        } catch (SQLException e) {
            e.printStackTrace();
            try {
                this.conn.rollback();  // Rollback nếu có lỗi
            } catch (SQLException rollbackEx) {
                rollbackEx.printStackTrace();
            }
        }
        return false;
    }

    public ResultSet get(PreparedStatement pre) {
        try {
            return pre.executeQuery();
        } catch (SQLException e) {
            e.printStackTrace();
            return null;
        }
    }

    
    public void releaseConnection() {
        // Nếu không dùng HikariCP
        try {
            if (conn != null && !conn.isClosed()) {
                conn.setAutoCommit(true);
                ConnectionPool.releaseConnection(conn);
            }
        } catch (SQLException e) {
            e.printStackTrace();
        }


        // Nếu dùng HikariCP
        try {
            if (conn != null && !conn.isClosed()) {
                conn.setAutoCommit(true);
                conn.close();
            }
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }
}
```
