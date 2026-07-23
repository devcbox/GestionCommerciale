# 🏪 GestionCommerciale — Multi-Tenant Retail & Inventory Management System

**GestionCommerciale** is a full-featured commercial management system (ERP/POS) built with **Laravel 10**, covering the entire retail cycle: purchasing, warehousing, stock, sales, and legally certified invoicing. It's designed to serve **multiple independent companies from a single codebase**, each operating on its own isolated database, with fine-grained, role-based access control down to individual menu actions.

A standout feature: the system integrates directly with **Benin's official e-invoicing platform (DGI / SYGMEF-eMCF)** to generate government-certified electronic invoices with QR codes.

---

## 🚀 Key Features

### 🏢 Multi-Company / Multi-Database Architecture
- Each company (`Entreprise`) can be provisioned with its own database.
- At login, the user selects the target company database; the connection is then dynamically switched for the entire session via a dedicated middleware.
- Company records, fiscal years (`Exercice`), and settings are managed independently per tenant.

### 👥 Role-Based Access Control (down to the action level)
- Custom roles (`Role`) with permissions defined **per menu and per action** (`view`, `create`, `edit`, `delete`) via a `menu_role` pivot table.
- Every route is protected by a `can_menu:{menu},{action}` middleware, enforced on top of standard authentication.

### 📦 Purchasing & Inventory
- Suppliers (`Fournisseur`) and supplier categories.
- Purchase orders (`CommandeAchat`) with line items and VAT handling.
- Goods receiving (`ReceptionCmdAchat`) linked back to purchase orders.
- Multi-warehouse stock (`Magasin`, `Stocke`) with stock adjustments and inter-warehouse transfers (`TransfertMagasin`).
- Physical inventory counts (`Inventaire`).
- Product catalog structured by family and category (`FamilleProduit`, `CategorieProduit`), with tariff categories for pricing (`CategorieTarifaire`).

### 🧾 Sales & Certified Invoicing
- Point-of-sale style invoicing (`Vente`) and quotes/proforma (`Proforma`).
- Client management with client categories (`CategorieClient`).
- Multiple payment modes (`ModePaiement`) and daily cash closing (`Fermetures`).
- **Government-certified invoices (`FactureNormalisee`)**: invoices are submitted to Benin's DGI SYGMEF-eMCF API, which returns an official `codeMECeFDGI`, a fiscal counter (NIM), and a QR code — required for tax-compliant retail invoicing in Benin.
- Credit note / invoice cancellation flow, cross-checked against the original certified invoice code.

### 📊 Dashboard & Reporting
- Operational dashboard (`TableauController`).
- Excel exports (via Maatwebsite/Laravel-Excel), e.g. company/data exports.

---

## 🛠 Tech Stack

| Technology | Role |
|---|---|
| Laravel 10 (PHP 8.1+) | Backend framework (MVC, Blade views) |
| MySQL | Database, one instance per company (multi-tenant) |
| Laravel Sanctum | Session/token authentication |
| Custom RBAC (Role/Menu pivot) | Fine-grained, per-menu, per-action permissions |
| Maatwebsite/Laravel-Excel (PhpSpreadsheet) | Excel import/export |
| Endroid QR-Code | QR code generation for certified invoices |
| Benin DGI SYGMEF-eMCF API | Real-time invoice certification (external government API) |
| Guzzle / cURL | HTTP client for the certification API |

---

## 🏗 Architecture Highlights

### Dynamic tenant database switching

```php
// EnsureDatabaseConnection middleware
if (Auth::check() && session()->has('selected_database')) {
    config(['database.connections.mysql.database' => session('selected_database')]);
    DB::purge('mysql');
    DB::reconnect('mysql');
}
```

At login, the available databases are listed and the user picks their company; from that point on, every query in the session runs against the selected tenant's database.

### Menu-level permission enforcement

```php
Route::get('/Produits', [ProduitController::class, 'Produits'])
    ->middleware('auth', 'can_menu:Produits,view');
```

