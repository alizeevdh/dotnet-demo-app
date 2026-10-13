# opmerkingen .net

wwaarom werk ik met een dockerfile.db

In het originele Compose-bestand staat schema.sql als bind mount:

yaml
- ./TodoApp/schema.sql:/docker-entrypoint-initdb.d/schema.sql:Z,ro

Dat werkte niet vanuit Jenkins. Jenkins draait in een container, maar de Docker-daemon draait op de VM. Het pad in de Jenkins-workspace bestaat op de VM niet, dus Docker maakte er een lege map van, en MariaDB crashte (Can't read from a directory 'stdin').

schema in de image (dockerfile)

FROM mariadb:11
COPY TodoApp/schema.sql /docker-entrypoint-initdb.d/01-schema.sql

Compose bouwt image (build: in plaats van image: mariadb:11). MariaDB voert alles in /docker-entrypoint-initdb.d/ automatisch uit wanneer de database voor het eerst start met een leeg volume.

## mijn interpretatie

het schema inbakken (Dockerfile.db)
Om het pad-probleem te omzeilen, laat ik Docker het schema.sql bestand direct in een eigen MariaDB container-image stoppen via een Dockerfile.

Zodra MariaDB voor de allereerste keer opstart met een leeg geheugen, kijkt hij uit zichzelf in die speciale map (/docker-entrypoint-initdb.d/) en voert hij jouw script automatisch uit.