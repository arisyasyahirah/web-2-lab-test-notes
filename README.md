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

Let me check the exercises from both labs first.

Looking at the PDFs:
- **Lab 7 Exercise**: Subject Management System — **NO JSTL, use scriptlets**, tied to session
- **Lab 8 Exercise**: Car Shop CRUD — **uses JSTL** (same pattern as Employee)



# another lab 8
Here's Lab 8 from scratch, step by step.

---

## STEP 1 — SQL (phpMyAdmin)

Open phpMyAdmin → click SQL tab → run this:

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

## STEP 2 — Create NetBeans Project

```
File → New Project
→ Java Web → Web Application → Next
→ Name: Lab8
→ Next → Server: Apache Tomcat
→ Finish
```

Then add libraries — right click `Libraries` folder:
```
Add JAR/Folder →
  1. mysql-connector-j.jar
  2. jstl-1.2.jar
```

---

## STEP 3 — Create `Employee.java` (Model/Bean)

```
Right click Source Packages
→ New → Java Package
→ Name: com.Model → Finish

Right click com.Model
→ New → Java Class
→ Name: Employee → Finish
```

This is just a **data container** — holds one employee's info:

```java
package com.Model;

public class Employee {

    // These variables match what we store in DB
    protected int    id;
    protected String name;
    protected String email;
    protected String position;

    // Empty constructor — required for JavaBean
    public Employee() {}

    // Constructor WITHOUT id
    // Used when INSERTING — DB auto generates the id
    public Employee(String name, String email, String position) {
        this.name     = name;
        this.email    = email;
        this.position = position;
    }

    // Constructor WITH id
    // Used when UPDATING or DISPLAYING — we already know the id
    public Employee(int id, String name, String email, String position) {
        this.id       = id;
        this.name     = name;
        this.email    = email;
        this.position = position;
    }

    // Getters and Setters
    // JSP reads these with ${employee.id}, ${employee.name} etc
    public int    getId()                  { return id; }
    public void   setId(int id)            { this.id = id; }

    public String getName()                { return name; }
    public void   setName(String name)     { this.name = name; }

    public String getEmail()               { return email; }
    public void   setEmail(String email)   { this.email = email; }

    public String getPosition()                { return position; }
    public void   setPosition(String position) { this.position = position; }
}
```

---

## STEP 4 — Create `EmployeeDAO.java` (Database Logic)

```
Right click Source Packages
→ New → Java Package
→ Name: com.DAO → Finish

Right click com.DAO
→ New → Java Class
→ Name: EmployeeDAO → Finish
```

DAO = **Data Access Object** — all SQL goes here, nothing else:

```java
package com.DAO;

import com.Model.Employee;
import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.util.ArrayList;
import java.util.List;

public class EmployeeDAO {

    // Database connection details
    private String jdbcURL      = "jdbc:mysql://localhost:3306/company";
    private String jdbcUsername = "root";
    private String jdbcPassword = "admin"; // change to YOUR mysql password

    // SQL statements written here so they are easy to find and change
    private static final String INSERT_SQL =
        "INSERT INTO employees (name, email, position) VALUES (?, ?, ?)";

    private static final String SELECT_BY_ID =
        "SELECT id, name, email, position FROM employees WHERE id = ?";

    private static final String SELECT_ALL =
        "SELECT * FROM employees";

    private static final String UPDATE_SQL =
        "UPDATE employees SET name=?, email=?, position=? WHERE id=?";

    private static final String DELETE_SQL =
        "DELETE FROM employees WHERE id=?";

    // Opens and returns a connection to the database
    // Called inside every method below
    protected Connection getConnection() {
        Connection conn = null;
        try {
            Class.forName("com.mysql.jdbc.Driver");
            conn = DriverManager.getConnection(
                       jdbcURL, jdbcUsername, jdbcPassword);
        } catch (SQLException e) {
            e.printStackTrace();
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
        return conn;
    }

    // CREATE — insert one employee into DB
    public void insertEmployee(Employee employee) throws SQLException {
        try (Connection conn = getConnection();
             PreparedStatement ps =
                 conn.prepareStatement(INSERT_SQL)) {

            // ? position 1 = name, 2 = email, 3 = position
            ps.setString(1, employee.getName());
            ps.setString(2, employee.getEmail());
            ps.setString(3, employee.getPosition());
            ps.executeUpdate();

        } catch (SQLException e) {
            e.printStackTrace();
        }
    }

    // READ ONE — get single employee by id (used for edit form)
    public Employee selectEmployee(int id) {
        Employee employee = null;
        try (Connection conn = getConnection();
             PreparedStatement ps =
                 conn.prepareStatement(SELECT_BY_ID)) {

            ps.setInt(1, id);
            ResultSet rs = ps.executeQuery();

            // rs.next() moves to first row
            // if row exists, build Employee object from it
            while (rs.next()) {
                String name     = rs.getString("name");
                String email    = rs.getString("email");
                String position = rs.getString("position");
                employee = new Employee(id, name, email, position);
            }

        } catch (SQLException e) {
            e.printStackTrace();
        }
        return employee;
    }

    // READ ALL — get every employee (used for list page)
    public List<Employee> selectAllEmployees() {
        List<Employee> employees = new ArrayList<>();
        try (Connection conn = getConnection();
             PreparedStatement ps =
                 conn.prepareStatement(SELECT_ALL)) {

            ResultSet rs = ps.executeQuery();

            // Each row becomes one Employee object added to list
            while (rs.next()) {
                int    id       = rs.getInt("id");
                String name     = rs.getString("name");
                String email    = rs.getString("email");
                String position = rs.getString("position");
                employees.add(new Employee(id, name, email, position));
            }

        } catch (SQLException e) {
            e.printStackTrace();
        }
        return employees;
    }

    // UPDATE — change existing employee's details
    public boolean updateEmployee(Employee employee) throws SQLException {
        boolean rowUpdated;
        try (Connection conn = getConnection();
             PreparedStatement ps =
                 conn.prepareStatement(UPDATE_SQL)) {

            // ? position 1=name, 2=email, 3=position, 4=id (WHERE clause)
            ps.setString(1, employee.getName());
            ps.setString(2, employee.getEmail());
            ps.setString(3, employee.getPosition());
            ps.setInt(4,    employee.getId());
            rowUpdated = ps.executeUpdate() > 0;
        }
        return rowUpdated;
    }

    // DELETE — remove employee by id
    public boolean deleteEmployee(int id) throws SQLException {
        boolean rowDeleted;
        try (Connection conn = getConnection();
             PreparedStatement ps =
                 conn.prepareStatement(DELETE_SQL)) {

            ps.setInt(1, id);
            rowDeleted = ps.executeUpdate() > 0;
        }
        return rowDeleted;
    }
}
```

---

## STEP 5 — Create `EmployeeServlet.java` (Controller)

This is the important part you asked about — creating servlet with web.xml:

```
Right click Source Packages
→ New → Java Package
→ Name: com.WEB → Finish

Right click com.WEB
→ New → Servlet
→ Name: EmployeeServlet → Next
```

You will see this screen — do this exactly:

```
Class Name : EmployeeServlet
Package    : com.WEB
→ Next

☑ TICK "Add information to deployment descriptor (web.xml)"

URL Pattern box — CHANGE it from /EmployeeServlet to just:
/

→ Finish
```

Now NetBeans auto-generates the class. You will see `@WebServlet` annotation at the top — **DELETE IT** because we are using web.xml instead:

```java
// DELETE THIS LINE that NetBeans generated:
// @WebServlet(name = "EmployeeServlet", urlPatterns = {"/EmployeeServlet"})
```

