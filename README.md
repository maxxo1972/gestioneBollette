# Gestione Bollette - MVP Laravel

## 1. Architettura

MVP Laravel 13.x con PHP 8.3+, MariaDB/MySQL, Blade, Bootstrap 5 via Vite, autenticazione session-based, storage PDF privato e servizi separati per upload e parsing.

Strati principali:

- `routes/web.php`: route web, login, area amministrativa protetta da `auth`.
- `app/Http/Controllers`: controller per dashboard, fornitori, bollette, ricevute placeholder e autenticazione.
- `app/Http/Requests`: validazione Laravel centralizzata.
- `app/Services/BillPdfStorageService.php`: upload, sostituzione, download e rimozione PDF in storage privato.
- `app/Services/BillParsing`: interfaccia e parser placeholder per luce, gas e acqua.
- `app/Models`: model Eloquent con relazioni e cast decimal/date.
- `resources/views`: Blade Bootstrap per area amministrativa.
- `database/migrations`: schema completo per MVP e predisposizione Fase 2.
- `database/seeders`: admin iniziale e regole Mirella.

Scelte di sicurezza:

- PDF salvati fuori dalla web root in `storage/app/private/bills/{year}/{utility_type}/{bill_id}.pdf`.
- Nome originale conservato in forma sanitizzata solo come metadato.
- Download solo tramite controller autenticato.
- Validazione su estensione, MIME e magic header `%PDF-`.
- Foreign key e indici su campi di filtro.
- Percentuali Mirella in tabella `sharing_rules`, non hardcoded nel codice applicativo.

## 2. Installazione progetto

Su una macchina di sviluppo, se si vuole partire da zero:

```bash
composer create-project laravel/laravel gestione-bollette "^13.0"
cd gestione-bollette
composer require barryvdh/laravel-dompdf smalot/pdfparser
npm install bootstrap @popperjs/core
```

Questo pacchetto contiene gia i file applicativi del MVP. Per installarlo su server:

```bash
sudo mkdir -p /var/www/gestione-bollette
sudo rsync -a ./ /var/www/gestione-bollette/
cd /var/www/gestione-bollette

cp .env.example .env
composer install --no-dev --optimize-autoloader
npm install
npm run build
php artisan key:generate
php artisan migrate --seed
php artisan config:cache
php artisan route:cache
php artisan view:cache
```

Account iniziale:

- Email: `admin@example.local`
- Password temporanea: `ChangeMe123!`

Cambiare la password subito dopo il primo accesso, aggiungendo una pagina dedicata o aggiornando il record con un comando Artisan/Tinker in ambiente controllato.

## 3. Configurazione database

Esempio MariaDB:

```sql
CREATE DATABASE gestione_bollette CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'gestione_bollette'@'localhost' IDENTIFIED BY 'change-this-password';
GRANT ALL PRIVILEGES ON gestione_bollette.* TO 'gestione_bollette'@'localhost';
FLUSH PRIVILEGES;
```

Configurare `.env`:

```dotenv
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=gestione_bollette
DB_USERNAME=gestione_bollette
DB_PASSWORD=change-this-password
BILLS_PDF_MAX_UPLOAD_KB=20480
TESSERACT_PATH=/usr/bin/tesseract
```

## 4. Migrations

File generati:

- `database/migrations/0001_01_01_000000_create_users_table.php`
- `database/migrations/2026_06_26_000100_create_suppliers_table.php`
- `database/migrations/2026_06_26_000200_create_bills_table.php`
- `database/migrations/2026_06_26_000300_create_electricity_bill_details_table.php`
- `database/migrations/2026_06_26_000400_create_gas_bill_details_table.php`
- `database/migrations/2026_06_26_000500_create_water_bill_details_table.php`
- `database/migrations/2026_06_26_000600_create_bill_line_items_table.php`
- `database/migrations/2026_06_26_000700_create_sharing_rules_table.php`
- `database/migrations/2026_06_26_000800_create_receipts_table.php`
- `database/migrations/2026_06_26_000900_create_receipt_items_table.php`

Note schema:

- Importi: `decimal(10,2)`.
- Consumi: `decimal(12,3)`.
- Costo medio: `decimal(10,4)`.
- Indici: `year`, `utility_type`, `supplier_id`, `extraction_status`.
- `suppliers` ha vincolo unico su `name + utility_type`.
- `supplier_id` su `bills` usa `restrictOnDelete`.
- Dettagli luce/gas/acqua sono 1:1 con `bills`.

