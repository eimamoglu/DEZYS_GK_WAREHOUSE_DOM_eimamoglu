# Middleware Engineering "Document Oriented Middleware using MongoDB" - Taskdescription

- Name: Elyesa Imamoglu
- Klasse: 4CHIT
- Datum: 2026-03-17

## Einführung

Diese Übung soll helfen die Funktionsweise und Einsatzmöglichkeiten eines dokumentenorientierten dezentralen Systems mit Hilfe des Frameworks Spring Data MongoDB oder einem Framework Ihrer Wahl zu demonstrieren. Die Daten werden in dieser Übung in einem NoSQL Repository gespeichert und verarbeitet.

Es handelt sich um ein Lagerstandort Beispiel, wie in Aufgabe "GK8.1 Spring Data and ORM". Die Daten aller Lagerstandorte sollen in der Zentrale persistiert und in einer NoSQL Datenbank gespeichert werden. Von hier aus koennen die Daten fuer verschiedene Fragestellungen des Betriebes (Management, Einkauf, Vertrieb,...) abgefragt werden.

## 2. Implementierung und Code-Dokumentation

### 2.1 Das Datenmodell (Model)
Die Klasse `ProductData` definiert die Struktur der Dokumente in der MongoDB. Durch die Annotation `@Id` wird das Feld `ID` als Primärschlüssel für MongoDB markiert.

```java
public class ProductData {
    @Id
    private String ID; // Interner MongoDB Key

    private String warehouseID;      // Identifikator des Lagerstandorts
    private String productID;        // Fachliche Artikelnummer
    private String productName;
    private String productCategory;
    private double productQuantity;
    
    // Getter und Setter ermöglichen den Zugriff durch das Framework
}
```

### 2.2 Datenzugriffsschicht (Repository)
Das Interface `WarehouseRepository` erbt von `MongoRepository`. Dies ermöglicht grundlegende CRUD-Operationen ohne manuellen Code. Zusätzlich wurden spezifische Suchmethoden definiert:

```java
public interface WarehouseRepository extends MongoRepository<ProductData, String> {
    // Findet ein einzelnes Produkt anhand der fachlichen ID
    public ProductData findByProductID(String productID);
    
    // Findet alle Produkte, die einem bestimmten Lager zugeordnet sind
    public List<ProductData> findByWarehouseID(String warehouseID);
}
```

### 2.3 Die REST-Schnittstelle (Controller)
Der `WarehouseController` bildet die API ab. Er verarbeitet HTTP-Anfragen und kommuniziert mit dem Repository.

**Wichtige Endpoints:**
* **POST `/api/product`**: Speichert ein neues Produkt.
* **GET `/api/warehouse/{id}`**: Liefert alle Produkte eines spezifischen Lagers.
* **DELETE `/api/product/{id}`**: Löscht ein Produkt anhand der ID.

```java
@RestController
@RequestMapping("/api") // Basis-Pfad für die API
public class WarehouseController {
    @Autowired
    private WarehouseRepository repository;

    // Implementierung der Abfrage für einen bestimmten Lagerstandort (EK-Anforderung)
    @GetMapping("/warehouse/{id}")
    public List<ProductData> getWarehouseById(@PathVariable String id) {
        return repository.findByWarehouseID(id);
    }

    // Speichern eines Produkts über das REST-Interface
    @PostMapping("/product")
    public ProductData addProduct(@RequestBody ProductData product) {
        return repository.save(product);
    }
}
```

### 2.4 Automatisierte Testdaten-Generierung (Data Generator)
Für die **Vertiefung** wurde eine separate Applikation erstellt, die über die REST-Schnittstelle der Middleware 300 Datensätze generiert. Dies simuliert ein externes System, das Daten an die Zentrale liefert.

```java
public class DataGenerator {
    public static void main(String[] args) {
        RestTemplate restTemplate = new RestTemplate();
        String url = "http://localhost:8080/api/product";
        
        // Loop zur Erzeugung von 300 Produkten verteilt auf 5 Lager
        for (int i = 1; i <= 300; i++) {
            int warehouseId = rand.nextInt(5) + 1; 
            ProductData p = new ProductData(
                String.valueOf(warehouseId), "PROD-" + i, ...
            );
            // Senden des Objekts als JSON per POST an die API
            restTemplate.postForObject(url, p, ProductData.class);
        }
    }
}
```

