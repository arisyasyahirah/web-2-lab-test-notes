# web-2-lab-test-notes

-- STEP 1

CREATE DATABASE lab7_db;
USE lab7_db;
CREATE TABLE students (
  id INT AUTO_INCREMENT PRIMARY KEY,
  matric_no VARCHAR(20) UNIQUE NOT NULL,
  password VARCHAR(50) NOT NULL,
  fullname VARCHAR(100) NOT NULL,
  profile_image MEDIUMBLOB
);

-- step 2
open netbeans, make new project
new project, web applicationi, name it (lab test), set server as tomcat
!!! add sql connector jar to libraries

-- Step 3 
Create packages and Model first.
Right-click Source Packages → New Package: com.lab.bean

Inside it make a New Java Class called StudentBean
(this has matricNo, fullname, password, base64Image, all with getters/setters.)
(start here)
package com.lab.bean;

public class StudentBean implements java.io.Serializable {
    private String matricNo;
    private String fullname;
    private String password;
    private String base64Image;

    public StudentBean() {}

    public String getMatricNo() { return matricNo; }
    public void setMatricNo(String matricNo) { this.matricNo = matricNo; }

    public String getFullname() { return fullname; }
    public void setFullname(String fullname) { this.fullname = fullname; }

    public String getPassword() { return password; }
    public void setPassword(String password) { this.password = password; }

    public String getBase64Image() { return base64Image; }
    public void setBase64Image(String base64Image) { this.base64Image = base64Image; }
}


-- setp 4
Create DAO (database logic).

New Package: com.lab.dao
New Class: StudentDAO — three methods: 
-registerStudent() (INSERT with BLOB), 
-loginStudent() (SELECT, converts BLOB→Base64), 
-deleteStudent() (DELETE).

remember getConnection() uses com.mysql.cj.jdbc.Driver 
and jdbc:mysql://localhost:3306/lab7_db.

(start here)
package com.lab.dao;

import com.lab.bean.StudentBean;
import java.io.ByteArrayOutputStream;
import java.io.InputStream;
import java.sql.*;
import java.util.Base64;

public class StudentDAO {

    private Connection getConnection() throws Exception {
        Class.forName("com.mysql.cj.jdbc.Driver");
        return DriverManager.getConnection(
            "jdbc:mysql://localhost:3306/lab7_db", "root", "");
    }

    // CREATE REGISTER STUDENT W IMAGE
    public boolean registerStudent(StudentBean student, InputStream imageStream) {
        try (Connection conn = getConnection()) {
            String sql = "INSERT INTO students " +
                         "(matric_no, password, fullname, profile_image) " +
                         "VALUES (?, ?, ?, ?)";
            PreparedStatement pstmt = conn.prepareStatement(sql);
            pstmt.setString(1, student.getMatricNo());
            pstmt.setString(2, student.getPassword());
            pstmt.setString(3, student.getFullname());
            pstmt.setBlob(4, imageStream);
            return pstmt.executeUpdate() > 0;
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }
    }

