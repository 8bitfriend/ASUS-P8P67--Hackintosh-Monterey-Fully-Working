# ASUS-P8P67--Hackintosh-Monterey-Fully-Working
Hackintosh on an Asus P8P67 with MacOS Monterey

Das Problem: „Phantom Codec“ bei ASUS P8P67 (BIOS ≥ 3602)

ASUS hat mit den BIOS-Versionen 3xxx (z. B. 3602) einen zweiten, nicht existierenden HD-Audio-Codec in der ACPI/DSDT hinterlegt. AppleHDAController erkennt zwei Codecs und weigert sich, die spezialisierte Persönlichkeit BuiltInHDA9D70 (die für viele Intel-Controller zuständig ist) an den tatsächlichen HDA-Controller pci8086,1c20 zu binden. Stattdessen wird nur die generische BuiltInHDA-Persönlichkeit mit der Basisklasse AppleHDAController geladen. In der Folge bleiben die internen Ein- und Ausgänge (Line-Out, Kopfhörer, Mikrofon) stumm.

Die Lösung: Ein codeless Companion-Kext, der eine eigene IOKit-Persönlichkeit bereitstellt, die den speziellen Controller-Treiber AppleHDA8086_9D70Controller direkt an pci8086,1c20 bindet – mit höherer Priorität (IOProbeScore). So wird der Phantom-Codec umgangen.

Instruktionen wurden mit Hilfe von KI erstellt und können Fehler enthalten.

Schritt 1: Den Companion-Kext erstellen

Öffne ein Terminal (unter macOS).

Lege die Kext-Ordnerstruktur an:

  bash
  mkdir -p ~/Desktop/HDA1C20Fix.kext/Contents

Erstelle die Info.plist mit dem Code-Editor nano:

  bash
  nano ~/Desktop/HDA1C20Fix.kext/Contents/Info.plist
  
Kopiere den folgenden Inhalt vollständig in die Datei (alles zwischen den Trennlinien):


xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>CFBundleDevelopmentRegion</key>
    <string>English</string>
    <key>CFBundleIdentifier</key>
    <string>com.hackintosh.HDA1C20Fix</string>
    <key>CFBundleInfoDictionaryVersion</key>
    <string>6.0</string>
    <key>CFBundleName</key>
    <string>HDA1C20Fix</string>
    <key>CFBundlePackageType</key>
    <string>KEXT</string>
    <key>CFBundleShortVersionString</key>
    <string>1.0.0</string>
    <key>CFBundleSignature</key>
    <string>????</string>
    <key>CFBundleVersion</key>
    <string>1.0.0</string>
    <key>IOKitPersonalities</key>
    <dict>
        <key>HDA1C20Fix</key>
        <dict>
            <key>CFBundleIdentifier</key>
            <string>com.apple.driver.AppleHDAController</string>
            <key>IOClass</key>
            <string>AppleHDA8086_9D70Controller</string>
            <key>IONameMatch</key>
            <array>
                <string>pci8086,1c20</string>
            </array>
            <key>IOProbeScore</key>
            <integer>4</integer>
            <key>IOProviderClass</key>
            <string>IOPCIDevice</string>
        </dict>
    </dict>
    <key>OSBundleLibraries</key>
    <dict>
        <key>com.apple.iokit.IOPCIFamily</key>
        <string>1.0.0</string>
        <key>com.apple.kpi.iokit</key>
        <string>16.0</string>
    </dict>
</dict>
</plist>


Speichern und schließen:

Drücke Ctrl + X, dann Y, dann Enter.


Dein Companion-Kext liegt jetzt fertig auf dem Schreibtisch: ~/Desktop/HDA1C20Fix.kext


Schritt 2: Kext in OpenCore einbinden

Kext in den OC-Ordner kopieren:
Ziehe die Datei HDA1C20Fix.kext vom Desktop in den Ordner EFI/OC/Kexts auf deiner EFI-Partition.
(Alternativ: cp -R ~/Desktop/HDA1C20Fix.kext /Volumes/EFI/EFI/OC/Kexts/)

In config.plist eintragen:
Öffne deine config.plist (z. B. mit ProperTree, PlistEdit Pro oder Xcode) und füge unter Kernel -> Add folgenden Eintrag hinzu:


xml
<dict>
    <key>BundlePath</key>
    <string>HDA1C20Fix.kext</string>
    <key>Enabled</key>
    <true/>
    <key>ExecutablePath</key>
    <string></string>
    <key>PlistPath</key>
    <string>Contents/Info.plist</string>
</dict>


Stelle sicher, dass der Eintrag zwischen den anderen Kext-Einträgen liegt und die Enabled-Angabe auf true steht.

Schritt 3: Layout-ID in den Boot-args setzen

Die passende Layout-ID für das P8P67 ist 7.

(In Einzelfällen kann auch 1, 2, 3 oder 12 funktionieren – bei deinem Board war es definitiv 7.)

Öffne in config.plist den Bereich NVRAM -> Add -> 7C436110-AB2A-4BBB-A880-FE41995C9F82.

Finde den Schlüssel boot-args (String).

Ändere den Wert so, dass er alcid=7 enthält.

Beispiel: keepsyms=1 debug=0x100 alcid=7 – wichtig ist, dass alcid=7 vorkommt.

Falls dort bereits ein anderes alcid=… steht, ersetze es.

Schritt 4: Neu starten und prüfen

Neustart und OpenCore hochfahren.

Öffne ein Terminal und prüfe, ob die Personality aktiv ist:

bash
ioreg -l -p IOService -n HDEF | grep -E "IOPersonalityPublisher|IOClass"
Du solltest sehen:

text
|   "IOPersonalityPublisher" = "com.hackintosh.HDA1C20Fix"
|   "IOClass" = "AppleHDA8086_9D70Controller"
Audio-Ausgaben überprüfen:

bash
system_profiler SPAudioDataType | grep -A5 "Internal Speakers"
Es sollte ein Eintrag „Internal Speakers“ oder „Line-Out“ auftauchen. Im Systembericht (Audio) erscheinen nun auch Kopfhörer, Mikrofon etc.

Warum dieser Kext unsichtbar bleibt
Der HDA1C20Fix.kext ist codeless – er besteht nur aus einer Info.plist und enthält kein ausführbares Programm. Deshalb taucht er nicht in kextstat auf. Das ist normal und kein Fehler. Entscheidend ist allein die IOKit-Persönlichkeit, die er injiziert (siehe ioreg-Check).

Zusätzliche Hinweise
BIOS Downgrade nicht nötig: Du kannst die aktuelle BIOS 3602 behalten.

AppleALC weiterhin erforderlich: Der Companion-Kext ersetzt nicht AppleALC, sondern sorgt nur dafür, dass der richtige Controller-Treiber anspringt. AppleALC liefert dann die Pin-Konfiguration für Layout 7.

Störungen nach Sleep? Falls der Ton nach dem Aufwachen weg ist, installiere zusätzlich den CodecCommander.kext, der die Codec-Power-States stabilisiert. In deinem Fall war es bislang nicht nötig.

Updates: Der Companion-Kext ist update-sicher; macOS-Updates ändern nicht deine OpenCore-Injektion. Du musst ihn nur dann anpassen, wenn Apple die AppleHDAController-Klassen komplett umbenennt (sehr unwahrscheinlich).

Damit hast du eine vollständige, saubere Lösung, die den BIOS-Bug dauerhaft umgeht und vollen nativen Sound auf deinem Asus P8P67 unter macOS ermöglicht.
