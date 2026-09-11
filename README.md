# Silk Wars Online

3D-Action-RPG im Browser. Three.js, ES6-Module, keine Build-Tools, direkt
GitHub-Pages-tauglich.

## Speichern

Ohne weitere Einrichtung speichert das Spiel im Browser (`localStorage`) —
kein Konto, keine Netzverbindung. Gespeichert wird automatisch alle 25
Sekunden, sobald sich etwas Wichtiges geändert hat, sowie beim Wegwischen der
Seite; von Hand über den Knopf im Charakterfenster (Taste C).

Für einen Speicherstand über mehrere Geräte hinweg trägst du in
`src/config/firebase.js` die Konfiguration aus der Firebase-Konsole ein.
Solange dort `null` steht, wird kein einziges Firebase-Modul geladen. In der
Konsole zusätzlich nötig: anonyme Anmeldung aktivieren, Firestore anlegen und
die Regeln aus `FirebaseAdapter.js` setzen. Am Spielcode ändert sich nichts.

## Starten

ES6-Module brauchen einen HTTP-Server, `file://` funktioniert nicht.

```bash
python3 -m http.server 8000
# http://localhost:8000
```

Für GitHub Pages: Repo-Inhalt so wie er ist pushen, Pages auf den Branch-Root
stellen. Kein Build-Schritt.

## Steuerung

| Eingabe | Wirkung |
|---|---|
| W A S D | laufen, relativ zur Kamera |
| Rechte Maustaste halten | Kamera drehen (Pointer Lock) |
| Mausrad | zoomen |
| Linksklick | Ziel wählen, ins Leere klicken hebt die Auswahl auf |
| Tab | nächstes Ziel nach Entfernung |
| 1 – 8 | Fähigkeiten der ersten Action-Bar |
| F1 – F8 | Fähigkeiten der zweiten Action-Bar |
| Leertaste | Ausweichen (zwei Ladungen) |
| T | Autoattack an/aus |
| Esc | Boden-Zielen abbrechen, sonst Ziel abwählen |
| C | Charakterfenster (Attribute verteilen) |
| K | Mastery- und Skillfenster |
| I | Ausrüstung und Gepäck |
| U | Notausstieg: zurück zum Ausgangspunkt, alles gelöst |
| F3 | Debug-Werte |

## Auf dem Handy spielen

Berührungsgeräte werden automatisch erkannt (`pointer: coarse`); am Rechner
lässt sich der Modus mit `?touch=1` in der Adresse erzwingen.

| Geste | Wirkung |
|---|---|
| Linker Daumen auf dem Ring | laufen (Stick mit Totzone) |
| Ein Finger über die Welt ziehen | Kamera drehen |
| Ein Finger kurz tippen | Ziel wählen oder Bodenpunkt bestätigen |
| Zwei Finger spreizen | zoomen |
| Zwei Finger kurz tippen | abbrechen (wie Rechtsklick) |
| Knöpfe rechts | Ausweichen, nächstes Ziel, Autoattack, Abbrechen |
| Knöpfe oben rechts | Charakter, Masteries, Ausrüstung |
| Action-Bar unten | direkt antippen |

Die Touch-Schicht ersetzt nichts: ein `InputRouter` führt Tastatur, Maus und
Finger zusammen, und Kamera, Bewegung und Zielsystem fragen weiterhin dieselbe
Schnittstelle ab. Bildschirmknöpfe und Tasten lösen denselben Befehl aus
(`input:command`), es gibt also keine zweite Steuerungslogik.

## Testmodus (Gamemaster)

Adresse mit `?gm=1` aufrufen:

```
https://DEIN-NAME.github.io/silk-wars-online/?gm=1
```

Dann gilt: Stufe 20, alle zwölf Masteries auf Maximum, **alle 91 Skills auf
Rang 5 freigeschaltet**, und jeder Gegenstand liegt doppelt im Gepäck — damit
sich auch Dual-Wield ausprobieren lässt. Zusätzliche Schalter:

| Taste | Wirkung |
|---|---|
| J | Unverwundbarkeit an/aus |
| N | keine Manakosten und keine Abklingzeiten |
| U | Notausstieg |

Auf dem Handy erscheinen für J und N zwei zusätzliche Knöpfe oben rechts.

Der Testmodus benutzt einen **eigenen Speicherplatz**: dein normaler
Spielstand bleibt unberührt, egal was du darin anstellst. Da 91 Skills nicht
auf 16 Slots passen, gibt es im Mastery-Fenster (K) neben jedem Skill einen
Knopf zum Ablegen auf die Leiste und darunter "Action-Bars leeren".

## Architektur

Eine Regel trägt das ganze Projekt: **`src/game/` importiert niemals `three`.**

```
src/
  core/     EventBus, Registry, GameLoop, Config, Mathe        (neutral)
  game/     Spielregeln, Zustand, Weltmathematik               (kein three)
    entities/   Entity, EntityState, EntityManager
    stats/      deriveStats.js    Attribute -> Kampfwerte
    ai/         AiController, AiSystem
    progression/ ProgressionSystem, SkillProgression, scaling.js
    world/      RespawnSystem, populate.js (Welt aus Gebietsdaten)
    persistence/ SaveSystem, SaveGame, createStorage,
                 adapters/ (local, memory, firebase)
    targeting/  TargetingSystem   Ziel-Auswahl und Boden-Zielen
    abilities/  AbilityExecutor, componentHandlers, CooldownController,
                CastSystem (Ladezeiten), Scheduler (verzögerte Schritte)
    components/ ResourcePool, StatusEffectController, Equipment
    combat/     ProjectileSystem, HazardSystem, TetherSystem, BarrierSystem,
                AutoAttackSystem, damage.js, factions.js
    data/       entityTemplates.js, abilities.js, statusEffects.js,
                masteries.js, items.js, regions.js, weaponProfiles.js
    ActionBarSystem.js            Slots -> ability:requested
  input/    InputController (Tastatur/Maus), TouchController,
            InputRouter                                        (kein Spielwissen)
  render/   Three.js
    models/     bodyTemplates.js  Körperbau als Daten
                rigs.js           Rig-Bauer (biped / quadruped)
                attachments.js    Waffen und Schilde als Daten
    animation/  clips.js (Keyframes als Daten), AnimationController,
                BipedLocomotion, QuadrupedLocomotion
    views/      CharacterView     Logikzustand -> Mesh
                NameplateLayer    Namen und HP-Balken über den Figuren
                WeaponTrail       Schwungspur der Klinge
    ViewManager.js                baut Modelle aus entity:spawned
    EntityPicker.js               Maus -> Entity / Bodenpunkt
    indicators.js                 Zielring und Boden-Reticle
    landmarks.js                  Wahrzeichen der Gebiete
    ProjectileLayer.js            Pfeile und Magiegeschosse
    HazardLayer.js                Bodenflächen und Wirbel
    TetherLayer.js                Ketten zwischen Entities
    CastLayer.js                  Ladeanimation
    effects.js                    Flächen- und Treffereffekte
    AssetRegistry.js              einzige Tür zwischen Gameplay und Geometrie
  ui/       HTML-Overlay
```

