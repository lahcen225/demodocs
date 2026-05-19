# Documentatie: Ansible, database aanpassen en website uitleg

Projectmap: `/home/lahcen/2526-pe2-lahcen225`

Belangrijke locaties:

- Ansible playbook: `/home/lahcen/2526-pe2-lahcen225/ansible/playbook.yml`
- Inventory: `/home/lahcen/2526-pe2-lahcen225/local-deploy/inventory.ini`
- Database schema: `/home/lahcen/2526-pe2-lahcen225/storage/schema.sql`
- Frontend: `/home/lahcen/2526-pe2-lahcen225/frontend`
- Backend: `/home/lahcen/2526-pe2-lahcen225/backend`

## 1. Wat is Ansible heel simpel?

Ansible is een tool waarmee je servers automatisch installeert en configureert.

In plaats van op elke machine manueel commando's uit te voeren, schrijf je in Ansible wat er moet gebeuren. Daarna voert Ansible dat uit op de juiste machines.

In dit project gebruikt Ansible drie machines:

- `frontend_vm` met IP `10.10.0.10`
- `backend_vm` met IP `10.10.0.20`
- `storage_vm` met IP `10.10.0.30`

Deze staan in:

```bash
/home/lahcen/2526-pe2-lahcen225/local-deploy/inventory.ini
```

De inventory zegt dus tegen Ansible: "dit zijn mijn servers".

## 2. Wat doet het Ansible playbook?

Het belangrijkste playbook staat hier:

```bash
/home/lahcen/2526-pe2-lahcen225/ansible/playbook.yml
```

Dat playbook configureert de drie delen van de applicatie.

### Storage tier

De storage machine is de database machine.

Ansible installeert en configureert hier PostgreSQL. PostgreSQL is de database waarin users, events en tickets worden opgeslagen.

Belangrijke tabellen:

- `users`
- `events`
- `tickets`

### Backend tier

De backend machine draait de Node.js API.

De backend verwerkt de logica van de website:

- registreren
- inloggen
- events ophalen
- events aanmaken
- tickets reserveren
- tickets betalen
- reservaties annuleren

De backend praat met PostgreSQL op de storage machine.

### Frontend tier

De frontend machine draait nginx.

Nginx toont de website in de browser en stuurt API-aanvragen door naar de backend.

De website is bereikbaar via:

```bash
http://localhost:8080
```

## 3. Deployment met Ansible

Eerst start je de Vagrant machines:

```bash
cd /home/lahcen/2526-pe2-lahcen225/local-deploy
vagrant up
```

Daarna deploy je de applicatie met Ansible vanuit de projectmap:

```bash
cd /home/lahcen/2526-pe2-lahcen225
ansible-playbook -i local-deploy/inventory.ini ansible/playbook.yml --ask-vault-pass
```

`--ask-vault-pass` is nodig omdat geheime waarden, zoals databasewachtwoorden en JWT secrets, in Ansible Vault zitten.

## 4. Hoe pas je manueel iets aan in de database?

Normaal moet je database-structuur en configuratie via Ansible aanpassen. Dat is beter omdat je wijzigingen dan reproduceerbaar zijn.

Maar soms wil je manueel kijken of tijdelijk iets aanpassen. Doe dat voorzichtig.

### Stap 1: ga naar de storage VM

```bash
cd /home/lahcen/2526-pe2-lahcen225/local-deploy
vagrant ssh storage
```

### Stap 2: open PostgreSQL

Gebruik de postgres gebruiker:

```bash
sudo -iu postgres psql ticketing
```

Je zit dan in de database `ticketing`.

### Stap 3: bekijk de tabellen

In `psql`:

```sql
\dt
```

Bekijken welke users bestaan:

```sql
SELECT id, email, role, created_at FROM users;
```

Bekijken welke events bestaan:

```sql
SELECT id, title, event_date, location, capacity, price_cents FROM events;
```

Bekijken welke tickets bestaan:

```sql
SELECT id, event_id, owner_id, status, reserved_until FROM tickets;
```

### Stap 4: wijzig iets met een transaction

Gebruik liefst altijd `BEGIN` en `COMMIT`. Dan kun je eerst controleren voor je definitief opslaat.

Voorbeeld: titel van een event aanpassen:

```sql
BEGIN;

UPDATE events
SET title = 'Nieuwe event naam'
WHERE id = 1;

SELECT id, title FROM events WHERE id = 1;

COMMIT;
```

Als je fout zit, gebruik je in plaats van `COMMIT`:

```sql
ROLLBACK;
```

### Voorbeeld: locatie van een event aanpassen

```sql
BEGIN;

UPDATE events
SET location = 'PXL Hasselt'
WHERE id = 1;

SELECT id, title, location FROM events WHERE id = 1;

COMMIT;
```

### Voorbeeld: capaciteit begrijpen