Your final `EmployeeServlet.java` should look like this:

```java
package com.WEB;

import com.DAO.EmployeeDAO;
import com.Model.Employee;
import java.io.IOException;
import java.sql.SQLException;
import java.util.List;
import javax.servlet.RequestDispatcher;
import javax.servlet.ServletException;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

// NO @WebServlet annotation here — mapping is in web.xml instead
public class EmployeeServlet extends HttpServlet {

    private EmployeeDAO employeeDAO;

    // init() runs ONCE when server starts
    // creates the DAO so it's ready to use
    @Override
    public void init() {
        employeeDAO = new EmployeeDAO();
    }

    // ALL post requests go to doGet
    // this means we handle everything in one place
    @Override
    protected void doPost(HttpServletRequest request,
                          HttpServletResponse response)
            throws ServletException, IOException {
        doGet(request, response);
    }

    // MAIN METHOD — reads URL path, decides what to do
    @Override
    protected void doGet(HttpServletRequest request,
                         HttpServletResponse response)
            throws ServletException, IOException {

        // getServletPath() reads the URL after project name
        // e.g. localhost:8080/Lab8/new    → action = "/new"
        // e.g. localhost:8080/Lab8/delete → action = "/delete"
        String action = request.getServletPath();

        try {
            switch (action) {
                case "/new":
                    showNewForm(request, response);
                    break;
                case "/insert":
                    insertEmployee(request, response);
                    break;
                case "/edit":
                    showEditForm(request, response);
                    break;
                case "/update":
                    updateEmployee(request, response);
                    break;
                case "/delete":
                    deleteEmployee(request, response);
                    break;
                default:
                    // /list and anything else → show the list
                    listEmployee(request, response);
                    break;
            }
        } catch (SQLException ex) {
            throw new ServletException(ex);
        }
    }

    // READ ALL
    // Gets all employees from DB, puts in request, forwards to list JSP
    private void listEmployee(HttpServletRequest request,
                               HttpServletResponse response)
            throws SQLException, IOException, ServletException {

        List<Employee> listEmployee = employeeDAO.selectAllEmployees();

        // setAttribute = pass data to JSP
        // JSP accesses this with ${listEmployee}
        request.setAttribute("listEmployee", listEmployee);

        RequestDispatcher dispatcher =
            request.getRequestDispatcher("employeeList.jsp");
        dispatcher.forward(request, response);
    }

    // SHOW ADD FORM
    // Just forwards to form JSP — no employee attribute set
    // JSP checks: employee == null → shows "Add New Employee"
    private void showNewForm(HttpServletRequest request,
                              HttpServletResponse response)
            throws ServletException, IOException {

        RequestDispatcher dispatcher =
            request.getRequestDispatcher("employeeForm.jsp");
        dispatcher.forward(request, response);
    }

    // SHOW EDIT FORM
    // Gets employee by id, puts in request, forwards to same form JSP
    // JSP checks: employee != null → shows "Edit Employee" pre-filled
    private void showEditForm(HttpServletRequest request,
                               HttpServletResponse response)
            throws SQLException, ServletException, IOException {

        // Read ?id=2 from the URL
        int id = Integer.parseInt(request.getParameter("id"));

        Employee existingEmployee = employeeDAO.selectEmployee(id);

        // setAttribute = pass employee object to JSP
        // JSP accesses this with ${employee.name} etc
        request.setAttribute("employee", existingEmployee);

        RequestDispatcher dispatcher =
            request.getRequestDispatcher("employeeForm.jsp");
        dispatcher.forward(request, response);
    }

    // INSERT (Create)
    // Reads form fields, builds Employee, inserts, redirects to list
    private void insertEmployee(HttpServletRequest request,
                                 HttpServletResponse response)
            throws SQLException, IOException {

        // getParameter reads what user typed in form fields
        String name     = request.getParameter("name");
        String email    = request.getParameter("email");
        String position = request.getParameter("position");

        // No id needed — DB auto generates it
        Employee newEmployee = new Employee(name, email, position);
        employeeDAO.insertEmployee(newEmployee);

        // sendRedirect sends user BACK to list page after insert
        response.sendRedirect("list");
    }

    // UPDATE
    // Reads form fields INCLUDING hidden id, updates, redirects to list
    private void updateEmployee(HttpServletRequest request,
                                 HttpServletResponse response)
            throws SQLException, IOException {

        // id comes from hidden input in the form
        int    id       = Integer.parseInt(request.getParameter("id"));
        String name     = request.getParameter("name");
        String email    = request.getParameter("email");
        String position = request.getParameter("position");

        // Need id this time so DB knows WHICH row to update
        Employee employee = new Employee(id, name, email, position);
        employeeDAO.updateEmployee(employee);

        response.sendRedirect("list");
    }

    // DELETE
    // Reads id from URL, deletes that row, redirects to list
    private void deleteEmployee(HttpServletRequest request,
                                 HttpServletResponse response)
            throws SQLException, IOException {

        // id comes from ?id=2 in the URL link
        int id = Integer.parseInt(request.getParameter("id"));
        employeeDAO.deleteEmployee(id);

        response.sendRedirect("list");
    }
}
```

---

## STEP 6 — Check `web.xml`

Because you ticked the web.xml box when creating the servlet, NetBeans auto-added this. Open `WEB-INF > web.xml` and verify it looks like this — also add the static file mappings:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"
         version="3.1">

    <!-- Auto-generated by NetBeans when you ticked the web.xml box -->
    <servlet>
        <servlet-name>EmployeeServlet</servlet-name>
        <servlet-class>com.WEB.EmployeeServlet</servlet-class>
    </servlet>
    <servlet-mapping>
        <servlet-name>EmployeeServlet</servlet-name>
        <url-pattern>/</url-pattern>  <!-- handles ALL URLs -->
    </servlet-mapping>

    <!-- ADD THESE — fixes CSS/JS/images breaking when / catches everything -->
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

</web-app>
```

---

## STEP 7 — Create JSP Views

```
Right click Web Pages
→ New → JSP
→ Name: employeeList → Finish
(repeat for employeeForm, error, index)
```

All 4 files already given in the previous response — copy them in exactly as they are.

---

## Full Flow Visual

```
Browser hits /list
      ↓
EmployeeServlet.doGet()
      ↓
action = "/list" → default case
      ↓
listEmployee() → DAO.selectAllEmployees()
      ↓
request.setAttribute("listEmployee", list)
      ↓
forward → employeeList.jsp
      ↓
JSTL <c:forEach> loops ${listEmployee} → shows table


User clicks Edit link (edit?id=2)
      ↓
EmployeeServlet.doGet()
      ↓
action = "/edit"
      ↓
showEditForm() → DAO.selectEmployee(2)
      ↓
request.setAttribute("employee", emp)
      ↓
forward → employeeForm.jsp
      ↓
JSTL <c:if test="${employee != null}"> → shows Edit form pre-filled
      ↓
User submits → POST /update
      ↓
updateEmployee() → DAO.updateEmployee()
      ↓
sendRedirect("list")
```

The key thing to remember is **web.xml maps `/` to EmployeeServlet**, so every URL goes through `doGet()`, and `getServletPath()` reads which part of the URL was typed to decide which private method to call.
////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////


# LAB EXERCISE

## LAB 7 EXERCISE — Subject Management System

### STEP 1 — SQL
```sql
USE lab7_db;
CREATE TABLE registered_subjects (
    id           INT AUTO_INCREMENT PRIMARY KEY,
    matric_no    VARCHAR(20) NOT NULL,
    subject_code VARCHAR(20) NOT NULL,
    subject_name VARCHAR(100) NOT NULL
);
```

---

### STEP 2 — `SubjectBean.java`
📁 `Source Packages > com.lab.bean > SubjectBean.java`
```java
package com.lab.bean;

