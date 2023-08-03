# Zero-downtime deployments mit Docker Swarm und Portainer

- [Zero-downtime deployments mit Docker Swarm und Portainer](#zero-downtime-deployments-mit-docker-swarm-und-portainer)
  - [Was sind Zero-downtime deployments?](#was-sind-zero-downtime-deployments)
  - [Rolling Updates vs Zero-downtime deployments](#rolling-updates-vs-zero-downtime-deployments)
  - [Wie funktioniert ein Zero-downtime deployment mit Docker Swarm und Portainer?](#wie-funktioniert-ein-zero-downtime-deployment-mit-docker-swarm-und-portainer)




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
|                    |                                                                                                                                      |



| Zero-downtime deployments |                                                                                                                                                                                                                                          |
| ------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Arbeitsweise              | Zero Downtime Deployments ermöglichen es, eine neue Version der Anwendung in Produktion einzuführen, ohne dass es zu Ausfallzeiten für die Benutzer kommt.                                                                               |
| Zeitliche Abstände        | Zero Downtime Deployments streben eine nahtlose Aktualisierung an, ohne dass es zu Zeitverzögerungen oder Unterbrechungen im Betrieb kommt.                                                                                              |
| Ausfallzeiten             | Der Hauptunterschied zu Rolling Updates besteht darin, dass Zero Downtime Deployments keine Ausfallzeiten haben. Die Anwendung bleibt kontinuierlich verfügbar, auch während der Aktualisierung.                                         |
| Versionierung             | Bei Zero Downtime Deployments werden die neuen Versionen der Anwendung schrittweise in Produktion gebracht, und erst wenn die neue Version als vollständig funktionsfähig und stabil bewertet wurde, wird die alte Version abgeschaltet. |
|                           |                                                                                                                                                                                                                                          |




## Wie funktioniert ein Zero-downtime deployment mit Docker Swarm und Portainer?

Genug Theorie. Wie setze ich jetzt ein Zero-downtime deployment mit Docker Swarm und Portainer um?

Folgende Dinge werden Vorrausgesetzt:

- Docker Swarm Cluster
- Repository indem das Projekt liegt mit einer Stack Yaml Datei
- Portainer

Da ich bereits in dem Artikel [Warum Portainer](https://www.ayedo.de/posts/warum-man-portainer-portainer-ansteller-der-konsole-nutzen-sollte/) gezeigt habe wie man eine Basic YAML Datei über Portainer ausrollt gehe ich hier nicht weiter darauf ein.

Der Fokus liegt viel eher auf den Befehlen die mir Docker Anbietet um ein Zero-downtime deployment zu erreichen und auf der verbindung zwischen Portainer und dem Repository.

Fangen wir an mit der YAML Datei. Diese sieht wie folgt aus:

```yaml
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
      - 80:80
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
Sagen wir nun wir haben eine neue Version von Wordpress und wollen diese ausrollen ohne das die Anwendung offline geht.

Hierfür können in der YAML Datei zwei Parameter gesetzt werden.

```yaml