- **Logik vs. Anzeige.** `src/game/` rechnet mit Zahlen und IDs. `src/render/`
  spiegelt das auf Meshes und entscheidet nie eine Spielregel. Verbunden wird
  beides über den EventBus und über Views, die pro Frame den Zustand auf ein
  Modell übertragen.
- **Fester Logiktakt (60 Hz), freier Rendertakt.** Kampf und Cooldowns laufen
  ab Phase 5 unabhängig von der Framerate identisch ab; die Views interpolieren.
- **Weltform als Mathematik.** `game/Heightfield.js` liefert die Bodenhöhe, das
  Terrain-Mesh wird daraus erzeugt — nie umgekehrt. Raycasting bleibt für
  Kamera-Kollision und (ab Phase 4) Maus-zu-Weltpunkt reserviert.
- **Daten statt Fallunterscheidungen.** Registries für Modelle, Körper, Rig-Typen,
  Animatoren und Ausrüstung. Später ebenso für Skills, Statuseffekte, Monster
  und Items.
- **Persistenz vorbereitet.** `PlayerState.toJSON()` liefert schon das Dokument,
  das in Phase 10 nach Firestore geht.

### Der Rig-Vertrag

Jeder Modell-Bauer liefert dieselbe Form zurück:

```js
{ id, kind, animation, root, parts, anchors, slots, materials, height,
  setAttachment(slot, object3D|null, transform), getAttachment(slot), dispose() }
```

Solange diese Form stimmt, ist die Herkunft egal — Primitive heute, ein
GLTF-Loader mit Kenney-/Quaternius-Modellen und Mixamo-Clips später. Weder
Animation noch Kampf-Code müssen dafür angefasst werden.

## Phasenplan

| Phase | Meilenstein | Status |
|---|---|---|
| 1 | Welt, Kamera, Bewegung, artikulierte Figur | **fertig** |
| 2 | Modell-Fabrik als Daten, Rig-Typen, sichtbare Ausrüstung | **fertig** |
| 3 | Entity-Kern, ResourcePool, Attribute, HP/Mana-Leisten | **fertig** |
| 4 | Zielsystem, Ziel-Frame, Boden-Reticle, zwei Action-Bars | **fertig** |
| 5 | AbilityExecutor, Komponenten-Handler, Dodge, Autoattack | **fertig** |
| 6 | Statuseffekte, Schadensformel, Krit | **fertig** |
| 7 | Monster und KI-Zustandsmaschine | **fertig** |
| 8 | XP/Level, Attribute, Mastery- und Skillpunkte | **fertig** |
| 9 | Ausrüstung, Inventar, Slot-Regeln, Imbue | **fertig** |
| 10 | Persistenz: Speicheradapter, Firebase vorbereitet | **fertig** |
| 11.1 | Animation und Aussehen | **fertig** |
| 11.2 | Inhalte: Monsterfähigkeiten, zweites Gebiet, mehr Skills | **fertig** |
| 12.1 | Kampf-Bausteine: Ladezeit, Projektilvarianten, Kombos, Elementfarben | **fertig** |
| 12.2 | Bogen vollständig (Bodeneffekte, Verbindungen) | **fertig** |
| 12.3 | Schwert | **fertig** |
| 12.4 | Speer und Schild | **fertig** |
| 12.5 | Großschwert und Dolche | **fertig** |
| 12.6 | Stab, Imbues und Nukes | **fertig** |
| 11.3 | Balance über simulierte Kämpfe | offen (nach den Skills) |

## Phase 1 — Fundament

- Prozedurales Terrain (240 × 240 m) mit ebener Startlichtung
- Third-Person-Kamera mit Zoom und Raycast-Kollision
- Kamerarelative WASD-Bewegung mit Beschleunigung und Kreis-Kollision
- Prozedurale Canvas-Texturen, 110 Bäume und 60 Felsen
- Schattenkasten, der dem Spieler folgt; Debug-Overlay (F3), `window.game`

## Phase 2 — Modell-Fabrik

- **Körperbau als Daten** (`bodyTemplates.js`): Proportionen, Palette, Rig-Typ
  und Ausrüstungs-Slots pro Gestalt. Sechs Vorlagen: Spieler, Dorfbewohner,
  Kriegerin, Jäger, Magierin, Waldwolf.
- **Zwei Rig-Bauer** über Registry: `biped` und `quadruped`. Ein neuer Rig-Typ
  ist ein neuer Eintrag, keine Änderung an bestehendem Code.
- **Sieben Ausrüstungsteile** als Daten mit eigenen Meshes: Kurzschwert, Dolch,
  Großschwert, Speer, Jagdbogen, Zauberstab, Rundschild.
- **Armhaltung folgt der Waffe.** Jedes Item bringt eine `pose` mit, die der
  Animator anwendet — der Bogen wird vorgestreckt, das Großschwert beidhändig,
  der Dolch schwingt fast frei mit. Keine Fallunterscheidung im Animator.
- **Waffenwechsel über Ereignis.** Die Tasten senden `entity:equipment` auf den
  Bus; Modell und HUD reagieren darauf. Genau diesen Weg nimmt ab Phase 9 auch
  das Ausrüstungssystem — die Tastatur verschwindet dann einfach.
- Vier NPCs und ein Wolf stehen in der Lichtung, jeder mit eigener Statur,
  Bewaffnung und Idle-Animation.

## Phase 3 — Entity-Kern

- **Entity = Zustand + Komponenten.** Keine Gott-Klasse: `Entity` hält heute
  einen `ResourcePool`, ab Phase 6 zusätzlich einen `StatusEffectController`,
  ab Phase 7 einen `AiController` — jeweils als eigenes Feld.
