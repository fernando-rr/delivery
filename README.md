# Delivery Platform - MVP Multi-tenant

## Repositórios do Projeto

Este projeto está dividido em repositórios separados:

- **Backend API:** [delivery-api](https://github.com/fernando-rr/delivery-api) - Laravel API multi-tenant
- **Frontend:** [delivery-front](https://github.com/fernando-rr/delivery-front) - Nuxt 3 SPA
- **Documentação:** Este repositório principal com documentação e arquitetura

## Links dos Repositórios

- 🔗 **Backend:** https://github.com/fernando-rr/delivery-api
- 🔗 **Frontend:** https://github.com/fernando-rr/delivery-front

---

# 1 — Visão geral (escopo ajustado)

Objetivo do MVP (ajustado):

* Plataforma onde cada restaurante tenha sua “loja” com DB próprio (clientes, produtos, cardápios, pedidos, faturas de pedidos).
* Banco central (SaaS) com restaurantes, planos/assinaturas, faturas de assinatura e gerência.
* Primeiro lançar pedidos via **site** (formulário público + painel do restaurante).
* Notificações via **WhatsApp** ficarão para o final (quando cliente iniciar conversa o WhatsApp é gratuito por 24h); pedidos pelo site **não** devem iniciar conversa por WhatsApp — usar **SMS** ou notificação in-app quando cliente estiver logado.
* MySQL será o motor; DB do tenant terá nome relacionado ao id da loja (recomendo prefixo `tenant_{id}`).

# 2 — Arquitetura recomendada (resumida)

* **Backend:** Laravel 12 API (Sanctum para SPA).

  * 2 conjuntos de migrations: `central` e `tenant`.
  * Dynamic DB connections para `tenant` (uma connection "tenant" que é reprovisionada com o nome do DB do restaurante no runtime).
  * Jobs/Queues (Redis 7.0+) para envios de SMS, geração de faturas/PDFs, processamento assíncrono.
* **Frontend:** Nuxt 3.14+ (SPA) — páginas públicas de pedido + painel restaurante (login).
* **DB:** MySQL 8.0+ central + múltiplos bancos MySQL (um por restaurante). Mesmo serviço MySQL pode conter muitos bancos.
* **Cache/Queue:** Redis 7.0+.
* **Storage:** S3/Spaces (arquivos, PDFs de nota/fatura).
* **Infra:** Docker Compose (traefik/nginx, app, mysql, redis, nuxt). Fazer deploy no VPS Locaweb.
* **Notificações:** SMS provider (Twilio/Zenvia/TWW) para pedidos pelo site. WhatsApp Cloud ou BSP só depois (opcional).

# 3 — Modelo de dados (central vs tenant)

## Central DB (SaaS)

Tabelas principais:

* `users` (saas admin / operadores do SaaS) — opcional
* `restaurants` (id, name, slug, db_name, plan_id, status, contact_phone, contact_email, created_at)
* `plans` (nome, features, price, billing_cycle)
* `subscriptions` (restaurant_id, plan_id, status, current_period_start, current_period_end)
* `subscription_invoices` (cobrancas de assinatura, amount, status, pdf_url, due_date, payment_provider_id)
* `billing_events` (logs de cobrança)
* `tenant_provision_logs` (audit)

Observação: `restaurants.db_name` terá o nome do DB do tenant — por exemplo `tenant_42` (recomendo prefixo para evitar colisões).

## Tenant DB (por restaurante)

Para cada banco novo criar tabelas:

* `products` (sku, name, price, description, active, category_id)
* `menus` / `menu_items` (organizar cardápios)
* `customers` (name, phone, email, address, external_id)
* `orders` (customer_id, total, status, payment_method, created_at)
* `order_items` (order_id, product_id, qty, price)
* `order_invoices` (order_id, invoice_number, amount, status: pending/paid/cancelled, pdf_url, created_at)
* `payments` (se integrar pagamentos)
* `notifications` (in-app notifications)
* `settings` (config do restaurante: taxas, tempo_preparo, envio_sms_enabled, whatsapp_enabled)

# 4 — Estratégia técnica para Multi-DB no Laravel

### 1) Padrão sugerido

* Mantenha um **central DB** com os metadados do tenant (nome do DB, plan, config).
* Crie um **connection template** `tenant` em `config/database.php`, e ao usar um tenant você faz `config(['database.connections.tenant.database' => $dbName])` + `DB::purge('tenant')` + `DB::reconnect('tenant')`.
* Migrations: mantenha `database/migrations/central` e `database/migrations/tenant`. Ao provisionar um tenant, execute as migrations do tenant com `--database=tenant` apontando para a connection dinâmica.

### 2) Criar DB do tenant (exemplo)

No signup do restaurante:

1. Criar registro `restaurants` no central.
2. Gerar db_name: `tenant_{id}`.
3. Criar DB no MySQL:

```php
DB::statement("CREATE DATABASE IF NOT EXISTS `{$dbName}` CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci");
```

4. Ajustar `config(['database.connections.tenant.database' => $dbName]); DB::purge('tenant'); DB::reconnect('tenant');`
5. Rodar migrations do tenant:

```php
\Artisan::call('migrate', [
  '--database' => 'tenant',
  '--path' => 'database/migrations/tenant',
  '--force' => true
]);
```

6. Seed iniciais (ex.: categorias, tempo padrão, exemplo de cardápio).

### 3) Riscos / limitações do approach DB-per-tenant

* **Número de databases**: MySQL suporta muitos DBs, mas performance de administração e backups é maior. Se você espera milhares+ de restaurantes, considere **single DB multi-tenant** (tenant_id) ou **schema-based** alternativas.
* **Operações em massa** (alterar uma coluna para todos os tenants) é mais custoso: precisa rodar migrations em cada DB. Scripts/CLI para rodar migration em lote são obrigatórios.
* **Conexões e pool:** gerenciar conexões dinâmicas pode aumentar uso de conexões; ajuste `max_connections` no MySQL se necessário.
* **Nomes de DB:** valide tamanho/caracteres; use prefixo `tenant_` + id numérico para simplicidade.

# 5 — Fluxo de pedido (site) — detalhe

1. Cliente abre site do restaurante (página pública/nuxt).
2. Cliente cria pedido (sem login ou com login):

   * Se sem login: pede nome, telefone (campo obrigatório), endereço; cria `customers` com telefone e `orders`.
3. Pedido é salvo no DB do tenant (chamando API que conecta ao DB do tenant).
4. Notificação automática:

   * **Não** enviar WhatsApp (para não iniciar conversa).
   * Opções:

     * Se cliente escolheu SMS (ou o restaurante configurou SMS), enfileirar job para SMS via provider.
     * Se cliente fez login no portal do restaurante (app), criar notificação in-app e mostrar status no painel cliente.
     * Opcional: notificar por e-mail.
5. Painel do restaurante (Nuxt) vê novo pedido e atualiza status: Recebido → Em preparo → Pronto → Entregue.
6. Quando trocar status, sistema gera `order_invoice` (se necessário) e pode enviar SMS/in-app conforme config do restaurante.

# 6 — Endpoints propostos (separados)

## Central API (Laravel)

* `POST /api/central/restaurants` — criar restaurante (signup SaaS) → provisiona DB tenant.
* `GET /api/central/restaurants` — listar restaurantes (admin).
* `GET /api/central/plans`, `POST /api/central/subscriptions` — gestão de planos e assinaturas.
* `POST /api/central/webhooks/payment` — receber webhooks de cobranças de assinatura.

## Tenant API (cada restaurante)

(servidos pelo mesmo backend, com middleware que determina tenant com base no header `X-Tenant-Id` **ou** subdomínio)

* `POST /api/orders` — criar pedido público.
* `GET /api/orders` — listar (auth restaurante).
* `GET /api/orders/{id}`
* `PATCH /api/orders/{id}/status` — atualizar status.
* `GET /api/products`, `POST /api/products`, etc.
* `GET /api/customers`, `POST /api/customers`
* `GET /api/order-invoices`, `POST /api/order-invoices/{id}/pay` (opcional).

# 7 — Autenticação e roteamento tenant

* Onboarding: restaurante cria conta → central API cria tenant + retorna subdomínio `loja-{slug}.seusite.com`.
* Recomendo usar **subdomínio por restaurante** (ex.: `meupizza.seusite.com`) — facilita seleção de tenant via Host header. Alternativa: enviar `X-Tenant-Id` em requests públicos.
* Middleware `IdentifyTenant`:

  * obtém `restaurant` pelo subdomínio (ou header),
  * carrega `db_name`,
  * configura `database.connections.tenant` dinamicamente.
* Use Sanctum token para users do restaurante (autenticação para painel).

# 8 — Faturas (diferenciar de pagamentos)

Você pediu “faturas” ao invés de “pagamentos” para pedidos — bom: uma fatura é um documento/registro de cobrança sobre um pedido.

Fluxo recomendado:

* Quando pedido finalizado (ou conforme regra do restaurante), gerar `order_invoice` com:

  * `invoice_number` (sequencial por tenant),
  * `amount`, `due_date`, `status` (pending/paid/cancelled),
  * gerar PDF com `laravel-dompdf` e armazenar em S3 (`pdf_url`).
* Integração de pagamento:

  * Se quiser integrar Pix/Cartão: gerar link de cobrança com Gerencianet/Pagar.me/MercadoPago/Stripe e salvar referência na fatura.
  * Em env de MVP você pode deixar faturas manualmente marcadas como pagas pelo restaurante.

# 9 — Processos assíncronos

* **Fila** (Redis):

  * Job: `CreateOrderInvoiceJob` — gera PDF e salva.
  * Job: `SendSmsJob` — envia SMS para cliente se configurado.
  * Job: `SyncPaymentStatusJob` — processa webhooks de pagamentos.
* **Retries** e logging obrigatórios.

# 10 — Backups e operações

* Backups separados:

  * **Central DB:** dump diário.
  * **Tenant DBs:** dump por tenant (cron) — pode ser `mysqldump tenant_42 > /backups/tenant_42.sql`. Automatizar e armazenar remoto (S3).
* Recomendação: rotacionar backups, retenção 30 dias.
* Migrations: plano para rodar migrations em lote (script que lê todos restaurants e executa migrate —database=tenant).

# 11 — Docker / Deploy (pontos rápidos)

* MySQL service com `max_allowed_packet`, `innodb_buffer_pool_size` ajustado se você tiver muitos tenants.
* Volumes por MySQL data.
* Redis container.
* Traefik para TLS.
* Healthchecks e logs rotacionados.

# 12 — Prioridade de implementação (passo-a-passo prático)

Fase 0 — preparação

1. Repositórios: `delivery-api` (Laravel 12), `delivery-front` (Nuxt 3.14+).
2. Provisionar VPS (Docker + Traefik + DNS).

Fase 1 — Core central (2–4 dias)

1. Modelos e migrations central (`restaurants`, `plans`, `subscriptions`, `subscription_invoices`).
2. Endpoint de signup que **provisiona tenant DB** (cria DB + roda migrations tenant).
3. Painel Admin SaaS (CRUD Planos, Restaurantes — o que você já tem nas imagens; completar CRUD assinaturas e faturas de assinatura).

Fase 2 — Tenant baseline (3–6 dias)

1. Criar migrations tenant (products, menus, customers, orders, order_items, order_invoices).
2. Implementar middleware `IdentifyTenant` e lógica de conexão dinâmica.
3. Endpoints tenant: produtos, cardápios, criar pedido (public) e painel para listar/atender pedido.

Fase 3 — Frontend MVP (3–7 dias)

1. Nuxt: formulário público de pedido (consumir `POST /api/orders` tenant).
2. Nuxt: painel restaurante (login, lista de pedidos, mudar status).
3. In-app notifications para clientes logados.

Fase 4 — Billing & faturas (2–4 dias)

1. Implementar `order_invoice` generation + PDF.
2. Subscription billing: geração de faturas de assinatura no central e integração com gateway (inicialmente manual/test).

Fase 5 — Notificações e SMS (2–3 dias)

1. Integrar provider SMS (Twilio/Zenvia) via jobs.
2. Config playlists no tenant `settings` (usar SMS para pedidos pelo site).

Fase 6 — WhatsApp (último) (tempo depende do onboarding Meta)

1. Onboarding Meta Business / WABA (pode levar dias).
2. Webhook WhatsApp + provider adapter (Meta Cloud + BSP opcional).
3. Logica para só enviar templates quando **cliente já iniciou conversa**.

# 13 — Boas práticas / recomendações finais

* Use prefixo no DB (`tenant_{id}`) — evita colisão.
* Scripts de administração: crie Artisan commands para rodar migrations/seed/backup para todos tenants.
* Monitore custo de SMS/WhatsApp e limite envios por tenant.
* Se crescer muito, reavalie strategy (migrar para single DB com tenant_id ou shard DBs por cluster).
* Crie um plano de rollback para migrations que alterem muitos tenants.

## Stack Tecnológica Atualizada

### Backend (Laravel 12)
- **PHP 8.3+** - Linguagem principal
- **Laravel 12** - Framework mais recente
- **MySQL 8.0+** - Banco de dados
- **Redis 7.0+** - Cache e filas
- **Laravel Sanctum** - Autenticação API
- **Laravel Horizon** - Monitoramento de filas
- **Laravel Telescope** - Debug e profiling
- **Laravel Pint** - Code style fixer
- **Laravel Pest** - Framework de testes

### Frontend (Nuxt 3.14+)
- **Node.js 20+** - Runtime
- **Nuxt 3.14+** - Framework Vue.js
- **Vue 3.5+** - Framework JavaScript
- **TypeScript 5.5+** - Linguagem tipada
- **Tailwind CSS 3.4+** - Framework CSS
- **Pinia 2.1+** - Gerenciamento de estado
- **Vite 6.0+** - Build tool
- **Vitest** - Framework de testes
- **pnpm 9.0+** - Package manager

### DevOps
- **Docker** - Containerização
- **Docker Compose** - Orquestração local
- **GitHub Actions** - CI/CD
- **Traefik** - Reverse proxy
- **Nginx** - Web server