## 5. Models

File generati:

- `app/Models/User.php`
- `app/Models/Supplier.php`
- `app/Models/Bill.php`
- `app/Models/ElectricityBillDetail.php`
- `app/Models/GasBillDetail.php`
- `app/Models/WaterBillDetail.php`
- `app/Models/BillLineItem.php`
- `app/Models/SharingRule.php`
- `app/Models/Receipt.php`
- `app/Models/ReceiptItem.php`

Relazioni principali:

- `Supplier hasMany Bill`
- `Bill belongsTo Supplier`
- `Bill hasOne ElectricityBillDetail`
- `Bill hasOne GasBillDetail`
- `Bill hasOne WaterBillDetail`
- `Bill hasMany BillLineItem`
- `Receipt hasMany ReceiptItem`
- `ReceiptItem belongsTo Receipt`
- `ReceiptItem belongsTo Bill`

## 6. Controllers

File generati:

- `app/Http/Controllers/AuthController.php`
- `app/Http/Controllers/DashboardController.php`
- `app/Http/Controllers/SupplierController.php`
- `app/Http/Controllers/BillController.php`
- `app/Http/Controllers/ReceiptController.php`

Funzioni MVP:

- Login/logout admin.
- Dashboard con conteggi e ultime 10 bollette.
- CRUD fornitori.
- CRUD bollette.
- Download PDF autenticato.
- Lista ricevute predisposta per Fase 2.

## 7. Services

File generati:

- `app/Services/BillPdfStorageService.php`
- `app/Services/BillParsing/BillParserInterface.php`
- `app/Services/BillParsing/AbstractBillParser.php`
- `app/Services/BillParsing/ElectricityBillParser.php`
- `app/Services/BillParsing/GasBillParser.php`
- `app/Services/BillParsing/WaterBillParser.php`
- `app/Services/BillParsing/BillParserManager.php`

`BillPdfStorageService` salva i PDF in:

```text
storage/app/private/bills/{year}/{utility_type}/{bill_id}.pdf
```

I parser placeholder non inventano dati: restituiscono array vuoto e stato `manual_review`.

## 8. Requests

File generati:

- `app/Http/Requests/LoginRequest.php`
- `app/Http/Requests/StoreSupplierRequest.php`
- `app/Http/Requests/UpdateSupplierRequest.php`
- `app/Http/Requests/StoreBillRequest.php`
- `app/Http/Requests/UpdateBillRequest.php`

Validazioni incluse:

- Tipologia utenza in `electricity`, `gas`, `water`.
- Stato estrazione in `pending`, `success`, `manual_review`, `error`.
- Fornitore coerente con tipo bolletta.
- PDF richiesto in creazione e opzionale in modifica.
- PDF massimo configurabile con `BILLS_PDF_MAX_UPLOAD_KB`.
- Date, anno e importi validati lato server.

## 9. Routes

File: `routes/web.php`

Route pubbliche:

- `GET /login`
- `POST /login`

Route protette da `auth`:

- `POST /logout`
- `GET /dashboard`
- CRUD `suppliers`
- CRUD `bills`
- `GET /bills/{bill}/pdf`
- `GET /receipts`

## 10. Views Blade

File generati:

- `resources/views/layouts/app.blade.php`
- `resources/views/auth/login.blade.php`
- `resources/views/dashboard/index.blade.php`
- `resources/views/suppliers/index.blade.php`
- `resources/views/suppliers/create.blade.php`
- `resources/views/suppliers/edit.blade.php`
- `resources/views/suppliers/_form.blade.php`
- `resources/views/bills/index.blade.php`
- `resources/views/bills/create.blade.php`
- `resources/views/bills/edit.blade.php`
- `resources/views/bills/_form.blade.php`
- `resources/views/bills/show.blade.php`
- `resources/views/receipts/index.blade.php`

Frontend:

- Bootstrap 5 via Vite.
- Layout responsive amministrativo.
- Tabelle con filtri base lato server.
- Form con CSRF nativo Laravel.

## 11. Seeder

File generati:

- `database/seeders/DatabaseSeeder.php`
- `database/seeders/AdminUserSeeder.php`
- `database/seeders/SharingRuleSeeder.php`

