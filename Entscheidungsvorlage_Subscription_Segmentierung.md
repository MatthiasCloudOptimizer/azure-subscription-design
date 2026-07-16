# Entscheidungsvorlage: Azure-Subscription-Segmentierung

**Datum:** 07.07.2026  
**Autor:** Matthias Braun, Microsoft MVP Azure  
**Lizenz:** [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/)

## Thema

Strategie zur Segmentierung von Azure Subscriptions

## 1. Zusammenfassung und Empfehlung

Dieses Dokument dient als Entscheidungsgrundlage für die Architektur der Azure-Umgebung.

Verglichen werden zwei Ansätze:

- ein monolithischer Ansatz mit wenigen Subscriptions, bei dem Workloads primär über Ressourcengruppen getrennt werden,
- ein granularer Ansatz mit Segmentierung nach Workload-Archetypen sowie getrennten Subscriptions für Produktions- und Nicht-Produktionsumgebungen.

Der granulare Ansatz leitet sich aus der Empfehlung des **Cloud Adoption Frameworks (CAF)** ab und bildet die Grundlage für die weitere Bewertung.

> **Unsere klare Empfehlung ist der granulare Ansatz.**

Er schafft eine sicherere, stabilere und langfristig kosteneffizientere Plattform, die mit den Anforderungen des Unternehmens skaliert. Der initiale Mehraufwand durch zusätzliche Subscriptions beschränkt sich im Wesentlichen auf organisatorische Prozesse.

Die technische Bereitstellung und Einbindung in die Azure-Plattform erfolgt standardisiert und weitgehend automatisiert mittels Terraform.

Gleichzeitig bedeutet der granulare Ansatz nicht, dass jede Komponente zwingend eine eigene Subscription benötigt. Zu diesem Ansatz gehört auch der gezielte Einsatz von **Shared Subscriptions** für Workloads oder Plattformbausteine, bei denen eine dedizierte Subscription keinen fachlichen, technischen oder wirtschaftlichen Mehrwert liefert oder der betriebliche Aufwand nicht im Verhältnis zum Nutzen steht.

## 2. Gegenüberstellung: „Sammel-Spokes“ vs. „Einzel-Anwendungs-Spokes“

Die aktuelle Architektur trennt bereits vorbildlich nach Produktions- und Nicht-Produktionsumgebungen auf Subscription-Ebene. Die zentrale Frage ist die Granularität innerhalb dieser Umgebungen.

### Grundprinzip

| Kriterium | Aktueller Ansatz: „Sammel-Spokes“ | Empfohlener Ansatz: „Anwendungs-Spokes“ |
|---|---|---|
| **Grundprinzip** | Eine Subscription pro Geschäftsbereich bzw. Verbund und Umgebung, z. B. `sub-dus-sait-nonprod-fdg`. Diese enthält die Ressourcengruppen für mehrere Anwendungen. | Eine Subscription pro einzelner Anwendung und Umgebung, z. B. `sub-app-atled-nonprod`. |
| **Sicherheit und Isolation** | **Mittel:** Ein Angreifer oder eine Fehlkonfiguration in einer Anwendung, z. B. App A, kann potenziell andere Anwendungen, z. B. Apps B und C, in derselben Subscription beeinträchtigen. Der Blast Radius umfasst alle Anwendungen im Spoke. | **Stark:** Native Isolation auf Subscription-Ebene. Ein Vorfall in App A ist vollständig von Apps B und C abgeschottet. Der Blast Radius bleibt minimal. |
| **Governance und Policies** | **Mittel:** Azure Policies gelten für alle Anwendungen in der Subscription. App-spezifische Policies müssen per Tagging und Filter umständlich zugewiesen werden. | **Stark:** Dedizierte und strikte Policies können einfach pro Anwendung zugewiesen werden. |
| **Limits und Quotas** | **Riskant:** Alle Anwendungen in der Subscription teilen sich Azure-Limits, z. B. vCPU-Quotas oder API-Request-Limits. Ein Lasttest von App A kann die API-Raten für die produktive App B aufbrauchen, wenn beide im selben Sammel-Spoke liegen. | **Robust:** Jede Anwendung erhält eigene dedizierte Limits. Eine gegenseitige Beeinflussung wird vermieden. |
| **Kostentransparenz** | **Schwach:** Die Kosten einer einzelnen Anwendung müssen über Tags der Ressourcengruppen aggregiert werden. Die Zuordnung geteilter Ressourcen ist schwierig. | **Exzellent:** Die Kosten einer Anwendung sind größtenteils direkt ihrer dedizierten Subscription zugeordnet und ohne zusätzliche Aggregation auswertbar. Gemeinsam genutzte Plattformressourcen müssen weiterhin separat verteilt bzw. verrechnet werden. |
| **Betrieb mit IaC und Terraform** | **Schwerfällig:** Ein großer Terraform-State verwaltet mehrere unabhängige Anwendungen. Plan- und Apply-Zyklen sind langsam, fehleranfällig und bergen das Risiko ungewollter Änderungen an anderen Anwendungen. | **Ideal:** Jede Anwendung hat einen eigenen, kleinen, fokussierten und unabhängigen Terraform-State. Änderungen sind schnell, isoliert und risikoarm. |
| **Netzwerkkosten bei App-zu-App-Kommunikation** | **Potenziell niedriger:** Anwendungen können direkt innerhalb desselben VNets kommunizieren. Bei zentralem Routing über den Hub entstehen ähnliche Kosten wie im granularen Modell. | **Potenziell höher:** Der Verkehr zwischen Anwendungen wird häufig über den Hub geführt. Die zusätzlichen Kosten sind ein bewusst akzeptierter Trade-off für bessere Isolation und Governance. |

