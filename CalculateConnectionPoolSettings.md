
# Berekenen van connection pool settings

  #### Wanneer `Maximum Pool Size` niet ingesteld wordt zullen database operaties bij piekgebruik van de toepassing niet gequeued worden en zal de toepassing meer connecties proberen te openen dan mogelijk.

### Maximum aantal toegelaten connecties
Het aantal database connecties is beperkt. Standaard is dit momenteel 100, maar kan op vraag verhoogd worden.
Ook het maximum aantal connecties op serverniveau is beperkt, en reservatie van niet gebruikte connecties kost resources. 

Zodra je meer connecties probeert te maken dan toegelaten krijg je een fout 
```sh
Npgsql.PostgresException: 53300: remaining connection slots are reserved for non-replication superuser connection
```
`3 connecties worden gereserveerd voor de superuser` - indien max connections op 100 staat, zijn er dus maar 97 connecties beschikbaar voor je toepassing.

Je kan de huidige limiet op een postgreSQL database opvragen met 
```sh
SHOW max_connections;
```

### Connectionpool limieten
De connectionpool zorgt voor hergebruik van connections. Zo moet er niet telkens opnieuw een verbinding opgezet worden met database, wat kostbare tijd kost.
In die pool zitten een aantal connecties; is er geen connectie beschikbaar beschikbaar voor uitvoering van een query en heeft de connection pool zijn maximum aantal connecties bereikt, dan zal de uitvoering gequeued worden. Is er geen connectie beschikbaar binnen een aanvaardbare tijd, dan krijg je volgende fout :

```sh
Npgsql.NpgsqlException (0x80004005): The connection pool has been exhausted, either raise MaxPoolSize (currently 124) or Timeout (currently 15 seconds)
```
Je kan dus ofwel het aantal connecties - `Maximum Pool Size` - in de pool omhoog doen, ofwel de timeout verhogen.

Een connectie wordt door de pool vrijgegeven nadat deze de `Connection Idle Lifetime` heeft overschreden.
Je kan ook `Minimum Pool Size` instellen, zodat er sowieso een aantal connecties proactief worden opgezet. Verder kan je ook instellen met hoeveel connecties de pool moet uitgebreid worden wanneer nieuwe connecties nodig zijn.

### Connectionpool Maximum Pool Size berekenen
Als de limiet van maximum connecties van de connection pool hoger staat dan die van de postgreSQL database, dan kan de applicatie dus ook proberen om meer connecties op te zetten dan toegelaten door de database.

#### Daarom moet de som van maximum aantal connecties (+3 van de superusers) in alle connectionpools naar eenzelfde database steeds kleiner zijn dan de database limiet.

Verder is het ook belangrijk te weten dat ```2 connectionstrings in eenzelfde toepassing een connectionpool delen als ze exact hetzelfde zijn```. Elke afwijking zorgt ervoor dat er een aparte pool wordt gebruikt.
Dit kan dienen om maximum toegelaten connecties door bv. Hangfire of Database migratie te scheiden van de connecties van je toepassing. Zo ben je zeker dat door een probleem in hangfire niet alle connecties van je toepassing worden verbruikt, en zo ben je zeker dat bij startup van de toepassing er 1 connectie effectief beschikbaar is voor de automatische database migratie.
Elke instantie van de toepassing zal ook een eigen connection pool hebben. 

Het aantal beschikbare connecties voor de toepassing wordt (en dus de waarde van  `Maximum Pool Size` ) : 
 ```
 Maximum Pool Size  = (PostgreSQL maxconnections - #superuserconnections) / #runningpods
 ```
 Dus met default van max 100 connecties, 3 superuserconnecties en 2 running pods : (100-3)/2 = 48 
 Indien je aparte connectionpools hebt voor een scheduler & databasemigratie :
  (PostgreSQL maxconnections - #superuserconnections - (#schedulerconnections * #runningpods) - (#dbmigrationconnections * #runningpods)) / #runningpods.
  Met bv 3 connecties voorzien voor de scheduler (hangfire) en 1 voor migratie :
  (100 -3 - (3 * 2) - (1 * 2))/2 = 44
  

 
 