- **Attribute wirken.** `deriveStats()` rechnet die vier Primärattribute in
  Kampfwerte um: Stärke -> physischer Schaden, Geschicklichkeit -> Kritchance
  und Kritschaden (nicht Schaden), Intelligenz -> Magieschaden und Mana,
  Vitalität -> Leben, Rüstung, Regeneration. Alle Koeffizienten stehen in
  `CONFIG.stats`.
- **Komponenten kennen den EventBus nicht.** Der `ResourcePool` meldet
  Änderungen per Callback, die `Entity` übersetzt das in `entity:resources`,
  `entity:damaged`, `entity:died`. Und zwar erst, wenn sich ein gerundeter
  Wert ändert — sonst würde das HUD 60-mal pro Sekunde neu schreiben.
- **Spawnen genügt.** Der `ViewManager` hört auf `entity:spawned` und baut das
  Modell selbst. Ab Phase 7 muss Monster-Code nicht wissen, dass es eine
  Anzeige gibt.
- **Entities als Daten** (`entityTemplates.js`): Name, Modell, Fraktion,
  Attribute, Startausrüstung und schon das spätere KI-Verhalten.
- HUD: Spielerrahmen mit Leben, Mana und Erfahrung; Namensschilder mit
  Lebensbalken über allen anderen Figuren, die bei Verletzung erscheinen und
  mit der Entfernung ausblenden.
- `EntityManager.nearest()` liefert bereits das nächste gültige Ziel — die
  Grundlage für die Ziel-Auswahl in Phase 4.

## Phase 4 — Zielen und Eingabe

Das Zielschema hat zwei getrennte Hälften, genau wie in GW2 oder Silkroad:

1. **Ein ausgewähltes Ziel.** Linksklick auf eine Figur oder Tab für das
   nächste nach Entfernung. Die Auswahl bleibt bestehen, während man sich frei
   bewegt, und geht erst bei 60 m Abstand verloren. Ring am Boden in der Farbe
   der Fraktion, Ziel-Rahmen mit Name, Stufe, Leben und laufender Entfernung.
2. **Davon unabhängig das Boden-Zielen.** Flächenfähigkeiten schalten einen
   Indikator ein: äußerer Ring für die Maximalreichweite um dich, innerer für
   den Wirkungsradius an der Maus. Zieht man weiter hinaus, rutscht der Punkt
   an die Reichweitengrenze und färbt sich gelb — überschreiten kann man sie
   nicht. Linksklick bestätigt, Rechtsklick oder Esc bricht ab.

- **Zwei Action-Bars zu je acht Slots** (1–8 und F1–F8). Ein Slot hält nur eine
  Ability-ID ohne Kategorie: Angriff, Buff, Imbue und Ausweichen sind
  gleichberechtigt zuweisbar.
- **Die Fähigkeit sagt, was die Eingabe braucht.** `targeting: 'target' |
  'ground' | 'self'` steht in den Ability-Daten; das `ActionBarSystem` prüft
  Ziel und Reichweite und meldet dann `ability:requested`. In Phase 5 hängt
  sich der AbilityExecutor an genau dieses Ereignis — an der Eingabe ändert
  sich dafür nichts.
- **Raycasting nur da, wo es hingehört:** `EntityPicker` beantwortet zwei
  Fragen — welche Entity liegt unter dem Zeiger, und auf welchen Bodenpunkt
  zeigt er. Zeigt die Maus über den Horizont, fängt eine waagerechte Ebene den
  Strahl ab, damit der Indikator nicht verschwindet.
- Die Waffentasten sind ins Ausrüstungsfenster (I) gewandert, weil die
  Zifferntasten jetzt der Action-Bar gehören.
- Meldungen über der Leiste bestätigen, was ankommt: "außer Reichweite
  (3.0 m von 2.6 m)", "kein gültiges Ziel", "Bodenpunkt wählen".

## Phase 5 — Der Ability-Executor

Der Kern der übernommenen Architektur: **es gibt keine Skill-Klassen.** Eine
Fähigkeit ist ein Datensatz mit einer Liste von Komponenten, und ein einziger
Executor arbeitet sie ab.

```js
'ability.firestorm': {
  targeting: 'ground', range: 14, cooldown: 8, mana: 22,
  components: [{
    type: 'aoe', at: 'point', radius: 3.6, vfx: 'fire',
    components: [{ type: 'damage', school: 'magic', base: 14, coefficient: 1.4 }],
  }],
}
```

- **Sechs Komponententypen, sechs Handler:** `damage`, `projectile`, `aoe`,
  `movement`, `resource`, `status_effect`. Jeder liegt in einer Registry, keiner
  kennt eine konkrete Fähigkeit.
- **Handler dürfen verschachteln.** Der aoe-Handler ruft für jeden Getroffenen
  wieder den Executor auf. "Fläche, die Schaden *und* einen Debuff verteilt"
  brauchte deshalb keine Zeile neuen Code — nur eine zweite Komponente in der
  Liste.
- **Geschosse tragen ihre Wirkung mit sich.** Ein Pfeil führt seine
  `onHit`-Komponenten mit und gibt sie beim Einschlag an denselben Executor
  zurück. Ein Pfeil nimmt damit exakt denselben Schadenspfad wie ein
  Schwerthieb.
- **Ausweichen ist eine ganz normale Fähigkeit** mit zwei Ladungen: eine
  `movement`-Komponente setzt einen Ausweichschritt, den die Bewegung ausführt
  (kein Teleport, ~5 m in 0,28 s), dazu eine `status_effect`-Komponente für die
  kurze Unverwundbarkeit. Keine Sonderlogik.
- **Autoattack pro Waffentyp** über `weaponProfiles.js`: Bogen schießt einen
  Pfeil, Schwert schlägt zu, Stab wirkt ein Magiegeschoss, leere Hand schlägt
  mit der Faust. Die Abklingzeit der Angriffsfähigkeit *ist* die
  Angriffsgeschwindigkeit — kein zweiter Timer.
- **Ladungen laufen über denselben Mechanismus wie Abklingzeiten**, damit die
  Pipeline keinen Sonderfall dafür kennt. Die Action-Bar zeigt beides: Sweep,
  Restsekunden und Ladungszähler.
- Sichtbar: fliegende Pfeile und Magiegeschosse mit leichter Zielsuche,
  Flächenringe, Treffer-Funken, Schadens- und Heilzahlen, dazu kurze
  Animations-Gesten (Schlag, Schuss, Zauber, Ausweichschritt), die über die
  Laufanimation gelegt werden.