## 3. Detaillierte Analyse der Peering-Kosten

Das Argument der Peering-Kosten ist valide. Die Kosten entstehen, wenn Daten ein virtuelles Netzwerk (VNet) verlassen und in ein anderes VNet übertragen werden.

In der empfohlenen Hub-Spoke-Architektur, die sich am Cloud Adoption Framework (CAF) orientiert, bedeutet das konkret:

- Die **Connectivity Subscription** übernimmt das zentrale Hub-VNet.
- Die **Identity Subscription** stellt zentrale Identitätsdienste wie Domain Controller in einem eigenen Spoke-VNet bereit.
- **Workload Subscriptions** enthalten jeweils die anwendungsspezifischen Spoke-VNets und sind ebenfalls mit dem Hub verbunden.

### Wann fallen Kosten an?

Kosten entstehen, wenn Daten zwischen virtuellen Netzwerken über Peering-Verbindungen übertragen werden.

Für In-Region-VNet-Peering in **Germany West Central** wird als Richtwert für den Stand 2026 etwa **0,01 € pro GB je durchquerter Peering-Verbindung** angesetzt.

Traffic innerhalb desselben VNets verursacht dagegen keine Peering-Kosten.

### Szenarioanalyse

| Szenario | Architektur und Traffic-Pfad | Kostenbewertung | Fazit |
|---|---|---|---|
| **Kommunikation innerhalb eines Workloads** | App-Server und Datenbank liegen in derselben Workload-Subscription und im selben VNet. Der Traffic verlässt das VNet nicht. | **0,00 €** | Für den Großteil des anwendungsinternen Traffics entstehen keine Peering-Kosten. |
| **Kommunikation mit dem Domain Controller** | Eine Anwendung im Workload-Spoke authentifiziert sich am Domain Controller in der Identity Subscription. Der Traffic verläuft über `Workload Spoke → Hub → Identity Spoke` und durchquert damit zwei Peering-Verbindungen. | Bei 15 GB pro Monat: `15 GB × 0,02 €/GB = 0,30 € pro Monat` für Hin- und Rückweg. Ein initialer Worst Case von 500 GB pro Monat läge bei etwa 10,00 € pro Monat. | Die realistischen Peering-Kosten für zentrale Dienste liegen im Cent-Bereich pro Anwendung und Monat. Sie sind daher kein entscheidender Kostenfaktor gegen eine granulare Segmentierung. |

## 4. Gegenüberstellung der direkten und indirekten Kosten

Der zentrale Trade-off besteht zwischen potenziell höheren direkten Netzwerk­kosten und deutlich reduzierten indirekten Kosten und Risiken.

