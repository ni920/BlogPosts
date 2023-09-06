# Zero-downtime deployments mit Docker Swarm und Portainer Teil 1/2

- [Zero-downtime deployments mit Docker Swarm und Portainer Teil 1/2](#zero-downtime-deployments-mit-docker-swarm-und-portainer-teil-12)
  - [Was sind Zero-downtime deployments?](#was-sind-zero-downtime-deployments)
  - [Rolling Updates vs Zero-downtime deployments](#rolling-updates-vs-zero-downtime-deployments)
  - [Wie funktioniert ein Zero-downtime deployment mit Docker Swarm und Portainer?](#wie-funktioniert-ein-zero-downtime-deployment-mit-docker-swarm-und-portainer)
    - [Healthcheck](#healthcheck)
    - [Zero-downtime deployment](#zero-downtime-deployment)


![](ZeroDowntimeDeployments.png)

## Was sind Zero-downtime deployments?

Bevor wir ans eingemachte gehen klären wir einmal die Frage was denn Zero-downtime deployments sind. Zero-downtime deployments sind Deployments die keine Ausfallzeit haben. Das bedeutet das die Anwendung die gerade deployed wird nicht offline ist. Das ist besonders wichtig bei Anwendungen die 24/7 laufen müssen.
Die hohe Verfügbarkeit von Anwendungen rückt immer mehr in den Fokus.
Ich mein wer will schon warten müssen wenn Netflix und co. ihre Anwendungen updaten.

 


## Rolling Updates vs Zero-downtime deployments

Beide Ansätze haben das gleiche Ziel, jedoch gibt es wesentliche Unterschiede.
Deswegen schauen wir uns einmal beide Ansätze anhand einer Tabelle an.

| Rolling Updates    |                                                                                                                                      |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------------------ |
| Arbeitsweise       | Neue Versionen werden schrittweise in den Produktionsbetrieb eingeführt, indem einzelne Instanzen schrittweise aktualisiert werden.  |
| Zeitliche Abstände | Die Aktualisierung erfordert eine gewisse Zeit, da sie schrittweise auf einzelnen Instanzen durchgeführt wird.                       |
| Ausfallzeiten      | Kurzzeitige Ausfallzeiten sind möglich, da bei jedem Schritt eine einzelne Instanz gestoppt und durch die neue Version ersetzt wird. |
| Versionierung      | Während der Aktualisierung können kurze Zeit verschiedene Versionen der Anwendung parallel laufen.                                   |




| Zero-downtime deployments |                                                                                                                                                                                                                                          |
| ------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Arbeitsweise              | Zero Downtime Deployments ermöglichen es, eine neue Version der Anwendung in Produktion einzuführen, ohne dass es zu Ausfallzeiten für die Benutzer kommt.                                                                               |
| Zeitliche Abstände        | Zero Downtime Deployments streben eine nahtlose Aktualisierung an, ohne dass es zu Zeitverzögerungen oder Unterbrechungen im Betrieb kommt.                                                                                              |
| Ausfallzeiten             | Der Hauptunterschied zu Rolling Updates besteht darin, dass Zero Downtime Deployments keine Ausfallzeiten haben. Die Anwendung bleibt kontinuierlich verfügbar, auch während der Aktualisierung.                                         |
| Versionierung             | Bei Zero Downtime Deployments werden die neuen Versionen der Anwendung schrittweise in Produktion gebracht, und erst wenn die neue Version als vollständig funktionsfähig und stabil bewertet wurde, wird die alte Version abgeschaltet. |





## Wie funktioniert ein Zero-downtime deployment mit Docker Swarm und Portainer?

Genug Theorie. Wie setze ich jetzt ein Zero-downtime deployment mit Docker Swarm und Portainer um?

Folgende Dinge werden Vorausgesetzt:

- Docker Swarm enabled
- Repository indem das Projekt liegt mit einer Stack Yaml Datei
- Portainer

Da ich bereits in dem Artikel [Warum Portainer](https://www.ayedo.de/posts/warum-man-portainer-portainer-ansteller-der-konsole-nutzen-sollte/) gezeigt habe wie man eine Basic YAML Datei über Portainer ausrollt gehe ich hier nicht weiter darauf ein.

Der Fokus liegt viel eher auf den Befehlen die mir Docker Anbietet um ein Zero-downtime deployment zu erreichen. 
In Teil 2 liegt der Fokus eher auf der Verbindung zwischen Portainer und dem Repository.

Fangen wir an mit der YAML Datei. Diese sieht wie folgt aus:

```yaml
version: '3.8'
services:
  db:
    image: mariadb:10.6.4-focal
    command: '--default-authentication-plugin=mysql_native_password'
    volumes:
      - db_data:/var/lib/mysql
    restart: always
    environment:
      - MYSQL_ROOT_PASSWORD=somewordpress
      - MYSQL_DATABASE=wordpress
      - MYSQL_USER=wordpress
      - MYSQL_PASSWORD=wordpress
    expose:
      - 3306
      - 33060
  wordpress:
    image: wordpress:latest
    volumes:
      - wp_data:/var/www/html
    ports:
      - 7002:80
    restart: always
    environment:
      - WORDPRESS_DB_HOST=db
      - WORDPRESS_DB_USER=wordpress
      - WORDPRESS_DB_PASSWORD=wordpress
      - WORDPRESS_DB_NAME=wordpress
volumes:
  db_data:
  wp_data:
```

Soweit nichts besonderes. Wir haben eine Wordpress Instanz mit einer Datenbank.
Wenn aktuell eine neue Version von Wordpress rauskommt muss  die Anwendung offline gehen um die neue Version auszurollen. Dies ist natürlich nicht optimal.

### Healthcheck

Der nächste Schritt ist deshalb einen Docker Healthcheck zu integrieren der überprüft ob eine Anwendung richtig läuft. 
Es würde prinzipiell auch funktionieren ohne einen Healthcheck, jedoch gibt container die selbst dann als laufend angezeigt werden wenn sie intern nicht mehr laufen.

```yaml
healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost"] # Testet einfach ob die Anwendung über Port 80 erreichbar ist
      interval: 30s
      timeout: 10s
      retries: 5
```

Hiernach können wir in Portainer überprüfen ob unsere Anwendung ```healthy``` ist.

![](ServiceHealthy.png)

Das sieht schonmal sehr gut aus.

Ich gehe hier nicht weiter auf den Healthcheck ein. Wer mehr darüber wissen will kann sich gerne die [Docker Manuals](https://docs.docker.com/compose/compose-file/05-services/#healthcheck) durchlesen.

###  Zero-downtime deployment

Nun da wir den Healthcheck haben können wir uns dem eigentlichen Zero-downtime deployment widmen.
Hierfür müssen wir unsere YAML Datei um ein paar Zeilen erweitern.

```yaml
deploy:
  replicas: 2 # Anzahl der Instanzen die gleichzeitig laufen sollen
  update_config: # Die Konfiguration für das Update
    order: start-first  # Die Reihenfolge in der die Instanzen geupdated werden sollen    
    failure_action: rollback # Was soll passieren wenn ein Update fehlschlägt

  rollback_config: # Die Konfiguration für das Rollback
    parallelism: 1 # Wie viele Instanzen sollen gleichzeitig zurückgerollt werden
    order: start-first # Die Reihenfolge in der die Instanzen zurückgerollt werden sollen

  restart_policy: # Die Konfiguration für den Restart
    condition: on-failure # Unter welchen Bedingungen soll ein Restart durchgeführt werden
```

Nun haben wir alles was wir brauchen um ein Zero-downtime deployment durchzuführen.
Den kompletten Stack findet ihr [hier](https://github.com/ni920/BlogExamples/blob/main/docker-compose.yml)

>Warum wir zwei Instanzen gleichzeitig laufen lassen? 
>
>Ganz einfach. Wenn wir nur eine Instanz laufen lassen und diese Instanz fehlschlägt haben wir keine Instanz mehr die die Anwendung bereitstellt. Deswegen lassen wir zwei Instanzen gleichzeitig laufen. Sollte eine Instanz fehlschlagen haben wir immer noch eine zweite Instanz die die Anwendung bereitstellt.
>In einem Swarm Cluster kann Docker so auch die Container über mehrere Nodes verteilen.
>
>Dadurch können wir eine hohe Verfügbarkeit der Anwendung erreichen.


Sollten wir nun unsere WordPress Instanz updaten wollen können wir dies ganz einfach über Portainer machen. 
Dies werde ich euch aber in Teil 2 dieses Beitrags zeigen, da es hier den Rahmen sprengen würde.



Quellen:
* [Portainer.io](https://www.portainer.io/)
* [Thumbnail (wurde stark bearbeitet)](https://unsplash.com/de/fotos/XbxothjMRL0)
* [Wordpress Compose](https://github.com/docker/awesome-compose/blob/master/official-documentation-samples/wordpress/README.md)