Der Executor hört auf `ability:requested` — dieselbe Meldung, die schon in
Phase 4 aus den Action-Bars kam. Ab Phase 7 senden Monster sie ebenfalls;
dafür ist an dieser Stelle nichts zu ändern.

## Phase 6 — Statuseffekte, Mitigation, Krit

**Die eine Schadensformel.** Jeder Treffer im Spiel läuft durch
`combat/damage.js` — Skill, Autoattack, Geschosseinschlag, Schaden über Zeit
und ab Phase 7 auch jeder Monsterangriff:

```
effektiv = roh × 100 / (100 + Verteidigung)
```

Abnehmender Ertrag statt harter Grenzen: 20 Rüstung nehmen 17 % weg, 100
nehmen die Hälfte, 200 zwei Drittel. Niemand ist je immun, ein schlechter
Build ist nur ineffizienter. Krit gilt für Waffen *und* Magie gleichermaßen.

**Vierzehn Statuseffekte als Registry-Daten**, keine Zeile Sonderlogik pro
Skill: Slow, Root, Stun, Freeze, Knockback, Knockdown, Burn, Poison, Bleed,
Silence, Vulnerable, Weaken, Invulnerable, Haste.

- **Gates statt Abfragen.** `canMove()`, `canAct()`, `canCast()` sind die
  einzige Stelle, an der Bewegung und Executor nachfragen. Kein System kennt
  Betäubung oder Verstummen beim Namen — ein neuer Effekt, der Bewegung
  sperrt, trägt einfach `blocks: ['move']` in seinen Daten.
- **Modifikatoren werden aufsummiert.** Slow (−40 %) und Hast (+25 %) ergeben
  zusammen Tempo 0,85; ein Blick auf `moveSpeedMultiplier()` genügt der
  Bewegung.
- **Stapelnde DoTs.** Brand stapelt bis dreifach, Gift bis fünffach; der
  Tickschaden wird ebenfalls mitigiert — sonst wäre Rüstung gegen Gift
  wertlos.
- **Rückstoß ist ein Schubs, kein Zustand.** Er nutzt denselben Mechanismus
  wie der Ausweichschritt, nur vom Verursacher weggerichtet.
- **Unverwundbarkeit greift jetzt wirklich:** Der Ausweichschritt aus Phase 5
  hatte sie schon angefordert — Treffer währenddessen melden nur noch "immun".
- **Waffenbann (Imbue)** addiert Magieschaden zum physischen Waffenschaden,
  und zwar abhängig vom Verhältnis STR:INT — mit Intelligenz 5 bringt er +7,4
  Schaden, mit Intelligenz 25 sind es +39,7. Jeder Treffer setzt zusätzlich
  Brand.
- Neu auf der Leiste: **Donnerschlag** (Slot 6) — Flächenschaden, Rückstoß und
  1,6 s Betäubung, drei Komponenten in einer Liste.
- HUD: Buff-/Debuff-Leiste unter dem Spielerrahmen und im Zielrahmen, mit
  Restzeit und Stapelzahl. Schaden über Zeit fliegt in eigener Farbe.

Auch die NPCs laufen ab jetzt durch das Bewegungssystem — sie bekommen nur
keine Eingabe. Dadurch wirkt Rückstoß schon vor der KI, und Phase 7 muss dafür
nichts umbauen.

## Phase 7 — Monster und KI

Ab hier ist es ein Spiel: die Wölfe wehren sich.

**Eine Zustandsmaschine pro Monster**, als Tabelle statt als switch:

```
wandern -> verfolgen -> angreifen -> zurückkehren (Leash)
```

- **Drei Verhaltenstypen aus den Template-Daten.** *Passiv* greift nie an (der
  Dorfbewohner und die Magierin nehmen bei einem Treffer Reißaus), *neutral*
  wehrt sich erst nach einem Treffer (Wachsoldatin, Jäger), *aggressiv* greift
  von sich aus an, sobald jemand in den Aggro-Radius kommt (Waldwölfe, 12 m).
- **Kein Parallelsystem.** Monster senden `ability:requested` wie die
  Action-Bar des Spielers. Reichweite, Abklingzeit, Mana, Mitigation, Krit und
  Statuseffekte prüft derselbe Executor. Die Wachsoldatin schlägt mit
  demselben `attack.greatsword` zu, das auch dir zur Verfügung steht, und der
  Jäger schießt echte Pfeile, die durch dieselbe Geschossphysik fliegen.
- **Angriffsreichweite kommt aus den Fähigkeiten-Daten**, nicht doppelt aus
  dem KI-Profil. Die Abklingzeit der Angriffsfähigkeit ist die
  Angriffsgeschwindigkeit.
- **Die Gates gelten auch für die KI.** Ein betäubter Wolf tut nichts, ein
  verlangsamter verfolgt langsamer, ein zurückgestoßener fliegt — alles über
  denselben StatusEffectController.
- **Leash:** Läuft man weg, kehrt das Monster zum Spawnpunkt zurück, füllt
  Leben auf und beruhigt sich. Kein endloser Zug quer über die Karte.
- **Respawn statt Testtasten.** Gefallene kommen von selbst zurück — Wölfe
  nach 20 s, NPCs nach 30–35 s, der Spieler nach 6 s am Ausgangspunkt mit
  Einblendung und Countdown. Die Tasten G und R sind damit weg.
- Monster laufen durch dieselbe Bewegung wie du: Terrainhöhe, Kollision mit
  Bäumen, Rückstoß.

Zur Balance: ein Waldwolf gegen einen frischen Charakter ist knapp. Wer nur
mit Autoattack dagegenhält, gewinnt nach etwa 8 Sekunden mit einem Viertel
Leben; wer sich gar nicht wehrt, fällt nach zehn. Der Biss setzt in 30 % der
Fälle Blutung — die Trefferchance ist ein Datenfeld der Komponente, keine
Sonderlogik.

## Phase 8 — Progression

**Kein Klassensystem.** Jeder Charakter startet identisch: Basis 5 auf allen
vier Attributen, 5 freie Attributpunkte, 5 Masterypunkte. Was daraus wird,
entscheidet allein die Punktevergabe.

- **Ein gemeinsamer Punkte-Pool** für Mastery-Level *und* Skill-Ränge. Ein
  Mastery-Level kostet einen Punkt, ein Skill-Rang ebenfalls — wer breit geht,
  hat viele Fähigkeiten auf niedrigem Rang, wer tief geht, wenige auf hohem.