De capaciteit van een event staat in de tabel `events`.

Maar tickets worden apart opgeslagen in de tabel `tickets`. Als een event capaciteit 5 heeft, bestaan er normaal 5 ticket-rijen voor dat event.

Dus als je alleen dit doet:

```sql
UPDATE events
SET capacity = 10
WHERE id = 1;
```

dan heb je nog niet automatisch 10 tickets. Je moet dan ook extra tickets toevoegen.

Voorbeeld om 5 extra beschikbare tickets toe te voegen:

```sql
BEGIN;

INSERT INTO tickets (event_id, status)
SELECT 1, 'AVAILABLE'
FROM generate_series(1, 5);

UPDATE events
SET capacity = 10
WHERE id = 1;

COMMIT;
```

### Voorbeeld: ticketstatus bekijken

Mogelijke ticketstatussen:

- `AVAILABLE`
- `RESERVED`
- `PAID`
- `CANCELLED`
- `EXPIRED`
- `CHECKED_IN`

Tickets per event tellen:

```sql
SELECT event_id, status, COUNT(*)
FROM tickets
GROUP BY event_id, status
ORDER BY event_id, status;
```

### Stap 5: psql afsluiten

```sql
\q
```

Daarna de VM verlaten:

```bash
exit
```

## 5. Wat moet je niet manueel aanpassen?

Pas niet zomaar deze dingen manueel aan op de VM's:

- `/etc/ticketing-backend.env`
- nginx configuratie
- systemd services
- databasewachtwoorden
- JWT secrets
- PostgreSQL configuratiebestanden

Die worden door Ansible beheerd. Als je ze manueel wijzigt, kan Ansible ze later opnieuw overschrijven.

Als je structureel iets wil veranderen, pas dan de Ansible files of applicatiecode aan en run daarna opnieuw het playbook.

## 6. Wat doet mijn website?

Jouw website is een ticketreservatie-systeem.

Er zijn twee soorten gebruikers:

- `organizer`
- `attendee`

Een organizer kan events aanmaken. Een attendee kan tickets reserveren en betalen.

### Belangrijkste workflow

1. Een gebruiker registreert of logt in.
2. Een organizer maakt een event aan.
3. Bij het aanmaken van een event maakt de backend automatisch tickets aan.
4. Een attendee bekijkt de events.
5. Een attendee reserveert een ticket.
6. Het ticket krijgt status `RESERVED`.
7. De attendee kan betalen.
8. Na betaling krijgt het ticket status `PAID`.

Als de attendee niet op tijd betaalt, kan de reservatie vervallen en wordt het ticket opnieuw beschikbaar.

## 7. Hoe werken de drie lagen samen?

### Frontend

De frontend is wat je in de browser ziet.

Voorbeelden van pagina's:

- homepagina
- loginpagina
- eventpagina
- mijn tickets
- mijn events
- nieuw event

De frontend doet zelf geen databasewerk. De frontend stuurt aanvragen naar de backend.

### Backend

De backend is de API.

Die controleert regels zoals:

- alleen organizers mogen events maken
- alleen attendees mogen tickets reserveren
- een organizer mag geen ticket reserveren voor zijn eigen event
- een ticket kan niet betaald worden als het niet gereserveerd is
- een betaald ticket kan niet zomaar geannuleerd worden
- er mag maar een beperkt aantal tickets per event zijn

### Storage

De storage tier is PostgreSQL.

Daarin staat de echte data:

- gebruikers
- events
- tickets
- reservatiestatussen

Als je de website opnieuw deployt met Ansible, blijft de data normaal bestaan. De database wordt niet zomaar gewist bij een gewone redeploy.

## 8. Handige controlecommando's

Frontend controleren:

```bash
curl -i http://localhost:8080/health
```

Backend controleren via frontend reverse proxy:

```bash
curl -i http://localhost:8080/api/health
```

Backend rechtstreeks op de backend VM controleren:

```bash
cd /home/lahcen/2526-pe2-lahcen225/local-deploy
vagrant ssh backend -c "curl -i http://127.0.0.1:3000/api/health"
```

Database controleren:

```bash
cd /home/lahcen/2526-pe2-lahcen225/local-deploy
vagrant ssh storage
sudo -iu postgres psql ticketing -c "SELECT 1 AS ok;"
```

## 9. Samenvatting

Ansible zorgt ervoor dat de drie machines automatisch juist worden ingesteld.

De website bestaat uit:

- frontend: browser en nginx
- backend: Node.js API met alle applicatielogica
- storage: PostgreSQL database

Manuele database-aanpassingen doe je alleen op de storage VM via `psql`, liefst met `BEGIN`, controle met `SELECT`, en daarna `COMMIT` of `ROLLBACK`.

Voor echte projectwijzigingen is Ansible de juiste plaats, niet manueel aanpassen op de VM.
