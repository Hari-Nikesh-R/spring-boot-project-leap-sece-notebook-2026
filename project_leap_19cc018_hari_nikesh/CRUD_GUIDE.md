# Spring Boot + MySQL (Docker) CRUD Guide

This guide walks you through connecting your Spring Boot project to a MySQL
database running in Docker, and building a simple CRUD (Create, Read, Update,
Delete) feature with Spring Data JPA.

## Prerequisites

- Docker Desktop installed and running
- This project opened in your IDE
- Java 25 / Maven (use the included `./mvnw` wrapper, no local Maven needed)

## 1. Start MySQL with Docker Compose

A `docker-compose.yml` is already in the project root. It starts:

- **mysql** — MySQL 8.4, database `leap_db`, user `leap_user` / password `leap_pass`, exposed on port `3306`
- **adminer** — a web UI to browse the database, at `http://localhost:8081`

Start it:

```bash
docker compose up -d
```

Check it's healthy:

```bash
docker compose ps
```

Stop it later with `docker compose down` (add `-v` to also wipe the data volume).

Log in to Adminer at `http://localhost:8081` with:
- System: MySQL
- Server: `mysql`
- Username: `leap_user`
- Password: `leap_pass`
- Database: `leap_db`

## 2. Project dependencies

`pom.xml` already includes:

- `spring-boot-starter-data-jpa` — Spring Data JPA + Hibernate
- `mysql-connector-j` — MySQL JDBC driver

## 3. Datasource configuration

`src/main/resources/application.properties` already points to the Dockerized DB:

```properties
spring.datasource.url=jdbc:mysql://localhost:3306/leap_db
spring.datasource.username=leap_user
spring.datasource.password=leap_pass
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver

spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
```

`ddl-auto=update` lets Hibernate auto-create/update tables from your `@Entity`
classes — convenient for learning, but in real projects you'd normally use
migrations (Flyway/Liquibase) instead.

## 4. Create an Entity

`src/main/java/.../student/Student.java`:

```java
package com.example.project_leap_19cc018_hari_nikesh.student;

import jakarta.persistence.*;

@Entity
@Table(name = "students")
public class Student {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;
    private String email;

    public Student() {}

    public Student(String name, String email) {
        this.name = name;
        this.email = email;
    }

    public Long getId() { return id; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }
}
```

## 5. Create a Repository

`src/main/java/.../student/StudentRepository.java`:

```java
package com.example.project_leap_19cc018_hari_nikesh.student;

import org.springframework.data.jpa.repository.JpaRepository;

public interface StudentRepository extends JpaRepository<Student, Long> {
}
```

`JpaRepository` already gives you `save`, `findAll`, `findById`, `deleteById`, etc.

## 6. Create a REST Controller

`src/main/java/.../student/StudentController.java`:

```java
package com.example.project_leap_19cc018_hari_nikesh.student;

import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/students")
public class StudentController {

    private final StudentRepository repository;

    public StudentController(StudentRepository repository) {
        this.repository = repository;
    }

    @GetMapping
    public List<Student> getAll() {
        return repository.findAll();
    }

    @GetMapping("/{id}")
    public ResponseEntity<Student> getOne(@PathVariable Long id) {
        return repository.findById(id)
                .map(ResponseEntity::ok)
                .orElse(ResponseEntity.notFound().build());
    }

    @PostMapping
    public Student create(@RequestBody Student student) {
        return repository.save(student);
    }

    @PutMapping("/{id}")
    public ResponseEntity<Student> update(@PathVariable Long id, @RequestBody Student updated) {
        return repository.findById(id)
                .map(existing -> {
                    existing.setName(updated.getName());
                    existing.setEmail(updated.getEmail());
                    return ResponseEntity.ok(repository.save(existing));
                })
                .orElse(ResponseEntity.notFound().build());
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> delete(@PathVariable Long id) {
        if (!repository.existsById(id)) {
            return ResponseEntity.notFound().build();
        }
        repository.deleteById(id);
        return ResponseEntity.noContent().build();
    }
}
```

## 7. Run the app

With MySQL still running in Docker:

```bash
./mvnw spring-boot:run
```

Hibernate will create the `students` table in `leap_db` on startup
(you can confirm this in Adminer).

## 8. Test the CRUD endpoints

Create:

```bash
curl -X POST http://localhost:8080/api/students \
  -H "Content-Type: application/json" \
  -d '{"name":"Ada Lovelace","email":"ada@example.com"}'
```

Read all:

```bash
curl http://localhost:8080/api/students
```

Read one:

```bash
curl http://localhost:8080/api/students/1
```

Update:

```bash
curl -X PUT http://localhost:8080/api/students/1 \
  -H "Content-Type: application/json" \
  -d '{"name":"Ada L.","email":"ada.l@example.com"}'
```

Delete:

```bash
curl -X DELETE http://localhost:8080/api/students/1
```

## Troubleshooting

- **`Communications link failure` / connection refused** — MySQL isn't up yet
  or isn't healthy. Run `docker compose ps` and `docker compose logs mysql`.
- **Port 3306 already in use** — you have another MySQL running locally; stop
  it, or change the host port in `docker-compose.yml` (e.g. `"3307:3306"`) and
  update `spring.datasource.url` to match.
- **Table not created** — check `spring.jpa.hibernate.ddl-auto=update` is set
  and that your entity has `@Entity` + a table name that doesn't collide with
  a reserved word.
