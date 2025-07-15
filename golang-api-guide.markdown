# Passo a Passo para Criar uma API em Golang com Echo e Docker Compose

Este guia descreve como criar uma API RESTful em Go usando o framework Echo, com integração de um banco de dados PostgreSQL via Docker Compose. A estrutura é modular, escalável e segue boas práticas de clean code, permitindo reutilização em qualquer projeto.

---

## Passo 1: Configurar o Ambiente
1. **Instalar o Go**: Certifique-se de que o Go (versão 1.21 ou superior) está instalado:
   ```bash
   go version
   ```
2. **Instalar o Docker e Docker Compose**: Verifique a instalação com:
   ```bash
   docker --version
   docker-compose --version
   ```
3. **Instalar Dependências do Go**:
   - Echo: Framework para a API.
   - GORM: ORM para interação com o banco de dados.
   - godotenv: Para variáveis de ambiente.
   Execute:
   ```bash
   go get github.com/labstack/echo/v4
   go get gorm.io/gorm
   go get gorm.io/driver/postgres
   go get github.com/joho/godotenv
   ```
4. **Inicializar o Projeto**:
   ```bash
   mkdir my-api
   cd my-api
   go mod init my-api
   ```

---

## Passo 2: Estruturar o Projeto
A estrutura do projeto é modular e inclui arquivos para Docker:

```
my-api/
├── cmd/
│   └── api/
│       └── main.go
├── internal/
│   ├── config/
│   │   └── config.go
│   ├── handler/
│   │   └── user.go
│   ├── model/
│   │   └── user.go
│   ├── repository/
│   │   └── user_repository.go
│   ├── service/
│   │   └── user_service.go
│   └── server/
│       └── server.go
├── .env
├── Dockerfile
├── docker-compose.yml
├── go.mod
└── go.sum
```

---

## Passo 3: Configurar o Docker Compose (`docker-compose.yml`)
Crie o arquivo `docker-compose.yml` na raiz do projeto para configurar a API e o PostgreSQL.

```yaml
version: '3.8'

services:
  api:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8080:8080"
    environment:
      - PORT=8080
      - DB_HOST=db
      - DB_PORT=5432
      - DB_USER=postgres
      - DB_PASSWORD=postgres
      - DB_NAME=mydb
    depends_on:
      - db
    networks:
      - app-network

  db:
    image: postgres:15
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=postgres
      - POSTGRES_DB=mydb
    volumes:
      - db-data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    networks:
      - app-network

networks:
  app-network:
    driver: bridge

volumes:
  db-data:
```

---

## Passo 4: Configurar o Dockerfile (`Dockerfile`)
Crie o arquivo `Dockerfile` na raiz do projeto para construir a imagem da API.

```dockerfile
FROM golang:1.21-alpine

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download

COPY . .

RUN go build -o main ./cmd/api

EXPOSE 8080

CMD ["./main"]
```

---

## Passo 5: Configurar Variáveis de Ambiente (`config/config.go`)
Crie o arquivo `internal/config/config.go` para carregar configurações, incluindo as do banco de dados.

```go
package config

import (
	"fmt"
	"os"

	"github.com/joho/godotenv"
)

// Config contém as configurações da aplicação.
type Config struct {
	Port     string
	DBHost   string
	DBPort   string
	DBUser   string
	DBPass   string
	DBName   string
}

// Load carrega as configurações a partir de variáveis de ambiente.
func Load() (*Config, error) {
	if err := godotenv.Load(); err != nil {
		fmt.Println("No .env file found, using system environment variables")
	}

	cfg := &Config{
		Port:   os.Getenv("PORT"),
		DBHost: os.Getenv("DB_HOST"),
		DBPort: os.Getenv("DB_PORT"),
		DBUser: os.Getenv("DB_USER"),
		DBPass: os.Getenv("DB_PASSWORD"),
		DBName: os.Getenv("DB_NAME"),
	}

	if cfg.Port == "" {
		cfg.Port = "8080"
	}
	if cfg.DBHost == "" || cfg.DBPort == "" || cfg.DBUser == "" || cfg.DBPass == "" || cfg.DBName == "" {
		return nil, fmt.Errorf("missing required database environment variables")
	}

	return cfg, nil
}

// DSN retorna a string de conexão para o PostgreSQL.
func (c *Config) DSN() string {
	return fmt.Sprintf("host=%s port=%s user=%s password=%s dbname=%s sslmode=disable",
		c.DBHost, c.DBPort, c.DBUser, c.DBPass, c.DBName)
}
```

