# Delivery Platform - Multi-tenant MVP

Plataforma de delivery multi-tenant onde cada restaurante possui seu próprio banco de dados isolado, com sistema central SaaS para gestão de planos e assinaturas.

## 🎯 Objetivo

Construir uma plataforma SaaS onde:
- **Sistema Central:** Gerencia restaurantes, planos, assinaturas e faturas de assinatura
- **Sistema Tenant:** Cada restaurante tem seu DB próprio (produtos, pedidos, clientes, cardápios)
- **Multi-tenant:** Arquitetura com DB por restaurante (padrão `tenant_{id}`)
- **API First:** Backend Laravel servindo frontend Nuxt via API
- **Escalável:** Preparado para crescimento com Redis, filas e cache

### Acesso

- **Frontend:** http://delivery.local
- **API:** http://delivery.local/api

Cada projeto possui seu próprio docker.

## 🏗️ Arquitetura

### Multi-tenant (DB por Restaurante)
- **Central DB:** `delivery_central` - restaurantes, planos, assinaturas
- **Tenant DBs:** `tenant_{id}` - produtos, pedidos, clientes, cardápios

### Stack Backend
- **Framework:** Laravel 12 (PHP 8.3)
- **Autenticação:** Laravel Sanctum
- **Database:** MySQL 8.0 (central + múltiplos tenant DBs)
- **Cache/Queue:** Redis 7.0
- **Storage:** S3/Spaces (futuro)

### Stack Frontend
- **Framework:** Nuxt 3.14
- **UI:** Vue 3.5 + TypeScript 5.5
- **Styling:** Tailwind CSS 3.4
- **State:** Pinia 2.1

### Endpoints Principais

**Central API:**
- `POST /api/central/restaurants` - Criar restaurante (provisiona tenant DB)
- `GET /api/central/plans` - Listar planos
- `POST /api/central/subscriptions` - Criar assinatura

**Tenant API:**
- `POST /api/orders` - Criar pedido
- `GET /api/orders` - Listar pedidos
- `GET /api/products` - Listar produtos
- `GET /api/customers` - Listar clientes

## 📋 Fluxo Multi-tenant

1. **Signup Restaurante:** API Central cria registro + provisiona `tenant_{id}` DB
2. **Migrations Tenant:** Executa migrations do tenant no novo DB
3. **Identificação:** Middleware identifica tenant via header/subdomínio
4. **Dynamic Connection:** Laravel conecta ao DB do tenant em runtime
5. **Isolamento:** Cada restaurante acessa apenas seus dados

## 🎯 Roadmap MVP

- [x] Infra Docker local
- [ ] Migrations central (restaurants, plans, subscriptions)
- [ ] Migrations tenant (products, orders, customers)
- [ ] API Central (CRUD restaurantes, provisionar tenant)
- [ ] Middleware IdentifyTenant
- [ ] API Tenant (produtos, pedidos)
- [ ] Frontend: formulário pedido público
- [ ] Frontend: painel restaurante
- [ ] Jobs assíncronos (Redis)
- [ ] Notificações SMS (Twilio/Zenvia)

## 🔐 Licença

MIT License
