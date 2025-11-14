# Spring Commerce: Building a Resilient E-commerce System with Spring Boot Microservices

This repository contains code for a microservices-based e-commerce application developed using Spring Boot. This project demonstrates how to build, test, and deploy microservices using Spring Cloud and Docker.

## Architecture Overview

The application is composed of multiple microservices:

- **Product Service**: Manages product information (MongoDB)
- **Order Service**: Handles customer orders (MySQL)
- **Inventory Service**: Tracks product inventory (MySQL)

![Architecture Overview](https://raw.githubusercontent.com/rahult18/springcommerce/main/project_architecture.png)

## Tech Stack

- **Java 21**
- **Spring Boot 3.x**
- **Spring Data MongoDB** (Product Service)
- **Spring Data JPA** (Order & Inventory Services)
- **MySQL** (Order & Inventory Services)
- **MongoDB** (Product Service)
- **Flyway** (Database migrations)
- **Lombok** (Reduce boilerplate code)
- **Docker & Docker Compose** (Containerization)
- **TestContainers** (Integration testing)