    // READ LOGIN FETCH STUDENT + CONVERT BLOB TO BASE 64
    public StudentBean loginStudent(String matricNo, String password) {
        StudentBean student = null;
        try (Connection conn = getConnection()) {
            String sql = "SELECT * FROM students " +
                         "WHERE matric_no = ? AND password = ?";
            PreparedStatement pstmt = conn.prepareStatement(sql);
            pstmt.setString(1, matricNo);
            pstmt.setString(2, password);
            ResultSet rs = pstmt.executeQuery();

            if (rs.next()) {
                student = new StudentBean();
                student.setMatricNo(rs.getString("matric_no"));
                student.setFullname(rs.getString("fullname"));

                // Convert BLOB → Base64
                Blob blob = rs.getBlob("profile_image");
                if (blob != null) {
                    InputStream is = blob.getBinaryStream();
                    ByteArrayOutputStream baos = new ByteArrayOutputStream();
                    byte[] buffer = new byte[4096];
                    int bytesRead;
                    while ((bytesRead = is.read(buffer)) != -1) {
                        baos.write(buffer, 0, bytesRead);
                    }
                    String base64 = Base64.getEncoder()
                                         .encodeToString(baos.toByteArray());
                    student.setBase64Image(base64);
                }
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
        return student;
    }

    // DELETE REMOVE STUDENT ACCOUNT
    public boolean deleteStudent(String matricNo) {
        try (Connection conn = getConnection()) {
            String sql = "DELETE FROM students WHERE matric_no = ?";
            PreparedStatement pstmt = conn.prepareStatement(sql);
            pstmt.setString(1, matricNo);
            return pstmt.executeUpdate() > 0;
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }
    }
}

-- step 5 
Create Controller (Servlet).

New Package: com.lab.controller
New Servlet: UserServlet. CRITICAL: when the wizard asks, CHECK "Add information to deployment descriptor (web.xml)" — this auto-creates the web.xml mapping instead of using @WebServlet.
Manually add @MultipartConfig(maxFileSize = 16177215) above the class — needed for file upload.
doPost() reads action parameter: if "register" → build StudentBean, get file Part from profileImage, call registerStudent(). If "login" → call loginStudent(), if success store bean in session (session.setAttribute("loggedUser", student)) and redirect to dashboard.jsp.
doGet() handles action=logout (session.invalidate()) and action=delete (delete from DB then invalidate).

📁 Source Packages > com.lab.controller > UserServlet.java
⚠️ When creating: CHECK "Add to web.xml" in wizard. DELETE any @WebServlet annotation generated.
(start here)
package com.lab.controller;

import com.lab.bean.StudentBean;
import com.lab.dao.StudentDAO;
import java.io.IOException;
import java.io.InputStream;
import javax.servlet.ServletException;
import javax.servlet.annotation.MultipartConfig;
import javax.servlet.http.*;
(javax= javaEE, jakarta=jakartaEE)

@MultipartConfig(maxFileSize = 16177215)  // MUST have for file upload
public class UserServlet extends HttpServlet {
 -1. create this here first

    private StudentDAO studentDAO = new StudentDAO();

    @Override
    protected void doPost(HttpServletRequest request,
                          HttpServletResponse response)
            throws ServletException, IOException {

        response.setContentType("text/html;charset=UTF-8");
        // read the hidden parameter to decide which action to execute
        String action = request.getParameter("action");

        if ("register".equals(action)) {

        //1. HANDLES REGISTRATION
            StudentBean newStudent = new StudentBean();
            newStudent.setMatricNo(request.getParameter("matricNo"));
            newStudent.setFullname(request.getParameter("fullname"));
            newStudent.setPassword(request.getParameter("password"));

            InputStream inputStream = null;
            Part filePart = request.getPart("profileImage");
            if (filePart != null && filePart.getSize() > 0) {
                inputStream = filePart.getInputStream();
            }

            boolean success = studentDAO.registerStudent(newStudent, inputStream);
            if (success) {
                response.getWriter().println(
                    "Registration Successful! <a href='login.html'>Login</a>");
            } else {
                response.getWriter().println("Registration Failed!");
            }
//2. HANLDLES LOGIN

        } else if ("login".equals(action)) {
            String matric = request.getParameter("matricNo");
            String pass   = request.getParameter("password");
            StudentBean student = studentDAO.loginStudent(matric, pass);

            if (student != null) {
// LOGIN SUCCESSFUL : STORE THE ENTIRE BEAN IN THE SESSION

                HttpSession session = request.getSession();
                session.setAttribute("loggedUser", student);  // Store in session
                response.sendRedirect("dashboard.jsp");
            } else {
                response.getWriter().println(
                    "Invalid Credentials! <a href='login.html'>Try Again</a>");
            }
        }
    }

    @Override
    protected void doGet(HttpServletRequest request,
                         HttpServletResponse response)
            throws ServletException, IOException {

        String action = request.getParameter("action");
        HttpSession session = request.getSession(false);

        if ("logout".equals(action)) {
// 3. HABDLES LOGOUT

            if (session != null) session.invalidate();
            response.sendRedirect("login.html");

        } else if ("delete".equals(action)) {
//4. HANDLES DELETE ACCOUNT

            if (session != null && session.getAttribute("loggedUser") != null) {
                StudentBean user = (StudentBean) session.getAttribute("loggedUser");
                studentDAO.deleteStudent(user.getMatricNo());
                session.invalidate();
            }
            response.sendRedirect("register.html");
        }
    }
}

-- step 6 📁 Web Pages > WEB-INF > web.xml — make sure this exists after wizard:
(tick the box and delete the last box leaving / only)
<servlet>
    <servlet-name>UserServlet</servlet-name>
    <servlet-class>com.lab.controller.UserServlet</servlet-class>
</servlet>
<servlet-mapping>
    <servlet-name>UserServlet</servlet-name>
    <url-pattern>/</url-pattern>
</servlet-mapping>

-- step 7 — Create the Views (HTML/JSP), last.

register.html — form with enctype="multipart/form-data", hidden <input type="hidden" name="action" value="register">, fields matricNo, fullname, password, file input profileImage.

(start here)
<!DOCTYPE html>
<html>
<head><title>Register</title></head>
<body>
<h2>Register New Student</h2>
<form action="UserServlet" method="POST" enctype="multipart/form-data">
    <input type="hidden" name="action" value="register">

    Matric No: <input type="text" name="matricNo" required><br><br>
    Full Name: <input type="text" name="fullname" required><br><br>
    Password:  <input type="password" name="password" required><br><br>
    Profile Picture: <input type="file" name="profileImage"
                            accept="image/*" required><br><br>
    <input type="submit" value="Register">
</form>
</body>
</html>

login.html — form posting to UserServlet, hidden action=login.
<!DOCTYPE html>
<html>
<body>
<h2>Student Login</h2>
<form action="UserServlet" method="POST">
    <input type="hidden" name="action" value="login">

    Matric No: <input type="text"     name="matricNo" required><br><br>
    Password:  <input type="password" name="password" required><br><br>
    <input type="submit" value="Login">
</form>
</body>
</html>

dashboard.jsp — scriptlet checks session.getAttribute("loggedUser"), if null redirect to login. Displays <%= user.getFullname() %> and image via data:image/jpeg;base64,<%= user.getBase64Image() %>. Logout/Delete links go to UserServlet?action=logout / ?action=delete.
<%@page contentType="text/html" pageEncoding="UTF-8"%>
<%@page import="com.lab.bean.StudentBean"%>
<!DOCTYPE html>
<html>
<head><title>Dashboard</title></head>
<body>

<%
    // Session security check — redirect if not logged in
    StudentBean user = (StudentBean) session.getAttribute("loggedUser");
    if (user == null) {
        response.sendRedirect("login.html");
        return;
    }
%>

<h2>Welcome, <%= user.getFullname() %>!</h2>
<p>Matric No: <%= user.getMatricNo() %></p>

<!-- Display image from Base64 -->

<-img src="data:image/jpeg;base64,<%= user.getBase64Image() %>"
     width="150" height="150" style="border-radius:50%"/><br><br>

<-a href="UserServlet?action=logout">Logout</a>
&nbsp (semicolon)&nbsp
<-a href="UserServlet?action=delete"
   onclick="return confirm('Delete your account permanently?')">
   Delete Account
</a>

</body>
</html>

Order summary for Lab 7: SQL table → StudentBean → StudentDAO → UserServlet (with web.xml + MultipartConfig) → register.html → login.html → dashboard.jsp.



//////////////////////////////////////////////////////////////////////////////

## LAB 8 — Full Code + File Placement

### STEP 1 — SQL
```sql
CREATE DATABASE IF NOT EXISTS Company;
USE Company;
CREATE TABLE IF NOT EXISTS employees (
  id       INT NOT NULL AUTO_INCREMENT,
  Name     VARCHAR(60),
  Email    VARCHAR(50),
  Position VARCHAR(15),
  PRIMARY KEY (id)
);
```

---

### STEP 2 — NetBeans Project Setup
```
New Project → Java Web → Web Application
Name: Lab8
Server: Apache Tomcat
→ Add Libraries: mysql-connector-j.jar AND jstl-1.2.jar
```

---

### STEP 3 — `Employee.java`
📁 `Source Packages > com.Model > Employee.java`
```java
package com.Model;

public class Employee {
    protected int id;
    protected String name;
    protected String email;
    protected String position;

    public Employee() {}

    // Constructor WITHOUT id (for INSERT)
    public Employee(String name, String email, String position) {
        this.name     = name;
        this.email    = email;
        this.position = position;
    }

    // Constructor WITH id (for UPDATE/display)
    public Employee(int id, String name, String email, String position) {
        this.id       = id;
        this.name     = name;
        this.email    = email;
        this.position = position;
    }

    public int    getId()       { return id; }
    public void   setId(int id) { this.id = id; }

    public String getName()          { return name; }
    public void   setName(String n)  { this.name = n; }

    public String getEmail()         { return email; }
    public void   setEmail(String e) { this.email = e; }

    public String getPosition()         { return position; }
    public void   setPosition(String p) { this.position = p; }
}
```

---

### STEP 4 — `EmployeeDAO.java`
📁 `Source Packages > com.DAO > EmployeeDAO.java`
```java
package com.DAO;

import com.Model.Employee;
import java.sql.*;
import java.util.*;

public class EmployeeDAO {

    private String jdbcURL      = "jdbc:mysql://localhost:3306/company";
    private String jdbcUsername = "root";
    private String jdbcPassword = "admin"; // change to your MySQL password

    // SQL constants
    private static final String INSERT_SQL =
        "INSERT INTO employees (name, email, position) VALUES (?, ?, ?)";
    private static final String SELECT_BY_ID =
        "SELECT id, name, email, position FROM employees WHERE id = ?";
    private static final String SELECT_ALL =
        "SELECT * FROM employees";
    private static final String DELETE_SQL =
        "DELETE FROM employees WHERE id = ?";
    private static final String UPDATE_SQL =
        "UPDATE employees SET name=?, email=?, position=? WHERE id=?";

    protected Connection getConnection() {
        Connection conn = null;
        try {
            Class.forName("com.mysql.jdbc.Driver");
            conn = DriverManager.getConnection(
                       jdbcURL, jdbcUsername, jdbcPassword);
        } catch (Exception e) { e.printStackTrace(); }
        return conn;
    }

    // CREATE
    public void insertEmployee(Employee emp) throws SQLException {
        try (Connection conn = getConnection();
             PreparedStatement ps = conn.prepareStatement(INSERT_SQL)) {
            ps.setString(1, emp.getName());
            ps.setString(2, emp.getEmail());
            ps.setString(3, emp.getPosition());
            ps.executeUpdate();
        }
    }

    // READ one
    public Employee selectEmployee(int id) {
        Employee emp = null;
        try (Connection conn = getConnection();
             PreparedStatement ps = conn.prepareStatement(SELECT_BY_ID)) {
            ps.setInt(1, id);
            ResultSet rs = ps.executeQuery();
            if (rs.next()) {
                emp = new Employee(id,
                    rs.getString("name"),
                    rs.getString("email"),
                    rs.getString("position"));
            }
        } catch (SQLException e) { e.printStackTrace(); }
        return emp;
    }

    // READ all
    public List<Employee> selectAllEmployees() {
        List<Employee> list = new ArrayList<>();
        try (Connection conn = getConnection();
             PreparedStatement ps = conn.prepareStatement(SELECT_ALL)) {
            ResultSet rs = ps.executeQuery();
            while (rs.next()) {
                list.add(new Employee(
                    rs.getInt("id"),
                    rs.getString("name"),
                    rs.getString("email"),
                    rs.getString("position")));
            }
        } catch (SQLException e) { e.printStackTrace(); }
        return list;
    }

    // DELETE
    public boolean deleteEmployee(int id) throws SQLException {
        boolean deleted;
        try (Connection conn = getConnection();
             PreparedStatement ps = conn.prepareStatement(DELETE_SQL)) {
            ps.setInt(1, id);
            deleted = ps.executeUpdate() > 0;
        }
        return deleted;
    }

    // UPDATE
    public boolean updateEmployee(Employee emp) throws SQLException {
        boolean updated;
        try (Connection conn = getConnection();
             PreparedStatement ps = conn.prepareStatement(UPDATE_SQL)) {
            ps.setString(1, emp.getName());
            ps.setString(2, emp.getEmail());
            ps.setString(3, emp.getPosition());
            ps.setInt(4, emp.getId());
            updated = ps.executeUpdate() > 0;
        }
        return updated;
    }
}
```

---

### STEP 5 — `EmployeeServlet.java`
📁 `Source Packages > com.WEB > EmployeeServlet.java`
```java
package com.WEB;

import com.DAO.EmployeeDAO;
import com.Model.Employee;
import java.io.IOException;
import java.sql.SQLException;
import java.util.List;
import javax.servlet.*;
import javax.servlet.annotation.WebServlet;
import javax.servlet.http.*;

@WebServlet("/")
public class EmployeeServlet extends HttpServlet {

    private EmployeeDAO employeeDAO;

    @Override
    public void init() {
        employeeDAO = new EmployeeDAO();
    }

    @Override
    protected void doPost(HttpServletRequest req, HttpServletResponse res)
            throws ServletException, IOException {
        doGet(req, res);  // forward all POST to doGet
    }

    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse res)
            throws ServletException, IOException {

        String action = req.getServletPath();

        try {
            switch (action) {
                case "/new":    showNewForm(req, res);    break;
                case "/insert": insertEmployee(req, res); break;
                case "/edit":   showEditForm(req, res);   break;
                case "/update": updateEmployee(req, res); break;
                case "/delete": deleteEmployee(req, res); break;
                default:        listEmployee(req, res);   break;
            }
        } catch (SQLException ex) {
            throw new ServletException(ex);
        }
    }

    // List all — default
    private void listEmployee(HttpServletRequest req, HttpServletResponse res)
            throws SQLException, IOException, ServletException {
        List<Employee> list = employeeDAO.selectAllEmployees();
        req.setAttribute("listEmployee", list);
        req.getRequestDispatcher("employeeList.jsp").forward(req, res);
    }

    // Show blank form for adding
    private void showNewForm(HttpServletRequest req, HttpServletResponse res)
            throws ServletException, IOException {
        req.getRequestDispatcher("employeeForm.jsp").forward(req, res);
    }

    // Show form pre-filled for editing
    private void showEditForm(HttpServletRequest req, HttpServletResponse res)
            throws SQLException, ServletException, IOException {
        int id = Integer.parseInt(req.getParameter("id"));
        Employee emp = employeeDAO.selectEmployee(id);
        req.setAttribute("employee", emp);
        req.getRequestDispatcher("employeeForm.jsp").forward(req, res);
    }

    // INSERT
    private void insertEmployee(HttpServletRequest req, HttpServletResponse res)
            throws SQLException, IOException {
        Employee emp = new Employee(
            req.getParameter("name"),
            req.getParameter("email"),
            req.getParameter("position"));
        employeeDAO.insertEmployee(emp);
        res.sendRedirect("list");
    }

    // UPDATE
    private void updateEmployee(HttpServletRequest req, HttpServletResponse res)
            throws SQLException, IOException {
        Employee emp = new Employee(
            Integer.parseInt(req.getParameter("id")),
            req.getParameter("name"),
            req.getParameter("email"),
            req.getParameter("position"));
        employeeDAO.updateEmployee(emp);
        res.sendRedirect("list");
    }

    // DELETE
    private void deleteEmployee(HttpServletRequest req, HttpServletResponse res)
            throws SQLException, IOException {
        int id = Integer.parseInt(req.getParameter("id"));
        employeeDAO.deleteEmployee(id);
        res.sendRedirect("list");
    }
}
```

---

### STEP 6 — `web.xml`
📁 `Web Pages > WEB-INF > web.xml` — add inside `<web-app>`:
```xml
<!-- Serve static files properly when @WebServlet("/") catches everything -->
<servlet-mapping>
    <servlet-name>default</servlet-name>
    <url-pattern>*.css</url-pattern>
</servlet-mapping>
<servlet-mapping>
    <servlet-name>default</servlet-name>
    <url-pattern>*.js</url-pattern>
</servlet-mapping>
<servlet-mapping>
    <servlet-name>default</servlet-name>
    <url-pattern>*.png</url-pattern>
</servlet-mapping>
```

---

### STEP 7 — Views

📁 `Web Pages > index.jsp`
```jsp
<%@page contentType="text/html" pageEncoding="UTF-8"%>
<!DOCTYPE html>
<html>
<head><title>Employee MVC</title></head>
<body>
<h1>Employee Management System</h1>
<ul>
    <li><a href="list">All Employees</a></li>
    <li><a href="new">Add New Employee</a></li>
</ul>
</body>
</html>
```

📁 `Web Pages > employeeList.jsp`
```jsp
<%@ page language="java" contentType="text/html; charset=UTF-8"%>
<%@ taglib uri="http://java.sun.com/jsp/jstl/core" prefix="c"%>
<!DOCTYPE html>
<html>
<head><title>Employee List</title></head>
<body>
<h2>List of Employees</h2>
<a href="new">Add New Employee</a><br><br>

<table border="1">
    <tr>
        <th>ID</th><th>Name</th><th>Email</th>
        <th>Position</th><th>Actions</th>
    </tr>

    <%-- JSTL loop through listEmployee attribute set by servlet --%>
    <c:forEach var="emp" items="${listEmployee}">
        <tr>
            <td><c:out value="${emp.id}"/></td>
            <td><c:out value="${emp.name}"/></td>
            <td><c:out value="${emp.email}"/></td>
            <td><c:out value="${emp.position}"/></td>
            <td>
                <a href="edit?id=<c:out value='${emp.id}'/>">Edit</a>
                &nbsp;
                <a href="delete?id=<c:out value='${emp.id}'/>">Delete</a>
            </td>
        </tr>
    </c:forEach>
</table>
</body>
</html>
```

📁 `Web Pages > employeeForm.jsp`
```jsp
<%@ page language="java" contentType="text/html; charset=UTF-8"%>
<%@ taglib uri="http://java.sun.com/jsp/jstl/core" prefix="c"%>
<!DOCTYPE html>
<html>
<head><title>Employee Form</title></head>
<body>

<%-- c:if switches form between ADD and EDIT mode --%>
<c:if test="${employee != null}">
    <h2>Edit Employee</h2>
    <form action="update" method="post">
    <input type="hidden" name="id" value="<c:out value='${employee.id}'/>">
</c:if>
<c:if test="${employee == null}">
    <h2>Add New Employee</h2>
    <form action="insert" method="post">
</c:if>

    Name:
    <input type="text" name="name"
           value="<c:out value='${employee.name}'/>" required><br><br>

    Email:
    <input type="text" name="email"
           value="<c:out value='${employee.email}'/>"><br><br>

    Position:
    <input type="text" name="position"
           value="<c:out value='${employee.position}'/>"><br><br>

    <input type="submit" value="Save">
</form>

<a href="list">Back to list</a>
</body>
</html>
```

📁 `Web Pages > error.jsp`
```jsp
<%@ page language="java" contentType="text/html; charset=UTF-8"
         isErrorPage="true" %>
<!DOCTYPE html>
<html>
<head><title>Error</title></head>
<body>
<h1>Error</h1>
<h2><%= exception.getMessage() %></h2>
</body>
</html>
```

---

## Quick Reference Cheat Sheet

| What | Lab 7 | Lab 8 |
|---|---|---|
| DB | `lab7_db` / `students` | `Company` / `employees` |
| Bean | `StudentBean` | `Employee` |
| DAO | `StudentDAO` | `EmployeeDAO` |
| Servlet | `UserServlet` (web.xml) | `EmployeeServlet` (@WebServlet) |
| Routing | `action` param | `getServletPath()` |
| Views | Scriptlets `<% %>` | JSTL `<c:forEach>` |
| Special | `@MultipartConfig` + BLOB | static file web.xml fix |
| Session | setAttribute/invalidate | none |

Good luck tomorrow! The two most important things to remember: **always create Bean → DAO → Servlet → JSP in that order**, and **don't forget `@MultipartConfig` for Lab 7 file upload** and **`jstl-1.2.jar` in Libraries for Lab 8**.