- **Kein linearer Skilltree.** Das Mastery-Level schaltet Fähigkeiten frei
  (Feuersturm ab Feuer 1, Waffenbann ab Feuer 3), Ränge gehen bis 10 und
  können das Mastery-Level nie übersteigen. Daraus entsteht die Entscheidung
  von selbst, ohne Baumstruktur.
- **Rang-Skalierung ohne Sonderfälle.** `scaleAbility()` liefert eine
  skalierte Kopie der Fähigkeit — Rang 1 macht 14 Grundschaden bei 3,60 m
  Radius und 8,0 s Abklingzeit, Rang 10 kommt auf 29,1 Schaden, 4,09 m und
  6,92 s. Die Registry-Daten bleiben unangetastet.
- **Zwölf Masteries**, sieben Waffen und fünf Magieschulen, jede mit passiven
  Boni pro Level. Auch Masteries ohne eigene Skills (Dolche, Schild, Speer,
  Stab) lohnen sich dadurch als Build-Entscheidung.
- **Synergien geben echte Boni**, generisch aus einer `effects`-Liste
  ausgewertet: Bogen 5 + Elektro 5 ergibt "Sturmschütze" mit +5 % Kritchance
  und +15 % Schaden auf den Schuss. Kein Sonderfall im Code — ein neuer
  Eintrag genügt.
- **Punktevergabe hängt nicht am Level.** `grantMasteryPoints(amount, source)`
  ist der generische Weg; Level-Ups nutzen ihn genauso wie später Bosskills
  oder Quests, auch nach Erreichen des Maximallevels.
- Zwei neue Fähigkeiten füllen die Magieschulen: **Kettenblitz** (springt über
  bis zu drei weitere Gegner, mit Abschwächung pro Sprung — der `chain`-Handler
  war der letzte fehlende Komponententyp) und **Fluch** (Verwundbar + Geschwächt).
- Nicht erlernte Fähigkeiten bleiben auf der Leiste sichtbar, aber ausgegraut,
  und der Executor lehnt sie ab. Der Rang steht als kleines "R2" im Slot.
- Fenster: **C** für Attribute mit Sofortwirkung auf die abgeleiteten Werte,
  **K** für Masteries, Ränge und Synergien, mit Knopf zum Ablegen auf die
  Action-Bar.

## Phase 9 — Ausrüstung

**Sieben Slots** (Haupthand, Nebenhand, Helm, Brust, Handschuhe, Hose, Schuhe)
und Items als Registry-Daten mit additiven Stat-Boni, die direkt in
`deriveStats` fließen. Volle Lederrüstung mit Eisenschwert bringt 100 auf 123
Leben, 4 auf 18 Rüstung und 21 auf 30 physischen Schaden.

- **Die Regeln stehen in den Item-Daten, nicht im Code.** `grip: 'two_hand'`
  räumt beim Anlegen beide Hände (Verdrängtes wandert zurück ins Gepäck) und
  sperrt die Nebenhand, solange die Waffe getragen wird. `dualWield: true` ist
  alles, was ein Dolch braucht, um zusätzlich in die Nebenhand zu dürfen —
  keine Sonderbehandlung für Dolche im Code.
- **Die Oberfläche kennt die Regeln nicht.** Das Fenster sendet nur Befehle
  (`equipment:equip`, `equipment:unequip`); die Equipment-Komponente
  entscheidet und liefert bei Ablehnung einen Grund zurück, den das HUD
  anzeigt: "Jagdbogen braucht beide Hände", "Erst ab Stufe 4", "Nicht für die
  Nebenhand geeignet".
- **Sichtbar am Charakter.** Waffen und Schild hingen schon an den Handankern;
  dazu kommen jetzt Helm und Brustpanzer an neuen Kopf- und Brustankern, in
  Leder- und Eisenausführung. Handschuhe, Hose und Schuhe wirken bisher nur
  über ihre Werte.
- **Die Waffe bestimmt die Autoattack** — über Item → Modell → Waffenprofil.
  Dolch sticht, Bogen schießt, Stab wirkt ein Magiegeschoss, Großschwert
  schlägt in die Fläche.
- **Beute.** Monster tragen Beutetabellen im Template; 40 Wölfe ließen im Test
  14 Gegenstände fallen. Auch NPCs tragen jetzt echte Items — die Wachsoldatin
  ihre Eichenklinge und ihren Eisenhelm, und beides kann sie verlieren.
- Der Bestand ist dicht: Anlegen, Umhängen, Verdrängen und Ablegen halten die
  Summe aus Gepäck und angelegten Stücken über alle Wege konstant.

## Phase 10 — Persistenz

Das war die Phase, für die seit Phase 1 vorgearbeitet wurde: weil der gesamte
Spielerzustand an einer Stelle liegt und `toJSON()` vollständig ist, besteht
das Speichern aus einem Aufruf. Es gab nichts umzubauen.

- **Drei Ablagen, eine Schnittstelle.** `load(slot)`, `save(slot, doc)`,
  `remove(slot)` — erfüllt von `LocalStorageAdapter`, `MemoryAdapter` (Rückfall
  im privaten Modus) und `FirebaseAdapter`. Weder das Spiel noch das
  SaveSystem kennen den Unterschied.
- **Firebase bleibt inaktiv, bis es gebraucht wird.** Die SDK-Module kommen
  per dynamischem Import; ohne Konfiguration lädt der Browser kein Byte davon.
- **Geladen wird, bevor der Spieler entsteht.** Dadurch muss nichts
  nachträglich in eine bestehende Entity gespiegelt werden — die Werte aus
  Ausrüstung und Masteries stehen beim ersten Frame korrekt.
- **Gespeichert wird nicht bei jedem Ereignis**, sondern gemerkt und im Takt
  geschrieben — sonst löste ein Kampf Dutzende Schreibvorgänge aus. Dazu beim
  Wegwischen der Seite, was auf dem Handy der Normalfall ist.
- **Versionsfeld und Migrationshaken** von Anfang an: ein Stand mit unbekannter
  Version wird abgewiesen statt halb geladen.
- Geprüft im Rundlauf über zwei Sitzungen: Stufe, Erfahrung, Attribute, freie
  Punkte, Mastery-Level, Skill-Ränge, Ausrüstung, Gepäck, Action-Bar-Belegung,
  Position, Spawnpunkt sowie Leben und Mana kommen unverändert zurück — auch
  die daraus abgeleiteten Werte.