public class SubjectBean implements java.io.Serializable {
    private int    id;
    private String matricNo;
    private String subjectCode;
    private String subjectName;

    public SubjectBean() {}

    public int    getId()                  { return id; }
    public void   setId(int id)            { this.id = id; }

    public String getMatricNo()            { return matricNo; }
    public void   setMatricNo(String m)    { this.matricNo = m; }

    public String getSubjectCode()         { return subjectCode; }
    public void   setSubjectCode(String c) { this.subjectCode = c; }

    public String getSubjectName()         { return subjectName; }
    public void   setSubjectName(String n) { this.subjectName = n; }
}
```

---

### STEP 3 — `SubjectDAO.java`
📁 `Source Packages > com.lab.dao > SubjectDAO.java`
```java
package com.lab.dao;

import com.lab.bean.SubjectBean;
import java.sql.*;
import java.util.*;

public class SubjectDAO {

    private Connection getConnection() throws Exception {
        Class.forName("com.mysql.cj.jdbc.Driver");
        return DriverManager.getConnection(
            "jdbc:mysql://localhost:3306/lab7_db", "root", "");
    }

    // CREATE — add new subject
    public boolean addSubject(SubjectBean subject) {
        try (Connection conn = getConnection()) {
            String sql = "INSERT INTO registered_subjects " +
                         "(matric_no, subject_code, subject_name) " +
                         "VALUES (?, ?, ?)";
            PreparedStatement ps = conn.prepareStatement(sql);
            ps.setString(1, subject.getMatricNo());
            ps.setString(2, subject.getSubjectCode());
            ps.setString(3, subject.getSubjectName());
            return ps.executeUpdate() > 0;
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }
    }