### 2.5 Initialisierung (CommandLineRunner)
In der Hauptklasse `Application` wurde die `run`-Methode genutzt, um beim Start der Applikation die Datenbank zu bereinigen und die initialen 10 Basis-Produkte für die **Grundlagen** zu laden.

```java
@Override
public void run(String... args) throws Exception {
   repository.deleteAll(); // Sicherstellung einer sauberen Testumgebung

   // Manuelles Seeding der 10 Basis-Produkte (3 Kategorien)
   repository.save(new ProductData("1","00-101","Orangensaft","Getraenk", 2500));
   // ... weitere 9 Produkte
   System.out.println("--- Grundlagen Daten geladen ---");
}
```

### 2.6 Konfiguration der Middleware (`application.properties`)
Damit die Spring Boot Applikation mit der gesicherten MongoDB kommunizieren kann, ist die Konfiguration der URI entscheidend. Hier wird der Root-User und die `authSource` definiert.

```properties
# Verbindung zur MongoDB im Docker
spring.data.mongodb.host=localhost
spring.data.mongodb.port=27017
spring.data.mongodb.database=warehouse_db
# ALTE ZEILEN (host/port) KÖNNEN RAUS, wenn URI genutzt wird
spring.data.mongodb.uri=mongodb://admin:password123@localhost:27017/warehouse_db?authSource=admin
```

### 2.7 Infrastruktur als Code (`docker-compose.yml`)
Die Datenbank wird als isolierter Container bereitgestellt. Die Umgebungsvariablen erzwingen die Erstellung eines Administrators beim ersten Start.

```yaml
services:
  mongodb:
    image: mongo:latest
    container_name: mongo-warehouse
    ports:
      - "27017:27017"
    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: password123
    volumes:
      - mongodb_data:/data/db
    restart: always

volumes:
  mongodb_data:
```

---

## Vertiefung Fragen

### Fragestellung 1: Bestandsanalyse pro Lagerstandort
**Ziel:** Die Zentrale muss wissen, wie viele Produkte insgesamt an jedem der 5 Standorte lagern, um die Logistik zu planen.

* **Logik:** Gruppierung aller Dokumente nach der `warehouseID` und Summierung der `productQuantity`.
* **Mongo Shell Befehl:**
```javascript
db.productData.aggregate([
  { 
    $group: { 
      _id: "$warehouseID", 
      Gesamtbestand: { $sum: "$productQuantity" },
      Anzahl_Artikel: { $sum: 1 } 
    } 
  },
  { $sort: { _id: 1 } }
])
```

---

### Fragestellung 2: Kritischer Lagerbestand (Low Stock Alert)
**Ziel:** Welche Produkte haben über **alle** Standorte hinweg einen Bestand von weniger als 10 Einheiten? (Wichtig für den Zentraleinkauf zur Nachbestellung).

* **Logik:** Filterung aller Produkte, deren Menge unter 10 liegt.
* **Mongo Shell Befehl:**
```javascript
db.productData.find(
  { productQuantity: { $lt: 10 } },
  { productName: 1, warehouseID: 1, productQuantity: 1, _id: 0 }
).sort({ productQuantity: 1 })
```

---

### Fragestellung 3: Kategorien-Verteilung (Sortimentsübersicht)
**Ziel:** Wie verteilt sich die Anzahl der Produkte auf die 6 verschiedenen Kategorien? (Analyse des Warensortiments).

* **Logik:** Gruppierung nach `productCategory` und Zählen der Einträge.
* **Mongo Shell Befehl:**
```javascript
db.productData.aggregate([
  { 
    $group: { 
      _id: "$productCategory", 
      Anzahl_Produkte: { $sum: 1 } 
    } 
  },
  { $sort: { Anzahl_Produkte: -1 } }
])
```

---