## Phase 11.1 — Animation und Aussehen

Bisher war jede Fähigkeit dieselbe halbe Sinuswelle am Arm. Jetzt sind
Animationen **Daten**: Keyframe-Kurven pro Gelenk in `clips.js`, additiv über
die Laufanimation gelegt.

- **Sieben Clips**: Schlag (Ausholen, Durchziehen, Nachschwingen), Stich,
  Bogenschuss (Ziehen, Lösen), Zauber, Ausweichschritt, Zucken bei Treffern
  und Sterben.
- **Additiv und überlagerbar.** Ein Treffer mitten im Schlag legt das Zucken
  darüber, ohne dass beide voneinander wissen. Nur der Todes-Clip übersteuert
  die Laufschicht und hält seine Endpose.
- **Ein Clip, zwei Rig-Arten.** Fehlende Gelenke werden übersprungen — deshalb
  beißt der Wolf beim selben `swing`-Clip mit Nacken und Kopf, während der
  Mensch die Schulter durchzieht. Eine Prüfung stellt sicher, dass jede Spur
  an mindestens einem Rig ankommt; ein Tippfehler im Gelenknamen fällt damit
  sofort auf.
- **Sterben ist eine Animation**, kein Umkippen mehr: die Figur sackt über
  1,1 s in den Boden, Beine knicken ein, Arme fallen zur Seite.
- **Schwungspur an der Klinge**, mit Vorlauf: sie setzt erst ein, wenn die
  Waffe tatsächlich durchzieht, und verjüngt sich zum Ende, statt einfach zu
  verschwinden.
- **Effekte pro Schule**: Feuer wirft aufsteigende Funken, die von der
  Schwerkraft wieder gekippt werden; Eis fährt als Stachelkranz aus dem Boden;
  der Kettenblitz zeichnet jetzt einen gezackten Strahl von Ziel zu Ziel.
- **Licht und Farbe**: filmische Tonwertkurve (ACES) gegen ausgebrannte
  Sonnenseiten, dazu ein kaltes Gegenlicht von der Schattenseite, das die
  Silhouetten vom Hintergrund abhebt.

## Phase 11.2 — Inhalte

**Monster kämpfen jetzt mit Verstand.** Das KI-Profil führt neben der
Autoattack eine Liste von Fähigkeiten mit Reichweitenfenster und Chance:

```js
abilities: [{ ability: 'attack.pounce', minRange: 4.5, maxRange: 10, chance: 0.7 }]
```

Der Wolf setzt zum Satz an, sobald die Lücke groß genug ist — im Nahkampf
beißt er weiter. Die Wachsoldatin rammt ihren Schild und betäubt, der Jäger
schießt Fesselpfeile, die Magierin Frostpfeile, die verlangsamen. Alles über
denselben Executor wie beim Spieler, also mit Mitigation, Krit und
Statuseffekten.

**Ein zweites Gebiet: das verfallene Lager** (Stufen 5–8), gut 70 m
nordwestlich. Zerbrochene Säulen, ein flackerndes Lagerfeuer und Kisten;
darin drei Wegelagerer mit zwei Dolchen, drei Schattenwölfe und ein Waldbär,
der mit der Pranke ausholt, zurückstößt und niederschlägt. Der Bär gibt als
einziger Gegner zusätzlich einen Masterypunkt — genau das, wofür
`grantMasteryPoints(amount, source)` in Phase 8 gebaut wurde.

- **Gebiete sind Daten** (`regions.js`): Mittelpunkt, Radius, Besatz,
  Wahrzeichen, freizuhaltender Kern. Die Welt setzt sich beim Start daraus
  zusammen, deterministisch über einen Seed, damit Spawn- und Rückkehrpunkte
  über Sitzungen hinweg gleich bleiben. Die Umgebung hält die Kerne frei —
  geprüft: null Hindernisse in beiden Gebietsmitten.
- **Die vier leeren Masteries haben Skills bekommen:** Schattenschritt
  (Dolche, setzt zum Ziel über), Aufspießen (Speer), Schildwall (Schild,
  −35 % erlittener Schaden bei −15 % Tempo) und Arkanstoß (Stab, Fläche mit
  Verstummen). Damit ist jede der zwölf Masteries spielbar.
- **Zwei neue Synergien:** Eisenwall (Schild + Großschwert) und Nachtklinge
  (Dolche + Dunkle Magie).
- **Sprung als Bewegungsart:** `mode: 'leap', toward: 'target'` berechnet
  Richtung und Weite aus dem Abstand, damit niemand über sein Ziel
  hinausschießt — genutzt von Satz und Schattenschritt.
- Eine Datenprüfung geht alle Templates, Fähigkeiten, Effekte, Items und
  Mastery-Skills durch: jeder Verweis löst auf.

## Phase 12.1 — Bausteine für die Skill-Liste

Die geplante Skill-Liste braucht Fähigkeiten, die das Komponentensystem noch
nicht ausdrücken konnte. Diese Etappe liefert die Bausteine — die Skills
selbst sind danach überwiegend Datenarbeit.

- **Ladezeit und Aufladen.** `castTime` an einer Fähigkeit startet einen
  sichtbaren Ladevorgang statt sofort auszulösen: Energiekugel am Charakter,
  die mit dem Fortschritt wächst, Ladebalken im HUD, Elementfarbe. Abgebrochen
  wird durch Loslaufen, Betäubung, Tod oder Esc — **Abklingzeit und Mana
  kostet nur das tatsächliche Auslösen.**
- **Projektil-Varianten**, alles Datenfelder: `count` und `spread` für den
  Fächer (fliegen geradeaus und treffen, was im Weg steht), `bounces` mit
  `falloff` für Sprünge von Gegner zu Gegner, `returns` für den späteren
  Waffenwurf, `homing` für Zielsuche.
- **Sequenzen und Wiederholungen.** `sequence` mit Verzögerungen pro Schritt
  und `repeat` als Kurzform sind die Grundlage aller 5-Hit-Kombos. Sie laufen
  über einen eigenen Scheduler im festen 60-Hz-Takt — bewusst nicht über
  `setTimeout`, sonst liefen Kombos bei Bildratenschwankungen auseinander.
- **Element-Farbsprache** durchgängig für Projektile, Flächen, Einschläge und
  Ladeeffekte: Feuer orange, Eis hellblau, Blitz weiß-blau und **gezackt statt
  rund**, dazu heilig, dunkel und arkan. Jede Fähigkeit trägt ein `element`;
  fehlt eine Modellangabe, wählt das Element die Geschossform.
