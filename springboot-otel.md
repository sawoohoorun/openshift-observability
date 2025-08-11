# Complete Spring Boot 3-tier Application with OpenTelemetry Guide - From Development to Production

This comprehensive guide covers everything from creating a Spring Boot application using Spring Initializr to deploying it on OpenShift with PostgreSQL database and OpenTelemetry distributed tracing.

## Table of Contents
1. [Project Creation from Spring Initializr](#1-project-creation-from-spring-initializr)
2. [Application Code Implementation](#2-application-code-implementation)
3. [PostgreSQL Database Setup](#3-postgresql-database-setup)
4. [Docker Image Build and Registry Push](#4-docker-image-build-and-registry-push)
5. [OpenTelemetry Tenant Configuration](#5-opentelemetry-tenant-configuration)
6. [Application Deployment to OpenShift](#6-application-deployment-to-openshift)
7. [Testing and Verification](#7-testing-and-verification)

---

## 1. Project Creation from Spring Initializr

### 1.1 Generate Project
1. Navigate to [https://start.spring.io/](https://start.spring.io/)
2. Configure the project:
   - **Project**: Maven
   - **Language**: Java
   - **Spring Boot**: 3.2.0
   - **Group**: `com.example`
   - **Artifact**: `employee-management`
   - **Name**: `employee-management`
   - **Description**: `Employee Management System with OpenTelemetry Tracing`
   - **Package name**: `com.example.employeemanagement`
   - **Packaging**: Jar
   - **Java**: 17

3. Add Dependencies:
   - Spring Web
   - Spring Data JPA
   - PostgreSQL Driver
   - Validation
   - Spring Boot Actuator

4. Click "GENERATE" and extract the downloaded ZIP

### 1.2 Project Structure Setup
```bash
cd employee-management
mkdir -p src/main/java/com/example/employeemanagement/{entity,repository,service,controller,dto,mapper,exception,config}
```

---

## 2. Application Code Implementation

### 2.1 Maven Configuration
**File**: `pom.xml`
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.0</version>
        <relativePath/>
    </parent>
    
    <groupId>com.example</groupId>
    <artifactId>employee-management</artifactId>
    <version>1.0.0</version>
    <packaging>jar</packaging>
    <name>employee-management</name>
    <description>Employee Management System with OpenTelemetry Tracing</description>
    
    <properties>
        <java.version>17</java.version>
        <opentelemetry.version>2.18.1</opentelemetry.version>
    </properties>
    
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>io.opentelemetry.instrumentation</groupId>
                <artifactId>opentelemetry-instrumentation-bom</artifactId>
                <version>${opentelemetry.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
    
    <dependencies>
        <!-- Spring Boot Starters -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-validation</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        
        <!-- PostgreSQL Driver -->
        <dependency>
            <groupId>org.postgresql</groupId>
            <artifactId>postgresql</artifactId>
            <scope>runtime</scope>
        </dependency>
        
        <!-- OpenTelemetry -->
        <dependency>
            <groupId>io.opentelemetry.instrumentation</groupId>
            <artifactId>opentelemetry-spring-boot-starter</artifactId>
        </dependency>
        
        <!-- Additional utilities -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
        
        <!-- Test Dependencies -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
    
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <layers>
                        <enabled>true</enabled>
                    </layers>
                    <excludes>
                        <exclude>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok</artifactId>
                        </exclude>
                    </excludes>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

### 2.2 Application Configuration
**File**: `src/main/resources/application.yml` (replace application.properties)
```yaml
spring:
  application:
    name: employee-management-service
  
  datasource:
    url: jdbc:postgresql://${DB_HOST:localhost}:${DB_PORT:5432}/${DB_NAME:employeedb}
    username: ${DB_USERNAME:appuser}
    password: ${DB_PASSWORD:password}
    driver-class-name: org.postgresql.Driver
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 30000
  
  jpa:
    hibernate:
      ddl-auto: validate
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQLDialect
        format_sql: true
    show-sql: false

# OpenTelemetry Configuration
otel:
  sdk:
    disabled: false
  service:
    name: ${spring.application.name}
  resource:
    attributes:
      service.namespace: employee-management
      service.version: 1.0.0
      deployment.environment: ${ENVIRONMENT:development}
      tenant: ${TENANT_NAME:tracing-db}
  exporter:
    otlp:
      endpoint: ${OTEL_ENDPOINT:http://localhost:4318}
      protocol: http/protobuf
      timeout: 10s
      headers:
        X-Scope-OrgID: ${TENANT_NAME:tracing-db}
  traces:
    sampler: parentbased_traceidratio
    sampler:
      arg: ${SAMPLING_RATE:1.0}
  instrumentation:
    jdbc:
      enabled: true
      statement-sanitizer:
        enabled: false
    spring-webmvc:
      enabled: true
    logback-appender:
      enabled: true

# Actuator Configuration
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      show-details: when-authorized
      probes:
        enabled: true
  health:
    livenessstate:
      enabled: true
    readinessstate:
      enabled: true
  tracing:
    enabled: true
    sampling:
      probability: ${SAMPLING_RATE:1.0}

# Logging with trace correlation
logging:
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level [%X{traceId},%X{spanId}] %logger{36} - %msg%n"
  level:
    io.opentelemetry: INFO
    org.springframework.web: DEBUG
    com.example.employeemanagement: DEBUG
```

### 2.3 Entity Classes

**File**: `src/main/java/com/example/employeemanagement/entity/EmployeeStatus.java`
```java
package com.example.employeemanagement.entity;

public enum EmployeeStatus {
    ACTIVE, INACTIVE, ON_LEAVE, TERMINATED
}
```

**File**: `src/main/java/com/example/employeemanagement/entity/Department.java`
```java
package com.example.employeemanagement.entity;

import jakarta.persistence.*;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Size;
import lombok.Data;
import lombok.NoArgsConstructor;
import lombok.AllArgsConstructor;
import org.hibernate.annotations.CreationTimestamp;
import org.hibernate.annotations.UpdateTimestamp;

import java.time.LocalDateTime;
import java.util.ArrayList;
import java.util.List;

@Entity
@Table(name = "departments")
@Data
@NoArgsConstructor
@AllArgsConstructor
public class Department {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @NotBlank
    @Size(min = 2, max = 100)
    @Column(nullable = false, unique = true)
    private String name;
    
    @Size(max = 500)
    private String description;
    
    @OneToMany(mappedBy = "department", cascade = CascadeType.ALL)
    private List<Employee> employees = new ArrayList<>();
    
    @CreationTimestamp
    @Column(nullable = false, updatable = false)
    private LocalDateTime createdAt;
    
    @UpdateTimestamp
    @Column(nullable = false)
    private LocalDateTime updatedAt;
}
```

**File**: `src/main/java/com/example/employeemanagement/entity/Employee.java`
```java
package com.example.employeemanagement.entity;

import jakarta.persistence.*;
import jakarta.validation.constraints.*;
import lombok.Data;
import lombok.NoArgsConstructor;
import lombok.AllArgsConstructor;
import org.hibernate.annotations.CreationTimestamp;
import org.hibernate.annotations.UpdateTimestamp;

import java.math.BigDecimal;
import java.time.LocalDateTime;

@Entity
@Table(name = "employees")
@Data
@NoArgsConstructor
@AllArgsConstructor
public class Employee {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @NotBlank(message = "First name is required")
    @Size(min = 2, max = 50)
    @Column(nullable = false, length = 50)
    private String firstName;
    
    @NotBlank(message = "Last name is required")
    @Size(min = 2, max = 50)
    @Column(nullable = false, length = 50)
    private String lastName;
    
    @Email(message = "Email should be valid")
    @NotBlank(message = "Email is required")
    @Column(nullable = false, unique = true)
    private String email;
    
    @NotNull(message = "Salary is required")
    @DecimalMin(value = "0.0", inclusive = false)
    @Column(nullable = false, precision = 10, scale = 2)
    private BigDecimal salary;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "department_id")
    private Department department;
    
    @Enumerated(EnumType.STRING)
    @Column(length = 20)
    private EmployeeStatus status = EmployeeStatus.ACTIVE;
    
    @CreationTimestamp
    @Column(nullable = false, updatable = false)
    private LocalDateTime createdAt;
    
    @UpdateTimestamp
    @Column(nullable = false)
    private LocalDateTime updatedAt;
}
```

### 2.4 Repository Layer

**File**: `src/main/java/com/example/employeemanagement/repository/EmployeeRepository.java`
```java
package com.example.employeemanagement.repository;

import com.example.employeemanagement.entity.Employee;
import com.example.employeemanagement.entity.EmployeeStatus;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;

import java.math.BigDecimal;
import java.util.List;
import java.util.Optional;

@Repository
public interface EmployeeRepository extends JpaRepository<Employee, Long> {
    
    Optional<Employee> findByEmail(String email);
    
    List<Employee> findByStatus(EmployeeStatus status);
    
    Page<Employee> findByDepartmentId(Long departmentId, Pageable pageable);
    
    @Query("SELECT e FROM Employee e WHERE e.salary BETWEEN :minSalary AND :maxSalary")
    List<Employee> findBySalaryRange(@Param("minSalary") BigDecimal minSalary, 
                                     @Param("maxSalary") BigDecimal maxSalary);
    
    @Query("SELECT e FROM Employee e WHERE LOWER(e.firstName) LIKE LOWER(CONCAT('%', :name, '%')) " +
           "OR LOWER(e.lastName) LIKE LOWER(CONCAT('%', :name, '%'))")
    Page<Employee> searchByName(@Param("name") String name, Pageable pageable);
}
```

**File**: `src/main/java/com/example/employeemanagement/repository/DepartmentRepository.java`
```java
package com.example.employeemanagement.repository;

import com.example.employeemanagement.entity.Department;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;

import java.util.Optional;

@Repository
public interface DepartmentRepository extends JpaRepository<Department, Long> {
    
    Optional<Department> findByName(String name);
    
    @Query("SELECT d FROM Department d LEFT JOIN FETCH d.employees WHERE d.id = :id")
    Optional<Department> findByIdWithEmployees(@Param("id") Long id);
}
```

### 2.5 DTO Classes

**File**: `src/main/java/com/example/employeemanagement/dto/CreateEmployeeDto.java`
```java
package com.example.employeemanagement.dto;

import jakarta.validation.constraints.*;
import lombok.Data;
import java.math.BigDecimal;

@Data
public class CreateEmployeeDto {
    @NotBlank(message = "First name is required")
    @Size(min = 2, max = 50)
    private String firstName;
    
    @NotBlank(message = "Last name is required")
    @Size(min = 2, max = 50)
    private String lastName;
    
    @Email(message = "Email should be valid")
    @NotBlank(message = "Email is required")
    private String email;
    
    @NotNull(message = "Salary is required")
    @DecimalMin(value = "0.0", inclusive = false)
    private BigDecimal salary;
    
    private Long departmentId;
}
```

**File**: `src/main/java/com/example/employeemanagement/dto/UpdateEmployeeDto.java`
```java
package com.example.employeemanagement.dto;

import com.example.employeemanagement.entity.EmployeeStatus;
import jakarta.validation.constraints.*;
import lombok.Data;
import java.math.BigDecimal;

@Data
public class UpdateEmployeeDto {
    @Size(min = 2, max = 50)
    private String firstName;
    
    @Size(min = 2, max = 50)
    private String lastName;
    
    @Email
    private String email;
    
    @DecimalMin(value = "0.0", inclusive = false)
    private BigDecimal salary;
    
    private Long departmentId;
    private EmployeeStatus status;
}
```

**File**: `src/main/java/com/example/employeemanagement/dto/EmployeeResponseDto.java`
```java
package com.example.employeemanagement.dto;

import com.example.employeemanagement.entity.EmployeeStatus;
import lombok.Data;
import java.math.BigDecimal;
import java.time.LocalDateTime;

@Data
public class EmployeeResponseDto {
    private Long id;
    private String firstName;
    private String lastName;
    private String email;
    private BigDecimal salary;
    private String departmentName;
    private EmployeeStatus status;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
}
```

### 2.6 Exception Handling

**File**: `src/main/java/com/example/employeemanagement/exception/ResourceNotFoundException.java`
```java
package com.example.employeemanagement.exception;

public class ResourceNotFoundException extends RuntimeException {
    public ResourceNotFoundException(String message) {
        super(message);
    }
}
```

**File**: `src/main/java/com/example/employeemanagement/exception/DuplicateResourceException.java`
```java
package com.example.employeemanagement.exception;

public class DuplicateResourceException extends RuntimeException {
    public DuplicateResourceException(String message) {
        super(message);
    }
}
```

**File**: `src/main/java/com/example/employeemanagement/exception/GlobalExceptionHandler.java`
```java
package com.example.employeemanagement.exception;

import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.validation.FieldError;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

import java.time.LocalDateTime;
import java.util.HashMap;
import java.util.Map;

@RestControllerAdvice
public class GlobalExceptionHandler {
    
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleResourceNotFound(ResourceNotFoundException ex) {
        ErrorResponse error = new ErrorResponse(
            HttpStatus.NOT_FOUND.value(),
            ex.getMessage(),
            LocalDateTime.now()
        );
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
    }
    
    @ExceptionHandler(DuplicateResourceException.class)
    public ResponseEntity<ErrorResponse> handleDuplicateResource(DuplicateResourceException ex) {
        ErrorResponse error = new ErrorResponse(
            HttpStatus.CONFLICT.value(),
            ex.getMessage(),
            LocalDateTime.now()
        );
        return ResponseEntity.status(HttpStatus.CONFLICT).body(error);
    }
    
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<Map<String, String>> handleValidationExceptions(
            MethodArgumentNotValidException ex) {
        Map<String, String> errors = new HashMap<>();
        ex.getBindingResult().getAllErrors().forEach((error) -> {
            String fieldName = ((FieldError) error).getField();
            String errorMessage = error.getDefaultMessage();
            errors.put(fieldName, errorMessage);
        });
        return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(errors);
    }
    
    public static class ErrorResponse {
        private int status;
        private String message;
        private LocalDateTime timestamp;
        
        public ErrorResponse(int status, String message, LocalDateTime timestamp) {
            this.status = status;
            this.message = message;
            this.timestamp = timestamp;
        }
        
        // Getters
        public int getStatus() { return status; }
        public String getMessage() { return message; }
        public LocalDateTime getTimestamp() { return timestamp; }
    }
}
```

### 2.7 Mapper

**File**: `src/main/java/com/example/employeemanagement/mapper/EmployeeMapper.java`
```java
package com.example.employeemanagement.mapper;

import com.example.employeemanagement.dto.CreateEmployeeDto;
import com.example.employeemanagement.dto.EmployeeResponseDto;
import com.example.employeemanagement.dto.UpdateEmployeeDto;
import com.example.employeemanagement.entity.Employee;
import org.springframework.stereotype.Component;

@Component
public class EmployeeMapper {
    
    public Employee toEntity(CreateEmployeeDto dto) {
        Employee employee = new Employee();
        employee.setFirstName(dto.getFirstName());
        employee.setLastName(dto.getLastName());
        employee.setEmail(dto.getEmail());
        employee.setSalary(dto.getSalary());
        return employee;
    }
    
    public EmployeeResponseDto toResponseDto(Employee employee) {
        EmployeeResponseDto dto = new EmployeeResponseDto();
        dto.setId(employee.getId());
        dto.setFirstName(employee.getFirstName());
        dto.setLastName(employee.getLastName());
        dto.setEmail(employee.getEmail());
        dto.setSalary(employee.getSalary());
        dto.setStatus(employee.getStatus());
        dto.setCreatedAt(employee.getCreatedAt());
        dto.setUpdatedAt(employee.getUpdatedAt());
        
        if (employee.getDepartment() != null) {
            dto.setDepartmentName(employee.getDepartment().getName());
        }
        
        return dto;
    }
    
    public void updateEntity(Employee employee, UpdateEmployeeDto dto) {
        if (dto.getFirstName() != null) {
            employee.setFirstName(dto.getFirstName());
        }
        if (dto.getLastName() != null) {
            employee.setLastName(dto.getLastName());
        }
        if (dto.getEmail() != null) {
            employee.setEmail(dto.getEmail());
        }
        if (dto.getSalary() != null) {
            employee.setSalary(dto.getSalary());
        }
        if (dto.getStatus() != null) {
            employee.setStatus(dto.getStatus());
        }
    }
}
```

### 2.8 Service Layer

**File**: `src/main/java/com/example/employeemanagement/service/EmployeeService.java`
```java
package com.example.employeemanagement.service;

import com.example.employeemanagement.dto.*;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;

public interface EmployeeService {
    EmployeeResponseDto createEmployee(CreateEmployeeDto createDto);
    EmployeeResponseDto updateEmployee(Long id, UpdateEmployeeDto updateDto);
    EmployeeResponseDto getEmployeeById(Long id);
    Page<EmployeeResponseDto> getAllEmployees(Pageable pageable);
    void deleteEmployee(Long id);
    Page<EmployeeResponseDto> searchEmployees(String query, Pageable pageable);
}
```

**File**: `src/main/java/com/example/employeemanagement/service/EmployeeServiceImpl.java`
```java
package com.example.employeemanagement.service;

import com.example.employeemanagement.dto.*;
import com.example.employeemanagement.entity.Department;
import com.example.employeemanagement.entity.Employee;
import com.example.employeemanagement.exception.DuplicateResourceException;
import com.example.employeemanagement.exception.ResourceNotFoundException;
import com.example.employeemanagement.mapper.EmployeeMapper;
import com.example.employeemanagement.repository.DepartmentRepository;
import com.example.employeemanagement.repository.EmployeeRepository;
import io.opentelemetry.api.trace.Span;
import io.opentelemetry.instrumentation.annotations.WithSpan;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
@Transactional
@Slf4j
public class EmployeeServiceImpl implements EmployeeService {
    
    private final EmployeeRepository employeeRepository;
    private final DepartmentRepository departmentRepository;
    private final EmployeeMapper employeeMapper;
    
    @Autowired
    public EmployeeServiceImpl(EmployeeRepository employeeRepository,
                               DepartmentRepository departmentRepository,
                               EmployeeMapper employeeMapper) {
        this.employeeRepository = employeeRepository;
        this.departmentRepository = departmentRepository;
        this.employeeMapper = employeeMapper;
    }
    
    @Override
    @WithSpan("create-employee")
    public EmployeeResponseDto createEmployee(CreateEmployeeDto createDto) {
        Span span = Span.current();
        span.setAttribute("employee.email", createDto.getEmail());
        span.setAttribute("operation.type", "create");
        
        log.info("Creating new employee with email: {}", createDto.getEmail());
        
        // Check if email already exists - this will generate a SELECT query
        if (employeeRepository.findByEmail(createDto.getEmail()).isPresent()) {
            throw new DuplicateResourceException("Employee with email already exists");
        }
        
        Employee employee = employeeMapper.toEntity(createDto);
        
        if (createDto.getDepartmentId() != null) {
            // This will generate another SELECT query
            Department department = departmentRepository.findById(createDto.getDepartmentId())
                .orElseThrow(() -> new ResourceNotFoundException("Department not found"));
            employee.setDepartment(department);
            span.setAttribute("department.id", createDto.getDepartmentId());
        }
        
        // This will generate an INSERT query
        Employee savedEmployee = employeeRepository.save(employee);
        span.setAttribute("employee.id", savedEmployee.getId());
        
        log.info("Successfully created employee with ID: {}", savedEmployee.getId());
        return employeeMapper.toResponseDto(savedEmployee);
    }
    
    @Override
    @WithSpan("update-employee")
    public EmployeeResponseDto updateEmployee(Long id, UpdateEmployeeDto updateDto) {
        Span span = Span.current();
        span.setAttribute("employee.id", id);
        span.setAttribute("operation.type", "update");
        
        // This will generate a SELECT query
        Employee employee = employeeRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("Employee not found"));
        
        employeeMapper.updateEntity(employee, updateDto);
        
        if (updateDto.getDepartmentId() != null) {
            // Another SELECT query
            Department department = departmentRepository.findById(updateDto.getDepartmentId())
                .orElseThrow(() -> new ResourceNotFoundException("Department not found"));
            employee.setDepartment(department);
        }
        
        // This will generate an UPDATE query
        Employee updatedEmployee = employeeRepository.save(employee);
        log.info("Successfully updated employee with ID: {}", id);
        return employeeMapper.toResponseDto(updatedEmployee);
    }
    
    @Override
    @Transactional(readOnly = true)
    @WithSpan("get-employee")
    public EmployeeResponseDto getEmployeeById(Long id) {
        Span.current().setAttribute("employee.id", id);
        // This will generate a SELECT query with JOIN
        Employee employee = employeeRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("Employee not found"));
        return employeeMapper.toResponseDto(employee);
    }
    
    @Override
    @Transactional(readOnly = true)
    @WithSpan("get-all-employees")
    public Page<EmployeeResponseDto> getAllEmployees(Pageable pageable) {
        Span span = Span.current();
        span.setAttribute("page.number", pageable.getPageNumber());
        span.setAttribute("page.size", pageable.getPageSize());
        
        // This will generate a paginated SELECT query
        Page<Employee> employeePage = employeeRepository.findAll(pageable);
        span.setAttribute("total.elements", employeePage.getTotalElements());
        
        return employeePage.map(employeeMapper::toResponseDto);
    }
    
    @Override
    @WithSpan("delete-employee")
    public void deleteEmployee(Long id) {
        Span span = Span.current();
        span.setAttribute("employee.id", id);
        span.setAttribute("operation.type", "delete");
        
        // SELECT query to check existence
        Employee employee = employeeRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("Employee not found"));
        
        // DELETE query
        employeeRepository.delete(employee);
        log.info("Successfully deleted employee with ID: {}", id);
    }
    
    @Override
    @Transactional(readOnly = true)
    @WithSpan("search-employees")
    public Page<EmployeeResponseDto> searchEmployees(String query, Pageable pageable) {
        Span span = Span.current();
        span.setAttribute("search.query", query);
        span.setAttribute("operation.type", "search");
        
        // This will generate a complex SELECT with LIKE queries
        Page<Employee> employeePage = employeeRepository.searchByName(query, pageable);
        span.setAttribute("search.results.count", employeePage.getTotalElements());
        
        return employeePage.map(employeeMapper::toResponseDto);
    }
}
```

### 2.9 Controller Layer

**File**: `src/main/java/com/example/employeemanagement/controller/EmployeeController.java`
```java
package com.example.employeemanagement.controller;

import com.example.employeemanagement.dto.*;
import com.example.employeemanagement.service.EmployeeService;
import io.opentelemetry.api.trace.Span;
import jakarta.validation.Valid;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/v1/employees")
@Slf4j
public class EmployeeController {
    
    private final EmployeeService employeeService;
    
    @Autowired
    public EmployeeController(EmployeeService employeeService) {
        this.employeeService = employeeService;
    }
    
    @PostMapping
    public ResponseEntity<EmployeeResponseDto> createEmployee(@Valid @RequestBody CreateEmployeeDto createDto) {
        Span span = Span.current();
        span.setAttribute("http.method", "POST");
        span.setAttribute("http.route", "/api/v1/employees");
        span.setAttribute("http.request.body.email", createDto.getEmail());
        
        log.info("Received request to create employee: {}", createDto.getEmail());
        EmployeeResponseDto response = employeeService.createEmployee(createDto);
        return ResponseEntity.status(HttpStatus.CREATED).body(response);
    }
    
    @GetMapping("/{id}")
    public ResponseEntity<EmployeeResponseDto> getEmployee(@PathVariable Long id) {
        Span span = Span.current();
        span.setAttribute("http.method", "GET");
        span.setAttribute("http.route", "/api/v1/employees/{id}");
        span.setAttribute("employee.id", id);
        
        return ResponseEntity.ok(employeeService.getEmployeeById(id));
    }
    
    @GetMapping
    public ResponseEntity<Page<EmployeeResponseDto>> getAllEmployees(Pageable pageable) {
        Span span = Span.current();
        span.setAttribute("http.method", "GET");
        span.setAttribute("http.route", "/api/v1/employees");
        span.setAttribute("page.number", pageable.getPageNumber());
        span.setAttribute("page.size", pageable.getPageSize());
        
        return ResponseEntity.ok(employeeService.getAllEmployees(pageable));
    }
    
    @PutMapping("/{id}")
    public ResponseEntity<EmployeeResponseDto> updateEmployee(
            @PathVariable Long id,
            @Valid @RequestBody UpdateEmployeeDto updateDto) {
        Span span = Span.current();
        span.setAttribute("http.method", "PUT");
        span.setAttribute("http.route", "/api/v1/employees/{id}");
        span.setAttribute("employee.id", id);
        
        return ResponseEntity.ok(employeeService.updateEmployee(id, updateDto));
    }
    
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteEmployee(@PathVariable Long id) {
        Span span = Span.current();
        span.setAttribute("http.method", "DELETE");
        span.setAttribute("http.route", "/api/v1/employees/{id}");
        span.setAttribute("employee.id", id);
        
        employeeService.deleteEmployee(id);
        return ResponseEntity.noContent().build();
    }
    
    @GetMapping("/search")
    public ResponseEntity<Page<EmployeeResponseDto>> searchEmployees(
            @RequestParam String query,
            Pageable pageable) {
        Span span = Span.current();
        span.setAttribute("http.method", "GET");
        span.setAttribute("http.route", "/api/v1/employees/search");
        span.setAttribute("search.query", query);
        
        return ResponseEntity.ok(employeeService.searchEmployees(query, pageable));
    }
    
    @GetMapping("/health")
    public ResponseEntity<String> health() {
        return ResponseEntity.ok("Employee Management Service is running!");
    }
}
```

### 2.10 Dockerfile

**File**: `Dockerfile` (in project root)
```dockerfile
# Multi-stage build for Spring Boot with OpenTelemetry
FROM bellsoft/liberica-openjdk-debian:17-cds AS builder
WORKDIR /builder

# Download OpenTelemetry Java Agent
RUN apt-get update && apt-get install -y curl && \
    curl -L -o opentelemetry-javaagent.jar \
    https://github.com/open-telemetry/opentelemetry-java-instrumentation/releases/latest/download/opentelemetry-javaagent.jar

# Copy Maven files and source
COPY .mvn .mvn
COPY mvnw .
COPY pom.xml .
COPY src src

# Build the application
RUN ./mvnw clean package -DskipTests

# Extract layers for better caching
RUN java -Djarmode=tools -jar target/*.jar extract --layers --destination extracted

# Runtime stage
FROM bellsoft/liberica-openjre-debian:17-cds
WORKDIR /application

# Create non-root user
RUN groupadd -r spring && useradd -r -g spring spring

# Copy OpenTelemetry agent
COPY --from=builder /builder/opentelemetry-javaagent.jar /opt/opentelemetry-javaagent.jar

# Copy extracted layers
COPY --from=builder /builder/extracted/dependencies/ ./
COPY --from=builder /builder/extracted/spring-boot-loader/ ./
COPY --from=builder /builder/extracted/snapshot-dependencies/ ./
COPY --from=builder /builder/extracted/application/ ./

# Set ownership
RUN chown -R spring:spring /application

USER spring

# Health check
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
    CMD curl -f http://localhost:8080/actuator/health || exit 1

# JVM settings optimized for containers
ENV JAVA_OPTS="-XX:MaxRAMPercentage=70.0 \
    -XX:+UseG1GC \
    -XX:+UseContainerSupport \
    -XX:+UseStringDeduplication \
    -XX:+ExitOnOutOfMemoryError"

# OpenTelemetry configuration
ENV OTEL_JAVAAGENT_ENABLED=true
ENV OTEL_INSTRUMENTATION_JDBC_STATEMENT_SANITIZER_ENABLED=false

EXPOSE 8080

# Run with OpenTelemetry agent
ENTRYPOINT ["sh", "-c", "java ${JAVA_OPTS} -javaagent:/opt/opentelemetry-javaagent.jar -jar application.jar"]
```

---

## 3. PostgreSQL Database Setup

### 3.1 Database Schema and Sample Data

**File**: `database-init.sql`
```sql
-- Create the database
CREATE DATABASE employeedb;

-- Connect to the database
\c employeedb;

-- Create user
CREATE USER appuser WITH PASSWORD 'securePass123';
GRANT ALL PRIVILEGES ON DATABASE employeedb TO appuser;

-- Grant schema permissions
GRANT ALL ON SCHEMA public TO appuser;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO appuser;
GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA public TO appuser;

-- Create departments table
CREATE TABLE IF NOT EXISTS departments (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL UNIQUE,
    description VARCHAR(500),
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- Create employees table
CREATE TABLE IF NOT EXISTS employees (
    id BIGSERIAL PRIMARY KEY,
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    email VARCHAR(255) NOT NULL UNIQUE,
    salary DECIMAL(10,2) NOT NULL,
    department_id BIGINT REFERENCES departments(id),
    status VARCHAR(20) DEFAULT 'ACTIVE',
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- Create indexes for better performance
CREATE INDEX IF NOT EXISTS idx_employee_email ON employees(email);
CREATE INDEX IF NOT EXISTS idx_employee_department ON employees(department_id);
CREATE INDEX IF NOT EXISTS idx_employee_status ON employees(status);
CREATE INDEX IF NOT EXISTS idx_employee_name ON employees(first_name, last_name);

-- Create update trigger for updated_at columns
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$ language 'plpgsql';

CREATE TRIGGER IF NOT EXISTS update_departments_updated_at 
    BEFORE UPDATE ON departments 
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER IF NOT EXISTS update_employees_updated_at 
    BEFORE UPDATE ON employees 
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

-- Insert sample departments
INSERT INTO departments (name, description) VALUES
('Engineering', 'Software development and engineering team'),
('Sales', 'Sales and business development team'),
('Human Resources', 'HR and talent management team'),
('Marketing', 'Marketing and communications team'),
('Finance', 'Financial operations and accounting team'),
('Operations', 'Operations and infrastructure team');

-- Insert sample employees
INSERT INTO employees (first_name, last_name, email, salary, department_id, status) VALUES
('John', 'Doe', 'john.doe@company.com', 85000.00, 1, 'ACTIVE'),
('Jane', 'Smith', 'jane.smith@company.com', 92000.00, 1, 'ACTIVE'),
('Bob', 'Johnson', 'bob.johnson@company.com', 78000.00, 2, 'ACTIVE'),
('Alice', 'Williams', 'alice.williams@company.com', 95000.00, 3, 'ACTIVE'),
('Charlie', 'Brown', 'charlie.brown@company.com', 72000.00, 4, 'ACTIVE'),
('Diana', 'Davis', 'diana.davis@company.com', 88000.00, 5, 'ACTIVE'),
('Eve', 'Miller', 'eve.miller@company.com', 91000.00, 1, 'ON_LEAVE'),
('Frank', 'Wilson', 'frank.wilson@company.com', 76000.00, 2, 'ACTIVE'),
('Grace', 'Taylor', 'grace.taylor@company.com', 89000.00, 6, 'ACTIVE'),
('Henry', 'Anderson', 'henry.anderson@company.com', 83000.00, 1, 'ACTIVE');

-- Grant permissions on all tables to appuser
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO appuser;
GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA public TO appuser;
```

### 3.2 PostgreSQL Deployment YAML

**File**: `postgresql-deployment.yaml`
```yaml
---
# PostgreSQL PersistentVolumeClaim
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgresql-pvc
  namespace: tracing-db
  labels:
    app: postgresql
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
  storageClassName: gp3-csi

---
# PostgreSQL Secret
apiVersion: v1
kind: Secret
metadata:
  name: postgresql-secret
  namespace: tracing-db
type: Opaque
data:
  username: YXBwdXNlcg==  # base64: appuser
  password: c2VjdXJlUGFzczEyMw==  # base64: securePass123
  postgres-password: YWRtaW5QYXNzMTIz  # base64: adminPass123

---
# PostgreSQL ConfigMap with initialization script
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgresql-init-script
  namespace: tracing-db
data:
  init-db.sql: |
    -- Create the database if it doesn't exist
    SELECT 'CREATE DATABASE employeedb'
    WHERE NOT EXISTS (SELECT FROM pg_database WHERE datname = 'employeedb')\gexec
    
    -- Connect to the database
    \c employeedb;
    
    -- Create user if not exists
    DO
    $do$
    BEGIN
       IF NOT EXISTS (
          SELECT FROM pg_catalog.pg_roles
          WHERE  rolname = 'appuser') THEN
          
          CREATE USER appuser WITH PASSWORD 'securePass123';
       END IF;
    END
    $do$;
    
    -- Grant permissions
    GRANT ALL PRIVILEGES ON DATABASE employeedb TO appuser;
    GRANT ALL ON SCHEMA public TO appuser;
    
    -- Create departments table
    CREATE TABLE IF NOT EXISTS departments (
        id BIGSERIAL PRIMARY KEY,
        name VARCHAR(100) NOT NULL UNIQUE,
        description VARCHAR(500),
        created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
        updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
    );
    
    -- Create employees table
    CREATE TABLE IF NOT EXISTS employees (
        id BIGSERIAL PRIMARY KEY,
        first_name VARCHAR(50) NOT NULL,
        last_name VARCHAR(50) NOT NULL,
        email VARCHAR(255) NOT NULL UNIQUE,
        salary DECIMAL(10,2) NOT NULL,
        department_id BIGINT REFERENCES departments(id),
        status VARCHAR(20) DEFAULT 'ACTIVE',
        created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
        updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
    );
    
    -- Create indexes
    CREATE INDEX IF NOT EXISTS idx_employee_email ON employees(email);
    CREATE INDEX IF NOT EXISTS idx_employee_department ON employees(department_id);
    CREATE INDEX IF NOT EXISTS idx_employee_status ON employees(status);
    CREATE INDEX IF NOT EXISTS idx_employee_name ON employees(first_name, last_name);
    
    -- Create update trigger function
    CREATE OR REPLACE FUNCTION update_updated_at_column()
    RETURNS TRIGGER AS $$
    BEGIN
        NEW.updated_at = CURRENT_TIMESTAMP;
        RETURN NEW;
    END;
    $$ language 'plpgsql';
    
    -- Create triggers
    DO $$
    BEGIN
        IF NOT EXISTS (SELECT 1 FROM pg_trigger WHERE tgname = 'update_departments_updated_at') THEN
            CREATE TRIGGER update_departments_updated_at 
                BEFORE UPDATE ON departments 
                FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
        END IF;
        
        IF NOT EXISTS (SELECT 1 FROM pg_trigger WHERE tgname = 'update_employees_updated_at') THEN
            CREATE TRIGGER update_employees_updated_at 
                BEFORE UPDATE ON employees 
                FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
        END IF;
    END $$;
    
    -- Insert sample data (only if tables are empty)
    INSERT INTO departments (name, description) 
    SELECT * FROM (VALUES
        ('Engineering', 'Software development and engineering team'),
        ('Sales', 'Sales and business development team'),
        ('Human Resources', 'HR and talent management team'),
        ('Marketing', 'Marketing and communications team'),
        ('Finance', 'Financial operations and accounting team'),
        ('Operations', 'Operations and infrastructure team')
    ) AS v(name, description)
    WHERE NOT EXISTS (SELECT 1 FROM departments WHERE departments.name = v.name);
    
    INSERT INTO employees (first_name, last_name, email, salary, department_id, status)
    SELECT * FROM (VALUES
        ('John', 'Doe', 'john.doe@company.com', 85000.00, 1, 'ACTIVE'),
        ('Jane', 'Smith', 'jane.smith@company.com', 92000.00, 1, 'ACTIVE'),
        ('Bob', 'Johnson', 'bob.johnson@company.com', 78000.00, 2, 'ACTIVE'),
        ('Alice', 'Williams', 'alice.williams@company.com', 95000.00, 3, 'ACTIVE'),
        ('Charlie', 'Brown', 'charlie.brown@company.com', 72000.00, 4, 'ACTIVE'),
        ('Diana', 'Davis', 'diana.davis@company.com', 88000.00, 5, 'ACTIVE'),
        ('Eve', 'Miller', 'eve.miller@company.com', 91000.00, 1, 'ON_LEAVE'),
        ('Frank', 'Wilson', 'frank.wilson@company.com', 76000.00, 2, 'ACTIVE'),
        ('Grace', 'Taylor', 'grace.taylor@company.com', 89000.00, 6, 'ACTIVE'),
        ('Henry', 'Anderson', 'henry.anderson@company.com', 83000.00, 1, 'ACTIVE')
    ) AS v(first_name, last_name, email, salary, department_id, status)
    WHERE NOT EXISTS (SELECT 1 FROM employees WHERE employees.email = v.email);
    
    -- Grant final permissions
    GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO appuser;
    GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA public TO appuser;

---
# PostgreSQL Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgresql
  namespace: tracing-db
  labels:
    app: postgresql
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgresql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: postgresql
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 999
        runAsGroup: 999
        fsGroup: 999
      containers:
      - name: postgresql
        image: registry.redhat.io/rhel8/postgresql-13:latest
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 5432
          name: postgresql
        env:
        - name: POSTGRESQL_USER
          valueFrom:
            secretKeyRef:
              name: postgresql-secret
              key: username
        - name: POSTGRESQL_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgresql-secret
              key: password
        - name: POSTGRESQL_ADMIN_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgresql-secret
              key: postgres-password
        - name: POSTGRESQL_DATABASE
          value: employeedb
        - name: POSTGRESQL_MAX_CONNECTIONS
          value: "100"
        - name: POSTGRESQL_SHARED_BUFFERS
          value: "128MB"
        resources:
          limits:
            memory: "2Gi"
            cpu: "1000m"
          requests:
            memory: "1Gi"
            cpu: "500m"
        livenessProbe:
          exec:
            command:
              - /usr/libexec/check-container
              - --live
          initialDelaySeconds: 120
          periodSeconds: 10
          timeoutSeconds: 10
          failureThreshold: 6
        readinessProbe:
          exec:
            command:
              - /usr/libexec/check-container
          initialDelaySeconds: 5
          periodSeconds: 10
          timeoutSeconds: 10
          failureThreshold: 3
        volumeMounts:
        - name: postgresql-data
          mountPath: /var/lib/pgsql/data
        - name: postgresql-init
          mountPath: /opt/app-root/src/postgresql-start/
        - name: dshm
          mountPath: /dev/shm
      volumes:
      - name: postgresql-data
        persistentVolumeClaim:
          claimName: postgresql-pvc
      - name: postgresql-init
        configMap:
          name: postgresql-init-script
      - name: dshm
        emptyDir:
          medium: Memory

---
# PostgreSQL Service
apiVersion: v1
kind: Service
metadata:
  name: postgresql-service
  namespace: tracing-db
  labels:
    app: postgresql
spec:
  type: ClusterIP
  ports:
  - port: 5432
    targetPort: 5432
    name: postgresql
  selector:
    app: postgresql
```

---

## 4. Docker Image Build and Registry Push

### 4.1 Build Application

```bash
# Navigate to project directory
cd employee-management

# Build the Spring Boot application
./mvnw clean package -DskipTests

# Verify the JAR was created
ls -la target/employee-management-*.jar
```

### 4.2 Build Docker Image

```bash
# Build the Docker image
docker build -t employee-management:1.0.0 .

# Tag for your registry (replace with your registry URL)
docker tag employee-management:1.0.0 your-registry.com/employee-management:1.0.0
docker tag employee-management:1.0.0 your-registry.com/employee-management:latest

# Verify images
docker images | grep employee-management
```

### 4.3 Push to Container Registry

```bash
# Login to your container registry
docker login your-registry.com

# Push the images
docker push your-registry.com/employee-management:1.0.0
docker push your-registry.com/employee-management:latest

# For OpenShift internal registry (alternative)
# First, expose the registry route if not already done
oc patch configs.imageregistry.operator.openshift.io/cluster --patch '{"spec":{"defaultRoute":true}}' --type=merge

# Get the registry route
REGISTRY_ROUTE=$(oc get route default-route -n openshift-image-registry -o jsonpath='{.spec.host}')

# Login to OpenShift registry
docker login -u $(oc whoami) -p $(oc whoami -t) $REGISTRY_ROUTE

# Tag and push to OpenShift registry
docker tag employee-management:1.0.0 $REGISTRY_ROUTE/tracing-db/employee-management:1.0.0
docker push $REGISTRY_ROUTE/tracing-db/employee-management:1.0.0
```

---

## 5. OpenTelemetry Tenant Configuration

### 5.1 Create New Tenant in TempoStack

**File**: `tempostack-update.yaml`
```yaml
apiVersion: tempo.grafana.com/v1alpha1
kind: TempoStack
metadata:
  name: tracing-tempo
  namespace: tracing-system
spec:
  storage:
    secret:
      name: tempo-storage-secret
      type: s3
    tls:
      enabled: true
      caName: minio-ca-bundle
  storageSize: 10Gi
  template:
    queryFrontend:
      jaegerQuery:
        enabled: true
    gateway:
      enabled: true
  tenants:
    mode: openshift
    authentication:
      - tenantName: tracing-demo
        tenantId: tracing-demo
      - tenantName: dev
        tenantId: dev
      - tenantName: prod
        tenantId: prod
      - tenantName: tracing-db
        tenantId: tracing-db
```

### 5.2 Deploy OpenTelemetry Collector for tracing-db Tenant

**File**: `otel-collector-tracing-db.yaml`
```yaml
---
# Create ServiceAccount for the collector
apiVersion: v1
kind: ServiceAccount
metadata:
  name: otel-collector-sa
  namespace: tracing-db

---
# Create ClusterRole for trace writing to tracing-db tenant
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: tempostack-traces-writer-tracing-db
rules:
- apiGroups:
  - 'tempo.grafana.com'
  resources:
  - tracing-db  # Must match tenant name in TempoStack
  resourceNames:
  - traces
  verbs:
  - 'create'

---
# Bind the role to ServiceAccount
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: tempostack-traces-writer-tracing-db
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: tempostack-traces-writer-tracing-db
subjects:
- kind: ServiceAccount
  name: otel-collector-sa
  namespace: tracing-db

---
# OpenTelemetry Collector Configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: otel-collector-config
  namespace: tracing-db
data:
  otel-collector-config.yaml: |
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318
            cors:
              allowed_origins:
                - "*"
      jaeger:
        protocols:
          grpc:
            endpoint: 0.0.0.0:14250
          thrift_binary:
            endpoint: 0.0.0.0:6832
          thrift_compact:
            endpoint: 0.0.0.0:6831
          thrift_http:
            endpoint: 0.0.0.0:14268
      zipkin:
        endpoint: 0.0.0.0:9411
    
    processors:
      batch:
        timeout: 10s
        send_batch_size: 1024
      memory_limiter:
        check_interval: 1s
        limit_percentage: 75
        spike_limit_percentage: 25
      resource:
        attributes:
        - key: service.namespace
          value: tracing-db
          action: upsert
        - key: deployment.environment
          value: production
          action: upsert
        - key: tenant
          value: tracing-db
          action: upsert
    
    exporters:
      otlp/tempo:
        endpoint: tempo-tracing-tempo-gateway.tracing-system.svc.cluster.local:8090
        tls:
          insecure: false
          ca_file: "/var/run/secrets/kubernetes.io/serviceaccount/service-ca.crt"
        auth:
          authenticator: bearertokenauth
        headers:
          X-Scope-OrgID: "tracing-db"
      
      debug:
        verbosity: detailed
        sampling_initial: 5
        sampling_thereafter: 200
    
    extensions:
      bearertokenauth:
        filename: "/var/run/secrets/kubernetes.io/serviceaccount/token"
      
      health_check:
        endpoint: 0.0.0.0:13133
    
    service:
      extensions: [bearertokenauth, health_check]
      pipelines:
        traces:
          receivers: [otlp, jaeger, zipkin]
          processors: [memory_limiter, resource, batch]
          exporters: [otlp/tempo, debug]

---
# OpenTelemetry Collector Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: otel-collector
  namespace: tracing-db
  labels:
    app: otel-collector
spec:
  replicas: 2
  selector:
    matchLabels:
      app: otel-collector
  template:
    metadata:
      labels:
        app: otel-collector
    spec:
      serviceAccountName: otel-collector-sa
      securityContext:
        runAsNonRoot: true
        runAsUser: 65534
      containers:
      - name: otel-collector
        image: otel/opentelemetry-collector-contrib:0.91.0
        imagePullPolicy: IfNotPresent
        command:
          - "/otelcol-contrib"
          - "--config=/etc/otelcol-contrib/otel-collector-config.yaml"
        ports:
        - containerPort: 4317
          name: otlp-grpc
          protocol: TCP
        - containerPort: 4318
          name: otlp-http
          protocol: TCP
        - containerPort: 14250
          name: jaeger-grpc
          protocol: TCP
        - containerPort: 14268
          name: jaeger-http
          protocol: TCP
        - containerPort: 9411
          name: zipkin
          protocol: TCP
        - containerPort: 13133
          name: health
          protocol: TCP
        env:
        - name: GOMEMLIMIT
          value: "160MiB"
        resources:
          limits:
            memory: 200Mi
            cpu: 100m
          requests:
            memory: 200Mi
            cpu: 100m
        livenessProbe:
          httpGet:
            path: /
            port: 13133
          initialDelaySeconds: 15
          periodSeconds: 20
        readinessProbe:
          httpGet:
            path: /
            port: 13133
          initialDelaySeconds: 5
          periodSeconds: 10
        volumeMounts:
        - name: otel-collector-config-vol
          mountPath: /etc/otelcol-contrib
      volumes:
      - name: otel-collector-config-vol
        configMap:
          name: otel-collector-config
          items:
          - key: otel-collector-config.yaml
            path: otel-collector-config.yaml

---
# OpenTelemetry Collector Service
apiVersion: v1
kind: Service
metadata:
  name: otel-collector
  namespace: tracing-db
  labels:
    app: otel-collector
spec:
  type: ClusterIP
  ports:
  - port: 4317
    targetPort: 4317
    name: otlp-grpc
    protocol: TCP
  - port: 4318
    targetPort: 4318
    name: otlp-http
    protocol: TCP
  - port: 14250
    targetPort: 14250
    name: jaeger-grpc
    protocol: TCP
  - port: 14268
    targetPort: 14268
    name: jaeger-http
    protocol: TCP
  - port: 9411
    targetPort: 9411
    name: zipkin
    protocol: TCP
  selector:
    app: otel-collector
```

---

## 6. Application Deployment to OpenShift

### 6.1 Create Namespace and Apply TempoStack Update

```bash
# Create the tracing-db namespace
oc new-project tracing-db

# Update TempoStack to include new tenant
oc apply -f tempostack-update.yaml

# Wait for TempoStack to be ready
oc wait --for=condition=Ready tempostack/tracing-tempo -n tracing-system --timeout=300s
```

### 6.2 Deploy PostgreSQL Database

```bash
# Deploy PostgreSQL
oc apply -f postgresql-deployment.yaml

# Wait for PostgreSQL to be ready
oc wait --for=condition=Ready pod -l app=postgresql -n tracing-db --timeout=300s

# Verify PostgreSQL is running
oc get pods -n tracing-db -l app=postgresql
oc logs -n tracing-db -l app=postgresql
```

### 6.3 Deploy OpenTelemetry Collector

```bash
# Deploy OTel Collector for tracing-db tenant
oc apply -f otel-collector-tracing-db.yaml

# Wait for collector to be ready
oc wait --for=condition=Ready pod -l app=otel-collector -n tracing-db --timeout=300s

# Verify collector is running
oc get pods -n tracing-db -l app=otel-collector
oc logs -n tracing-db -l app=otel-collector
```

### 6.4 Spring Boot Application Deployment

**File**: `employee-management-deployment.yaml`
```yaml
---
# Application ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: employee-app-config
  namespace: tracing-db
  labels:
    app: employee-management
data:
  application.properties: |
    # Database Configuration
    spring.datasource.url=jdbc:postgresql://postgresql-service:5432/employeedb
    spring.datasource.driver-class-name=org.postgresql.Driver
    spring.jpa.hibernate.ddl-auto=validate
    spring.jpa.show-sql=false
    spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.PostgreSQLDialect
    
    # OpenTelemetry Configuration
    otel.service.name=employee-management-service
    otel.exporter.otlp.endpoint=http://otel-collector:4318
    otel.instrumentation.jdbc.enabled=true
    otel.instrumentation.jdbc.statement-sanitizer.enabled=false
    otel.traces.sampler=parentbased_traceidratio
    otel.traces.sampler.arg=1.0
    otel.resource.attributes=tenant=tracing-db,service.namespace=tracing-db
    
    # Actuator Configuration
    management.endpoints.web.exposure.include=health,info,metrics,prometheus
    management.health.probes.enabled=true
    management.tracing.enabled=true
    management.tracing.sampling.probability=1.0

---
# Application Secret for Database
apiVersion: v1
kind: Secret
metadata:
  name: employee-app-secret
  namespace: tracing-db
type: Opaque
data:
  db-username: YXBwdXNlcg==  # base64: appuser
  db-password: c2VjdXJlUGFzczEyMw==  # base64: securePass123

---
# Employee Management Application Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: employee-management
  namespace: tracing-db
  labels:
    app: employee-management
    version: v1
spec:
  replicas: 2
  selector:
    matchLabels:
      app: employee-management
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  template:
    metadata:
      labels:
        app: employee-management
        version: v1
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/actuator/prometheus"
    spec:
      serviceAccountName: default
      securityContext:
        runAsNonRoot: true
        runAsUser: 1001
        runAsGroup: 1001
        fsGroup: 1001
      initContainers:
      - name: wait-for-db
        image: postgres:13-alpine
        command:
        - sh
        - -c
        - |
          echo "Waiting for PostgreSQL to be ready..."
          until pg_isready -h postgresql-service -p 5432 -U appuser; do
            echo "PostgreSQL is not ready yet. Waiting..."
            sleep 2
          done
          echo "PostgreSQL is ready!"
        env:
        - name: PGUSER
          valueFrom:
            secretKeyRef:
              name: employee-app-secret
              key: db-username
        - name: PGPASSWORD
          valueFrom:
            secretKeyRef:
              name: employee-app-secret
              key: db-password
      - name: wait-for-otel
        image: curlimages/curl:latest
        command:
        - sh
        - -c
        - |
          echo "Waiting for OpenTelemetry Collector to be ready..."
          until curl -s http://otel-collector:13133/; do
            echo "OTel Collector is not ready yet. Waiting..."
            sleep 2
          done
          echo "OTel Collector is ready!"
      containers:
      - name: employee-management
        image: your-registry.com/employee-management:1.0.0  # Replace with your actual registry
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
          name: http
          protocol: TCP
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "production"
        - name: DB_HOST
          value: "postgresql-service"
        - name: DB_PORT
          value: "5432"
        - name: DB_NAME
          value: "employeedb"
        - name: DB_USERNAME
          valueFrom:
            secretKeyRef:
              name: employee-app-secret
              key: db-username
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: employee-app-secret
              key: db-password
        - name: OTEL_ENDPOINT
          value: "http://otel-collector:4318"
        - name: ENVIRONMENT
          value: "production"
        - name: TENANT_NAME
          value: "tracing-db"
        - name: SAMPLING_RATE
          value: "1.0"
        - name: JAVA_OPTS
          value: "-XX:MaxRAMPercentage=70.0 -XX:+UseG1GC -XX:+UseContainerSupport -XX:+UseStringDeduplication"
        - name: OTEL_JAVAAGENT_ENABLED
          value: "true"
        - name: OTEL_INSTRUMENTATION_JDBC_STATEMENT_SANITIZER_ENABLED
          value: "false"
        - name: OTEL_SERVICE_NAME
          value: "employee-management-service"
        - name: OTEL_EXPORTER_OTLP_ENDPOINT
          value: "http://otel-collector:4318"
        - name: OTEL_RESOURCE_ATTRIBUTES
          value: "tenant=tracing-db,service.namespace=tracing-db,deployment.environment=production"
        resources:
          limits:
            memory: "1Gi"
            cpu: "1000m"
          requests:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8080
          initialDelaySeconds: 90
          periodSeconds: 30
          timeoutSeconds: 10
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        startupProbe:
          httpGet:
            path: /actuator/health
            port: 8080
          initialDelaySeconds: 20
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 30
        volumeMounts:
        - name: app-config
          mountPath: /config
          readOnly: true
        - name: tmp-volume
          mountPath: /tmp
      volumes:
      - name: app-config
        configMap:
          name: employee-app-config
      - name: tmp-volume
        emptyDir: {}

---
# Employee Management Service
apiVersion: v1
kind: Service
metadata:
  name: employee-management-service
  namespace: tracing-db
  labels:
    app: employee-management
spec:
  type: ClusterIP
  ports:
  - port: 8080
    targetPort: 8080
    name: http
    protocol: TCP
  selector:
    app: employee-management

---
# Employee Management Route for external access
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: employee-management-route
  namespace: tracing-db
  labels:
    app: employee-management
  annotations:
    haproxy.router.openshift.io/timeout: "300s"
    haproxy.router.openshift.io/balance: "roundrobin"
spec:
  to:
    kind: Service
    name: employee-management-service
  port:
    targetPort: http
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
  wildcardPolicy: None

---
# Horizontal Pod Autoscaler
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: employee-management-hpa
  namespace: tracing-db
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: employee-management
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
      - type: Pods
        value: 4
        periodSeconds: 15
      ```

### 6.5 Deploy the Spring Boot Application

```bash
# Deploy the Employee Management application
oc apply -f employee-management-deployment.yaml

# Wait for deployment to be ready
oc wait --for=condition=Available deployment/employee-management -n tracing-db --timeout=600s

# Verify all pods are running
oc get pods -n tracing-db

# Check application logs
oc logs -f deployment/employee-management -n tracing-db

# Get the application route
APP_ROUTE=$(oc get route employee-management-route -n tracing-db -o jsonpath='{.spec.host}')
echo "Application URL: https://$APP_ROUTE"
```

### 6.6 Verify All Components

```bash
# Check all resources in tracing-db namespace
oc get all -n tracing-db

# Check persistent volumes
oc get pvc -n tracing-db

# Check services
oc get svc -n tracing-db

# Check routes
oc get routes -n tracing-db

# Check pod status
oc get pods -n tracing-db -o wide
```

---

## 7. Testing and Verification

### 7.1 Health Check Verification

```bash
# Get the application route
APP_ROUTE=$(oc get route employee-management-route -n tracing-db -o jsonpath='{.spec.host}')

# Test application health
curl -k "https://$APP_ROUTE/actuator/health"

# Test database connectivity
curl -k "https://$APP_ROUTE/actuator/health/db"

# Test readiness
curl -k "https://$APP_ROUTE/actuator/health/readiness"

# Test liveness
curl -k "https://$APP_ROUTE/actuator/health/liveness"
```

### 7.2 Database Operations Testing

```bash
# Test GET all employees
curl -k "https://$APP_ROUTE/api/v1/employees"

# Test GET employee by ID
curl -k "https://$APP_ROUTE/api/v1/employees/1"

# Create a new employee (this will generate INSERT traces)
curl -k -X POST "https://$APP_ROUTE/api/v1/employees" \
  -H "Content-Type: application/json" \
  -d '{
    "firstName": "Test",
    "lastName": "Employee",
    "email": "test.employee@company.com",
    "salary": 75000,
    "departmentId": 1
  }'

# Search employees (this will generate LIKE queries)
curl -k "https://$APP_ROUTE/api/v1/employees/search?query=John"

# Update an employee (this will generate UPDATE traces)
curl -k -X PUT "https://$APP_ROUTE/api/v1/employees/1" \
  -H "Content-Type: application/json" \
  -d '{
    "salary": 90000,
    "status": "ACTIVE"
  }'

# Get paginated results
curl -k "https://$APP_ROUTE/api/v1/employees?page=0&size=5&sort=firstName,asc"
```

### 7.3 Load Testing for Tracing

```bash
# Generate multiple requests to create diverse traces
for i in {1..50}; do
  # Create employees
  curl -k -X POST "https://$APP_ROUTE/api/v1/employees" \
    -H "Content-Type: application/json" \
    -d "{
      \"firstName\": \"User$i\",
      \"lastName\": \"Test$i\",
      \"email\": \"user$i@loadtest.com\",
      \"salary\": $((70000 + RANDOM % 30000)),
      \"departmentId\": $((1 + RANDOM % 6))
    }" &
  
  # Search operations
  curl -k "https://$APP_ROUTE/api/v1/employees/search?query=User" &
  
  # Get operations
  curl -k "https://$APP_ROUTE/api/v1/employees/$((1 + RANDOM % 10))" &
  
  # Pagination
  curl -k "https://$APP_ROUTE/api/v1/employees?page=$((RANDOM % 3))&size=10" &
  
  if [ $((i % 10)) -eq 0 ]; then
    echo "Generated $i requests..."
    sleep 2
  fi
done

wait
echo "Load testing completed!"
```

### 7.4 OpenTelemetry Tracing Verification

#### Check OpenTelemetry Collector Status

```bash
# Check OTel Collector logs
oc logs -f deployment/otel-collector -n tracing-db

# Check if traces are being received
oc logs deployment/otel-collector -n tracing-db | grep -i "traces received"

# Check collector health
COLLECTOR_POD=$(oc get pod -l app=otel-collector -n tracing-db -o jsonpath='{.items[0].metadata.name}')
oc exec $COLLECTOR_POD -n tracing-db -- curl http://localhost:13133/
```

#### Verify Traces in Jaeger UI

```bash
# Get Jaeger UI route (assuming you have it deployed from the previous setup)
JAEGER_ROUTE=$(oc get route jaeger-ui -n tracing-system -o jsonpath='{.spec.host}' 2>/dev/null)

if [ -n "$JAEGER_ROUTE" ]; then
  echo "Jaeger UI available at: http://$JAEGER_ROUTE"
  echo "Look for service: employee-management-service"
  echo "Tenant: tracing-db"
else
  echo "Jaeger UI route not found. Check if Jaeger is deployed."
fi

# Alternative: Check traces via Tempo Gateway
TEMPO_GATEWAY_ROUTE=$(oc get route tempo-gateway -n tracing-system -o jsonpath='{.spec.host}' 2>/dev/null)

if [ -n "$TEMPO_GATEWAY_ROUTE" ]; then
  echo "Tempo Gateway available at: https://$TEMPO_GATEWAY_ROUTE"
  echo "Tenant-specific URL: https://$TEMPO_GATEWAY_ROUTE/api/traces/v1/tracing-db"
fi
```

#### Access OpenShift Console Distributed Tracing

1. Log into OpenShift Console
2. Navigate to **Observe → Traces**
3. Select namespace: `tracing-db`
4. Look for traces from `employee-management-service`

### 7.5 JDBC Tracing Analysis

When viewing traces in the distributed tracing UI, you should see:

#### Trace Structure for Employee Creation:
```
POST /api/v1/employees (total: ~500ms)
├── EmployeeController.createEmployee (480ms)
│   └── EmployeeServiceImpl.createEmployee (450ms)
│       ├── EmployeeRepository.findByEmail (50ms)
│       │   └── SELECT employees WHERE email = ? (45ms)
│       ├── DepartmentRepository.findById (30ms)
│       │   └── SELECT departments WHERE id = ? (25ms)
│       └── EmployeeRepository.save (300ms)
│           ├── INSERT INTO employees (...) VALUES (...) (280ms)
│           └── Connection.commit (15ms)
```

#### Expected JDBC Span Attributes:
- **db.system**: postgresql
- **db.connection_string**: jdbc:postgresql://postgresql-service:5432/employeedb
- **db.statement**: The actual SQL query (e.g., "SELECT * FROM employees WHERE email = ?")
- **db.operation**: SELECT, INSERT, UPDATE, DELETE
- **db.sql.table**: employees, departments
- **thread.name**: The thread executing the query

#### Key SQL Operations to Verify:
1. **Employee Email Check**: `SELECT * FROM employees WHERE email = ?`
2. **Department Lookup**: `SELECT * FROM departments WHERE id = ?`
3. **Employee Creation**: `INSERT INTO employees (first_name, last_name, email, salary, department_id, status, created_at, updated_at) VALUES (?, ?, ?, ?, ?, ?, ?, ?)`
4. **Employee Search**: `SELECT * FROM employees WHERE LOWER(first_name) LIKE LOWER(CONCAT('%', ?, '%')) OR LOWER(last_name) LIKE LOWER(CONCAT('%', ?, '%'))`
5. **Paginated Queries**: `SELECT * FROM employees ORDER BY first_name ASC LIMIT ? OFFSET ?`

### 7.6 Troubleshooting Common Issues

#### Application Not Starting
```bash
# Check pod events
oc describe pod -l app=employee-management -n tracing-db

# Check application logs
oc logs deployment/employee-management -n tracing-db --previous

# Check init container logs
oc logs deployment/employee-management -n tracing-db -c wait-for-db
oc logs deployment/employee-management -n tracing-db -c wait-for-otel
```

#### Database Connection Issues
```bash
# Test database connectivity from a debug pod
oc run debug-db --image=postgres:13-alpine --rm -it --restart=Never -n tracing-db -- psql -h postgresql-service -U appuser -d employeedb

# Check PostgreSQL logs
oc logs deployment/postgresql -n tracing-db

# Verify database initialization
oc exec deployment/postgresql -n tracing-db -- psql -U appuser -d employeedb -c "\dt"
```

#### OpenTelemetry Issues
```bash
# Check OTel Collector configuration
oc get configmap otel-collector-config -n tracing-db -o yaml

# Test OTel Collector connectivity
oc port-forward svc/otel-collector -n tracing-db 4318:4318 &
curl http://localhost:4318/v1/traces

# Check Tempo Gateway connectivity
oc exec deployment/otel-collector -n tracing-db -- curl -v tempo-tracing-tempo-gateway.tracing-system.svc.cluster.local:8090

# Verify tenant permissions
oc auth can-i create traces --as=system:serviceaccount:tracing-db:otel-collector-sa --subresource=tempo.grafana.com/tracing-db
```

#### No Traces Appearing
```bash
# Enable debug logging in application
oc set env deployment/employee-management -n tracing-db LOGGING_LEVEL_IO_OPENTELEMETRY=DEBUG

# Check if spans are being created
oc logs deployment/employee-management -n tracing-db | grep -i "span\|trace"

# Verify OpenTelemetry Java agent is loaded
oc logs deployment/employee-management -n tracing-db | grep -i "opentelemetry"

# Check OTel Collector debug output
oc logs deployment/otel-collector -n tracing-db | grep -i "ResourceSpans"
```

### 7.7 Performance Monitoring

```bash
# Check application metrics
curl -k "https://$APP_ROUTE/actuator/metrics"

# Check specific database metrics
curl -k "https://$APP_ROUTE/actuator/metrics/hikaricp.connections.active"
curl -k "https://$APP_ROUTE/actuator/metrics/jdbc.connections.active"

# Check JVM metrics
curl -k "https://$APP_ROUTE/actuator/metrics/jvm.memory.used"

# Check HTTP request metrics
curl -k "https://$APP_ROUTE/actuator/metrics/http.server.requests"
```

### 7.8 Scaling and Load Testing

```bash
# Scale the application
oc scale deployment employee-management --replicas=5 -n tracing-db

# Watch HPA status
oc get hpa -n tracing-db -w

# Generate sustained load
for i in {1..1000}; do
  curl -k "https://$APP_ROUTE/api/v1/employees" &
  curl -k "https://$APP_ROUTE/api/v1/employees/search?query=test" &
  
  if [ $((i % 50)) -eq 0 ]; then
    echo "Executed $i requests..."
    sleep 1
  fi
done
```

---

## Summary

This complete guide covers:

1. **Project Creation**: Step-by-step Spring Boot 3-tier application creation from Spring Initializr
2. **Application Development**: Complete implementation with proper layered architecture
3. **Database Setup**: PostgreSQL deployment with persistent storage and sample data
4. **Container Images**: Docker build and registry push procedures
5. **OpenTelemetry Configuration**: Multi-tenant setup with dedicated collector for `tracing-db` tenant
6. **OpenShift Deployment**: Complete deployment manifests with health checks and scaling
7. **Testing and Verification**: Comprehensive testing procedures including JDBC tracing verification

The application demonstrates:
- **Presentation Layer**: REST controllers with OpenTelemetry HTTP instrumentation
- **Business Layer**: Service classes with custom tracing spans
- **Data Layer**: JPA repositories with automatic JDBC tracing
- **Cross-cutting Concerns**: Exception handling, validation, logging with trace correlation

All JDBC operations (SELECT, INSERT, UPDATE, DELETE) are automatically traced and visible in the distributed tracing UI with detailed SQL statement information and database performance metrics.
