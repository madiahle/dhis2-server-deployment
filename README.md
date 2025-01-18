# dhis2-server-deployment

#!/bin/bash

# Arrêter le script en cas d'erreur
set -e

echo "=== Déploiement automatique de DHIS2 ==="

# Mise à jour et installation des outils nécessaires
echo "=== Mise à jour du système et installation des outils ==="
sudo apt-get update -y && sudo apt-get upgrade -y
sudo apt-get install -y net-tools openssh-server vim curl unzip openjdk-17-jdk

# Configuration du fuseau horaire
echo "=== Configuration du fuseau horaire ==="
sudo dpkg-reconfigure tzdata

# Installation et configuration de PostgreSQL avec PostGIS
echo "=== Installation de PostgreSQL et PostGIS ==="
sudo sh -c 'echo "deb https://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
curl -fsSL https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/postgresql.gpg
sudo apt-get update -y
sudo apt-get install -y postgresql-16 postgresql-16-postgis-3

echo "=== Démarrage et activation de PostgreSQL ==="
sudo systemctl start postgresql
sudo systemctl enable postgresql

echo "=== Création de l'utilisateur et de la base de données DHIS2 ==="
sudo -u postgres createuser -SDRP dhis
sudo -u postgres createdb -O dhis dhis
sudo -u postgres psql -c "create extension postgis;" dhis
sudo -u postgres psql -c "create extension btree_gin;" dhis
sudo -u postgres psql -c "create extension pg_trgm;" dhis

# Création de l'utilisateur DHIS2
echo "=== Création de l'utilisateur système DHIS2 ==="
sudo useradd -d /home/dhis -m dhis -s /bin/false
echo "Veuillez définir le mot de passe pour l'utilisateur dhis :"
sudo passwd dhis
sudo -u dhis mkdir -p /home/dhis/config
sudo -u dhis nano /home/dhis/config/dhis.conf


echo "# ----------------------------------------------------------------------
# Database connection
# ----------------------------------------------------------------------
# JDBC driver class
connection.driver_class = org.postgresql.Driver
# Database connection URL
connection.url = jdbc:postgresql:dhis
# Database username
connection.username = dhis
# Database password
connection.password = dhis" > dhis.conf


# Téléchargement et configuration de Tomcat
echo "=== Téléchargement et configuration de Tomcat ==="
wget https://dlcdn.apache.org/tomcat/tomcat-9/v9.0.98/bin/apache-tomcat-9.0.98.zip -P /home/dhis/
sudo -u dhis unzip /home/dhis/apache-tomcat-9.0.98.zip -d /home/dhis/
sudo -u dhis mv /home/dhis/apache-tomcat-9.0.98 /home/dhis/tomcat-dhis
sudo chown -R dhis:dhis /home/dhis/tomcat-dhis/

# Configuration des variables d'environnement pour Tomcat
echo "=== Configuration des variables d'environnement pour Tomcat ==="
sudo -u dhis bash -c 'cat > /home/dhis/tomcat-dhis/bin/setenv.sh <<EOF
export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64
export DHIS2_HOME=/home/dhis/config
export CATALINA_OPTS="-Xmx2G -Xms1G -XX:+UseG1GC -Dfile.encoding=UTF-8"
EOF'
sudo chmod +x /home/dhis/tomcat-dhis/bin/setenv.sh


#Edit this /home/dhis/tomcat-dhis/conf/server.xml to ensure having below content 

<Connector port="8080" protocol="HTTP/1.1"
  connectionTimeout="20000"
  redirectPort="8443"
  relaxedQueryChars="[]" />


# Téléchargement et déploiement de DHIS2
echo "=== Téléchargement et déploiement de DHIS2 ==="
wget https://releases.dhis2.org/41/dhis2-stable-41.2.0.war -P /home/dhis/
sudo -u dhis mv /home/dhis/dhis2-stable-41.2.0.war /home/dhis/tomcat-dhis/webapps/ROOT.war

# Démarrage de Tomcat
echo "=== Démarrage de Tomcat ==="
sudo -u dhis /home/dhis/tomcat-dhis/bin/startup.sh

echo "=== Déploiement terminé ! ==="
echo "Accédez à DHIS2 à l'adresse suivante : http://<votre_ip>:8080"