## Demo Applikation

* Download Docker for MongoDB  
  `docker pull mongo`  

* Run Docker for MongoDB (using port 27017, name mongo)  
  `docker run -d -p 27017:27017 --name mongo mongo`  

* Run MongoShell on Docker Instance  
  `docker exec -it mongo bash`  
  `mongosh`  

* Execute MongoShell Commands    
  `show dbs`  
  `use local`   
  `db.startup_log.countDocuments();`    

* Accessing Data with MongoDB and Spring  
  - Build and Run Example  
	  `gradle clean bootRun`  

  - Check Data in MongoDB.  
    `docker exec -it mongo bash`
    `mongosh`
    `use test`
    `db.warehouseData.find()`  

## Fragestellung für Protokoll

+ Nennen Sie 4 Vorteile eines NoSQL Repository im Gegensatz zu einem relationalen DBMS

Flexibilität (Schemalosigkeit): Datenstrukturen können jederzeit ohne aufwendige Tabellen-Migrationen (ALTER TABLE) angepasst werden.

Skalierbarkeit: NoSQL-Systeme sind für horizontales Skalieren ausgelegt (Verteilung auf viele günstige Server statt eines großen High-End-Servers).

Performance: Sehr hohe Schreibgeschwindigkeiten bei großen Datenmengen, da keine komplexen Relationen und Joins berechnet werden müssen.

Umgang mit unstrukturierten Daten: Ideal für moderne Datenformate wie JSON, Dokumente oder Graphen, die nicht in starre Spalten/Zeilen passen.

+ Nennen Sie 4 Nachteile eines NoSQL Repository im Gegensatz zu einem relationalen DBMS

Keine Standardisierung: Es gibt kein universelles SQL; jede NoSQL-Datenbank (MongoDB, Cassandra, Neo4j) hat eine eigene Abfragesprache.

Fehlende ACID-Garantie: Viele NoSQL-Systeme opfern die strikte Konsistenz für Geschwindigkeit ("Eventual Consistency").

Datenredundanz: Um Performance zu gewinnen, werden Daten oft dupliziert (Denormalisierung) statt über Fremdschlüssel referenziert.

Komplexität bei Analysen: Komplexe Abfragen, die in SQL einfache Joins wären, müssen oft mühsam in der Applikationslogik oder über Aggregation-Pipelines gelöst werden.

+ Welche Schwierigkeiten ergeben sich bei der Zusammenführung der Daten?

Inkonsistente Datentypen: Da kein Schema erzwungen wird, kann das Feld productQuantity in einem Dokument ein String und im anderen ein Integer sein.

ID-Konflikte: Bei der Fusionierung mehrerer Lager müssen globale Eindeutigkeiten der Primärschlüssel sichergestellt werden.

Fehlende Integrität: Es gibt keine eingebauten Fremdschlüssel-Checks, die verhindern, dass verwaiste Datensätze entstehen.

+ Welche Arten von NoSQL Datenbanken gibt es? & Nennen Sie einen Vertreter für jede Art?

Es gibt vier Hauptarten von NoSQL-Datenbanken:
1. Dokumentenorientierte Datenbanken (z.B. MongoDB)
2. Key-Value Stores (z.B. Redis)
3. Spaltenorientierte Datenbanken (z.B. Cassandra)
4. Graphdatenbanken (z.B. Neo4j, ArangoDB)

+ Beschreiben Sie die Abkürzungen CA, CP und AP in Bezug auf das CAP Theorem

+ Das CAP-Theorem besagt, dass ein verteiltes System nur zwei der drei folgenden Eigenschaften gleichzeitig garantieren kann:
- Consistency (C): Alle Knoten sehen die gleichen Daten zur gleichen Zeit. Es gibt keine Inkonsistenzen.
- Availability (A): Jeder Anfrage erhält eine Antwort, auch wenn einige Knoten ausgefallen sind. Das System bleibt erreichbar.
- Partition Tolerance (P): Das System funktioniert weiter, auch wenn es Netzwerkpartitionen gibt, d.h. Teile des Systems nicht miteinander kommunizieren können.