- **Krit-Bonus pro Fähigkeit** (`critBonus`) für den geladenen Schuss.

Fünf Bogen-Skills zeigen es: **Geladener Schuss** (1 s, +25 % Kritchance),
**Goldener Schuss** (2 s, sehr hoher Schaden), **Fächerschuss** (5 Pfeile,
traf im Test 5 verschiedene Gegner), **Sprungschuss** (43 → 35 → 28 Schaden
über drei Ziele) und **Wuchtschuss** (Rückstoß und Verwundbarkeit).

## Phase 12.2 — Bogen vollständig

Drei weitere Bausteine, danach war der Rest Datenarbeit:

- **Bodeneffekte** (`HazardSystem`): eine Fläche, die bestehen bleibt und in
  festen Abständen ihre Komponenten auf alles anwendet, was darin steht —
  über denselben Executor wie jede andere Fähigkeit. Sie kann feststehen,
  in eine Richtung wandern oder an einer Entity kleben. Damit sind Nagelfeld
  und Pfeilwirbel derselbe Mechanismus, und die Lavaflächen und
  Elementarspuren aus der Liste sind später nur noch Daten.
- **Verbindungen** (`TetherSystem`): eine Kette zwischen zwei Entities, die
  eine Höchstentfernung erzwingt. Sichtbar als durchhängende Linie, die umso
  straffer wird, je weiter das Ziel zieht — sie besteht dauerhaft, nicht nur
  im Moment des Treffers.
- **Geschosse auf Bodenpunkte:** ein Pfeil kann jetzt einen Punkt ansteuern
  statt einen Gegner. Er fliegt an allem vorbei und detoniert erst dort.

Damit hat der Bogen **zwölf Fähigkeiten**, gestaffelt über die Mastery-Level 1
bis 10: Schuss, Bogenschlag, Wuchtschuss, Fächerschuss, Sprungschuss rückwärts,
Sprungschuss, Geladener Schuss, Sprengpfeil, Fesselkette, Goldener Schuss,
Nagelschuss und Pfeilwirbel.

Im Test: der Sprengpfeil verschonte einen Gegner, der genau im Flugweg stand,
und traf drei am Zielpunkt; das Nagelfeld hielt einen Wolf vier Sekunden am
Boden; der Wirbel zog durch eine Viererreihe und traf jeden zweimal; die Kette
hielt einen Fluchtversuch auf 30 m bei exakt 8 m fest.

## Phase 12.3 — Schwert

Elf Fähigkeiten, gestaffelt über die Mastery-Level 1 bis 10: Hieb, Stich,
Deckung, Lähmen, Fünfschlag, Niederschlag, Luftklingen, Todesstoß,
Klingensprung, Klingenwurf, Klingenwirbel und Klingenzug.

Drei Bausteine kamen dafür dazu:

- **Bedingte Fähigkeiten.** `requires: { targetStatus: 'knockdown' }` — der
  Todesstoß lässt sich nur gegen liegende Gegner einsetzen und meldet sonst
  "nur gegen niedergeschlagene Ziele".
- **Bewegung des Ziels.** Die movement-Komponente kennt jetzt `at: 'target'`
  mit `toward: 'caster'`: der Klingenzug reißt alles im Umkreis des
  steckenden Schwerts zu dir heran (im Test von 11–13 m auf unter 1 m).
- **Block-Haltung** als Statuseffekt: −60 % erlittener Schaden bei −40 % Tempo,
  über dieselbe Mitigation wie alles andere.

Dazu zwei neue Geschossformen: die geworfene Klinge überschlägt sich im Flug
und **verschwindet währenddessen aus der Hand**, bis sie zurückkehrt; die
Luftklingen fliegen als flache Sicheln nacheinander los.

### Ein Fehler, der ohne Test nie aufgefallen wäre

Das geworfene Schwert flog nach dem Treffer geradeaus weiter, statt
umzukehren. Ursache war die Lenkung: sie mischte den alten mit dem neuen
Richtungsvektor und normalisierte danach. Bei einer Kehrtwende heben sich
(1,0) und (−1,0) unterwegs auf, und das Ergebnis wurde wieder auf die alte
Richtung normiert. Die Lenkung interpoliert jetzt den **Winkel** — damit sind
180-Grad-Wenden möglich, und zurückkehrende Waffen funktionieren überhaupt
erst.

## Phase 12.4 — Speer und Schild

Speer (neun): Aufspießen, Speerwurf, Speerdeckung, Fünfstoß, Speerwirbel,
Speersprung, Sturmangriff, Verschleiern, Emporschleudern.
Schild (fünf): Schildwall, Verhöhnen, Schildwurf, Bodenstoß, Schutzbund.

Vier Bausteine kamen dazu:

- **Barrieren.** Eine Mauer ist eine Reihe runder Kollider — dieselbe Form,
  die Bäume und Felsen schon benutzen. Die Bewegung musste dafür nichts Neues
  lernen, sie fragt jetzt zusätzlich zu den festen auch bewegliche Hindernisse
  ab. Im Test blieb ein anstürmender Wolf an der Mauer bei x=7 hängen und kam
  nicht weiter als x=8.
- **Spott.** Der taunt-Handler setzt das Ziel der KI direkt und sperrt es für
  die Dauer — auch gegen neue Treffer von anderer Seite. Drei Wölfe, die auf
  einen Dorfbewohner losgingen, drehten alle zum Spieler.
- **Verschleierung.** Ein Statuseffekt, den die Aggro-Suche übergeht: der Wolf
  bemerkte den verschleierten Spieler auf 5,7 m nicht, obwohl sein
  Aggro-Radius bei 12 m liegt. Preis: 35 % weniger Schaden, und die Figur wird
  sichtbar halbdurchsichtig.
- **Verbündete.** `isAlly()` fasst weiter als die Fraktionsgleichheit, damit
  der Schutzbund auf Dorfbewohner und Wachen wirkt. Auf einen Gegner
  angewandt, passiert schlicht nichts.

**Emporschleudern** ist ehrlich gesagt ein Kompromiss: die Bewegung kennt
keine Y-Achse, ein echter Flugzustand wäre ein größerer Umbau. Das Ziel wird
deshalb logisch niedergeschlagen, während ein Animations-Clip das Modell
sichtbar hochschnellen und hart landen lässt. Wer genau hinsieht, merkt: das
Ziel fliegt, wird aber währenddessen nicht seitlich weggeschoben.