    // READ — get all subjects for ONE student only
    public List<SubjectBean> getSubjectsByMatric(String matricNo) {
        List<SubjectBean> list = new ArrayList<>();
        try (Connection conn = getConnection()) {
            String sql = "SELECT * FROM registered_subjects " +
                         "WHERE matric_no = ?";
            PreparedStatement ps = conn.prepareStatement(sql);
            ps.setString(1, matricNo);
            ResultSet rs = ps.executeQuery();
            while (rs.next()) {
                SubjectBean s = new SubjectBean();
                s.setId(rs.getInt("id"));
                s.setMatricNo(rs.getString("matric_no"));
                s.setSubjectCode(rs.getString("subject_code"));
                s.setSubjectName(rs.getString("subject_name"));
                list.add(s);
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
        return list;
    }

    // READ ONE — get single subject by id (for edit form)
    public SubjectBean getSubjectById(int id) {
        SubjectBean s = null;
        try (Connection conn = getConnection()) {
            String sql = "SELECT * FROM registered_subjects WHERE id = ?";
            PreparedStatement ps = conn.prepareStatement(sql);
            ps.setInt(1, id);
            ResultSet rs = ps.executeQuery();
            if (rs.next()) {
                s = new SubjectBean();
                s.setId(rs.getInt("id"));
                s.setMatricNo(rs.getString("matric_no"));
                s.setSubjectCode(rs.getString("subject_code"));
                s.setSubjectName(rs.getString("subject_name"));
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
        return s;
    }

    // UPDATE — edit subject
    public boolean updateSubject(SubjectBean subject) {
        try (Connection conn = getConnection()) {
            String sql = "UPDATE registered_subjects " +
                         "SET subject_code = ?, subject_name = ? " +
                         "WHERE id = ? AND matric_no = ?";
            PreparedStatement ps = conn.prepareStatement(sql);
            ps.setString(1, subject.getSubjectCode());
            ps.setString(2, subject.getSubjectName());
            ps.setInt(3, subject.getId());
            ps.setString(4, subject.getMatricNo());
            return ps.executeUpdate() > 0;
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }
    }

    // DELETE — remove subject
    public boolean deleteSubject(int id, String matricNo) {
        try (Connection conn = getConnection()) {
            // matric_no check ensures student can only delete their OWN subject
            String sql = "DELETE FROM registered_subjects " +
                         "WHERE id = ? AND matric_no = ?";
            PreparedStatement ps = conn.prepareStatement(sql);
            ps.setInt(1, id);
            ps.setString(2, matricNo);
            return ps.executeUpdate() > 0;
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }
    }
}
```

---

### STEP 4 — `SubjectServlet.java`
📁 `Source Packages > com.lab.controller > SubjectServlet.java`

⚠️ When creating: **CHECK "Add to web.xml"**, delete any `@WebServlet` generated
```java
package com.lab.controller;

import com.lab.bean.StudentBean;
import com.lab.bean.SubjectBean;
import com.lab.dao.SubjectDAO;
import java.io.IOException;
import java.util.List;
import javax.servlet.ServletException;
import javax.servlet.http.*;

public class SubjectServlet extends HttpServlet {

    private SubjectDAO subjectDAO = new SubjectDAO();

    @Override
    protected void doGet(HttpServletRequest request,
                         HttpServletResponse response)
            throws ServletException, IOException {

        response.setContentType("text/html;charset=UTF-8");

        // SESSION CHECK — get logged in student
        HttpSession session = request.getSession(false);
        if (session == null || session.getAttribute("loggedUser") == null) {
            response.sendRedirect("login.html");
            return;
        }

        // Get matric_no from session
        StudentBean user = (StudentBean) session.getAttribute("loggedUser");
        String matricNo  = user.getMatricNo();

        String action = request.getParameter("action");
        if (action == null) action = "view";

        switch (action) {

            case "view":
                // READ — fetch subjects for this student only
                List<SubjectBean> list = subjectDAO.getSubjectsByMatric(matricNo);
                request.setAttribute("subjectList", list);
                request.getRequestDispatcher("subject/viewSubjects.jsp")
                       .forward(request, response);
                break;

            case "add":
                // Show add form
                request.getRequestDispatcher("subject/registerSubject.jsp")
                       .forward(request, response);
                break;

            case "edit":
                // Show edit form pre-filled
                int editId = Integer.parseInt(request.getParameter("id"));
                SubjectBean toEdit = subjectDAO.getSubjectById(editId);
                request.setAttribute("subject", toEdit);
                request.getRequestDispatcher("subject/updateSubject.jsp")
                       .forward(request, response);
                break;

            case "delete":
                // DELETE
                int delId = Integer.parseInt(request.getParameter("id"));
                subjectDAO.deleteSubject(delId, matricNo);
                response.sendRedirect("SubjectServlet?action=view");
                break;

            default:
                response.sendRedirect("SubjectServlet?action=view");
                break;
        }
    }

    @Override
    protected void doPost(HttpServletRequest request,
                          HttpServletResponse response)
            throws ServletException, IOException {

        response.setContentType("text/html;charset=UTF-8");

        // SESSION CHECK
        HttpSession session = request.getSession(false);
        if (session == null || session.getAttribute("loggedUser") == null) {
            response.sendRedirect("login.html");
            return;
        }

        StudentBean user = (StudentBean) session.getAttribute("loggedUser");
        String matricNo  = user.getMatricNo();

        String action = request.getParameter("action");

        if ("insert".equals(action)) {
            // CREATE
            SubjectBean s = new SubjectBean();
            s.setMatricNo(matricNo);  // from session, not form!
            s.setSubjectCode(request.getParameter("subjectCode"));
            s.setSubjectName(request.getParameter("subjectName"));
            subjectDAO.addSubject(s);
            response.sendRedirect("SubjectServlet?action=view");

        } else if ("update".equals(action)) {
            // UPDATE
            SubjectBean s = new SubjectBean();
            s.setId(Integer.parseInt(request.getParameter("id")));
            s.setMatricNo(matricNo);  // from session, not form!
            s.setSubjectCode(request.getParameter("subjectCode"));
            s.setSubjectName(request.getParameter("subjectName"));
            subjectDAO.updateSubject(s);
            response.sendRedirect("SubjectServlet?action=view");
        }
    }
}
```

📁 `Web Pages > WEB-INF > web.xml` — add inside `<web-app>`:
```xml
<servlet>
    <servlet-name>SubjectServlet</servlet-name>
    <servlet-class>com.lab.controller.SubjectServlet</servlet-class>
</servlet>
<servlet-mapping>
    <servlet-name>SubjectServlet</servlet-name>
    <url-pattern>/SubjectServlet</url-pattern>
</servlet-mapping>
```

---

### STEP 5 — Views (SCRIPTLETS — no JSTL!)

📁 `Web Pages > subject > viewSubjects.jsp`
```jsp
<%@page contentType="text/html" pageEncoding="UTF-8"%>
<%@page import="com.lab.bean.SubjectBean"%>
<%@page import="com.lab.bean.StudentBean"%>
<%@page import="java.util.List"%>
<!DOCTYPE html>
<html>
<head><title>My Subjects</title></head>
<body>

<%
    // Session check
    StudentBean user = (StudentBean) session.getAttribute("loggedUser");
    if (user == null) {
        response.sendRedirect("login.html");
        return;
    }
%>

<h2>Welcome, <%= user.getFullname() %> — Your Subjects</h2>
<a href="SubjectServlet?action=add">+ Register New Subject</a><br><br>

<table border="1">
    <tr>
        <th>ID</th>
        <th>Subject Code</th>
        <th>Subject Name</th>
        <th>Actions</th>
    </tr>

    <%
        // Get list from servlet setAttribute
        List<SubjectBean> list =
            (List<SubjectBean>) request.getAttribute("subjectList");

        if (list != null && !list.isEmpty()) {
            for (SubjectBean s : list) {
    %>
    <tr>
        <td><%= s.getId() %></td>
        <td><%= s.getSubjectCode() %></td>
        <td><%= s.getSubjectName() %></td>
        <td>
            <a href="SubjectServlet?action=edit&id=<%= s.getId() %>">Edit</a>
            &nbsp;
            <a href="SubjectServlet?action=delete&id=<%= s.getId() %>"
               onclick="return confirm('Delete this subject?')">Delete</a>
        </td>
    </tr>
    <%
            } // end for
        } else {
    %>
    <tr>
        <td colspan="4">No subjects registered yet.</td>
    </tr>
    <%
        } // end if
    %>
</table>

<br>
<a href="UserServlet?action=logout">Logout</a>
</body>
</html>
```

📁 `Web Pages > subject > registerSubject.jsp`
```jsp
<%@page contentType="text/html" pageEncoding="UTF-8"%>
<%@page import="com.lab.bean.StudentBean"%>
<!DOCTYPE html>
<html>
<head><title>Register Subject</title></head>
<body>

<%
    StudentBean user = (StudentBean) session.getAttribute("loggedUser");
    if (user == null) {
        response.sendRedirect("login.html");
        return;
    }
%>

<h2>Register New Subject</h2>

<form action="SubjectServlet" method="POST">
    <input type="hidden" name="action" value="insert">

    Subject Code:
    <input type="text" name="subjectCode" required><br><br>

    Subject Name:
    <input type="text" name="subjectName" required><br><br>

    <input type="submit" value="Register Subject">
</form>

<br>
<a href="SubjectServlet?action=view">Back to My Subjects</a>
</body>
</html>
```

📁 `Web Pages > subject > updateSubject.jsp`
```jsp
<%@page contentType="text/html" pageEncoding="UTF-8"%>
<%@page import="com.lab.bean.SubjectBean"%>
<%@page import="com.lab.bean.StudentBean"%>
<!DOCTYPE html>
<html>
<head><title>Update Subject</title></head>
<body>

<%
    StudentBean user = (StudentBean) session.getAttribute("loggedUser");
    if (user == null) {
        response.sendRedirect("login.html");
        return;
    }

    // Get subject passed from servlet
    SubjectBean s = (SubjectBean) request.getAttribute("subject");
%>

<h2>Update Subject</h2>

<form action="SubjectServlet" method="POST">
    <input type="hidden" name="action" value="update">
    <input type="hidden" name="id"     value="<%= s.getId() %>">

    Subject Code:
    <input type="text" name="subjectCode"
           value="<%= s.getSubjectCode() %>" required><br><br>

    Subject Name:
    <input type="text" name="subjectName"
           value="<%= s.getSubjectName() %>" required><br><br>

    <input type="submit" value="Update Subject">
</form>

<br>
<a href="SubjectServlet?action=view">Back to My Subjects</a>
</body>
</html>
```

---

## LAB 8 EXERCISE — Car Shop CRUD (with JSTL)

### STEP 1 — SQL
```sql
CREATE DATABASE IF NOT EXISTS carshop;
USE carshop;
CREATE TABLE IF NOT EXISTS CarPricelist (
    Car_id   INT NOT NULL AUTO_INCREMENT,
    Brand    VARCHAR(15),
    Model    VARCHAR(30),
    Cyclinder INT,
    Price    DOUBLE,
    PRIMARY KEY (Car_id)
);
```

---

### STEP 2 — NetBeans Project Setup
```
New Project → Java Web → Web Application
Name: CarShop
Server: Apache Tomcat
→ Add Libraries: mysql-connector-j.jar AND jstl-1.2.jar
```

---

### STEP 3 — `Car.java`
📁 `Source Packages > com.Model > Car.java`
```java
package com.Model;

public class Car {
    private int    carId;
    private String brand;
    private String model;
    private int    cyclinder;
    private double price;

    public Car() {}

    // Constructor WITHOUT id — for INSERT
    public Car(String brand, String model, int cyclinder, double price) {
        this.brand     = brand;
        this.model     = model;
        this.cyclinder = cyclinder;
        this.price     = price;
    }

    // Constructor WITH id — for UPDATE/display
    public Car(int carId, String brand, String model,
               int cyclinder, double price) {
        this.carId     = carId;
        this.brand     = brand;
        this.model     = model;
        this.cyclinder = cyclinder;
        this.price     = price;
    }

    public int    getCarId()              { return carId; }
    public void   setCarId(int carId)     { this.carId = carId; }

    public String getBrand()              { return brand; }
    public void   setBrand(String brand)  { this.brand = brand; }

    public String getModel()              { return model; }
    public void   setModel(String model)  { this.model = model; }

    public int    getCyclinder()              { return cyclinder; }
    public void   setCyclinder(int cyclinder) { this.cyclinder = cyclinder; }

    public double getPrice()               { return price; }
    public void   setPrice(double price)   { this.price = price; }
}
```

---

### STEP 4 — `CarDAO.java`
📁 `Source Packages > com.DAO > CarDAO.java`
```java
package com.DAO;

import com.Model.Car;
import java.sql.*;
import java.util.*;

public class CarDAO {

    private String jdbcURL      = "jdbc:mysql://localhost:3306/carshop";
    private String jdbcUsername = "root";
    private String jdbcPassword = "admin"; // change to your password

    private static final String INSERT_SQL =
        "INSERT INTO CarPricelist (Brand, Model, Cyclinder, Price) " +
        "VALUES (?, ?, ?, ?)";
    private static final String SELECT_ALL =
        "SELECT * FROM CarPricelist";
    private static final String SELECT_BY_ID =
        "SELECT * FROM CarPricelist WHERE Car_id = ?";
    private static final String UPDATE_SQL =
        "UPDATE CarPricelist SET Brand=?, Model=?, Cyclinder=?, Price=? " +
        "WHERE Car_id=?";
    private static final String DELETE_SQL =
        "DELETE FROM CarPricelist WHERE Car_id=?";

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
    public void insertCar(Car car) throws SQLException {
        try (Connection conn = getConnection();
             PreparedStatement ps = conn.prepareStatement(INSERT_SQL)) {
            ps.setString(1, car.getBrand());
            ps.setString(2, car.getModel());
            ps.setInt(3,    car.getCyclinder());
            ps.setDouble(4, car.getPrice());
            ps.executeUpdate();
        }
    }

    // READ all
    public List<Car> selectAllCars() {
        List<Car> list = new ArrayList<>();
        try (Connection conn = getConnection();
             PreparedStatement ps = conn.prepareStatement(SELECT_ALL)) {
            ResultSet rs = ps.executeQuery();
            while (rs.next()) {
                list.add(new Car(
                    rs.getInt("Car_id"),
                    rs.getString("Brand"),
                    rs.getString("Model"),
                    rs.getInt("Cyclinder"),
                    rs.getDouble("Price")));
            }
        } catch (SQLException e) { e.printStackTrace(); }
        return list;
    }

    // READ one
    public Car selectCarById(int id) {
        Car car = null;
        try (Connection conn = getConnection();
             PreparedStatement ps = conn.prepareStatement(SELECT_BY_ID)) {
            ps.setInt(1, id);
            ResultSet rs = ps.executeQuery();
            if (rs.next()) {
                car = new Car(
                    rs.getInt("Car_id"),
                    rs.getString("Brand"),
                    rs.getString("Model"),
                    rs.getInt("Cyclinder"),
                    rs.getDouble("Price"));
            }
        } catch (SQLException e) { e.printStackTrace(); }
        return car;
    }

    // UPDATE
    public boolean updateCar(Car car) throws SQLException {
        boolean updated;
        try (Connection conn = getConnection();
             PreparedStatement ps = conn.prepareStatement(UPDATE_SQL)) {
            ps.setString(1, car.getBrand());
            ps.setString(2, car.getModel());
            ps.setInt(3,    car.getCyclinder());
            ps.setDouble(4, car.getPrice());
            ps.setInt(5,    car.getCarId());
            updated = ps.executeUpdate() > 0;
        }
        return updated;
    }

    // DELETE
    public boolean deleteCar(int id) throws SQLException {
        boolean deleted;
        try (Connection conn = getConnection();
             PreparedStatement ps = conn.prepareStatement(DELETE_SQL)) {
            ps.setInt(1, id);
            deleted = ps.executeUpdate() > 0;
        }
        return deleted;
    }
}
```

---

### STEP 5 — `CarServlet.java`
📁 `Source Packages > com.WEB > CarServlet.java`
```java
package com.WEB;

import com.DAO.CarDAO;
import com.Model.Car;
import java.io.IOException;
import java.sql.SQLException;
import java.util.List;
import javax.servlet.*;
import javax.servlet.annotation.WebServlet;
import javax.servlet.http.*;

@WebServlet("/")
public class CarServlet extends HttpServlet {

    private CarDAO carDAO;

    @Override
    public void init() {
        carDAO = new CarDAO();
    }

    @Override
    protected void doPost(HttpServletRequest req, HttpServletResponse res)
            throws ServletException, IOException {
        doGet(req, res);
    }

    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse res)
            throws ServletException, IOException {

        String action = req.getServletPath();

        try {
            switch (action) {
                case "/new":    showNewForm(req, res);  break;
                case "/insert": insertCar(req, res);    break;
                case "/edit":   showEditForm(req, res); break;
                case "/update": updateCar(req, res);    break;
                case "/delete": deleteCar(req, res);    break;
                default:        listCars(req, res);     break;
            }
        } catch (SQLException ex) {
            throw new ServletException(ex);
        }
    }

    // List all cars — default
    private void listCars(HttpServletRequest req, HttpServletResponse res)
            throws SQLException, IOException, ServletException {
        List<Car> list = carDAO.selectAllCars();
        req.setAttribute("carList", list);
        req.getRequestDispatcher("carList.jsp").forward(req, res);
    }

    // Show blank add form
    private void showNewForm(HttpServletRequest req, HttpServletResponse res)
            throws ServletException, IOException {
        req.getRequestDispatcher("carForm.jsp").forward(req, res);
    }

    // Show edit form pre-filled
    private void showEditForm(HttpServletRequest req, HttpServletResponse res)
            throws SQLException, ServletException, IOException {
        int id = Integer.parseInt(req.getParameter("id"));
        Car car = carDAO.selectCarById(id);
        req.setAttribute("car", car);
        req.getRequestDispatcher("carForm.jsp").forward(req, res);
    }

    // INSERT
    private void insertCar(HttpServletRequest req, HttpServletResponse res)
            throws SQLException, IOException {
        Car car = new Car(
            req.getParameter("brand"),
            req.getParameter("model"),
            Integer.parseInt(req.getParameter("cyclinder")),
            Double.parseDouble(req.getParameter("price")));
        carDAO.insertCar(car);
        res.sendRedirect("list");
    }

    // UPDATE
    private void updateCar(HttpServletRequest req, HttpServletResponse res)
            throws SQLException, IOException {
        Car car = new Car(
            Integer.parseInt(req.getParameter("carId")),
            req.getParameter("brand"),
            req.getParameter("model"),
            Integer.parseInt(req.getParameter("cyclinder")),
            Double.parseDouble(req.getParameter("price")));
        carDAO.updateCar(car);
        res.sendRedirect("list");
    }

    // DELETE
    private void deleteCar(HttpServletRequest req, HttpServletResponse res)
            throws SQLException, IOException {
        int id = Integer.parseInt(req.getParameter("id"));
        carDAO.deleteCar(id);
        res.sendRedirect("list");
    }
}
```

---

### STEP 6 — web.xml static fix
📁 `Web Pages > WEB-INF > web.xml`
```xml
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

### STEP 7 — Views (JSTL)

📁 `Web Pages > index.jsp`
```jsp
<%@page contentType="text/html" pageEncoding="UTF-8"%>
<!DOCTYPE html>
<html>
<head><title>Car Shop</title></head>
<body>
<h1>Car Shop Management System</h1>
<ul>
    <li><a href="list">View All Cars</a></li>
    <li><a href="new">Add New Car</a></li>
</ul>
</body>
</html>
```

📁 `Web Pages > carList.jsp`
```jsp
<%@ page language="java" contentType="text/html; charset=UTF-8"%>
<%@ taglib uri="http://java.sun.com/jsp/jstl/core" prefix="c"%>
<!DOCTYPE html>
<html>
<head><title>Car Price List</title></head>
<body>

<h2>Car Price List</h2>
<a href="new">+ Add New Car</a><br><br>

<table border="1">
    <tr>
        <th>ID</th>
        <th>Brand</th>
        <th>Model</th>
        <th>Cyclinder</th>
        <th>Price (RM)</th>
        <th>Actions</th>
    </tr>

    <c:forEach var="car" items="${carList}">
        <tr>
            <td><c:out value="${car.carId}"/></td>
            <td><c:out value="${car.brand}"/></td>
            <td><c:out value="${car.model}"/></td>
            <td><c:out value="${car.cyclinder}"/></td>
            <td><c:out value="${car.price}"/></td>
            <td>
                <a href="edit?id=<c:out value='${car.carId}'/>">Edit</a>
                &nbsp;
                <a href="delete?id=<c:out value='${car.carId}'/>"
                   onclick="return confirm('Delete this car?')">Delete</a>
            </td>
        </tr>
    </c:forEach>
</table>

</body>
</html>
```

📁 `Web Pages > carForm.jsp`
```jsp
<%@ page language="java" contentType="text/html; charset=UTF-8"%>
<%@ taglib uri="http://java.sun.com/jsp/jstl/core" prefix="c"%>
<!DOCTYPE html>
<html>
<head><title>Car Form</title></head>
<body>

<%-- Switch between ADD and EDIT mode --%>
<c:if test="${car != null}">
    <h2>Edit Car</h2>
    <form action="update" method="post">
    <input type="hidden" name="carId"
           value="<c:out value='${car.carId}'/>">
</c:if>
<c:if test="${car == null}">
    <h2>Add New Car</h2>
    <form action="insert" method="post">
</c:if>

    Brand:
    <input type="text" name="brand"
           value="<c:out value='${car.brand}'/>" required><br><br>

    Model:
    <input type="text" name="model"
           value="<c:out value='${car.model}'/>" required><br><br>

    Cyclinder:
    <input type="number" name="cyclinder"
           value="<c:out value='${car.cyclinder}'/>" required><br><br>

    Price (RM):
    <input type="number" step="0.01" name="price"
           value="<c:out value='${car.price}'/>" required><br><br>

    <input type="submit" value="Save">
</form>

<br>
<a href="list">Back to List</a>
</body>
</html>
```

---

## Final File Structure Summary

```
LAB 7 PROJECT
├── Source Packages
│   ├── com.lab.bean
│   │   ├── StudentBean.java      (from main lab)
│   │   └── SubjectBean.java      (exercise)
│   ├── com.lab.dao
│   │   ├── StudentDAO.java       (from main lab)
│   │   └── SubjectDAO.java       (exercise)
│   └── com.lab.controller
│       ├── UserServlet.java      (from main lab)
│       └── SubjectServlet.java   (exercise)
└── Web Pages
    ├── register.html             (from main lab)
    ├── login.html                (from main lab)
    ├── dashboard.jsp             (from main lab)
    ├── WEB-INF/web.xml           (both servlets mapped here)
    └── subject/
        ├── viewSubjects.jsp      (exercise — scriptlets)
        ├── registerSubject.jsp   (exercise — scriptlets)
        └── updateSubject.jsp     (exercise — scriptlets)

LAB 8 PROJECT
├── Source Packages
│   ├── com.Model
│   │   └── Car.java
│   ├── com.DAO
│   │   └── CarDAO.java
│   └── com.WEB
│       └── CarServlet.java
└── Web Pages
    ├── index.jsp
    ├── carList.jsp               (JSTL c:forEach)
    ├── carForm.jsp               (JSTL c:if for add/edit)
    └── WEB-INF/web.xml           (static file fix)
```

---

## Key Differences to Remember

| | Lab 7 Exercise | Lab 8 Exercise |
|---|---|---|
| JSP style | **Scriptlets** `<% %>` | **JSTL** `<c:forEach>` |
| Routing | `action` param in URL | `getServletPath()` |
| Servlet mapping | **web.xml** (no annotation) | **@WebServlet("/")** |
| Session | Must get `matricNo` from session | No session needed |
| DB filter | `WHERE matric_no = ?` | No filter, show all |

Good luck tomorrow!
Here's a template you can reuse for ANY MVC CRUD question. I'll show you exactly what to change.

---

## What to Change Checklist

```
When you get a new question, change these things:

1. DATABASE NAME   → carshop, bookstore, clinic etc
2. TABLE NAME      → CarPricelist, books, patients etc
3. CLASS NAME      → Car, Book, Patient etc
4. PACKAGE NAMES   → com.Model, com.DAO, com.WEB (can keep same)
5. VARIABLES       → match the table columns
6. SERVLET NAME    → CarServlet, BookServlet etc
7. JSP NAMES       → carList.jsp, bookList.jsp etc
8. ATTRIBUTE NAMES → "carList", "bookList" etc (setAttribute)
```

---

## TEMPLATE — SQL

```sql
-- CHANGE: database name, table name, columns to match question
CREATE DATABASE IF NOT EXISTS YOUR_DB_NAME;
USE YOUR_DB_NAME;

CREATE TABLE IF NOT EXISTS YOUR_TABLE_NAME (
    id        INT NOT NULL AUTO_INCREMENT,
    column1   VARCHAR(50),   -- change to match question
    column2   VARCHAR(50),   -- change to match question
    column3   INT,           -- change type to match (INT, DOUBLE, VARCHAR)
    column4   DOUBLE,        -- add/remove columns as needed
    PRIMARY KEY (id)
);
```

**Car Shop example:**
```sql
CREATE DATABASE IF NOT EXISTS carshop;
USE carshop;
CREATE TABLE IF NOT EXISTS CarPricelist (
    Car_id    INT NOT NULL AUTO_INCREMENT,
    Brand     VARCHAR(15),
    Model     VARCHAR(30),
    Cyclinder INT,
    Price     DOUBLE,
    PRIMARY KEY (Car_id)
);
```

---

## TEMPLATE — Bean (Model)

```java
package com.Model;

// CHANGE: class name to match your entity e.g. Car, Book, Patient
public class YourEntity {

    // CHANGE: variables to match your table columns
    private int    id;       // always need this
    private String field1;   // match column type: String, int, double
    private String field2;
    private int    field3;
    private double field4;

    // Empty constructor — always keep this
    public YourEntity() {}

    // Constructor WITHOUT id — used for INSERT
    // CHANGE: parameters to match your fields
    public YourEntity(String field1, String field2,
                      int field3, double field4) {
        this.field1 = field1;
        this.field2 = field2;
        this.field3 = field3;
        this.field4 = field4;
    }

    // Constructor WITH id — used for UPDATE and display
    // CHANGE: parameters to match your fields
    public YourEntity(int id, String field1, String field2,
                      int field3, double field4) {
        this.id     = id;
        this.field1 = field1;
        this.field2 = field2;
        this.field3 = field3;
        this.field4 = field4;
    }

    // Getters and Setters — CHANGE names to match fields
    public int    getId()                { return id; }
    public void   setId(int id)          { this.id = id; }

    public String getField1()            { return field1; }
    public void   setField1(String f)    { this.field1 = f; }

    public String getField2()            { return field2; }
    public void   setField2(String f)    { this.field2 = f; }

    public int    getField3()            { return field3; }
    public void   setField3(int f)       { this.field3 = f; }

    public double getField4()            { return field4; }
    public void   setField4(double f)    { this.field4 = f; }
}
```

**Car Shop example:**
```java
package com.Model;

public class Car {

    private int    carId;
    private String brand;
    private String model;
    private int    cyclinder;
    private double price;

    public Car() {}

    // INSERT — no id
    public Car(String brand, String model, int cyclinder, double price) {
        this.brand     = brand;
        this.model     = model;
        this.cyclinder = cyclinder;
        this.price     = price;
    }

    // UPDATE/display — with id
    public Car(int carId, String brand, String model,
               int cyclinder, double price) {
        this.carId     = carId;
        this.brand     = brand;
        this.model     = model;
        this.cyclinder = cyclinder;
        this.price     = price;
    }

    public int    getCarId()                  { return carId; }
    public void   setCarId(int carId)         { this.carId = carId; }

    public String getBrand()                  { return brand; }
    public void   setBrand(String brand)      { this.brand = brand; }

    public String getModel()                  { return model; }
    public void   setModel(String model)      { this.model = model; }

    public int    getCyclinder()              { return cyclinder; }
    public void   setCyclinder(int cyclinder) { this.cyclinder = cyclinder; }

    public double getPrice()                  { return price; }
    public void   setPrice(double price)      { this.price = price; }
}
```

---

## TEMPLATE — DAO

```java
package com.DAO;

// CHANGE: import your entity class
import com.Model.YourEntity;
import java.sql.*;
import java.util.*;

// CHANGE: class name
public class YourEntityDAO {

    // CHANGE: database name in URL + password
    private String jdbcURL      = "jdbc:mysql://localhost:3306/YOUR_DB_NAME";
    private String jdbcUsername = "root";
    private String jdbcPassword = "";

    // CHANGE: table name and column names in all SQL below
    private static final String INSERT_SQL =
        "INSERT INTO YOUR_TABLE (col1, col2, col3, col4) " +
        "VALUES (?, ?, ?, ?)";

    private static final String SELECT_ALL =
        "SELECT * FROM YOUR_TABLE";

    private static final String SELECT_BY_ID =
        "SELECT * FROM YOUR_TABLE WHERE id = ?";

    private static final String UPDATE_SQL =
        "UPDATE YOUR_TABLE " +
        "SET col1=?, col2=?, col3=?, col4=? " +
        "WHERE id=?";

    private static final String DELETE_SQL =
        "DELETE FROM YOUR_TABLE WHERE id=?";

    // NEVER CHANGE THIS — just update driver class name if needed
    protected Connection getConnection() {
        Connection conn = null;
        try {
            Class.forName("com.mysql.cj.jdbc.Driver");
            conn = DriverManager.getConnection(
                       jdbcURL, jdbcUsername, jdbcPassword);
        } catch (Exception e) { e.printStackTrace(); }
        return conn;
    }

    // CREATE — CHANGE: setter names to match your entity
    public void insertEntity(YourEntity entity) throws SQLException {
        try (Connection conn = getConnection();
             PreparedStatement ps = conn.prepareStatement(INSERT_SQL)) {

            // CHANGE: match ? positions to your INSERT_SQL columns
            ps.setString(1, entity.getField1());
            ps.setString(2, entity.getField2());
            ps.setInt(3,    entity.getField3());
            ps.setDouble(4, entity.getField4());
            ps.executeUpdate();

        } catch (SQLException e) { e.printStackTrace(); }
    }

    // READ ALL — CHANGE: entity class name, getter names
    public List<YourEntity> selectAll() {
        List<YourEntity> list = new ArrayList<>();
        try (Connection conn = getConnection();
             PreparedStatement ps = conn.prepareStatement(SELECT_ALL)) {

            ResultSet rs = ps.executeQuery();
            while (rs.next()) {
                // CHANGE: column names to match your table
                list.add(new YourEntity(
                    rs.getInt("id"),
                    rs.getString("col1"),
                    rs.getString("col2"),
                    rs.getInt("col3"),
                    rs.getDouble("col4")));
            }
        } catch (SQLException e) { e.printStackTrace(); }
        return list;
    }

    // READ ONE — CHANGE: entity class name, column names
    public YourEntity selectById(int id) {
        YourEntity entity = null;
        try (Connection conn = getConnection();
             PreparedStatement ps = conn.prepareStatement(SELECT_BY_ID)) {

            ps.setInt(1, id);
            ResultSet rs = ps.executeQuery();
            if (rs.next()) {
                entity = new YourEntity(
                    rs.getInt("id"),
                    rs.getString("col1"),
                    rs.getString("col2"),
                    rs.getInt("col3"),
                    rs.getDouble("col4"));
            }
        } catch (SQLException e) { e.printStackTrace(); }
        return entity;
    }

    // UPDATE — CHANGE: setter names, match ? positions to UPDATE_SQL
    public boolean updateEntity(YourEntity entity) throws SQLException {
        boolean updated;
        try (Connection conn = getConnection();
             PreparedStatement ps = conn.prepareStatement(UPDATE_SQL)) {

            ps.setString(1, entity.getField1());
            ps.setString(2, entity.getField2());
            ps.setInt(3,    entity.getField3());
            ps.setDouble(4, entity.getField4());
            ps.setInt(5,    entity.getId()); // WHERE id — always last
            updated = ps.executeUpdate() > 0;
        }
        return updated;
    }

    // DELETE — NEVER CHANGE THIS, just the method name
    public boolean deleteEntity(int id) throws SQLException {
        boolean deleted;
        try (Connection conn = getConnection();
             PreparedStatement ps = conn.prepareStatement(DELETE_SQL)) {

            ps.setInt(1, id);
            deleted = ps.executeUpdate() > 0;
        }
        return deleted;
    }
}
```

---

## TEMPLATE — Servlet

```java
package com.WEB;

// CHANGE: import your DAO and Entity
import com.DAO.YourEntityDAO;
import com.Model.YourEntity;
import java.io.IOException;
import java.sql.SQLException;
import java.util.List;
import javax.servlet.*;
import javax.servlet.http.*;

// NO @WebServlet — using web.xml instead
// CHANGE: class name
public class YourEntityServlet extends HttpServlet {

    // CHANGE: DAO type and name
    private YourEntityDAO dao;

    @Override
    public void init() {
        dao = new YourEntityDAO(); // CHANGE: DAO class name
    }

    @Override
    protected void doPost(HttpServletRequest request,
                          HttpServletResponse response)
            throws ServletException, IOException {
        doGet(request, response); // NEVER CHANGE THIS
    }

    @Override
    protected void doGet(HttpServletRequest request,
                         HttpServletResponse response)
            throws ServletException, IOException {

        String action = request.getServletPath(); // NEVER CHANGE THIS

        try {
            switch (action) {
                // NEVER CHANGE these case names
                case "/new":    showNewForm(request, response);  break;
                case "/insert": insertEntity(request, response); break;
                case "/edit":   showEditForm(request, response); break;
                case "/update": updateEntity(request, response); break;
                case "/delete": deleteEntity(request, response); break;
                default:        listEntity(request, response);   break;
            }
        } catch (SQLException ex) {
            throw new ServletException(ex);
        }
    }

    // READ ALL
    // CHANGE: "entityList" attribute name, "entityList.jsp" JSP name
    private void listEntity(HttpServletRequest request,
                             HttpServletResponse response)
            throws SQLException, IOException, ServletException {

        List<YourEntity> list = dao.selectAll();
        request.setAttribute("entityList", list); // CHANGE attribute name
        request.getRequestDispatcher("entityList.jsp") // CHANGE jsp name
               .forward(request, response);
    }

    // SHOW ADD FORM
    // CHANGE: "entityForm.jsp" JSP name
    private void showNewForm(HttpServletRequest request,
                              HttpServletResponse response)
            throws ServletException, IOException {

        request.getRequestDispatcher("entityForm.jsp") // CHANGE jsp name
               .forward(request, response);
    }

    // SHOW EDIT FORM
    // CHANGE: "entity" attribute name, selectById method, JSP name
    private void showEditForm(HttpServletRequest request,
                               HttpServletResponse response)
            throws SQLException, ServletException, IOException {

        int id = Integer.parseInt(request.getParameter("id"));
        YourEntity entity = dao.selectById(id);
        request.setAttribute("entity", entity); // CHANGE attribute name
        request.getRequestDispatcher("entityForm.jsp") // CHANGE jsp name
               .forward(request, response);
    }

    // INSERT
    // CHANGE: getParameter names to match your form fields
    private void insertEntity(HttpServletRequest request,
                               HttpServletResponse response)
            throws SQLException, IOException {

        // CHANGE: parameter names match your HTML form input names
        YourEntity entity = new YourEntity(
            request.getParameter("field1"),
            request.getParameter("field2"),
            Integer.parseInt(request.getParameter("field3")),
            Double.parseDouble(request.getParameter("field4")));

        dao.insertEntity(entity);
        response.sendRedirect("list"); // NEVER CHANGE
    }

    // UPDATE
    // CHANGE: getParameter names to match your form fields
    private void updateEntity(HttpServletRequest request,
                               HttpServletResponse response)
            throws SQLException, IOException {

        YourEntity entity = new YourEntity(
            Integer.parseInt(request.getParameter("id")), // hidden field
            request.getParameter("field1"),
            request.getParameter("field2"),
            Integer.parseInt(request.getParameter("field3")),
            Double.parseDouble(request.getParameter("field4")));

        dao.updateEntity(entity);
        response.sendRedirect("list"); // NEVER CHANGE
    }

    // DELETE — NEVER CHANGE THIS
    private void deleteEntity(HttpServletRequest request,
                               HttpServletResponse response)
            throws SQLException, IOException {

        int id = Integer.parseInt(request.getParameter("id"));
        dao.deleteEntity(id);
        response.sendRedirect("list"); // NEVER CHANGE
    }
}
```

---

## TEMPLATE — `web.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app>

    <!-- CHANGE: servlet-name and servlet-class to match yours -->
    <servlet>
        <servlet-name>YourEntityServlet</servlet-name>
        <servlet-class>com.WEB.YourEntityServlet</servlet-class>
    </servlet>
    <servlet-mapping>
        <servlet-name>YourEntityServlet</servlet-name>
        <url-pattern>/</url-pattern> <!-- NEVER CHANGE -->
    </servlet-mapping>

    <!-- NEVER CHANGE THESE 3 — fixes static files -->
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

</web-app>
```

---

## TEMPLATE — `entityList.jsp`

```jsp
<%@ page language="java" contentType="text/html; charset=UTF-8"%>
<%@ taglib uri="http://java.sun.com/jsp/jstl/core" prefix="c"%>
<!DOCTYPE html>
<html>
<head>
    <title>YOUR TITLE</title><!-- CHANGE -->
    <link rel="stylesheet" href="https://stackpath.bootstrapcdn.com/bootstrap/4.3.1/css/bootstrap.min.css">
</head>
<body>

<nav class="navbar navbar-expand-md navbar-dark" style="background-color: tomato">
    <a href="" class="navbar-brand">YOUR APP NAME</a><!-- CHANGE -->
    <ul class="navbar-nav">
        <li><a href="<%=request.getContextPath()%>/list" class="nav-link">
            YOUR ENTITY<!-- CHANGE e.g. Cars, Books -->
        </a></li>
    </ul>
</nav>

<br>
<div class="container">
    <h3 class="text-center">List of YOUR ENTITY</h3><!-- CHANGE -->
    <hr>
    <a href="<%=request.getContextPath()%>/new" class="btn btn-success">
        Add New YOUR ENTITY <!-- CHANGE -->
    </a>
    <br><br>

    <table class="table table-bordered">
        <thead>
            <tr>
                <th>ID</th>
                <th>Field 1</th>  <!-- CHANGE to your column names -->
                <th>Field 2</th>
                <th>Field 3</th>
                <th>Field 4</th>
                <th>Actions</th>
            </tr>
        </thead>
        <tbody>
            <%-- CHANGE: "entityList" and "entity" to match setAttribute name --%>
            <c:forEach var="entity" items="${entityList}">
                <tr>
                    <td><c:out value="${entity.id}"/></td>
                    <%-- CHANGE: getter names e.g. entity.brand, entity.model --%>
                    <td><c:out value="${entity.field1}"/></td>
                    <td><c:out value="${entity.field2}"/></td>
                    <td><c:out value="${entity.field3}"/></td>
                    <td><c:out value="${entity.field4}"/></td>
                    <td>
                        <a href="edit?id=<c:out value='${entity.id}'/>"
                           class="btn btn-primary btn-sm">Edit</a>
                        &nbsp;
                        <a href="delete?id=<c:out value='${entity.id}'/>"
                           class="btn btn-danger btn-sm"
                           onclick="return confirm('Delete this?')">Delete</a>
                    </td>
                </tr>
            </c:forEach>
        </tbody>
    </table>
</div>

</body>
</html>
```

---

## TEMPLATE — `entityForm.jsp`

```jsp
<%@ page language="java" contentType="text/html; charset=UTF-8"%>
<%@ taglib uri="http://java.sun.com/jsp/jstl/core" prefix="c"%>
<!DOCTYPE html>
<html>
<head>
    <title>YOUR TITLE</title><!-- CHANGE -->
    <link rel="stylesheet" href="https://stackpath.bootstrapcdn.com/bootstrap/4.3.1/css/bootstrap.min.css">
</head>
<body>

<nav class="navbar navbar-expand-md navbar-dark" style="background-color: tomato">
    <a href="" class="navbar-brand">YOUR APP NAME</a><!-- CHANGE -->
</nav>

<br>
<div class="container col-md-5">
<div class="card">
<div class="card-body">

    <%-- CHANGE "entity" to match your setAttribute name --%>
    <c:if test="${entity != null}">
        <form action="update" method="post">
        <input type="hidden" name="id" value="<c:out value='${entity.id}'/>">
    </c:if>
    <c:if test="${entity == null}">
        <form action="insert" method="post">
    </c:if>

        <h2>
            <c:if test="${entity != null}">Edit YOUR ENTITY</c:if><!-- CHANGE -->
            <c:if test="${entity == null}">Add New YOUR ENTITY</c:if><!-- CHANGE -->
        </h2>

        <!-- CHANGE: label, input name, and ${entity.field1} for each field -->
        <div class="form-group">
            <label>Field 1 Label</label>
            <input type="text" name="field1" class="form-control"
                   value="<c:out value='${entity.field1}'/>" required>
        </div>

        <div class="form-group">
            <label>Field 2 Label</label>
            <input type="text" name="field2" class="form-control"
                   value="<c:out value='${entity.field2}'/>">
        </div>

        <!-- For INT fields use type="number" -->
        <div class="form-group">
            <label>Field 3 Label</label>
            <input type="number" name="field3" class="form-control"
                   value="<c:out value='${entity.field3}'/>">
        </div>

        <!-- For DOUBLE fields use type="number" step="0.01" -->
        <div class="form-group">
            <label>Field 4 Label</label>
            <input type="number" step="0.01" name="field4" class="form-control"
                   value="<c:out value='${entity.field4}'/>">
        </div>

        <button type="submit" class="btn btn-success">Save</button>
        <a href="list" class="btn btn-secondary">Cancel</a>

    </form>

</div>
</div>
</div>

</body>
</html>
```

---

## Quick Change Summary Card

```
NEW QUESTION COMES IN → follow this order:

1. SQL        → change DB name,