Kombinationen:
- CA: Fokus auf Konsistenz und Verfügbarkeit (klassische SQL-Datenbanken ohne Sharding).
- CP: Fokus auf Konsistenz und Fehlertoleranz (z. B. MongoDB im Standard).
- AP: Fokus auf Verfügbarkeit und Fehlertoleranz (z. B. Cassandra).

+ Mit welchem Befehl koennen Sie den Lagerstand eines Produktes aller Lagerstandorte anzeigen.

```
db.productData.find({ "productID": "PROD-100" })
```

+ Mit welchem Befehl koennen Sie den Lagerstand eines Produktes eines bestimmten Lagerstandortes anzeigen.

````
db.productData.find({ "warehouseID": "3", "productID": "PROD-100" })
````


## Links und Dokumente
* [Was bedeutet NoSQL](https://www.oracle.com/at/database/nosql/what-is-nosql)
* [Accessing Data with MongoDB](https://spring.io/guides/gs/accessing-data-mongodb/)
* [MongoDB Installation](https://docs.mongodb.com/manual/administration/install-community/)
* [mongo Shell Quick Reference](https://docs.mongodb.com/manual/reference/mongo-shell/)
* [mongo Shell Query Reference](https://www.mongodb.com/docs/manual/tutorial/query-embedded-documents/)
* [Grundlagen Spring Framework](https://spring.io/)
* [Spring Boot](https://spring.io/guides/gs/spring-boot/)
* [Spring Data MongoDB](https://spring.io/projects/spring-data-mongodb)
* [Spring RESTful Web Service](https://spring.io/guides/gs/rest-service/#use-maven)
* NoSQL Introduction
  - [NoSQL on w3resource](https://www.w3resource.com/mongodb/nosql.php)  
  - [Introduction to NoSQL Database](https://www.edureka.co/blog/introduction-to-nosql-database/)  
  - [NoSQL im Überblick](https://www.heise.de/ct/artikel/NoSQL-im-Ueberblick-1012483.html)  
  - [Introduction to NoSQL Databases on YouTube ](https://www.youtube.com/watch?v=2yQ9TGFpDuM)  


## Mongo Shell Abfragen  
  
Link to [Mongo Shell Query and Projection Operators](https://docs.mongodb.com/manual/reference/operator/query/)

Den Demo-Abfragen liegt folgende Datenstruktur zu Grunde:   
   `{  `  
   `    warehouseID: '1',   `   
   `    warehouseName: 'Linz Bahnhof',   `   
   `   timestamp: '2022-01-02 01:00:00',   `   
   `    warehousePostalCode: 4010,`    
   `   warehouseCity: 'Linz',`   
   `   warehouseCountrz: 'Austria',`   
   `   productData: [`  
   `      { productID: '00-443175', productName: 'Bio Orangensaft Sonne', productQuantity: 2500 },`    
   `      { productID: '00-871895', productName: 'Bio Apfelsaft Gold', productQuantity: 3420 },`    
   `      { productID: '01-926885', productName: 'Ariel Waschmittel Color', productQuantity: 478 },`     
   `   ]`   
    `}`
  
* Filtern nach dem Lagerstandort 1    
`db.productData.find( { 
	"warehouseID": "1"
} )`


* Filtern nach Lagerstandort 1 und dem Produkt mit dem Namen "Bio Apfelsaft Gold"  
`db.productData.find( { 
	"warehouseID": "1",
        "productName": "Bio Apfelsaft Gold"
} )`

* Filtern nach allen Produkten, die einen Lagerbestand unter 500 Stueck haben.  
`db.productData.find( { 
	"productQuantity": { $lte: 500 }
} )`

* Filtern nach Lagerstandort 1 und einem Lagerbestand unter 500 Stueck haben.  
`db.productData.find( { 
    "warehouseID": "1",
    "productQuantity": { $lte: 500 }
} )`

* Filtern nach allen Produkten der Produktkategorien.  
`db.productData.find( { 
     productCategory: { $in: [ "Waschmittel", "Getraenk" ] } 
} )`