Admin iniziale:

```text
Name: Admin
Email: admin@example.local
Password: ChangeMe123!
```

Regole Mirella:

| utility_type | beneficiary_name | percentage | calculation_base | active |
| --- | --- | ---: | --- | --- |
| electricity | Mirella | 32.00 | total_amount | true |
| water | Mirella | 50.00 | total_amount | true |
| gas | Mirella | 0.00 | total_amount | false |

## 12. Apache VirtualHost

File: `apache/gestione-bollette.conf`

Installazione pacchetti Ubuntu:

```bash
sudo apt update
sudo apt install -y apache2 mariadb-server unzip git curl
sudo apt install -y php8.3 php8.3-fpm php8.3-cli php8.3-mysql php8.3-mbstring php8.3-xml php8.3-curl php8.3-zip php8.3-bcmath php8.3-intl php8.3-gd
sudo apt install -y tesseract-ocr tesseract-ocr-ita
```

Abilitazione Apache:

```bash
sudo cp apache/gestione-bollette.conf /etc/apache2/sites-available/gestione-bollette.conf
sudo a2enmod rewrite proxy_fcgi setenvif headers
sudo a2enconf php8.3-fpm
sudo a2ensite gestione-bollette.conf
sudo apache2ctl configtest
sudo systemctl reload apache2
sudo systemctl enable --now php8.3-fpm
```

Per HTTPS usare Certbot o un certificato interno e migrare il VirtualHost su `*:443`.

## 13. Permessi Linux

Utente consigliato: proprietario deploy, gruppo `www-data`.

```bash
sudo chown -R deploy:www-data /var/www/gestione-bollette
sudo find /var/www/gestione-bollette -type f -exec chmod 0640 {} \;
sudo find /var/www/gestione-bollette -type d -exec chmod 0750 {} \;
sudo chmod -R ug+rwX /var/www/gestione-bollette/storage /var/www/gestione-bollette/bootstrap/cache
sudo chmod 0750 /var/www/gestione-bollette/public
sudo find /var/www/gestione-bollette/storage/app/private -type f -name '*.pdf' -exec chmod 0640 {} \;
```

I PDF restano fuori da `public` e non sono serviti direttamente da Apache.

## 14. Test funzionali

Checklist minima:

1. `php artisan migrate:fresh --seed` completa senza errori.
2. Login con `admin@example.local` e `ChangeMe123!`.
3. Creazione fornitore luce, gas e acqua.
4. Creazione bolletta con PDF valido.
5. Verifica file in `storage/app/private/bills/{year}/{utility_type}/{id}.pdf`.
6. Download PDF da utente autenticato.
7. Logout e verifica che `/bills/{id}/pdf` rimandi al login.
8. Filtri elenco bollette per anno, tipo, fornitore e stato.
9. Modifica bolletta con sostituzione PDF.
10. Eliminazione fornitore non collegato a bollette.
11. Tentativo eliminazione fornitore collegato: deve fallire.
12. Verifica dashboard: conteggi e ultime 10 bollette.

Comandi utili:

```bash
php artisan about
php artisan route:list
php artisan migrate:fresh --seed
php artisan test
```

## 15. Prossimi step per Fase 2

- Implementare estrazione testo con `smalot/pdfparser`.
- Aggiungere fallback OCR con Tesseract per PDF scannerizzati.
- Creare parser specifici per fornitore e tipo utenza.
- Completare form dettagli luce/gas/acqua.
- Generare ricevute PDF con Dompdf.
- Calcolare ricevute da `sharing_rules`.
- Aggiungere dashboard multi-anno e report esportabili.
- Aggiungere cambio password admin e gestione utenti.
- Aggiungere test Feature automatizzati per upload e autorizzazioni.

## Backup database consigliato

Script: `scripts/backup-db.sh`

Esempio manuale:

```bash
sudo DB_NAME=gestione_bollette DB_USER=gestione_bollette DB_PASSWORD='change-this-password' bash scripts/backup-db.sh
```

Esempio cron giornaliero:

```cron
30 2 * * * DB_NAME=gestione_bollette DB_USER=gestione_bollette DB_PASSWORD='change-this-password' /var/www/gestione-bollette/scripts/backup-db.sh >> /var/log/gestione-bollette-backup.log 2>&1
```