Crie um arquivo `.env` na raiz do projeto:
```
PORT=8080
DB_HOST=db
DB_PORT=5432
DB_USER=postgres
DB_PASSWORD=postgres
DB_NAME=mydb
```

---

## Passo 6: Definir o Modelo (`model/user.go`)
Crie o modelo com anotações para o GORM e JSON.

```go
package model

import "gorm.io/gorm"

// User representa um usuário na aplicação.
type User struct {
	gorm.Model
	ID    int    `json:"id" gorm:"primaryKey"`
	Name  string `json:"name" gorm:"type:varchar(100)"`
	Email string `json:"email" gorm:"type:varchar(100);unique"`
}
```

---

## Passo 7: Configurar o Repositório (`repository/user_repository.go`)
Crie o repositório para interagir com o PostgreSQL usando GORM.

```go
package repository

import (
	"my-api/internal/model"

	"gorm.io/gorm"
)

// UserRepository define a interface para operações com usuários.
type UserRepository interface {
	Create(user *model.User) error
	FindByID(id int) (*model.User, error)
	FindAll() ([]*model.User, error)
}

// userRepository é a implementação do repositório com GORM.
type userRepository struct {
	db *gorm.DB
}

// NewUserRepository cria uma nova instância do repositório.
func NewUserRepository(db *gorm.DB) UserRepository {
	return &userRepository{db: db}
}

// Create adiciona um novo usuário.
func (r *userRepository) Create(user *model.User) error {
	return r.db.Create(user).Error
}

// FindByID busca um usuário por ID.
func (r *userRepository) FindByID(id int) (*model.User, error) {
	var user model.User
	if err := r.db.First(&user, id).Error; err != nil {
		return nil, err
	}
	return &user, nil
}

// FindAll retorna todos os usuários.
func (r *userRepository) FindAll() ([]*model.User, error) {
	var users []*model.User
	if err := r.db.Find(&users).Error; err != nil {
		return nil, err
	}
	return users, nil
}
```

---

## Passo 8: Configurar o Serviço (`service/user_service.go`)
O serviço contém a lógica de negócios.

```go
package service

import (
	"my-api/internal/model"
	"my-api/internal/repository"
)

// UserService define a interface para operações de negócio com usuários.
type UserService interface {
	CreateUser(user *model.User) error
	GetUserByID(id int) (*model.User, error)
	GetAllUsers() ([]*model.User, error)
}

// userService é a implementação do serviço.
type userService struct {
	repo repository.UserRepository
}

// NewUserService cria uma nova instância do serviço.
func NewUserService(repo repository.UserRepository) UserService {
	return &userService{repo: repo}
}

// CreateUser cria um novo usuário.
func (s *userService) CreateUser(user *model.User) error {
	return s.repo.Create(user)
}

// GetUserByID retorna um usuário por ID.
func (s *userService) GetUserByID(id int) (*model.User, error) {
	return s.repo.FindByID(id)
}

// GetAllUsers retorna todos os usuários.
func (s *userService) GetAllUsers() ([]*model.User, error) {
	return s.repo.FindAll()
}
```

---

## Passo 9: Configurar os Handlers (`handler/user.go`)
Os handlers lidam com as requisições HTTP.

```go
package handler

import (
	"net/http"
	"strconv"

	"my-api/internal/model"
	"my-api/internal/service"

	"github.com/labstack/echo/v4"
)

// UserHandler gerencia as rotas relacionadas a usuários.
type UserHandler struct {
	service service.UserService
}

// NewUserHandler cria uma nova instância do handler.
func NewUserHandler(service service.UserService) *UserHandler {
	return &UserHandler{service: service}
}

// CreateUser cria um novo usuário.
func (h *UserHandler) CreateUser(c echo.Context) error {
	var user model.User
	if err := c.Bind(&user); err != nil {
		return c.JSON(http.StatusBadRequest, map[string]string{"error": "invalid request"})
	}
	if err := h.service.CreateUser(&user); err != nil {
		return c.JSON(http.StatusInternalServerError, map[string]string{"error": err.Error()})
	}
	return c.JSON(http.StatusCreated, user)
}

// GetUserByID retorna um usuário por ID.
func (h *UserHandler) GetUserByID(c echo.Context) error {
	id, err := strconv.Atoi(c.Param("id"))
	if err != nil {
		return c.JSON(http.StatusBadRequest, map[string]string{"error": "invalid ID"})
	}
	user, err := h.service.GetUserByID(id)
	if err != nil {
		return c.JSON(http.StatusNotFound, map[string]string{"error": err.Error()})
	}
	return c.JSON(http.StatusOK, user)
}

// GetAllUsers retorna todos os usuários.
func (h *UserHandler) GetAllUsers(c echo.Context) error {
	users, err := h.service.GetAllUsers()
	if err != nil {
		return c.JSON(http.StatusInternalServerError, map[string]string{"error": err.Error()})
	}
	return c.JSON(http.StatusOK, users)
}
```