| Kostenart | Aktueller Ansatz: „Sammel-Spokes“ | Empfohlener Ansatz: „Anwendungs-Spokes“ |
|---|---|---|
| **Direkte Kosten: Peering** | **Baseline:** Kosten für Traffic zu zentralen Diensten über Hub und Identity fallen an. Traffic zwischen Apps im selben Spoke ist kostenlos. | **Inkrementell höher:** Identische Baseline-Kosten sowie zusätzliche, doppelte Peering-Kosten für App-zu-App-Traffic, der nun über den Hub geleitet werden muss. |
| **Indirekte Kosten: Risiken** | **Hoch:** Ein Sicherheitsvorfall oder Ressourcenengpass in einer Anwendung kann sich auf alle anderen Anwendungen in derselben Subscription ausbreiten. | **Niedrig:** Ein Vorfall ist auf genau eine Anwendung beschränkt. Das Geschäftsrisiko ist deutlich reduziert. |
| **Indirekte Kosten: Betriebsaufwand** | **Hoch:** Permanenter Aufwand für die Verwaltung großer Terraform-States und die aufwendige Kostenanalyse über Tagging. | **Niedrig:** Geringerer operativer Aufwand durch kleine, unabhängige IaC-Workflows und native Kostentransparenz pro Anwendung. |

### Fazit zur Kostenanalyse

Der empfohlene Ansatz führt zu potenziell höheren, aber weiterhin geringen und planbaren direkten Kosten für die App-zu-App-Kommunikation.

Dieser bewusste Trade-off finanziert:

- eine deutliche Reduktion indirekter Kosten und Risiken,
- geringeren Betriebsaufwand,
- robustere Deployments,
- verbesserte Kostentransparenz,
- einen deutlich reduzierten Blast Radius bei Sicherheits- oder Betriebsvorfällen.

Für Anwendungslandschaften mit besonders hohem Kommunikationsaufkommen zwischen mehreren Komponenten kann eine gemeinsame Platzierung innerhalb derselben Subscription bzw. desselben Workload-Spokes geprüft werden. Dadurch lassen sich Netzwerk- und Transitkosten minimieren, ohne die grundlegenden Architekturprinzipien aufzugeben.

## 5. Ausrichtung an Microsoft Frameworks

Der empfohlene Ansatz setzt die Designprinzipien des **Cloud Adoption Frameworks** und des **Well-Architected Frameworks** für Azure Landing Zones konsequent um.

Die dort empfohlene **Subscription Democratization** ermöglicht Skalierung, Governance und Sicherheit durch klar getrennte Verantwortungs- und Isolationsgrenzen.

| WAF-Säule | Beitrag des granularen Ansatzes |
|---|---|
| **Sicherheit** | Stärkere Isolation und klar begrenzter Zugriff pro Workload. |
| **Zuverlässigkeit** | Reduzierter Blast Radius bei technischen Fehlern, Fehlkonfigurationen oder Sicherheitsvorfällen. |
| **Kostenoptimierung** | Native Kostentransparenz pro Anwendung ohne komplexe Tagging-Auswertungen. |
| **Operative Exzellenz** | Kleine, unabhängige IaC-States und klar abgegrenzte Verantwortlichkeiten. |
| **Leistungseffizienz** | Dedizierte Ressourcen-Kontingente pro Workload und geringere gegenseitige Beeinflussung. |
| **Nachhaltigkeit** | Die isolierte und transparente Struktur unterstützt eine gezieltere Ressourcenplanung und vermeidet unnötige Überdimensionierung. |

## 6. Schlussfolgerung

Die granulare Segmentierung nach Anwendung und Umgebung ist der vorzuziehende Zielzustand für die Azure-Plattform.

Sie bietet gegenüber Sammel-Spokes insbesondere Vorteile bei:

- Sicherheit und Isolation,
- Governance und Policy-Zuweisung,
- Limits und Quotas,
- Kostentransparenz,
- Terraform-State-Management,
- Fehler- und Auswirkungsbegrenzung.

Die zusätzlichen Peering-Kosten sind in typischen Szenarien gering, planbar und gegenüber den reduzierten Betriebs- und Geschäftsrisiken vertretbar.

Eine Subscription sollte dennoch nicht pauschal pro Komponente eingeführt werden. Shared Subscriptions bleiben sinnvoll, wenn sie unter fachlichen, technischen und wirtschaftlichen Gesichtspunkten die bessere Lösung darstellen.
