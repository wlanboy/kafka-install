# kafka-install

Ansible-Playbooks für die Neuinstallation von Apache Kafka und Apache ZooKeeper auf
RedHat-VMs, als Binary-Download vom internen Nexus/Artifactory-Mirror (kein
Docker-Container), gestartet jeweils als systemd User-Unit (`systemctl --user`)
des `ansible_user` bzw. eines dedizierten Service-Users. TLS und SASL/PLAIN sind
von Anfang an aktiv.

Struktur und Vorgehen orientieren sich an [kafka-config](../kafka-config) (dort
insbesondere `kminion.yml`: Download vom Nexus-Mirror, Lingering, systemd
User-Unit). Das JAAS-/SASL-Format folgt [kafka-config/files/kafka_server_jaas.conf](../kafka-config/files/kafka_server_jaas.conf)
und [kafka_mirror/secrets/kafka_server_jaas.conf](../kafka_mirror/secrets/kafka_server_jaas.conf).

`kafka-config` überschreibt gezielt einzelne Werte in einer bereits vorhandenen
`/etc/kafka/server.properties`. Dieses Repo macht dagegen die Erstinstallation
(Binary, vollständige Config, Service). Auf einer frisch damit installierten VM
lässt sich anschließend wieder `kafka-config` für laufende Config-Änderungen
verwenden.

## Voraussetzungen

- Java ist auf den VMs bereits installiert (z. B. via RPM/Satellite/Foreman,
  nicht Teil dieser Playbooks). Pfad in `kafka_java_home` / `zookeeper_java_home`
  eintragen (`java -version` bzw. `alternatives --list` auf der VM prüfen).
- Keystore/Truststore (JKS) liegen bereits auf den VMs (Zertifikatsverteilung ist
  nicht Teil dieser Playbooks, analog zu kafka-config, wo SSL-Konfiguration
  ebenfalls unangetastet bleibt). Pfade in `kafka_ssl_*` / `zookeeper_ssl_*`
  eintragen.
- `ansible_user` auf den Ziel-VMs kann per `become`/sudo root werden (für
  Verzeichnis-Anlage, `loginctl enable-linger`, Paketverzeichnis-Rechte) und läuft
  danach auch als der Service-User selbst (`become_user`).
- `loginctl enable-linger` funktioniert (erfordert i. d. R. `systemd-logind`, auf
  RHEL Standard), nötig, damit die `--user`-Unit auch ohne aktive Login-Session
  bzw. nach einem Reboot automatisch startet.
- Passwörter (`kafka_inter_broker_password`, `kafka_jaas_users`,
  `kafka_ssl_*_password`, `zookeeper_ssl_*_password`, `zookeeper_kafka_sasl_password`)
  NIEMALS im Klartext committen. Mit `ansible-vault` verschlüsseln, z. B.:
  ```bash
  ansible-vault encrypt_string 'geheimes-passwort' --name 'kafka_inter_broker_password'
  ```
  und den Output anstelle des Klartextwerts in `inventory/<env>/group_vars/all.yml`
  einfügen (bzw. in eine separate `group_vars/vault.yml` auslagern und mit
  `--ask-vault-pass` / `--vault-password-file` ausführen).

## Installation

ZooKeeper muss zuerst installiert werden: `install-kafka.yml` berechnet den
`zookeeper.connect`-String aus der `[zookeeper]`-Inventory-Gruppe (die Gruppe muss
befüllt sein, ZooKeeper selbst muss zum Start von Kafka aber bereits laufen).

```bash
# 1. ZooKeeper-Ensemble installieren
ansible-playbook -i inventory/<env>/hosts.ini install-zookeeper.yml

# 2. Kafka-Broker installieren (verbindet sich gegen das ZooKeeper-Ensemble aus Schritt 1)
ansible-playbook -i inventory/<env>/hosts.ini install-kafka.yml
```

Beide Playbooks laufen mit `serial: 1` und `any_errors_fatal: true`. Bricht ein Host
beim Health-Check (Warten auf den Client-/Listener-Port) ab, werden keine weiteren
Hosts angefasst.

Erneutes Ausführen ist idempotent: Archiv-Download/-Entpacken, Config-Rendering und
Unit-Datei laufen bei jedem Durchlauf, der Service wird aber nur neu gestartet
(`notify`-Handler), wenn sich tatsächlich etwas geändert hat.

## Was die Playbooks tun

### `install-zookeeper.yml` (Hosts: `[zookeeper]`)

1. Prüft `zookeeper_mirror_url` (Pflicht, kein Default) und `zookeeper_id` je Host.
2. Legt Install- (`zookeeper_install_dir`, Default `/opt/zookeeper`) und
   Datenverzeichnis (`zookeeper_data_dir`) an, Owner = `zookeeper_service_user`.
3. Lädt `apache-zookeeper-<version>-bin.tar.gz` vom Nexus-Mirror direkt auf den
   Zielhost und entpackt es (`--strip-components=1`, damit der Inhalt direkt unter
   `zookeeper_install_dir` liegt, unabhängig vom Versions-Ordnernamen im Archiv).
