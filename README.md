# zmall
项目目录结构
```go
zmall/
├── api/
│   └── http/
│       ├── handlers/
│       └── routes.go
├── config/
│   └── config.go
├── internal/
│   ├── models/
│   ├── services/
│   ├── repositories/
│   └── middleware/
├── pkg/
│   ├── utils/
│   └── logger/
├── scripts/
├── sql/
├── main.go
├── go.mod
└── go.sum
```

各目录说明
* api：存放与外部交互的接口相关代码，如 HTTP API。
        http：HTTP 接口相关代码。
        handlers：处理 HTTP 请求的代码。
        routes.go：定义 HTTP 路由。
* config：存放项目配置相关代码。
* internal：存放项目内部核心代码。
    models：存放数据模型定义。
    services：存放业务逻辑服务代码。
    repositories：存放数据访问层代码。
    middleware：存放中间件代码。
* pkg：存放可复用的工具包。
    utils：通用工具函数。
    logger：日志记录相关代码。
* scripts：存放项目脚本，如部署脚本等。
    sql：存放数据库 SQL 脚本。 
* main.go：项目入口文件。
* go.mod 和 go.sum：Go 模块管理文件。

# 分步实现方案
## 步骤 1：项目初始化与基础配置
目标：初始化项目，创建基础目录结构，配置项目依赖和基础配置。
操作：
创建项目目录 zmall，并初始化 Go 模块。

bash
mkdir zmall
cd zmall
go mod init github.com/yourusername/zmall
创建上述目录结构。
在 config/config.go 中定义基础配置结构。
```go

package config

type Config struct {
    HTTPPort     string
    DBHost       string
    DBPort       string
    DBUser       string
    DBPassword   string
    DBName       string
}

func LoadConfig() *Config {
return &Config{
    HTTPPort:     "8080",
    DBHost:       "localhost",
    DBPort:       "3306",
    DBUser:       "user",
    DBPassword:   "password",
    DBName:       "zmall",
    }
}

```

## 步骤 2：数据库连接与模型定义
目标：连接 MySQL 数据库，定义商品、用户、订单等核心数据模型。
操作：
在 go.mod 中添加 GORM 和 MySQL 驱动依赖。

```bash
go get -u gorm.io/gorm
go get -u gorm.io/driver/mysql
```

在 internal/models 下定义数据模型。

product.go
```go
package models

import "gorm.io/gorm"

type Product struct {
gorm.Model
Name  string  `gorm:"not null" json:"name"`
Price float64 `gorm:"not null" json:"price"`
}
````
在 internal/repositories 中实现数据库操作。
product_repo.go
```go
package repositories

import (
"gorm.io/gorm"
"zmall/internal/models"
)

type ProductRepository struct {
    db *gorm.DB
}

func NewProductRepository(db *gorm.DB) *ProductRepository {
    return &ProductRepository{db: db}
}

func (r *ProductRepository) Create(product *models.Product) error {
    return r.db.Create(product).Error
}

func (r *ProductRepository) GetAll() ([]models.Product, error) {
    var products []models.Product
    err := r.db.Find(&products).Error
    return products, err
}
```

## 步骤 3：HTTP 服务搭建与路由配置
目标：使用 Hertz 框架搭建 HTTP 服务，配置商品相关路由。
操作：
添加 Hertz 依赖。

```bash
go get github.com/cloudwego/hertz/pkg/app/server
```

在 api/http/routes.go 中定义路由。

routes.go
```go
package http

import (
"context"
"zmall/api/http/handlers"
"github.com/cloudwego/hertz/pkg/app"
"github.com/cloudwego/hertz/pkg/app/server"
)

func SetupRoutes(h *server.Hertz, productHandler *handlers.ProductHandler) {
    h.POST("/products", func(c context.Context, ctx *app.RequestContext) {
    productHandler.Create(c, ctx)
    })
    h.GET("/products", func(c context.Context, ctx *app.RequestContext) {
    productHandler.GetAll(c, ctx)
    })
}

```

在 api/http/handlers 中实现请求处理。

product_handler.go
```go
package handlers

import (
"context"
"zmall/internal/models"
"zmall/internal/repositories"
"github.com/cloudwego/hertz/pkg/app"
"github.com/cloudwego/hertz/pkg/protocol/consts"
)

type ProductHandler struct {
    productRepo *repositories.ProductRepository
}

func NewProductHandler(productRepo *repositories.ProductRepository) *ProductHandler {
    return &ProductHandler{productRepo: productRepo}
}

func (h *ProductHandler) Create(c context.Context, ctx *app.RequestContext) {
    var product models.Product
    if err := ctx.Bind(&product); err != nil {
    ctx.JSON(consts.StatusBadRequest, app.H{"error": err.Error()})
    return
    }
    if err := h.productRepo.Create(&product); err != nil {
    ctx.JSON(consts.StatusInternalServerError, app.H{"error": err.Error()})
    return
    }
    ctx.JSON(consts.StatusCreated, product)
}

func (h *ProductHandler) GetAll(c context.Context, ctx *app.RequestContext) {
    products, err := h.productRepo.GetAll()
    if err != nil {
    ctx.JSON(consts.StatusInternalServerError, app.H{"error": err.Error()})
    return
}
    ctx.JSON(consts.StatusOK, products)
}

```

## 步骤 4：主程序整合
目标：在 main.go 中整合配置、数据库连接、HTTP 服务和路由。
操作：

main.go
```go
package main

import (
"gorm.io/driver/mysql"
"gorm.io/gorm"
"zmall/api/http"
"zmall/api/http/handlers"
"zmall/config"
"zmall/internal/models"
"zmall/internal/repositories"
"github.com/cloudwego/hertz/pkg/app/server"
)

func main() {
cfg := config.LoadConfig()

    // 初始化数据库连接
    dsn := fmt.Sprintf("%s:%s@tcp(%s:%s)/%s?charset=utf8mb4&parseTime=True&loc=Local",
        cfg.DBUser, cfg.DBPassword, cfg.DBHost, cfg.DBPort, cfg.DBName)
    db, err := gorm.Open(mysql.Open(dsn), &gorm.Config{})
    if err != nil {
        panic("failed to connect database")
    }
    db.AutoMigrate(&models.Product{})

    // 初始化仓库和处理器
    productRepo := repositories.NewProductRepository(db)
    productHandler := handlers.NewProductHandler(productRepo)

    // 初始化 Hertz 服务器
    h := server.Default()
    http.SetupRoutes(h, productHandler)

    // 启动服务器
    h.Spin()
}

```

## 步骤 5：扩展功能
目标：逐步添加用户管理、订单管理、购物车等功能，实现完整的电商业务流程。
操作：
重复步骤 2 - 4，为用户、订单等模型创建对应的数据库操作、HTTP 接口和业务逻辑。
添加认证中间件，实现用户登录、注册功能。
实现购物车功能，允许用户添加、删除商品。
实现订单创建、支付、查询功能。
## 步骤 6：可观测性与优化
目标：添加日志记录、监控和链路追踪功能，优化系统性能。
操作：
在 pkg/logger 中实现日志记录功能。
集成 Prometheus 进行系统监控。
集成 OpenTelemetry 进行链路追踪。
通过以上步骤，你可以逐步完成一个完整度较高的单机电商项目。