Every protected route declares which menu and which action (`view`, `create`, `edit`, `delete`) it requires. Permissions are configured per role from the **Menu Permissions** admin screen and checked against the `menu_role` pivot table on every request.

### Certified invoice flow

1. A sale (`Vente`) is finalized and its data is sent to the SYGMEF-eMCF API for confirmation.
2. The DGI's response provides a certification code (`codeMECeFDGI`), a fiscal counter, a NIM, and a timestamp.
3. A QR code is generated from this data and stored on the `FactureNormalisee` record, alongside the invoice totals and VAT breakdown.
4. Credit notes reference the original certified code and are validated against it before being processed.

---

## 🧱 Core Data Model (non-exhaustive)

| Table / Model | Description |
|---|---|
| `Entreprise` | Tenant company, each mapped to its own database |
| `Utilisateur` / `Role` / `Menu` | Users, roles, and menu-based permissions |
| `Fournisseur` / `CategorieFournisseur` | Suppliers |
| `Produit` / `FamilleProduit` / `CategorieProduit` | Product catalog |
| `Magasin` / `Stocke` | Warehouses and stock levels |
| `CommandeAchat` / `DetailCommandeAchat` | Purchase orders |
| `ReceptionCmdAchat` / `DetailReceptionCmdAchat` | Goods receiving |
| `TransfertMagasin` / `DetailTransfertMagasin` | Inter-warehouse transfers |
| `Inventaire` / `DetailInventaire` | Physical stock counts |
| `Client` / `CategorieClient` | Customers |
| `Vente` / `DetailVente` | Sales / invoices |
| `Proforma` / `DetailProforma` | Quotes |
| `FactureNormalisee` | DGI-certified invoice records (MECeF code, QR code, VAT totals) |
| `ModePaiement` / `Fermetures` / `DetailFermetures` | Payment methods and daily cash closing |
| `Exercice` | Fiscal years |
| `CategorieTarifaire` | Pricing tiers |

---

## 🚀 Installation

### Prerequisites
- PHP 8.1+
- Composer
- MySQL
- Node.js (for asset compilation via Vite)

### Steps

```bash
# 1. Clone the project
git clone https://github.com/your-account/GestionCommerciale.git
cd GestionCommerciale

# 2. Install PHP dependencies
composer install

# 3. Set up the environment
cp .env.example .env
php artisan key:generate
```

Configure your database connection in `.env`:

```env
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=commerce
DB_USERNAME=root
DB_PASSWORD=your_password
```

```bash
# 4. Run migrations (and seeders if available)
php artisan migrate --seed

# 5. Install and build front-end assets
npm install
npm run build

# 6. Serve the application
php artisan serve
```

### DGI / SYGMEF-eMCF certification

To use the certified-invoicing feature, a valid connection to Benin's DGI SYGMEF-eMCF sandbox or production API is required (device token, IFU, and the `storage/certificates/cacert.pem` certificate bundle used for the cURL calls). Without this configuration, standard (non-certified) sales and quotes still work normally.

---

## 🔐 Authentication & Access

- Login requires selecting the target company database in addition to credentials.
- Once authenticated, every page and action is gated by the `auth` and `can_menu:{menu},{action}` middleware pair.
- An **Administrator** role has access to system-wide settings: companies, users, roles & menu permissions, fiscal years, payment modes, and pricing tiers.



---

## 📈 What This Project Demonstrates

- Designing and operating a **multi-tenant Laravel application** with per-company database isolation
- Building a **custom, menu-level RBAC system** from scratch
- Integrating with a **real external government API** (Benin DGI SYGMEF-eMCF) for legally certified invoicing
- Managing a full commercial cycle: purchasing → receiving → stock → sales → certified invoicing
- QR code generation and Excel import/export
- Structuring a large-scale Laravel application (30+ models, dozens of controllers) around real business workflows

---

## 👨‍💻 Author

**Franck Dieu-donné AYENAN**
Backend Developer — Laravel