4. Schreibt `myid` (aus der Host-Variable `zookeeper_id`) ins Datenverzeichnis.
5. Rendert `zookeeper_jaas.conf` (SASL/DIGEST-MD5-User für eingehende
   Kafka-Verbindungen, aus `zookeeper_kafka_sasl_username`/`_password`) und `zoo.cfg`
   (TLS via `secureClientPort` + Keystore/Truststore, SASL, sowie die
   `server.X=host:2888:3888`-Ensemble-Liste, automatisch aus allen Hosts der
   `[zookeeper]`-Gruppe berechnet).
6. Aktiviert Lingering und installiert `zookeeper.service` unter
   `~/.config/systemd/user/`, startet den Service über `systemctl --user`.
7. Health-Check: wartet auf den (secure) Client-Port.

### `install-kafka.yml` (Hosts: `[kafka]`)

1. Prüft `kafka_mirror_url` (Pflicht, kein Default), `kafka_broker_id` je Host und
   dass die `[zookeeper]`-Gruppe nicht leer ist.
2. Berechnet `zookeeper.connect` aus allen Hosts der `[zookeeper]`-Gruppe (Host +
   `secureClientPort`/`clientPort` je nach `zookeeper_tls_enabled`) plus Chroot
   (`kafka_zookeeper_chroot`, Default `/kafka`).
3. Legt Install- (`kafka_install_dir`, Default `/opt/kafka`) und Datenverzeichnis
   (`kafka_data_dir`, `log.dirs`) an.
4. Lädt `kafka_<scala-version>-<version>.tgz` vom Nexus-Mirror und entpackt es
   (`--strip-components=1`).
5. Rendert `kafka_server_jaas.conf` (SASL/PLAIN, `KafkaServer` mit
   Inter-Broker-User + `kafka_jaas_users`, `KafkaClient` für Kafka-Bordmittel auf dem
   Host, `Client` für die Authentifizierung des Brokers gegen ZooKeeper) und
   `server.properties` (Listener `SASL_SSL`, Keystore/Truststore, `zookeeper.connect`,
   Partitionierung/Replikation aus den `kafka_*`-Defaults).
6. Aktiviert Lingering und installiert `kafka.service` unter
   `~/.config/systemd/user/`, startet den Service über `systemctl --user`.
7. Health-Check: wartet auf den Listener-Port (`kafka_listener_port`, Default 9093).

## Wichtige Variablen (`inventory/<env>/group_vars/all.yml`)

| Variable | Bedeutung |
| --- | --- |
| `kafka_mirror_url` / `zookeeper_mirror_url` | Nexus-URL zum jeweiligen `.tgz`/`.tar.gz` (Pflicht) |
| `kafka_install_dir` / `zookeeper_install_dir` | Zielverzeichnis auf der VM (Default `/opt/kafka` bzw. `/opt/zookeeper`) |
| `kafka_java_home` / `zookeeper_java_home` | JVM-Pfad, RedHat-typisch unter `/usr/lib/jvm/...` |
| `kafka_ssl_*` / `zookeeper_ssl_*` | Keystore/Truststore-Pfade + Passwörter (Dateien müssen bereits auf der VM liegen) |
| `kafka_sasl_mechanism` | SASL-Mechanismus, Default `PLAIN` |
| `kafka_inter_broker_username`/`_password` | Inter-Broker-SASL-User (KafkaServer-Block) |
| `kafka_jaas_users` | Dict weiterer SASL-Client-User (Producer/Consumer/Monitoring) |
| `kafka_super_users` | Kafka-ACL-Superuser, Default `User:<kafka_inter_broker_username>` |
| `zookeeper_kafka_sasl_username`/`_password` | User, mit dem sich der Broker gegen ZooKeeper authentifiziert |
| `kafka_broker_id` / `zookeeper_id` | Pflicht-Host-Variable in `hosts.ini`, eindeutig je Ensemble-Mitglied |
| `kafka_zookeeper_chroot` | ZK-Chroot-Pfad für dieses Kafka-Cluster, Default `/kafka` |

Broker-/Node-IDs werden nicht in `group_vars`, sondern direkt als Host-Variable
in `inventory/<env>/hosts.ini` gepflegt (siehe dort), da sie pro Host eindeutig sein
müssen.

## Bedienung (systemctl --user)

```bash
systemctl --user status kafka.service        # bzw. zookeeper.service
journalctl --user -u kafka.service -f
systemctl --user restart kafka.service
```

Auf der VM als der jeweilige Service-User ausführen (oder per SSH mit
`XDG_RUNTIME_DIR=/run/user/<uid>` für einen anderen User).

## Troubleshooting

- Health-Check-Timeout nach dem Rollout: Unit-Logs prüfen
  (`journalctl --user -u kafka.service -n 200`). Häufigste Ursachen sind ein
  falscher `kafka_java_home`, ein falscher Keystore/Truststore-Pfad oder
  -Passwort, oder ein nicht erreichbares `zookeeper.connect` (ZooKeeper noch
  nicht installiert/gestartet).
- `loginctl enable-linger` schlägt fehl: `systemd-logind` prüfen. Ohne Lingering
  startet die `--user`-Unit nur, solange eine Login-Session des Service-Users aktiv
  ist.
- ZooKeeper-Ensemble bildet kein Quorum: `zookeeper_id` je Host in `hosts.ini`
  prüfen (muss eindeutig sein und zur `myid`-Datei passen) sowie Erreichbarkeit der
  Quorum-/Election-Ports 2888/3888 zwischen den ZK-Hosts (Firewall).
