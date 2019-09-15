---
layout: instance/blog-post
title: JAXenter Interview
tagline: "„Jedes Entwicklungsteam hat eigene Möglichkeiten, das Gold aus dem Inhalt seiner Logdaten zu heben“"
language: 🇩🇪
---

Im Zuge der DevOpsCon in Berlin hatten [Nikolaus](http://www.nikolauswinter.de) und ich die Möglichkeit,
unseren Talk zum Thema Log Management in in Container-Umgebungen näher vorzustellen. Hier die Mitschrift
unseres Interviews, welches auf [jaxenter.de](https://jaxenter.de/log-management-microservices-architecture-interview-84584)
erschienen ist.

<!--more-->

Microservice-Architekturen liegen im Trend, haben aber auch eine ganze Reihe neuer Herausforderungen im Gepäck. Beispielsweise, wenn es um das Thema Logging geht. Im Zuge der DevOpsCon 2019 haben wir uns mit Nikolaus Winter (shopping24 internet Group) und Torsten Köster (shopping24 internet group) darüber unterhalten, welche typischen Probleme es gibt und wie man sie mithilfe geeigneter Tools lösen kann.

JAXenter: Ihr beschäftigt Euch auf der DevOpsCon  2019 mit Logging. Wo liegen da die neuen Herausforderungen im Kontext der viel beschworenen Microservice -Architekturen?

Torsten Köster: Microservices externalisieren zuvor interne Aufrufketten. Entsprechend mehr Lognachrichten fallen an. Dazu kommt, dass Cloud und Container-Umgebungen wesentlich volatiler sind als “der guten alte Server”. Die Anwendung wandert zwischen den Compute-Knoten und Ihre Logdateien werden abgeräumt. Will man deren Inhalt im Nachhinein auswerten, muss man sie aktiv verarbeiten und zentral speichern.


Nikolaus Winter

Nikolaus Winter: Durch die Microservice-Architektur gibt es deutlich mehr Quellen für Logs. Da ist es wichtig, den Überblick zu behalten und mit ein wenig smarter Governance dafür zu sorgen, dass Logs nicht nur in einen Index geschoben, sondern dort auch gefunden werden können.

Durch die Microservice-Architektur gibt es deutlich mehr Quellen für Logs. Da ist es wichtig, den Überblick zu behalten.

JAXenter: Welche Lösungsansätze schlagt Ihr vor bzw. gibt es Best Practices, die sich herauskristallisiert haben?

Nikolaus Winter: Zunächst einmal: Allein das Beschäftigen mit dem Thema Logging ist schon einmal ein super Start. Wer dieses Thema frühzeitig mitdenkt und auch den Aufwand einplant, erspart sich im Laufe des Projektes viel Stress.

Falls man die Log-Analyse nachträglich etablieren will, empfehlen wir, zunächst nur ein Subset der Applikationen zu verarbeiten und dann nach und nach weitere hinzuzunehmen. In der Zwischenzeit sollte stets sichergestellt sein, dass der Logging Stack weiterhin richtig dimensioniert ist und die Logs erfolgreich durchsucht werden können.

JAXenter: Habt Ihr ein Lieblingstool, dass Ihr empfehlen könnt?

Torsten Köster: Wir sind große Freunde von Open Source Software und propagieren (auch in unseren Talks) den „ELG“-Stack:

Elasticsearch hat sich als der De-Facto-Standard für das Speichern von Lognachrichten etabliert.
Logstash kann auf kleinem CPU- und Speicher-Footprint viele Tausend Nachrichten in der Sekunde bewegen und verfügt über ein stetig wachsendes Plugin-Ökosystem
Graylog sticht mit seinem Index-Management hervor. Nachrichten können beim Empfang in sogenannte Streams eingeordnet werden. Auf diesen können Zugriffsrechte individuell verteilt werden. Von der Analyseoberfläche her hat Graylog in der aktuellen Version zu Kibana aufgeholt.
Aber es gibt im Containerumfeld spannende neue Tools wie z.B. Loki, die es zu beobachten gilt.

JAXenter: Welche Rolle spielt Docker in Sachen Logging?

Nikolaus Winter: Bevor unsere Anwendungen in Docker liefen, haben große Anwendungen auf langlebigen Servern die Logs in der Regel nach fachlichen oder technischen Kriterien sortiert ins Dateisystem geschrieben. Bei Bedarf wurden sie direkt dort durchsucht, ggf. wurden sie archiviert. Ein ordentlicher Logging-Stack mit zeitnaher Verarbeitung war eher die Ausnahme.

Haben wir eine Microservice-Architektur mit Docker, können wir uns dies so nicht mehr leisten. Die Logs müssen den Server am besten sofort verlassen und da liegt es nahe, sie direkt in einen Stack zu schieben, der sie nutzbar macht.

DevOpsCon Istio Cheat Sheet
Free: BRAND NEW DevOps Istio Cheat Sheet
Ever felt like service mesh chaos is taking over? Then our brand new Istio cheat sheet is the right one for you! DevOpsCon speaker Michael Hofmann has summarized Istio’s most important commands and functions. Download FOR FREE now & sort out your microservices architecture!

Download here
Torsten Köster: Im Container-Umfeld ist das Logging nach STDOUT der kleinste gemeinsame Nenner, der in allen Containern funktioniert. Selbstverständlich kann man auch in eine Logdatei im Docker-Dateisystem schreiben und diese von Filebeat auslesen lassen. Aber wozu? Im Endeffekt schränkt man mit solchen Konstruktion die Portabilität seiner Container massiv ein.

JAXenter: Worum geht es Euch in Eurer Session – was ist die Kernbotschaft, die Ihr den Besuchern mit auf den Weg geben möchtet?

Ich sehe Log-Management als Service für erfolgreiche, autonome Entwicklungsteams.

Nikolaus Winter: Wir sind überzeugt, dass es mit OpenSource Tools möglich ist, Logging-Daten zu beherrschen. Eine teure kommerzielle Lösung ist nicht notwendig, kann aber auch ein valider Weg sein. Aber dann bitte ohne uns 😉

Torsten Köster: Ich sehe Log-Management als Service für erfolgreiche, autonome Entwicklungsteams. Die meisten Entwickler können (und wollen) sich nicht mit Log-Management oder Versionsupdates im Logging-Stack beschäftigen. Das ist auch fein, denn sie sind Experten auf anderen Gebieten. Sobald das Team aber die Verarbeitung seiner Logs verändern möchte (z.B. mit erweiterten Filtern), muss dies problemlos möglich sein. Jedes Entwicklungsteam hat eigene Bedürfnisse und Möglichkeiten, das Gold aus dem Inhalt seiner Logdaten zu heben.