## Phase 12.5 / 12.6 — Großschwert, Dolche, Stab, Bänne und Nukes

Damit ist die Skill-Liste umgesetzt: **106 Fähigkeiten, davon 91 für den
Spieler**, verteilt auf zwölf Masteries.

| Mastery | Anzahl |
|---|---|
| Schwert | 12 |
| Bogen | 12 |
| Speer | 9 |
| Feuer / Eis / Elektro | je 9 |
| Großschwert, Dolche | je 8 |
| Stab | 7 |
| Schild | 5 |

**Bänne (Imbues) mit echten Wahrscheinlichkeiten**, wie in der Liste
vorgegeben: Feuer 30 % Brand / 5 % Blenden, Eis 30 % Verlangsamen / 5 %
Einfrieren, Blitz 30 % Überschlag auf einen zweiten Gegner / 5 % Betäubung.
Über 60 Schläge mit dem Blitzbann traten im Test beide Auslöser auf.

**Der Bann färbt die Fähigkeiten ein.** Dolch- und Stabfähigkeiten tragen
`element: 'imbue'` — erst zur Laufzeit wird daraus Feuer, Eis oder Blitz, mit
passender Farbe, Geschossform und elementtypischer Zugabe. Dieselbe
Klingenhatz entlädt sich je nach Bann anders, ohne dass es sie dreimal gibt.

**Neue Bausteine dieser Etappe:**

- **Kegelflächen** (`shape: 'cone'`) für den Schrei — im Test traf er den
  Gegner vorne und den hinter dem Rücken nicht.
- **Einschlag aus der Höhe** (`falling`): die Wirkung kommt verzögert aus der
  Spiellogik, während die Anzeige sichtbar etwas herabfallen lässt und den
  Zielkreis pulsieren lässt. Meteor und Gletschersturz haben damit echte
  Flugzeit statt Sofortschaden.
- **Streuung** (`jitter`) für den Elementarhagel, damit mehrere Einschläge
  nicht übereinander liegen.
- **Sprung auf einen Bodenpunkt** für den Wuchtsprung.

**Achtzehn Nukes** entstehen aus sechs Bauformen mal drei Elementen: schnelles
Geschoss, geladene Lanze, Nova um den Wirker, stehendes Feld, Kettenblitz und
Einschlag aus dem Himmel. Die Form bestimmt Verhalten und Optik, das Element
Farbe und Zusatzeffekt — so unterscheiden sie sich sichtbar und nicht nur im
Namen.

### Prüfung

Ein Durchlauf löst **jede der 91 Spielerfähigkeiten einmal aus**: 90 gingen
durch, die einzige Ablehnung war der Todesstoß mit der korrekten Begründung
"nur gegen niedergeschlagene Ziele". Dazu die übliche Datenprüfung — jeder
Verweis auf Effekte, Komponenten, Items und Mastery-Skills löst auf.

## Nachbesserung: nach dem Tod festgefahren

Gemeldet und behoben. Drei Änderungen am Wiedereinstieg:

- **Sicherheitsnetz.** Das Respawn-System prüft jetzt jeden Takt, ob jemand
  tot ist, für den kein Wiedereinstieg eingeplant ist — und holt ihn nach.
  Geht ein Todesereignis verloren, bleibt niemand mehr dauerhaft liegen.
- **Gründlicheres Aufräumen.** Beim Wiedereinstieg werden zusätzlich
  Geschwindigkeit, Ausweichschritt, laufende Zauber und das KI-Ziel
  zurückgesetzt.
- **Spawnschutz.** 2,5 s Unverwundbarkeit nach dem Wiedereinstieg, damit man
  nicht direkt wieder in einer noch laufenden Flächenwirkung steht.

Dazu der **Notausstieg** auf Taste U (und als Knopf in der Touch-Steuerung):
er löst Ketten, beendet Zustände und Ausweichschritte, bricht laufende Zauber
ab und stellt den Charakter an seinen Ausgangspunkt — unabhängig davon, was
ihn festgehalten hat.

## Bekannte Grenzen (bewusst)

- Handschuhe, Hose und Schuhe haben noch keine Meshes; ihnen fehlen die Anker
  am Rig.
- Der Firebase-Adapter ist geschrieben, aber noch nie gegen ein echtes Projekt
  gelaufen — der lokale Weg ist der getestete.
- Nur ein Speicherplatz, keine Charakterauswahl.
- Dunkle und Heilige Magie haben bisher nur je ein bis zwei Skills — die
  Skill-Liste nennt Nukes ausdrücklich nur für Feuer, Eis und Elektro.
- Nichts davon ist austariert: Schaden, Kosten und Abklingzeiten sind
  Erstwerte. Das ist der Inhalt der Balance-Phase.
- Keine Gesichter, keine Mimik, keine Umhänge — das Rig bleibt bewusst
  Low-Poly.
- Der Waldbär nutzt dasselbe Vierbeiner-Rig wie die Wölfe, nur breiter und
  schwerer.
- Emporschleudern hebt das Modell, nicht die Spielfigur — es gibt keinen
  echten Flugzustand mit Y-Achse.
- Zwischen den Gebieten liegt leeres Gelände; Wege, Übergänge und ein
  Levelhinweis beim Betreten fehlen noch.
- Kein Handel, keine Haltbarkeit, keine Aufwertung — Gegenstände sind Beute
  oder Startausrüstung.
- Monster haben noch keine eigenen Skills über die Autoattack hinaus; die
  Pipeline dafür steht (ein zweiter Eintrag im KI-Profil genügt).
- Schaden ist Rohschaden. Rüstung, Magieresistenz und Krit setzen sich in
  Phase 6 davor; die Werte werden schon berechnet, aber noch nicht angewandt.
- Erfahrung wird angezeigt, aber noch nicht vergeben — Level-Ups in Phase 8.
- Hast wirkt bisher auf Tempo und Schaden, noch nicht auf Abklingzeiten.
- Skills wandern per Knopf aus dem Mastery-Fenster auf die Leiste; Drag & Drop
  fehlt noch.
- Attributpunkte lassen sich nicht zurücksetzen (kein Respec).
- Kein Speichern; `PlayerState` ist vorbereitet, Firebase kommt in Phase 10.
- Three.js kommt per CDN-Importmap. Sobald echte Assets dazukommen, ist der
  Moment, über Vite zu reden.