---

## Passo 10: Configurar o Servidor (`server/server.go`)
Crie o arquivo para configurar o servidor Echo e conectar ao banco de dados.

```go
package server

import (
	"fmt"
	"my-api/internal/config"
	"my-api/internal/handler"
	"my-api/internal/repository"
	"my-api/internal/service"

	"github.com/labstack/echo/v4"
	"github.com/labstack/echo/v4/middleware"
	"gorm.io/driver/postgres"
	"gorm.io/gorm"
)

// Server encapsula o servidor Echo.
type Server struct {
	*echo.Echo
	config *config.Config
}

// NewServer cria uma nova instância do servidor.
func NewServer(cfg *config.Config) *Server {
	e := echo.New()

	// Configurar middlewares
	e.Use(middleware.Logger())
	e.Use(middleware.Recover())

	// Conectar ao banco de dados
	db, err := gorm.Open(postgres.Open(cfg.DSN()), &gorm.Config{})
	if err != nil {
		panic(fmt.Sprintf("failed to connect to database: %v", err))
	}

	// Inicializar dependências
	userRepo := repository.NewUserRepository(db)
	userService := service.NewUserService(userRepo)
	userHandler := handler.NewUserHandler(userService)

	// Registrar rotas
	e.POST("/users", userHandler.CreateUser)
	e.GET("/users/:id", userHandler.GetUserByID)
	e.GET("/users", userHandler.GetAllUsers)

	return &Server{
		Echo:   e,
		config: cfg,
	}
}

// Start inicia o servidor.
func (s *Server) Start() error {
	addr := fmt.Sprintf(":%s", s.config.Port)
	return s.Echo.Start(addr)
}
```

---

## Passo 11: Configurar o Ponto de Entrada (`main.go`)
Crie o arquivo `cmd/api/main.go`.

```go
package main

import (
	"log"
	"my-api/internal/config"
	"my-api/internal/server"
)

func main() {
	// Carregar configurações
	cfg, err := config.Load()
	if err != nil {
		log.Fatalf("failed to load config: %v", err)
	}

	// Inicializar o servidor
	srv := server.NewServer(cfg)
	if err := srv.Start(); err != nil {
		log.Fatalf("failed to start server: %v", err)
	}
}
```

---

## Passo 12: Testar a API
1. **Iniciar os Containers**:
   ```bash
   docker-compose up --build
   ```
   Isso constrói e inicia a API (em `http://localhost:8080`) e o PostgreSQL.

2. **Testar as Rotas**:
   - **Criar usuário**:
     ```bash
     curl -X POST http://localhost:8080/users -H "Content-Type: application/json" -d '{"id":1,"name":"John Doe","email":"john@example.com"}'
     ```
   - **Buscar usuário por ID**:
     ```bash
     curl http://localhost:8080/users/1
     ```
   - **Listar todos os usuários**:
     ```bash
     curl http://localhost:8080/users
     ```

3. **Parar os Containers**:
   ```bash
   docker-compose down
   ```

---

## Passo 13: Adicionar Boas Práticas
- **Validação de Entrada**: Use o middleware de validação do Echo ou `github.com/go-playground/validator/v10`.
- **Logging**: Configure logs estruturados com `logrus` ou `zap`.
- **Documentação**: Gere documentação da API com Swagger/OpenAPI.
- **Testes**: Escreva testes unitários para handlers, serviços e repositórios usando o pacote `testing`.
- **Migrações**: Use `golang-migrate` para gerenciar esquemas de banco de dados.
- **Segurança**: Adicione autenticação com JWT ou OAuth.

---

## Passo 14: Reutilizar o Template
Para usar este template em novos projetos:
1. Copie a estrutura de diretórios e arquivos.
2. Atualize o nome do módulo em `go.mod` (ex.: `my-api` para `new-project`).
3. Ajuste o `.env` e `docker-compose.yml` conforme necessário (ex.: nome do banco, portas).
4. Execute `docker-compose up` para iniciar a API e o banco de dados.

---

## Passo 15: Próximos Passos
- **Autenticação**: Implemente JWT ou OAuth.
- **CI/CD**: Configure pipelines com GitHub Actions.
- **Monitoramento**: Integre Prometheus e Grafana.
- **Escalabilidade**: Considere adicionar um balanceador de carga ou orquestração com Kubernetes.